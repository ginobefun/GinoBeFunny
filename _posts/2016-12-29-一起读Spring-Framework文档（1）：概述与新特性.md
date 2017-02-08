---
title: 一起读Spring Framework文档（1）：概述与新特性
date: 2016-12-29 10:15:49
tags: [Spring, Spring Framework, 一起读文档]
categories: Spring
link_title: spring_framework_reference_notes_part1_overview_and_newfeature
toc_number: false
---
主要包括Spring Framework的介绍、快速入门、模块体系以及4.x各个版本的新特性和改进。
<!-- more -->

**阅读说明：**
1. Spring经过多年的发展，现在已经有多个成熟的项目，包括Spring Framework、Spring Boot、Spring Cloud、Spring Data等，但是大家最熟知的应该还是Spring Framework，在这一系列的博文中，如无特殊说明，Spring指的皆为Spring Framework；
2. Spring中很大一部分我没有用过，在笔记中可能存在错误描述，欢迎指正。另外如果有些词不好翻译，会保留英文单词；
3. 该笔记阅读的文档是Spring最新的4.3.5版本，官网地址可能随着新版本的发布而替换；
4. 部分内容会因为个人喜好进行增减，如需完整阅读，可以阅读官网原文。

# 第一部分 Spring概述
- 轻量级的开源框架；
- 潜在的一站式企业级应用开发解决方案；
- 模块化的设计；
- 非侵入式的设计；
- 支持声明式的事务管理；
- 提供全功能的MVC框架；
- 能将AOP透明地集成；

