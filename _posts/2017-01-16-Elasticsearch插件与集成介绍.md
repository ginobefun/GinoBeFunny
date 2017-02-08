---
title: Elasticsearch插件与集成介绍
date: 2017-01-16 15:35:53
tags: [Elasticsearch, Elasticsearch插件]
categories: Elasticsearch
link_title: elasticsearch_plugins_and_integrations
---
Elasticsearch的能力虽然已经非常强大，但是它也提供了基于插件的扩展功能，基于此我们可以扩展查询、分词、监控、脚本等能力。
这是学习Elasticsearch插件的第一篇，主要是阅读[官方文档](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/index.html)的笔记，介绍官方的一些插件和优秀的社区插件；后面一篇主要是通过源码来深入学习Elasticsearch插件的开发，并通过实战开发一个自定义的插件。

<!-- more -->
## Plugin Management
- Plugins are a way to enhance the core Elasticsearch functionality in a custom manner. They range from adding custom mapping types, custom analyzers, native scripts, custom discovery and more.
- Site plugins and mixed plugins are deprecated and will be removed in 5.0.0. Instead, site plugins should either be migrated to Kibana or should use a standalone web server.
- The *plugin* script is used to install, list, and remove plugins. 


    sudo bin/plugin install [plugin_name]   #Core Elasticsearch plugins
    sudo bin/plugin install [org]/[user|component]/[version] #Community and non-core plugins
    sudo bin/plugin install [url]           #Custom URL or file system
    sudo bin/plugin list                    #Listing plugins
    sudo bin/plugin remove [plugin_name]    #Removing plugins
    
## API Extension Plugins
API extension plugins add new functionality to Elasticsearch by adding new APIs or features, usually to do with search or mapping.

- [Core][Delete By Query Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/plugins-delete-by-query.html)：Elasticsearch 1.0版本有一个delete-by-query的API，由于存在兼容性、一致性和可靠性的问题而被[移除](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/delete-by-query-plugin-reason.html)，该插件使用scroll获取文档ID和版本然后使用bulk进行批量删除；
- [carrot2 Plugin](https://github.com/carrot2/elasticsearch-carrot2)：可以自动地将相似的文档组织起来，并且给每个文档的群组分类贴上相应的较为用户可以理解的标签。这样的聚类也可以看做是一种动态的针对每个搜索和命中结果集合的动态 facet。可以在[Carrot2 demo page](http://search.carrot2.org/stable/search?query=elasticsearch&results=200&view=foamtree)体验一下这个工具。
- [Elasticsearch Image Plugin](https://github.com/kzwang/elasticsearch-image)：基于[LIRE](https://github.com/dermotte/lire)的相似图片搜索插件；
- [Entity Resolution Plugin](https://github.com/YannBrrd/elasticsearch-entity-resolution)：基于贝叶斯概率模型去除重复数据的插件；
- [SQL language Plugin](https://github.com/NLPchina/elasticsearch-sql/)：支持采用SQL查询的ES插件；
- [Elasticsearch Taste Plugin](https://github.com/codelibs/elasticsearch-taste)：基于用户和内容推荐的ES插件；

## Analysis Plugins
Analysis plugins extend Elasticsearch by adding new analyzers, tokenizers, token filters, or character filters to Elasticsearch.

- [Core][ICU](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/analysis-icu.html)：使用ICU实现的一个针对亚洲语言的分词器插件；
- [Core][SmartCN](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/analysis-smartcn.html)：官方提供的一个基于概率的针对中文或中英混合的分词器；
- [Combo Analysis Plugin](https://github.com/yakaz/elasticsearch-analysis-combo/)：通常一个分析器里只能配置一个分词器，该插件支持能配置多个分词器组合；
- [IK Analysis Plugin](https://github.com/medcl/elasticsearch-analysis-ik)：一个非常流行的中文分析器插件，迁移自Lucene的IK分析器；
- [Mmseg Analysis Plugin](https://github.com/medcl/elasticsearch-analysis-mmseg)：基于MMSEG算法的中文分析器，在中英混合时分词效果较差；
- [Pinyin Analysis Plugin](https://github.com/medcl/elasticsearch-analysis-pinyin)：将中文转换为拼音的分析器，支持首字母和连接符配置；

## Discovery Pluginsedit
Discovery plugins extend Elasticsearch by adding new discovery mechanisms that can be used instead of [Zen Discovery](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/modules-discovery-zen.html).

- [Core]AWS Cloud/Azure Cloud/GCE Cloud：官方提供的基于各种云服务的插件；
- [ZooKeeper Discovery Plugin](https://github.com/grmblfrz/elasticsearch-zookeeper)：基于ZooKeeper的Elasticsearch集群发现插件；
- [Kubernetes Discovery Plugin](https://github.com/fabric8io/elasticsearch-cloud-kubernetes)：使用K8 API单播发现插件；

## Security Plugins
- [Kerberos/SPNEGO Realm](https://github.com/codecentric/elasticsearch-shield-kerberos-realm)：基于Kerberos/SPNEGO验证HTTP和传输请求。
- [Readonly REST](https://github.com/sscarduzio/elasticsearch-readonlyrest-plugin)：只对外暴露查询相关的操作，拒绝删除和更新操作。

## Integrations
Integrations are not plugins, but are external tools or modules that make it easier to work with Elasticsearch.

- [JDBC importer](https://github.com/jprante/elasticsearch-jdbc)：通过JDBC将数据库的数据导入到Elasticsearch中；
- [Kafka Standalone Consumer(Indexer)](https://github.com/BigDataDevs/kafka-elasticsearch-consumer)：读取kafka消息并处理，最终批量写入到Elasticsearch中；
- [Mongolastic](https://github.com/ozlerhakan/mongolastic)：将MongoDB的数据迁移到Elasticsearch中；
- [Scrutineer](https://github.com/Aconex/scrutineer)：比较Elasticsearch和数据库中数据的一致性；

## Help for plugin authors
插件描述文件plugin-descriptor.properties可以参考：https://github.com/elastic/elasticsearch/blob/2.4/dev-tools/src/main/resources/plugin-metadata/plugin-descriptor.properties

## 参考资料
[Elasticsearch Plugins and Integrations](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/index.html)

