---
title: 动手学dubbo之Container与SPI
date: 2017-07-13 16:43:01
tags: [dubbo, RPC, SPI, 分布式服务框架, 入门, 教程, 实例]
categories: dubbo
link_title: learning_dubbo_container
toc_number: false
---

在[动手学dubbo之初体验](http://ginobefunny.com/post/learning_dubbo_quick_start/)一文中我们了解了dubbo的架构，接下来的几篇文章我会根据阅读Quick Start里面的demo源码来深入学习dubbo的实现。这一篇主要学习Container的原理、实现和作用。
<!-- more -->

# 一、从启动类Main开始
我们从dubbo-demo-provider\bin\start.bat脚本中可以看出provider启动的入口为com.alibaba.dubbo.container.Main。下面是其具体的源码以及我个人增加的注释：

```java
public class Main {

    private static final String CONTAINER_KEY = "dubbo.container";

    private static final String SHUTDOWN_HOOK_KEY = "dubbo.shutdown.hook";
    
    private static final Logger logger = LoggerFactory.getLogger(Main.class);

    private static final ExtensionLoader<Container> loader = ExtensionLoader.getExtensionLoader(Container.class);
    
    private static volatile boolean running = true;

    public static void main(String[] args) {
        try {
            // 1. 获取需要启动的容器的SPI key
            if (args == null || args.length == 0) {
                // 1.1 如启动参数未传入指定的容器，则使用dubbo.properties配置文件的dubbo.container属性值
                String config = ConfigUtils.getProperty(CONTAINER_KEY, loader.getDefaultExtensionName());
                args = Constants.COMMA_SPLIT_PATTERN.split(config);
            }

            // 2. 获取需要启动的容器的SPI实现类
            final List<Container> containers = new ArrayList<Container>();
            for (int i = 0; i < args.length; i ++) {
                containers.add(loader.getExtension(args[i]));
            }
            logger.info("Use container type(" + Arrays.toString(args) + ") to run dubbo serivce.");

            // 3. 优雅停机处理
            if ("true".equals(System.getProperty(SHUTDOWN_HOOK_KEY))) {
	            Runtime.getRuntime().addShutdownHook(new Thread() {
	                public void run() {
	                    for (Container container : containers) {
	                        try {
	                            container.stop();
	                            logger.info("Dubbo " + container.getClass().getSimpleName() + " stopped!");
	                        } catch (Throwable t) {
	                            logger.error(t.getMessage(), t);
	                        }
	                        synchronized (Main.class) {
                                running = false;
	                            Main.class.notify();
	                        }
	                    }
	                }
	            });
            }

            // 4. 启动容器
            for (Container container : containers) {
                container.start();
                logger.info("Dubbo " + container.getClass().getSimpleName() + " started!");
            }
            System.out.println(new SimpleDateFormat("[yyyy-MM-dd HH:mm:ss]").format(new Date()) + " Dubbo service server started!");
        } catch (RuntimeException e) {
            logger.error(e.getMessage(), e);
            System.exit(1);
        }

        synchronized (Main.class) {
            while (running) {
                try {
                    Main.class.wait();
                } catch (Throwable e) {
                }
            }
        }
    }
}
```

从上面的代码可以看出，其中最重要的就是如何根据 *dubbo.container* 的配置找到对应的容器服务实现并调用start()方法执行启动，dubbo是通过SPI来实现的。

# 二、SPI机制
## 2.1 Java SPI简介
SPI 全称为 Service Provider Interface，是JDK内置的一种服务提供发现机制，目前有不少框架用它来做服务的扩展发现。它是一种动态发现服务的机制，比如有个接口，想运行时动态的给它添加实现，你只需要根据SPI的规则添加一个实现和配置即可。

Java SPI机制的约定如下：
1) 在META-INF/services/目录中创建以接口全限定名命名的文件，文件内容为具体实现类的全限定名；
2) 使用ServiceLoader类动态加载META-INF中的实现类；
3) 如SPI的实现类为Jar则需要放在主程序classpath中；
4) 具体实现类必须要有无参构造方法；

## 2.2 Java SPI的Demo
以下Demo代码已上传至github：https://github.com/ginobefun/learning_projects/tree/master/learning-spi