## 1. Spring快速入门
Spring Framework Reference主要介绍的是框架的详细信息，如果是新手，建议从官的[Guides](https://spring.io/guides)的实例入手（比如[Building REST services with Spring](https://spring.io/guides/tutorials/bookmarks/)这个就超级棒），这些实例很多都是基于Spring Boot来运行。

## 2. Spring介绍
Spring为开发Java提供了基础架构的支持，基于普通的POJO使得开发J2EE更加轻松。

### 2.1 依赖注入和控制反转
通常一个Java程序包含了大量的对象需要管理，而这些对象之间又存在各种依赖关系。这时虽然一些设计模式如工厂模式、构造器、服务定位的方式可以采用，但是更好的方式利用一个统一的模式：我们只需描述是什么以及在哪里使用即可，剩下的一起交给框架去实现。

Spring的控制反转组件（*Inversion of Control, IOC*）开箱即用，其通过一系列的组件来形成统一的模式来实现。Spring的IOC是Spring最经典的设计之一，越来越多的公司采用这种方案。

关于IOC，应该读读Martin Fowler大神的《[Inversion of Control Containers and the Dependency Injection pattern](http://martinfowler.com/articles/injection.html)》一文。

### 2.2 Spring的模块
Spring现在已经发展到包括大约20个模块，按功能可分为：核心容器、数据访问与集成、Web开发、面向切面（AOP）、类代理、消息和测试，整体框架图如下所示：

![Spring Modules](http://oi46mo3on.bkt.clouddn.com/8_spring_ref/spring_modules.png)

#### 2.2.1 核心容器
- **spring-core**和**spring-beans**是Spring最基础的模块，提供了控制反转（IOC）和依赖注入（DI）特性。**BeanFactory**接口解耦了程序中的配置和依赖关系。
- **spring-context**使得访问对象更加便捷，还增加了对国际化、事件传播、资源加载的支持，以及透明化创建上下文的能力。**ApplicationContext**是该模块的核心接口。
- **spring-context-support**提供了常见的第三方库集成到Spring上下文的支持，如缓存（EhCache, Guava, JCache）、通知（JavaMail）、调度（CommonJ, Quartz）和模板引擎（FreeMarker, JasperReports, Velocity）。
- **spring-expression**提供了一个强大的表达式语言（EL）来查询和操作运行时的对象图。该表达式语言支持属性赋值、方法调用、访问集合、算术和逻辑运算等能力。

#### 2.2.2 AOP和类代理（Instrumentation）
- **spring-aop**提供了面向切面编程（*Aspect Oriented Programming, AOP*）的能力，利用方法拦截器和切入点能使原始方法和新的逻辑完全分离。使用源码级的元数据功能，还可以将行为信息合并到你的代码中。
- **spring-aspects**集成了AspectJ（另外一个流行的AOP利器）；
- **spring-instrument**为应用服务器提供了类代理支持（*注：这个地方叫做类代码不知道正确与否，主要是参考了[Java SE 6 新特性: Instrumentation 新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/)一文 随着后面的阅读再来更正*）和类加载器实现。
- **spring-instrument-tomcat**模块是为Tomcat服务器实现的类代理。

#### 2.2.3 消息
**spring-messaging**模块为信息通信处理提供了一套包含抽象概念的实现，包括*Message*，*MessageChannel*，*MessageHandler*等，另外还提供了一些易用的注解。

#### 2.2.4 数据访问与集成
- **spring-jdbc** 模块提供了一个JDBC抽象层，消除冗长的JDBC编码和数据库厂商特有的错误码。
- **spring-tx**模块支持编程和声明式事务，而这只需实现特定的接口甚至是简单的POJO。
- **spring-orm**模块集成了流行的对象关系映射（ORM）API，包括JPA、JDO和Hibernate。
- **spring-oxm**模块提供了“对象XML映射”抽象实现，支持JAXB, Castor, XMLBeans, JiBX和XStream。
- **spring-jms**模块包含了生产和消费信息的功能。4.1版本后还提供了与**spring-messaging**模块的集成。

#### 2.2.5 Web开发
- **spring-web**模块提供了基本的面向Web的集成特性，例如方文件上传功能、使用Servlet的监听器以及一个面向Web应用程序上下文IoC容器的初始化。它还包含一个HTTP客户端和远程访问相关的部分。
- **spring-webmvc**模块包含Spring的模型-视图-控制器（*MVC*）和REST Web服务实现。Spring MVC框架使得领域模型代码和Web展示的代码完全分离，并能与其他Spring功能无缝集成。
- **spring-webmvc-portlet**模块为Portlet环境提供了Spring MVC实现。

#### 2.2.6 测试
**spring-test**模块支持Spring这些组件基于Junit或TestNG的单元测试和集成测试。它提供了一致的上下文加载和缓存，另外它还使得对象的Mock更加容易。

### 2.3 Spring的使用场景
++典型的基于Spring的Web应用++

![典型的基于Spring的Web应用](http://oi46mo3on.bkt.clouddn.com/8_spring_ref/spring_use_for_web.png)

- 持久化层可以利用**spring-orm**来实现对象关系映射，用**spring-tx**来声明事务；
- 逻辑层可以基于POJO，并使用IOC容器来管理对象；另外还可以使用邮件发送通知，远程服务实现RPC；
- 展示层可以使用**spring-webmvc**分离逻辑和展示；

++与其他Web开发框架集成++

![与其他Web开发框架集成](http://oi46mo3on.bkt.clouddn.com/8_spring_ref/spring_use_for_other_web_framework.png)

++远程服务调用++

![远程服务调用](http://oi46mo3on.bkt.clouddn.com/8_spring_ref/spring_use_for_remoting.png)

++EJB集成++

![EJB集成](http://oi46mo3on.bkt.clouddn.com/8_spring_ref/spring_use_for_ejb.png)

#### 2.3.1 依赖管理与命名约定
- Maven依赖仓库：Maven的中央仓库和一些特别为Spring建立的公共Maven仓库；
- 命名约定：GroupId为*org.springframework*，ArtifactId为各个模块名如*spring-aop*；
- 最小化依赖原则：不必依赖所有的模块，按需依赖；

Maven依赖配置：

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>4.3.5.RELEASE</version>
            <scope>runtime</scope>
        </dependency>
    </dependencies>

Gradle依赖配置：

    repositories {
        mavenCentral()
        // and optionally...
        maven { url "http://repo.spring.io/release" }
    }
    
    dependencies {
        compile("org.springframework:spring-context:4.3.5.RELEASE")
        testCompile("org.springframework:spring-test:4.3.5.RELEASE")
    }

#### 2.3.2 日志框架
日志框架依赖的问题对于Spring框架非常的重要，因为：
- 它是Spring仅有的必须的外部依赖；
- 应用程序本身也离不开日志；
- Spring所集成的第三方框架本身也会依赖日志框架；

Spring采取的方案是在**spring-core**模块中依赖**commons-logging**，其他模块通过传递依赖来依赖该模块。这样的好处是你无须做特别的处理，程序会在classpath或特定的路径寻找最合适的日志框架，即使找不到也会有看着不错的日志输出。

另外一种选择是放弃**commons-logging**改用SLF4J（*Simple Logging Facade for Java，基于门面模式本身并不负责记录日志、支持常用的日志框架*），依赖配置如下：

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>4.3.5.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>commons-logging</groupId>
                    <artifactId>commons-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>1.5.8</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.5.8</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.5.8</version>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.14</version>
        </dependency>
    </dependencies>

# 第二部分 Spring框架4.x版本新特性
## 3. Spring 4.0版本新特性与改进
Spring自从2004年发布以来，经历了多个重大版本的演变。在Spring 2.0中提供了XML命名空间和AspectJ支持；在Spring 2.5中拥抱注解驱动的配置；在Spring 3.0在框架代码中大量使用Java 5+的特性，如*@Configuration*。

Spring 4.0是目前的最新主要版本，首次支持Java 8的功能，当前你也可以使用旧版本的Java，但是最低要求是Java 6。

在这个版本中，Spring清除了很多过时的类和方法。具体升级时可以参考[migration guide for upgrading to Spring 4.0](https://github.com/spring-projects/spring-framework/wiki/Migrating-from-earlier-versions-of-the-spring-framework)指南。

### 3.1 改进快速入门体验
网站改版，同时例子补充得更完整更易理解

### 3.2 移除废弃的包和类
同时对第三方依赖的框架也进行了版本升级

### 3.3 对Java 8的支持
包括Lambda表达式、方法引用、java.time；

### 3.4 对Java EE 6和7的支持
- J2EE 6：JPA 2.0, Servlet 3.0;
- J2EE 7：JMS 2.0, JTA 1.2, JPA 2.1, Bean Validation 1.1, JSR-236;

### 3.5 使用Groovy的领域专业语言（DSL）定义Bean
举个栗子：

    def reader = new GroovyBeanDefinitionReader(myApplicationContext)
    reader.beans {
        dataSource(BasicDataSource) {
            driverClassName = "org.hsqldb.jdbcDriver"
            url = "jdbc:hsqldb:mem:grailsDB"
            username = "sa"
            password = ""
            settings = [mynew:"setting"]
        }
        sessionFactory(SessionFactory) {
            dataSource = dataSource
        }
        myService(MyService) {
            nestedBean = { AnotherBean bean ->
                dataSource = dataSource
            }
        }
    }

### 3.6 核心容器改进
- 注入Bean时支持泛型作为一个标识符，如@Autowired Repository<Customer> customerRepository；
- Spring的元注解支持定制来实现暴露源注解的特定属性；
- 集合和数组属性在装配时支持排序；
- @Lazy注解允许在注入点和@Bean定义时使用；
- 引入新的@Description注解；
- 通过@Conditional注解实现了条件过滤Bean，和@Profile不同之处在于@Conditional可以通过自定义编程来过滤；
- 基于CGLIB的代理类不再需要默认的构造函数，从而实现了无须调用构造函数的代理；
- 通过LocaleContext支持对时区进行管理；

### 3.7 Web开发改进
- 全面采用Servlet 3；
- 增加@RestController注解，这样无须在RequestMapping的方法上添加@ResponseBody注解；
- AsyncRestTemplate支持非阻塞异步REST请求；
- Spring MVC中更易理解的时区机制；

### 3.8 WebSocket、SockJS以及STOMP通信
- 增加了**spring-websocket**模块支持JSR-356;
- 增加了**spring-messaging**模块支持STOMP，另外该工程还对消息通信进行了抽象以便给其他模块集成；

### 3.9 测试方面的改进
- 几乎所有的Spring注解都可以在测试代码中使用；
- Active bean definition profiles可以通过自定义编码来实现；
- **spring-core**工程的SocketUtils类可以扫描本地空闲的TCP和UDP端口；
- 大部分的Servlet相关的Mock类都升级到Servlet 3.0版本了；

## 4. Spring 4.1版本新特性与改进
### 4.1 JMS的改进
- Spring 4.1引进了一个更简单的方式来注册JMS监听，即给Bean的方法上增加@JmsListener注解，同时XML命名空间上也增加了jms:annotation-driven，使用JmsListenerConfigurer还可以实现完全自定义编程的注册。
- 由于**spring-messaging**模块的抽象定义，消息监听可以变得更灵活。另外像@Payload、@Header、@Headers、@SendTo这样的注解的引入使得代码编写更加清晰；
- JmsMessageOperations的引入使得JmsTemplate调用更加便捷；
- JmsTemplate支持同步的请求应答；
- 可以给多个<jms:listener/>设置优先级；
- 支持JMS 2.0的共享消费者；

### 4.2 缓存改进
- 在无须更改现有的缓存配置下支持JCache (JSR-107)；
- 使用CacheResolver支持运行期的缓存，从而缓存的value参数不是必须的了；
- 支持更多操作级的定制，如缓存使用、缓存管理、缓存key生成；
- 通过类级别的@CacheConfig注解允许一些配置在类级别共享，而不需要任何额外的缓存操作；
- 通过CacheErrorHandler更好的异常处理；
- Cache接口增加了新的方法putIfAbsent；

### 4.3 Web开发改进
- 基于资源处理现有的支持ResourceHttpRequestHandler 已经扩展了新的抽象ResourceResolver，ResourceTransformer和ResourceUrlProvider；许多内置的实现支持多版本的URL资源、定位gzip压缩资源、生成HTML 5声明等能力。
- @RequestParam、@RequestHeader和@MatrixVariable这些控制器的方法参数支持Java 8的java.util.Optional;
- 当一个服务已经返回ListenableFuture时，可以使用其作为返回来替代原本采用的DeferredResult；
- 按照相互依赖的顺序关系来调用@ModelAttribute；
- Jackson的@JsonView注解可以直接在带有@ResponseBody和ResponseEntity的控制器方法上使用；
- Jackson支持JSONP；
- 声明一个@ControllerAdvice的Bean可以在控制器方法已经结束但响应写入之前的时机被调用；
- HttpMessageConverter支持三个新的选项：Gson、Protobuf以及Jackson基于XML的序列化；
- JSP这样的视图文件可以通过名字定义指向控制器方法的链接；
- ResponseEntity提供了一个内建风格的API来声明服务器端的响应，例如方法ResponseEntity.ok()；
- RequestEntity提供了一个内置风格的API来声明客户端REST代码对HTTP请求；
- MVC方面：视图解析器现在可以配置内容negotiation、默认已支持视图控制器重定向和设置响应状态、默认已支持自定义的路由；
- 支持Groovy的标记模板；

### 4.4 WebSocket通信改进
- 支持SockJS客户端；
- 当STOMP客户端订阅和取消订阅时会发布新的上下文事件SessionSubscribeEvent和SessionUnsubscribeEvent；
- @SendToUser只针对单一会话不需要身份验证；
- @MessageMapping方法使用“.”替换原来的“/”作为路径分隔符；

### 4.5 测试方面的改进
- 通过TestTransaction API可以测试事务；
- 通过@Sql和@SqlConfig注解为每个类或方法执行SQL脚本；
- 通过@TestPropertySource注解实现测试属性值覆盖应用和系统的属性值；

## 5. Spring 4.2版本新特性与改进
### 5.1 核心容器的改进
- 支持Java 8接口默认方法上的@Bean注解解析；
- Configration类可以在普通的组件类上使用@Import注解；
- Confugration类可以通过@Order注解实现排序；
- @Resource注入点支持@Lazy声明；
- 事件处理机制引入了基于注解模型的能力；
- 对注释的属性别名的声明和查找提供了先天的支持；
- 元注解的查找进行了多项优化；
- DefaultConversionService和DefaultFormattingConversionService提供了对字符、金额、时区的转换能力；

### 5.2 数据访问的改进
- 通过AspectJ支持javax.transaction.Transactional；
- 支持Hibernate ORM 5.0；
- 内置的数据库可以自动分配唯一的名称；

### 5.3 JMS的改进
- 通过JmsListenerContainerFactory可以控制autoStartup属性；
- 可以给每个监听的容器配置回复的Destination；
- 在同一方法上可以配置多个@JmsListener；

### 5.4 Web开发的改进
- 支持HTTP流和服务器发送事件；
- 内置支持CORS，包括全局和本地的配置；
- HTTP缓存机制优化包括新的CacheControl构造器、ETag/Last-Modified改进；
- 支持自定义的Mapping注解；
- 通过AbstractHandlerMethodMapping在运行时注册和取消注册请求Mapping；
- @Controller方法返回类型支持java.util.concurrent.CompletableFuture；
- RestTemplate集成okhttp；

### 5.5 WebSocket通信改进
- 暴露有关连接用户和订阅状态的信息；
- 解决跨服务器的集群用户destinations；
- StompSubProtocolErrorHandler来自定义STOMP错误处理；

### 5.6 测试方面的改进
- 基于JUnit的集成测试，现在可以使用JUnit的规则，而不是执行 SpringJUnit4ClassRunner。这使得基于Spring集成测试可以通过JUnit的执行器或第三方执行器（如MockitoJUnitRunner）来执行；
- 为HtmlUnit提供先天支持；
- AopTestUtils是一种新的测试工具类，它允许开发者获得隐藏着一个或多个Spring代理基础目标对象的引用；
- ReflectionTestUtils现在支持设置和获取static属性，包括常量；
- @Commit替换原来的 @Rollback(false)；

## 6. Spring 4.3版本新特性与改进
### 6.1 核心容器改进
- 核心容器异常时提供更丰富的元数据信息；
- Bean属性的getters/setters支持Java 8的默认方法；
- 当注入一个Primary的Bean时，懒加载的候选Bean不会被立即创建；
- @Configuration 类支持构造函数注入；
- @Scheduled适用于任何范围的Bean；

### 6.2 数据访问的改进
jdbc:initialize-database和jdbc:embedded-database可以给每个脚本应用单独的可配置separator；

### 6.3 缓存方面的改进
- 在指定Key上面的并发调用变成同步的，使得缓存值只会被计算一次；该特性需要通过@Cacheable的sync属性开启；
- Cache接口增加了get(Object key, Callable<T> valueLoader)方法；
- ConcurrentMapCacheManager和ConcurrentMapCache现在可以通过storeByValue属性进行缓存的序列化；

### 6.4 JMS的改进
- @SendTo现在可以在类级别指定共用的一个回复destination。
- @JmsListener和@JmsListeners现在可以用作元注解来创建自定义的支持属性覆盖的复合注解；

### 6.5 Web开发的改进
- 内置支持HTTP HEAD和HTTP OPTIONS；
- 新的注解：@GetMapping、@PostMapping、@PutMapping、@DeleteMapping和@PatchMapping；
- 新的注解：@RequestScope、@SessionScope和@ApplicationScope；
- @ResponseStatus支持类级别并被所有方法继承；
- 在HTTP消息转换时采用一致的字符集处理；
- AsyncRestTemplate 支持请求拦截；

### 6.6 WebSocket通信改进
@SendTo现在可以在类级别指定共用的一个回复destination。

### 6.7 测试方面的改进
- Spring测试上下文需要Junit 4.12以上的版本；
- SpringJUnit4ClassRunner增加新的别名SpringRunner；
- 测试的有关注解现在可以在接口中声明；

### 6.8 第三方框架版本更新
- Hibernate ORM 5.2
- Hibernate Validator 5.3
- Jackson 2.8
- OkHttp 3.x
- Tomcat 8.5
- Netty 4.1
- Undertow 1.4
- WildFly 10.1

