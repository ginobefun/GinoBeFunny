---
title: 基于word2vec和Elasticsearch实现个性化搜索
date: 2017-03-28 15:51:02
tags: [word2vec, Elasticsearch, 个性化, 搜索, 推荐]
categories: Elasticsearch
link_title: personalized_search_implemention_based_word2vec_and_elasticsearch
---

在[word2vec学习小记](http://ginobefunny.com/post/learning_word2vec/)一文中我们曾经学习了word2vec这个工具，它基于神经网络语言模型并在其基础上进行优化，最终能获取词向量和语言模型。在我们的商品搜索系统里，采用了word2vec的方式来计算用户向量和商品向量，并通过Elasticsearch的function_score评分机制和自定义的脚本插件来实现个性化搜索。

<!-- more --> 

# 背景介绍

先来看下[维基百科](https://en.wikipedia.org/wiki/Personalized_search)上对于个性化搜索的定义和介绍：

> Personalized search refers to web search experiences that are tailored specifically to an individual's interests by incorporating information about the individual beyond specific query provided. Pitkow et al. describe two general approaches to personalizing search results, one involving modifying the user's query and the other re-ranking search results.

由此我们可以得到两个重要的信息：
1. 个性化搜索需要充分考虑到用户的偏好，将**用户感兴趣的内容**优先展示给用户；
2. 另外是对于实现个性化的方式上主要有**查询修改和对搜索结果的重排序**两种。

而对我们电商网站来说，个性化搜索的重点是当用户搜索某个关键字，如【卫衣】时，能将用户最感兴趣最可能购买的商品（如用户偏好的品牌或款式）优先展示给用户，以提升用户体验和点击转化。

# 设计思路
1. 在此之前我们曾经有一般的个性化搜索实现，其主要是通过计算用户和商品的一些重要属性（比如品牌、品类、性别等）的权重，然后得到一个用户和商品之间的关联系数，然后根据该系数进行重排序。
2. 但是这一版从效果来看并不是很好，我个人觉得主要的原因有以下几点：用户对商品的各个属性的重视程度并不是一样的，另外考虑的商品的属性并不全，且没有去考虑商品和商品直接的关系；
3. 在新的版本的设计中，我们考虑通过**用户的浏览记录这种时序数据**来获取用户和商品以及商品和商品直接的关联关系，其核心就是通过类似于语言模型的词出现的顺序来训练向量表示结果；
4. 在获取用户向量和商品向量表示后，我们就可以**根据向量直接的距离来计算相关性**，从而将用户感兴趣的商品优先展示；

# 实现细节
## 商品向量的计算
1. 根据用户最近某段时间（如30天内）的浏览记录，获取得到浏览SKN的列表并使用空格分隔；核心的逻辑如下面的SQL所示：


    select concat_ws(' ', collect_set(product_skn)) as skns 
    from 
     (select uid, cast(product_skn as string) as product_skn, click_time_stamp 
      from product_click_record 
      where date_id <= $date_id and date_id >= $date_id_30_day_ago
      order by uid, click_time_stamp) as a 
    group by uid;

2. 将该SQL的执行结果写入文件作为word2vec训练的输入；
3. 调用word2vec执行训练，并保存训练的结果：


    time ./word2vec -train $prepare_file -output $result_file -cbow 1 -size 20 
    -window 8 -negative 25 -hs 0 -sample 1e-4 -threads 20 -iter 15 

4. 读取训练结果的向量，保存到搜索库的商品向量表。

## 用户向量的计算
1. 在计算用户向量时采用了一种简化的处理，即通过用户最近某段时间（如30天内）的商品浏览记录，根据这些商品的向量进行每一维的求平均值处理来计算用户向量，核心的逻辑如下：


    vec_list = []
    for i in range(feature_length):
        vec_list.append("avg(coalesce(b.vec[%s], 0))" % (str(i)))
    vec = ', '.join(vec_list)
    
    
    select a.uid as uid, array(%s) as vec 
    from 
     (select * from product_click_record where date_id <= $date_id and date_id >= $date_id_30_day_ago) as a
    left outer join
     (select * from product_w2v where date_id = $date_id) as b
    on a.product_skn = b.product_skn
    group by a.uid;

2. 将计算获取的用户向量，保存到Redis里供搜索服务获取。

## 搜索服务时增加个性化评分
1. 商品索引重建构造器在重建索引时设置商品向量到product_index的某个字段中，比如下面例子的productFeatureVector字段；
2. 搜索服务在默认的cross_fields相关性评分的机制下，需要增加个性化的评分，这个可以通过**function_score**来实现。

```java
Map<String, Object> scriptParams = new HashMap<>();
scriptParams.put("field", "productFeatureVector");
scriptParams.put("inputFeatureVector", userVector);
scriptParams.put("version", version);
Script script = new Script("feature_vector_scoring_script", ScriptService.ScriptType.INLINE, "native", scriptParams);
functionScoreQueryBuilder.add(ScoreFunctionBuilders.scriptFunction(script));
```

3. 这里采用了[elasticsearch-feature-vector-scoring插件](https://github.com/ginobefun/elasticsearch-feature-vector-scoring)来进行相关性评分，其核心是向量的余弦距离表示，具体见下面一小节的介绍。在脚本参数中，field表示索引中保存商品特征向量的字段；inputFeatureVector表示输入的向量，在这里为用户的向量；
4. 这里把version参数单独拿出来解释一下，因为每天计算出来的向量是不一样的，向量的每一维并没有对应商品的某个具体的属性（至少我们现在看不出来这种关联），因此我们要特别避免不同时间计算出来的向量的之间计算相关性。在实现的时候，我们是通过一个中间变量来表示最新的版本，即在完成商品向量和用户向量的计算和推送给搜索之后，再更新这个中间向量；搜索的索引构造器定期轮询这个中间变量，当发现发生更新之后，就将商品的特征向量批量更新到ES中，在后面的搜索中就可以采用新版本的向量了；

## elasticsearch-feature-vector-scoring插件
这是我自己写的一个插件，具体的使用可以看下[项目主页](https://github.com/ginobefun/elasticsearch-feature-vector-scoring)，其核心也就一个类，我将其主要的代码和注释贴一下：

```java
public class FeatureVectorScoringSearchScript extends AbstractSearchScript {
    public static final ESLogger LOGGER = Loggers.getLogger("feature-vector-scoring");
    public static final String SCRIPT_NAME = "feature_vector_scoring_script";
    private static final double DEFAULT_BASE_CONSTANT = 1.0D;
    private static final double DEFAULT_FACTOR_CONSTANT = 1.0D;

    // field in index to store feature vector
    private String field;

    // version of feature vector, if it isn't null, it should match version of index
    private String version;

    // final_score = baseConstant + factorConstant * cos(X, Y)
    private double baseConstant;

    // final_score = baseConstant + factorConstant * cos(X, Y)
    private double factorConstant;

    // input feature vector
    private double[] inputFeatureVector;

    // cos(X, Y) = Σ(Xi * Yi) / ( sqrt(Σ(Xi * Xi)) * sqrt(Σ(Yi * Yi)) )
    // the inputFeatureVectorNorm is sqrt(Σ(Xi * Xi))
    private double inputFeatureVectorNorm;

    public static class ScriptFactory implements NativeScriptFactory {
        @Override
        public ExecutableScript newScript(@Nullable Map<String, Object> params) throws ScriptException {
            return new FeatureVectorScoringSearchScript(params);
        }

        @Override
        public boolean needsScores() {
            return false;
        }
    }

    private FeatureVectorScoringSearchScript(Map<String, Object> params) throws ScriptException {
        this.field = (String) params.get("field");
        String inputFeatureVectorStr = (String) params.get("inputFeatureVector");
        if (this.field == null || inputFeatureVectorStr == null || inputFeatureVectorStr.trim().length() == 0) {
            throw new ScriptException("Initialize script " + SCRIPT_NAME + " failed!");
        }

        this.version = (String) params.get("version");
        this.baseConstant = params.get("baseConstant") != null ? Double.parseDouble(params.get("baseConstant").toString()) : DEFAULT_BASE_CONSTANT;
        this.factorConstant = params.get("factorConstant") != null ? Double.parseDouble(params.get("factorConstant").toString()) : DEFAULT_FACTOR_CONSTANT;

        String[] inputFeatureVectorArr = inputFeatureVectorStr.split(",");
        int dimension = inputFeatureVectorArr.length;
        double sumOfSquare = 0.0D;
        this.inputFeatureVector = new double[dimension];
        double temp;
        for (int index = 0; index < dimension; index++) {
            temp = Double.parseDouble(inputFeatureVectorArr[index].trim());
            this.inputFeatureVector[index] = temp;
            sumOfSquare += temp * temp;
        }

        this.inputFeatureVectorNorm = Math.sqrt(sumOfSquare);
        LOGGER.debug("FeatureVectorScoringSearchScript.init, version:{}, norm:{}, baseConstant:{}, factorConstant:{}."
                , this.version, this.inputFeatureVectorNorm, this.baseConstant, this.factorConstant);
    }

    @Override
    public Object run() {
        if (this.inputFeatureVectorNorm == 0) {
            return this.baseConstant;
        }

        if (!doc().containsKey(this.field) || doc().get(this.field) == null) {
            LOGGER.error("cannot find field {}.", field);
            return this.baseConstant;
        }

        String docFeatureVectorStr = ((ScriptDocValues.Strings) doc().get(this.field)).getValue();
        return calculateScore(docFeatureVectorStr);
    }

    public double calculateScore(String docFeatureVectorStr) {
        // 1. check docFeatureVector
        if (docFeatureVectorStr == null) {
            return this.baseConstant;
        }

        docFeatureVectorStr = docFeatureVectorStr.trim();
        if (docFeatureVectorStr.isEmpty()) {
            return this.baseConstant;
        }

        // 2. check version and get feature vector array of document
        String[] docFeatureVectorArr;
        if (this.version != null) {
            String versionPrefix = version + "|";
            if (!docFeatureVectorStr.startsWith(versionPrefix)) {
                return this.baseConstant;
            }

            docFeatureVectorArr = docFeatureVectorStr.substring(versionPrefix.length()).split(",");
        } else {
            docFeatureVectorArr = docFeatureVectorStr.split(",");
        }

        // 3. check the dimension of input and document
        int dimension = this.inputFeatureVector.length;
        if (docFeatureVectorArr == null || docFeatureVectorArr.length != dimension) {
            return this.baseConstant;
        }

        // 4. calculate the relevance score of the two feature vector
        double sumOfSquare = 0.0D;
        double sumOfProduct = 0.0D;
        double tempValueInDouble;
        for (int i = 0; i < dimension; i++) {
            tempValueInDouble = Double.parseDouble(docFeatureVectorArr[i].trim());
            sumOfProduct += tempValueInDouble * this.inputFeatureVector[i];
            sumOfSquare += tempValueInDouble * tempValueInDouble;
        }

        if (sumOfSquare == 0) {
            return this.baseConstant;
        }

        double cosScore = sumOfProduct / (Math.sqrt(sumOfSquare) * inputFeatureVectorNorm);
        return this.baseConstant + this.factorConstant * cosScore;
    }
}
```

# 总结与后续改进
- 基于word2vec、Elasticsearch和自定义的脚本插件，我们就实现了一个个性化的搜索服务，相对于原有的实现，新版的点击率和转化率都有大幅的提升；
- 基于word2vec的商品向量还有一个可用之处，就是可以用来实现相似商品的推荐；
- 但是以我个人的理解，使用word2vec来实现个性化搜索或个性化推荐是有一定局限性的，因为它只能处理用户点击历史这样的时序数据，而无法全面的去考虑用户偏好，这个还是有很大的改进和提升的空间；
- 后续的话我们会更多的参考业界的做法，更多地更全面地考虑用户的偏好，另外还需要考虑时效性的问题，以优化商品排序和推荐。

# 参考资料
- [Personalized search Wiki](https://en.wikipedia.org/wiki/Personalized_search)
- [搜索下一站：个性化搜索基本方法和简单实验](http://blog.csdn.net/soso_blog/article/details/6050346)
- [京东基于大数据技术的个性化电商搜索引擎](http://www.infoq.com/cn/presentations/jingdong-personalized-search-engine-based-on-big-data-technology)
- [淘宝为什么还不能实现个性化推荐和搜索？](https://www.zhihu.com/question/35574888)