定义一个用于计算商品搜索得分的接口ScoreService ：
```java
package com.gino.demo.spi.api;

public interface ScoreService {
    /**
     * 计算商品搜索的得分
     * @param tdidfScore 文本相关性得分
     * @param cosScore   用户和商品偏好性得分
     * @return 计算最终得分
     */
    double calScore(double tdidfScore, double cosScore);
}
```

SearchService类使用SPI获取服务实现，执行得分计算并排序：
```java
public class SearchService {
    public static Map<String, Double> search(Map<String, Double> tdidfScoreMap, Map<String, Double> cosScoreMap) {
        ScoreService scoreService;
        ServiceLoader<ScoreService> loader = ServiceLoader.load(ScoreService.class);
        if (loader.iterator().hasNext()) {
            scoreService = loader.iterator().next();
        } else {
            throw new IllegalStateException("Cannot find score services.");
        }

        System.out.println("Use Score Service: " + scoreService.getClass().getName());
        Map<String, Double> finalScoreMap = new HashMap<>();
        tdidfScoreMap.forEach((pId, tdidfScore) -> {
            finalScoreMap.put(pId, scoreService.calScore(tdidfScore, cosScoreMap.get(pId)));
        });

        Map<String, Double> resultMap = new LinkedHashMap<>();
        finalScoreMap.keySet().stream()
                .sorted(Comparator.comparing(pId -> finalScoreMap.get(pId), Comparator.comparingDouble(s -> s)).reversed())
                .forEachOrdered(pId -> {
                    resultMap.put(pId, finalScoreMap.get(pId));
                });

        return resultMap;
    }
}
```

这里有两种计算最终得分的方式，第一种是直接返回用户和商品的个性化得分的ReplaceScoreService：
```java
package com.gino.demo.spi.score;
import com.gino.demo.spi.api.ScoreService;

public class ReplaceScoreService implements ScoreService {
    public double calScore(double tdidfScore, double cosScore) {
        return cosScore;
    }
}
```
根据Java SPI的约定，还需要在META-INF/services/下新建com.gino.demo.spi.api.ScoreService文件，内容为：
```
com.gino.demo.spi.score.ReplaceScoreService
```

另外一种得分是将两者相乘以得到最终得分的MultiplyScoreService：
```java
package com.gino.demo.spi.score;
import com.gino.demo.spi.api.ScoreService;

public class MultiplyScoreService implements ScoreService {
    public double calScore(double tdidfScore, double cosScore) {
        return tdidfScore * cosScore;
    }
}
```
同时也在该工程的META-INF/services/下新建com.gino.demo.spi.api.ScoreService文件，内容为：
```
com.gino.demo.spi.score.MultiplyScoreService
```

在应用的代码中，就可以通过依赖不同的maven工程来实现采用不同的得分计算方式，比如我们采用MultiplyScoreService的时候，maven依赖如下：
```xml
    <artifactId>demo-spi-app</artifactId>
    <groupId>com.gino.demo</groupId>
    <version>1.0-SNAPSHOT</version>
    <modelVersion>4.0.0</modelVersion>
    <packaging>jar</packaging>

    <dependencies>
		<dependency>
            <groupId>com.gino.demo</groupId>
            <artifactId>demo-spi-api</artifactId>
            <version>1.0-SNAPSHOT</version>
		</dependency>
		<dependency>
            <groupId>com.gino.demo</groupId>
            <artifactId>demo-spi-search</artifactId>
            <version>1.0-SNAPSHOT</version>
		</dependency>
		<dependency>
            <groupId>com.gino.demo</groupId>
            <artifactId>demo-spi-multiply</artifactId>
            <version>1.0-SNAPSHOT</version>
		</dependency>
    </dependencies>
```
对应的APP代码如下：
```java
public class App {
    public static void main(String[] args) {
        Map<String, Double> tdidfScoreMap = new HashMap<>();
        tdidfScoreMap.put("product1", Double.valueOf(0.3D));
        tdidfScoreMap.put("product2", Double.valueOf(0.5D));
        tdidfScoreMap.put("product3", Double.valueOf(0.8D));

        Map<String, Double> cosScoreMap = new HashMap<>();
        cosScoreMap.put("product1", Double.valueOf(0.2D));
        cosScoreMap.put("product2", Double.valueOf(0.7D));
        cosScoreMap.put("product3", Double.valueOf(0.4D));

        Map<String, Double> resultMap = SearchService.search(tdidfScoreMap, cosScoreMap);
        System.out.println("Search Result: " + resultMap);
    }
}
```

执行后控制台输出：
```
Use Score Service: com.gino.demo.spi.score.MultiplyScoreService
Search Result: {product2=0.35, product3=0.32000000000000006, product1=0.06}
```

如果修改maven依赖为依赖demo-spi-replace，则会调用ReplaceScoreService进行得分计算。

## 2.3 dubbo的SPI
通过查看ExtensionLoader源码发现，在dubbo里并没有直接采用Java SPI，而是参考其重新设计了一套SPI机制，在[Dubbo的ExtensionLoader](http://www.hiwzc.com/java-spi-dubbo-extensionloader.html)文章里列举了两者之间的差别主要有：

1） ServiceLoader是采用迭代器遍历的方式实现的，而Dubbo为每种实现指定一个名称，由名称和服务共同确定一个实现，这样做的好处是，可以为成套的服务接口指定相同的名称，比如指定使用dubbo协议后，协议使用的其他扩展点就自动加载名称为dubbo的实现。此外，指定服务名称可以根据名称来获取扩展点实现实例，不像ServiceLoader那样在遍历过程中创建永远不会使用的服务实例。
2）Dubbo提供了一种类似IoC的机制，即一个扩展点可以直接setter注入其它扩展点；
3）基于线程安全和性能的考虑，Dubbo采用了ConcurrentMap来缓存实现类的实例；
4）Dubbo要求服务必须是一个接口；
5）ServiceLoader在解析配置出错时会抛出异常，如果捕获了这种异常，而不进行额外的处理，那么后面需要这种实例时，由于没有成功实例化，又会抛出新的异常，而新抛出的异常不能指示真正的错误原因。dubbo的实现是将解析配置时发生的异常保存起来，当访问这种实例时，通过查找保存的异常，抛出真正的原因。

## 2.4 ExtensionLoader核心源码
### 2.4.1 SPI注解
Dubbo首先定义了一个SPI注解，只有标记了该注解的服务，Dubbo SPI机制才能为其加载具体实现。value属性用于配置该服务的默认实现名称。

### 2.4.2 Adaptive注解
Adaptive注解标注一个扩展点的Adaptive实现，一个扩展点最多只能有一个Adaptive实现。Adaptive标注的实现不提供具体的功能，而是作为一个适配器，根据不同的情况选择具体的实现。这个有点抽象，在后续碰到的时候再结合具体的例子和源码学习。

### 2.4.3 Activate注解
Activate注解用于配置扩展点实现的激活条件和排列顺序。

### 2.4.4 Holder辅助类
Holder类用于保存一个值，并通过给值添加volatile来保证线程可见性。

### 2.4.5 ExtensionLoader的静态成员和实例成员
```java
    // SERVICES_DIRECTORY、DUBBO_DIRECTORY和DUBBO_INTERNAL_DIRECTORY定义了3个配置文件查询目录，
    // 即META-INF/dubbo、META-INF/dubbo/internal和META-INF/services，ExtensionLoader支持从这三个地方加载扩展点配置。
    private static final String SERVICES_DIRECTORY = "META-INF/services/";
    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";
    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";

    // EXTENSION_LOADERS用于缓存所有扩展点的ExtensionLoader实例。
    private static final Map<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();

    // EXTENSION_INSTANCES用于缓存所有扩展点实现的实例。
    private static final Map<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();

    // type成员记录了该加载器要加载的扩展点类型，即标注了SPI注解的接口。
    private final Class<?> type;

    // objectFactory是获取对象的工厂
    private final ExtensionFactory objectFactory;

    // cachedNames及其他的实例成员缓存了扩展点相关的信息
    private final Map<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();

    // ......
```

### 2.4.6 获取ExtensionLoader的工厂方法
```java
    // 首先判断扩展点是否为空，是否是接口，是否标注了SPI注解，如果都满足，则查看该扩展点是否已经创建过加载器实例，
    // 如果没有，则调用构造方法创建一个加载器实例并缓存起来。
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null)
            throw new IllegalArgumentException("Extension type == null");
        if(!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        if(!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type + 
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }
        
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }

    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

### 2.4.7 获取扩展点实现
```java
    // 首先判断是否指定了扩展点名称，如果没有指定，则抛出异常，如果指定的名称为true，则返回默认的扩展点。
    // 然后查看缓存的实例中有没有指定的实现，如果没有，则创建指定的实现。如果有则直接返回缓存的实例。
    public T getExtension(String name) {
		if (name == null || name.length() == 0)
		    throw new IllegalArgumentException("Extension name == null");
		if ("true".equals(name)) {
		    return getDefaultExtension();
		}
		Holder<Object> holder = cachedInstances.get(name);
		if (holder == null) {
		    cachedInstances.putIfAbsent(name, new Holder<Object>());
		    holder = cachedInstances.get(name);
		}
		Object instance = holder.get();
		if (instance == null) {
		    synchronized (holder) {
	            instance = holder.get();
	            if (instance == null) {
	                instance = createExtension(name);
	                holder.set(instance);
	            }
	        }
		}
		return (T) instance;
	}

    // 首先查看缓存的扩展点实现类中有没有包含这个扩展，如果没有则报错，因为ExtensionLoader只解析一次并缓存所有的扩展点实现类，
    // 此行为是getExtensionClasses实现的。如果找到了扩展点实现类，则先从缓存EXTENSION_INSTANCES中查看是否已经存在该实现类的实例化对象，
    // 如果没有找到，则创建新的实例并缓存到EXTENSION_INSTANCES中，否则使用找到的实例，
    // 然后调用injectExtension方法注入该扩展点依赖的其他扩展实现，并为该实例创建所有包装类。
    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && wrapperClasses.size() > 0) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```

# 三、dubbo里有哪些Container?
## 3.1 Container SPI定义
```java
// 默认使用SpringContainer
@SPI("spring")
public interface Container {
    void start();
    
    void stop();
}
```

## 3.2 dubbo里有哪些Container实现

![dubbo_containers.png](http://upload-images.jianshu.io/upload_images/3709321-35f17414c406361c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.3 以SpringContainer为例了解如何实现dubbo SPI 
```java
package com.alibaba.dubbo.container.spring;

import com.alibaba.dubbo.common.logger.Logger;
import com.alibaba.dubbo.common.logger.LoggerFactory;
import com.alibaba.dubbo.common.utils.ConfigUtils;
import com.alibaba.dubbo.container.Container;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class SpringContainer implements Container {

    private static final Logger logger = LoggerFactory.getLogger(SpringContainer.class);
    public static final String SPRING_CONFIG = "dubbo.spring.config";
    public static final String DEFAULT_SPRING_CONFIG = "classpath*:META-INF/spring/*.xml";
    static ClassPathXmlApplicationContext context;

    public static ClassPathXmlApplicationContext getContext() {
        return context;
    }

    public void start() {
        String configPath = ConfigUtils.getProperty(SPRING_CONFIG);
        if (configPath == null || configPath.length() == 0) {
            configPath = DEFAULT_SPRING_CONFIG;
        }
      
        // 使用ClassPathXmlApplicationContext加载指定目录下的Spring配置文件
        context = new ClassPathXmlApplicationContext(configPath.split("[,\\s]+"));
        context.start();
    }

    public void stop() {
        try {
            if (context != null) {
                context.stop();
                context.close();
                context = null;
            }
        } catch (Throwable e) {
            logger.error(e.getMessage(), e);
        }
    }
}
```

在META-INF/dubbo/internal下新建文件com.alibaba.dubbo.container.Container，并保存内容为：
```
spring=com.alibaba.dubbo.container.spring.SpringContainer
```

# 四、小结
- 本文主要是阅读了dubbo中container模块的源码，通过学习了解到其核心在于dubbo SPI来实现服务发现，从而启动特定的容器；
- 通过阅读源码，可以发现dubbo能很好地与Spring进行集成，但是它们之间的关系并非耦合。另外通过SPI机制，能够非常容易地进行功能扩展，这也是我认为dubbo架构设计中非常棒的一个部分；
- 在dubbo源码中，常使用的Container包括Log4jContainer、LogbackContainer和SpringContainer，通过对dubbo SPI的学习，我们也可以扩展实现自定义的Container。

# 参考资源
- [dubbo用户指南](http://dubbo.io/User+Guide-zh.htm)
- [dubbo开源代码](https://github.com/alibaba/dubbo)
- [Java SPI机制-3：Dubbo的ExtensionLoader](http://www.hiwzc.com/java-spi-dubbo-extensionloader.html)

**动手学dubbo系列**
1. [动手学dubbo之初体验](http://ginobefunny.com/post/learning_dubbo_quick_start/)
2. [动手学dubbo之Container与SPI](http://ginobefunny.com/post/learning_dubbo_container/)