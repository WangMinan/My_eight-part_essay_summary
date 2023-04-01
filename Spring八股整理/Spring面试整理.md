<div align="center">
    <h1>
        🥬Spring面试整理
    </h1>
</div>


## 1. Spring refresh 流程

**要求**

* 掌握 refresh 的 12 个步骤

### 1.0 Spring refresh 概述

refresh 是 `AbstractApplicationContext` 中的一个方法，负责初始化 `ApplicationContext` 容器，容器必须调用 refresh 才能正常工作。它的内部主要会调用 12 个方法，我们把它们称为 refresh 的 12 个步骤：

1. prepareRefresh

2. obtainFreshBeanFactory

3. prepareBeanFactory

4. postProcessBeanFactory

5. invokeBeanFactoryPostProcessors

6. registerBeanPostProcessors

7. initMessageSource

8. initApplicationEventMulticaster

9. onRefresh

10. registerListeners

11. finishBeanFactoryInitialization

12. finishRefresh


<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230327113829969.png" alt="image-20230327113829969" style="zoom:50%;" />

> ***功能分类***
>
> * 1 为准备环境
>
> * 2 3 4 5 6 为准备 `BeanFactory`
>
> * 7 8 9 10 12 为准备 `ApplicationContext`
>
> * 11 为初始化 `BeanFactory` 中非延迟单例 bean



### 1.1 prepareRefresh

* 这一步创建和准备了 Environment 对象，它作为 ApplicationContext 的一个成员变量

* Environment 对象的作用之一是为后续 @Value，值注入时提供键值信息
* Environment 分成三个主要部分
  * systemProperties - 保存 java 环境键值
  * systemEnvironment - 保存系统环境键值
  * 自定义 PropertySource - 保存自定义键值，例如来自于 *.properties 文件的键值

![image-20210902181639048](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902181639048.png)

**案例演示: 如何获取@Value中的内容**

```java
// 如何获得和解析 @Value 内容
public class TestEnvironment {
    public static void main(String[] args) throws NoSuchFieldException, IOException {
        // 1) 获得 @Value 的值
        System.out.println("=======================> 仅获取 @Value 值");
        // 配置解析器
        QualifierAnnotationAutowireCandidateResolver resolver = 
        	new QualifierAnnotationAutowireCandidateResolver();
        Object name = 
        	resolver.getSuggestedValue(
        		new DependencyDescriptor(
        			Bean1.class.getDeclaredField("name"), false));
        System.out.println(name);

        // 2) 解析 @Value 的值
        System.out.println("=======================> 获取 @Value 值, 并解析${}");
        Object javaHome = resolver.getSuggestedValue(
        	new DependencyDescriptor(Bean1.class.getDeclaredField("javaHome"), false));
        System.out.println(javaHome);
        System.out.println(getEnvironment().resolvePlaceholders(javaHome.toString()));

        // 3) 解析 SpEL 表达式
        System.out.println("=======================> 获取 @Value 值, 并解析#{}");
        Object expression = resolver.getSuggestedValue(
        	new DependencyDescriptor(Bean1.class.getDeclaredField("expression"), false));
        System.out.println(expression);
        String v1 = getEnvironment().resolvePlaceholders(expression.toString());
        System.out.println(v1);
        System.out.println(
        	new StandardBeanExpressionResolver()
        		.evaluate(v1, new BeanExpressionContext(new DefaultListableBeanFactory(),null)));
    }

    private static Environment getEnvironment() throws IOException {
        // 读取系统环境变量
        StandardEnvironment env = new StandardEnvironment();
        // 补充指定自定义文件
        env.getPropertySources().addLast(
        	new ResourcePropertySource("jdbc", new ClassPathResource("jdbc.properties")));
        return env;
    }

    static class Bean1 {
        @Value("hello")
        private String name;

        @Value("${jdbc.username}")
        private String javaHome;

        // SpringEL表达式,先解析${}内部分,再连起来解析外部
        @Value("#{'class version:' + '${java.class.version}'}")
        private String expression;
    }
}
```

输出结果如下

```
=======================> 仅获取 @Value 值
hello
=======================> 获取 @Value 值, 并解析${}
${jdbc.username}
root
=======================> 获取 @Value 值, 并解析#{}
#{'class version:' + '${java.class.version}'}
#{'class version:' + '61.0'}
class version:61.0

进程已结束,退出代码0
```



### 1.2 obtainFreshBeanFactory

* 这一步获取（或创建） BeanFactory，它也是作为 ApplicationContext 的一个成员变量
* BeanFactory 的作用是负责 bean 的创建、依赖注入和初始化，bean 的各项特征由 BeanDefinition 定义
  * BeanDefinition 作为 bean 的设计蓝图，规定了 bean 的特征，如单例多例、依赖关系、初始销毁方法等
  * BeanDefinition 的来源有多种多样，可以是通过 xml 获得、配置类获得、组件扫描获得，也可以是编程添加
* 所有的 BeanDefinition 会存入 BeanFactory 中的 beanDefinitionMap 集合

![image-20210902182004819](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902182004819.png)

**代码演示**

```java
// 演示各种 BeanDefinition 的来源
public class TestBeanDefinition {
    public static void main(String[] args) {
        System.out.println("========================> 一开始");
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        System.out.println(Arrays.toString(beanFactory.getBeanDefinitionNames()));

        System.out.println("========================> 1) 从 xml 获取 ");
        XmlBeanDefinitionReader reader1 = new XmlBeanDefinitionReader(beanFactory);
        reader1.loadBeanDefinitions(new ClassPathResource("bd.xml"));
        System.out.println(Arrays.toString(beanFactory.getBeanDefinitionNames()));

        System.out.println("========================> 2) 从配置类获取 ");
        beanFactory.registerBeanDefinition("config1", 
            BeanDefinitionBuilder.genericBeanDefinition(Config1.class).getBeanDefinition());

        ConfigurationClassPostProcessor postProcessor = new ConfigurationClassPostProcessor();
        postProcessor.postProcessBeanDefinitionRegistry(beanFactory);
        System.out.println(Arrays.toString(beanFactory.getBeanDefinitionNames()));

        System.out.println("========================> 3) 扫描获取 ");
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(beanFactory);
        scanner.scan("day04.refresh.sub");
        System.out.println(Arrays.toString(beanFactory.getBeanDefinitionNames()));
    }

    static class Bean1 {

    }

    static class Bean2 {

    }

    static class Config1 {
        @Bean
        public Bean2 bean2() {
            return new Bean2();
        }
    }
}
```

输出结果如下

```
========================> 一开始
[]
========================> 1) 从 xml 获取 
[bean1]
========================> 2) 从配置类获取 
[bean1, config1, bean2]
========================> 3) 扫描获取 
[bean1, config1, bean2, bean3, org.springframework.context.annotation.internalConfigurationAnnotationProcessor, org.springframework.context.annotation.internalAutowiredAnnotationProcessor, org.springframework.context.annotation.internalCommonAnnotationProcessor, org.springframework.context.event.internalEventListenerProcessor, org.springframework.context.event.internalEventListenerFactory]

进程已结束,退出代码0
```



### 1.3 prepareBeanFactory

* 这一步会进一步完善 BeanFactory，为它的各项成员变量赋值
* beanExpressionResolver 用来解析 SpEL，常见实现为 StandardBeanExpressionResolver
* propertyEditorRegistrars 会注册类型转换器，值注入时将字符串类型转化为其他类型
  * 它在这里使用了 ResourceEditorRegistrar 实现类
  * 并应用 ApplicationContext 提供的 Environment 完成 ${ } 解析
* registerResolvableDependency 来注册 beanFactory 以及 ApplicationContext，让它们也能用于依赖注入
* beanPostProcessors 是 bean 后处理器集合，会工作在 bean 的生命周期各个阶段，此处会添加两个：
  * ApplicationContextAwareProcessor 用来解析 Aware 接口
  * ApplicationListenerDetector 用来识别容器中 ApplicationListener 类型的 bean

![image-20210902182541925](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902182541925.png)



### 1.4 postProcessBeanFactory

* 这一步是空实现，留给子类扩展。
  * 一般 Web 环境的 ApplicationContext 都要利用它注册新的 Scope，完善 Web 下的 BeanFactory
* 这里体现的是模板方法设计模式

### 1.5 invokeBeanFactoryPostProcessors

* 这一步会调用 beanFactory 后处理器
* beanFactory 后处理器，充当 beanFactory 的扩展点，可以用来补充或修改 BeanDefinition
* 常见的 beanFactory 后处理器有
  * ConfigurationClassPostProcessor – 解析 @Configuration、@Bean、@Import、@PropertySource 等
  * PropertySourcesPlaceHolderConfigurer – 替换 BeanDefinition 中的 ${ }
  * MapperScannerConfigurer – 补充 Mapper 接口对应的 BeanDefinition

![image-20210902183232114](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902183232114.png)



### 1.6 registerBeanPostProcessors

* 这一步是继续从 beanFactory 中找出 bean 后处理器，添加至 beanPostProcessors 集合中
* bean 后处理器，充当 bean 的扩展点，可以工作在 bean 的实例化、依赖注入、初始化阶段，常见的有：
  * AutowiredAnnotationBeanPostProcessor 功能有：解析 @Autowired，@Value 注解
  * CommonAnnotationBeanPostProcessor 功能有：解析 @Resource，@PostConstruct，@PreDestroy
  * AnnotationAwareAspectJAutoProxyCreator 功能有：为符合切点的目标 bean 自动创建代理

![image-20210902183520307](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902183520307.png)

**代码样例**

```java
public class TestBeanPostProcessor {

    public static void main(String[] args) {
        GenericApplicationContext context = new GenericApplicationContext();
        DefaultListableBeanFactory beanFactory = context.getDefaultListableBeanFactory();
        // 基础语句
        beanFactory.registerBeanDefinition("bean1", 
                                       BeanDefinitionBuilder
                                           .genericBeanDefinition(Bean1.class)
                                           .getBeanDefinition());
        beanFactory.registerBeanDefinition("bean2", 
                                       BeanDefinitionBuilder
                                           .genericBeanDefinition(Bean2.class)
                                           .getBeanDefinition());
        beanFactory.registerBeanDefinition("bean3", 
                                       BeanDefinitionBuilder
                                           .genericBeanDefinition(Bean3.class)
                                           .getBeanDefinition());
        beanFactory.registerBeanDefinition("aspect1", 
                                       BeanDefinitionBuilder
                                           .genericBeanDefinition(Aspect1.class)
                                           .getBeanDefinition());
        // 添加用于处理@Autowired的bean后处理器
        beanFactory.registerBeanDefinition("processor1",
                BeanDefinitionBuilder
                   .genericBeanDefinition(
                       AutowiredAnnotationBeanPostProcessor.class).getBeanDefinition());
        // 添加用于处理@Resource的bean后处理器
        beanFactory.registerBeanDefinition("processor2",
                BeanDefinitionBuilder.genericBeanDefinition(
                    CommonAnnotationBeanPostProcessor.class).getBeanDefinition());
        // 添加用于处理@Aspect的bean后处理器
        beanFactory.registerBeanDefinition("processor3",       						
            BeanDefinitionBuilder.genericBeanDefinition(                                                   	 				AnnotationAwareAspectJAutoProxyCreator.class).getBeanDefinition());
        context.refresh();
        beanFactory.getBean(Bean1.class).foo();
    }

    static class Bean1 {
        Bean2 bean2;
        Bean3 bean3;

        @Autowired
        public void setBean2(Bean2 bean2) {
            System.out.println("发生了依赖注入..." + bean2);
            this.bean2 = bean2;
        }

        @Resource
        public void setBean3(Bean3 bean3) {
            System.out.println("发生了依赖注入..." + bean3);
            this.bean3 = bean3;
        }

        public void foo() {
            System.out.println("foo");
        }
    }

    static class Bean2 {

    }

    static class Bean3 {

    }

    // 切面
    @Aspect
    static class Aspect1 {
        @Before("execution(* foo())")
        public void before() {
            System.out.println("before...");
        }
    }
}
```

在仅注册bean1,bean2,bean3与aspect1，无**bean的后处理器**的情况下输出情况如下：

```
foo

进程已结束,退出代码0
```

并没有执行`@Autowired`与`@Resource`,也没有执行切面`@Aspect`，仅执行了foo方法

加上bean后处理器后，输出变为

```
发生了依赖注入...day04.refresh.TestBeanPostProcessor$Bean3@64ec96c6
发生了依赖注入...day04.refresh.TestBeanPostProcessor$Bean2@2f666ebb
before...
foo
```

与我们期待的结果一致了



### 1.7 initMessageSource

从beanFactory回到了application context，给其中一些对象赋值

* 这一步是为 ApplicationContext 添加 messageSource 成员，实现**国际化**功能
* 去 beanFactory 内找名为 messageSource 的 bean，如果没有，则提供空的 MessageSource 实现

![image-20210902183819984](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902183819984.png)



### 1.8 initApplicationContextEventMulticaster

* 这一步为 `ApplicationContext` 添加事件广播器成员，即 `applicationContextEventMulticaster`
* 它的作用是发布事件给监听器
* 去 beanFactory 找名为 `applicationEventMulticaster` 的 bean 作为事件广播器，若没有，会创建默认的事件广播器
* 之后就可以调用 `ApplicationContext.publishEvent(事件对象)` 来发布事件

![image-20210902183943469](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902183943469.png)



### 1.9 onRefresh

* 这一步是空实现，留给子类扩展
  * SpringBoot 中的子类在这里**准备了 WebServer**，即内嵌 web 容器
* 体现的是模板方法设计模式



### 1.10 registerListeners

* 这一步会从多种途径找到事件监听器，并添加至 `applicationEventMulticaster`
* 事件监听器顾名思义，用来接收事件广播器发布的事件，有如下来源
  * 事先编程添加的
  * 来自容器中的 bean
  * 来自于 `@EventListener` 的解析
* 要实现事件监听器，只需要实现 `ApplicationListener` 接口，重写其中 `onApplicationEvent(E e)` 方法即可

![image-20210902184343872](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902184343872.png)



### 1.11 finishBeanFactoryInitialization

* 这一步会将 beanFactory 的成员补充完毕，并初始化所有非延迟单例 bean
* `conversionService` 也是一套转换机制，[作为对`PropertyEditor`的补充](#1.3 prepareBeanFactory).
* `embeddedValueResolvers` 即内嵌值解析器，用来解析 @Value 中的 ${ }，借用的是 Environment 的功能
* singletonObjects 即单例池，缓存所有单例对象
  * 对象的创建都分三个阶段，每一阶段都有不同的 bean 后处理器参与进来，扩展功能

![image-20210902184641623](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902184641623.png)



### 1.12 finishRefresh

* 这一步会为 `ApplicationContext` 添加 `lifecycleProcessor` 成员，用来控制容器内需要生命周期管理的 bean
* 如果容器中有名称为 `lifecycleProcessor` 的 bean 就用它，否则创建默认的生命周期管理器
* 准备好生命周期管理器，就可以实现
  * 调用 context 的 start，即可触发所有实现 LifeCycle 接口 bean 的 start
  * 调用 context 的 stop，即可触发所有实现 LifeCycle 接口 bean 的 stop
* 发布 `ContextRefreshed` 事件，整个 refresh 执行完成

![image-20210902185052433](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210902185052433.png)



## 2. Spring bean 生命周期

**要求**

* 掌握 Spring bean 的生命周期

**bean 生命周期 概述**

bean 的生命周期从调用 beanFactory 的 getBean 开始，到这个 bean 被销毁，可以总结为以下七个阶段：

1. 处理名称，检查缓存
2. 处理父子容器
3. 处理 dependsOn
4. 选择 scope 策略
5. 创建 bean
6. 类型转换处理
7. 销毁 bean

> ***注意***
>
> * 划分的阶段和名称并不重要，重要的是理解整个过程中做了哪些事情



### 2.1 处理名称，检查缓存

```java
protected <T> T doGetBean(
        String name, 
        @Nullable Class<T> requiredType, 
        @Nullable Object[] args, 
        boolean typeCheckOnly) throws BeansException
```

* 这一步会处理别名，将别名解析为实际名称
* 对 FactoryBean 也会特殊处理，如果以 & 开头表示要获取 FactoryBean 本身，否则表示要获取其产品
* 这里针对单例对象会检查一级、二级、三级缓存
  * singletonFactories 三级缓存，存放**单例工厂**对象
  * earlySingletonObjects 二级缓存，存放**单例工厂的产品**对象
    * 如果发生循环依赖，产品是代理；无循环依赖，产品是原始对象
  * singletonObjects 一级缓存，存放**单例成品**对象



### 2.2 处理父子容器

* 如果当前容器根据名字找不到这个 bean，此时若父容器存在，则执行父容器的 getBean 流程

* 父子容器的 bean 名称可以重复

  

### 2.3 处理 dependsOn

* 如果当前 bean 有通过 dependsOn 指定了**非显式依赖的 bean**，这一步会提前创建这些 dependsOn 的 bean 
* 所谓非显式依赖，就是指两个 bean 之间不存在直接依赖关系，但需要控制它们的创建先后顺序



### 2.4 选择 scope 策略

scope理解为"从XX范围内找到bean"更贴切。

#### 2.4.0 补充知识：Spring Scope

> cope用来声明容器中的对象所应该处的限定场景或者说该对象的存活时间，即容器在对象进入其相应的scope之前生成并装配这些对象，在该对象不再处于这些scope的限定之后，容器通常会销毁这些对象。

+ **singleton**

  singleton**是容器默认的scope**，所以写和不写没有区别。scope为singleton的时候，**在Spring的IoC容器中只存在一个实例**，所有对该对象的引用将共享这个实例。**该实例从容器启动，并因为第一次被请求而初始化后，将一直存活到容器退出**，也就是说，它与IoC容器“几乎”拥有相同的寿命。

  singleton的bean所具有的属性

  - **对象实例数量**：singleton类型的bean定义，在一个容器中只存在一个共享实例，**所有对该类型bean的依赖都引用这一单一实例**
  - **对象存活时间**：singleton类型bean定义，从容器启动，到它第一次被请求而实例化开始，只要容器不销毁或退出，该类型的单一实例就会**一直存活**

+ **prototype**

  对于那些请求方**不能共享的对象实例**，应该将其bean定义的scope设置为prototype。这样，每个请求方可以得到**自己对应的一个对象实例**。通常，声明为prototype的scope的bean定义类型，都是一些有状态的，比如保存每个顾客信息的对象.

+ **request、session、global session**

  这三个scope类型是Spring2.0之后新增加的，它们不像上面两个那么通用，**它们只适用于Web应用程序**，通常是与`XmlWebApplicationContext`共同使用(注意：只能使用scope属性才能使用这三种，也就是必须使用XSD文档声明的XML配置文件格式)

  + request

    在Spring容器中，即`XmlWebApplicationContext`会为每**个HTTP请求创建一个全新的Request-Processor对象**供当前请求使用，**当请求结束后，该对象实生命周期就结束**。当同时有10个HTTP请求进来的时候，容器会分别针对这10个请求返回10个全新的RequestProcessor对象实例，且它们之间互不干扰。所以严格意义上说request可以看作prototype的一种特例，除了request的场景更加具体.

  + session

    放到session中的最普遍的信息就是用户登录信息，Spring容器会为每个独立的session创建属于它们自己全新的UserPreferences对象实例。与request相比，除了拥有session scope的bean比request scope的bean可能更长的存活时间，其他没什么差别.

  + global session

    global session只有应用在基于portlet的Web应用程序中才有意义，它映射到portlet的global范围的session。如果在普通的基于servlet的Web应用中使了用这个类型的scope，容器会将其作为普通的session类型的scope来对待

#### 2.4.1 一个案例与三种策略

我们这里有一个bean。这个bean在被初始化和销毁前会打印输出，我们来看一下下列三种scope的情况。

```java
static class Bean1 {
    @PostConstruct
    public void init() {
        LoggerUtils.get().debug("{} - init", this);
    }

    @PreDestroy
    public void destroy() {
        LoggerUtils.get().debug("{} - destroy", this);
    }
}
```

* **对于 singleton scope**，首先到单例池去获取 bean，如果有则直接返回，没有再进入创建流程

  ```java
  // 单例 bean 从 refresh 被创建, 到 close 被销毁, BeanFactory 会记录哪些 bean 要调用销毁方法
  private static void testSingletonScope() {
      GenericApplicationContext context = new GenericApplicationContext();
      context.registerBean("bean1", Bean1.class);
      context.registerBean(CommonAnnotationBeanPostProcessor.class);
      context.refresh(); // 直接调用getBean方法被初始化
      context.close(); // 调用销毁方法进行销毁
  }
  ```

  输出如下

  ```
  [DEBUG] 20:41:22.022 [main] - day04.bean.TestScope$Bean1@7eac9008 - init 
  [DEBUG] 20:41:22.045 [main] - day04.bean.TestScope$Bean1@7eac9008 - destroy 
  ```

  可以看到初始化和销毁方法都被正确地执行了

* **对于 prototype scope**，每次都会进入创建流程

  ```java
  // 多例 bean 从首次 getBean 被创建, 到调用 BeanFactory 的 destroyBean 被销毁
  private static void testPrototypeScope() {
      GenericApplicationContext context = new GenericApplicationContext();
      // 注册时需要声明bean1不是单例而是多例
      context.registerBean("bean1", Bean1.class, bd -> bd.setScope("prototype"));
      context.registerBean(CommonAnnotationBeanPostProcessor.class);
      context.refresh(); // 未初始化
      
  	// 此时(通过getBean使用时)才初始化
      Bean1 bean = context.getBean(Bean1.class);
      // 没谁记录该 bean 要调用销毁方法, 需要我们自行调用
      context.getDefaultListableBeanFactory().destroyBean(bean);
  
      context.close();
  }
  ```

  注意懒加载机制 输出如下

  ```
  [DEBUG] 20:49:54.410 [main] - day04.bean.TestScope$Bean1@6ab778a - init 
  [DEBUG] 20:49:54.414 [main] - day04.bean.TestScope$Bean1@6ab778a - destroy 
  ```

* **对于自定义 scope**，例如 request，首先到 request 域获取 bean，如果有则直接返回，没有再进入创建流程

  ```java
  // request bean 从首次 getBean 被创建, 到 request 结束前被销毁
  private static void testRequestScope() {
      GenericApplicationContext context = new GenericApplicationContext();
      // 注册一个RequestScope
      context.getDefaultListableBeanFactory().registerScope("request", new RequestScope());
      // 注册bean 声明其scope为RequestScope
      context.registerBean("bean1", Bean1.class, bd -> bd.setScope("request"));
      context.registerBean(CommonAnnotationBeanPostProcessor.class);
      context.refresh();
  
      for (int i = 0; i < 2; i++) {
          new Thread(() -> {
              MockHttpServletRequest request = new MockHttpServletRequest();
              // 每个 webRequest 对象会记录哪些 bean 要调用销毁方法
              ServletWebRequest webRequest = new ServletWebRequest(request);
              RequestContextHolder.setRequestAttributes(webRequest);
  
              // bean被存入request的作用域 对同一个request多次执行获取bean操作获取的是同一个bean
              Bean1 bean = context.getBean(Bean1.class);
              LoggerUtils.get().debug("{}", bean);
              LoggerUtils.get().debug("{}", request.getAttribute("bean1"));
  
              // request 请求结束前调用这些销毁方法
              webRequest.requestCompleted();
          }).start();
      }
  }
  ```

  通过多个线程进行调用 输出如下

  ```
  [DEBUG] 21:06:10.985 [Thread-0] - day04.bean.TestScope$Bean1@1f669b69 - init 
  [DEBUG] 21:06:10.985 [Thread-1] - day04.bean.TestScope$Bean1@3ed2f85a - init 
  [DEBUG] 21:06:10.989 [Thread-0] - day04.bean.TestScope$Bean1@1f669b69 
  [DEBUG] 21:06:10.989 [Thread-1] - day04.bean.TestScope$Bean1@3ed2f85a 
  [DEBUG] 21:06:10.990 [Thread-0] - day04.bean.TestScope$Bean1@1f669b69 
  [DEBUG] 21:06:10.990 [Thread-1] - day04.bean.TestScope$Bean1@3ed2f85a 
  [DEBUG] 21:06:10.990 [Thread-0] - day04.bean.TestScope$Bean1@1f669b69 - destroy 
  [DEBUG] 21:06:10.990 [Thread-1] - day04.bean.TestScope$Bean1@3ed2f85a - destroy 
  
  进程已结束,退出代码0
  ```

  可见在每一个线程(request)中的bean是一致的



### 2.5 创建bean

总体流程

![image-20230328213702796](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230328213702796.png)

#### 2.5.1 创建 bean 实例

| **要点**                             | **总结**                                                     |
| ------------------------------------ | ------------------------------------------------------------ |
| 有自定义 TargetSource 的情况         | 由 AnnotationAwareAspectJAutoProxyCreator 创建代理返回       |
| Supplier 方式创建 bean 实例          | 为 Spring 5.0 新增功能，方便编程方式创建  bean  实例         |
| FactoryMethod 方式  创建 bean  实例  | ① 分成静态工厂与实例工厂；② 工厂方法若有参数，需要对工厂方法参数进行解析，利用  resolveDependency；③ 如果有多个工厂方法候选者，还要进一步按权重筛选 |
| AutowiredAnnotationBeanPostProcessor | ① 优先选择带  @Autowired  注解的构造；② 若有唯一的带参构造，也会入选 |
| mbd.getPreferredConstructors         | 选择所有公共构造，这些构造之间按权重筛选                     |
| 采用默认构造                         | 如果上面的后处理器和 BeanDefiniation 都没找到构造，采用默认构造，即使是私有的 |

#### 2.5.2 依赖注入

| **要点**                             | **总结**                                                     |
| ------------------------------------ | ------------------------------------------------------------ |
| AutowiredAnnotationBeanPostProcessor | 识别   @Autowired  及 @Value  标注的成员，封装为  InjectionMetadata 进行依赖注入 |
| CommonAnnotationBeanPostProcessor    | 识别   @Resource  标注的成员，封装为  InjectionMetadata 进行依赖注入 |
| resolveDependency                    | 用来查找要装配的值，可以识别：① Optional；② ObjectFactory 及 ObjectProvider；③ @Lazy  注解；④ @Value  注解（${  }, #{ }, 类型转换）；⑤ 集合类型（Collection，Map，数组等）；⑥ 泛型和  @Qualifier（用来区分类型歧义）；⑦ primary  及名字匹配（用来区分类型歧义） |
| AUTOWIRE_BY_NAME                     | 根据成员名字找 bean 对象，修改 mbd 的 propertyValues，不会考虑简单类型的成员 |
| AUTOWIRE_BY_TYPE                     | 根据成员类型执行 resolveDependency 找到依赖注入的值，修改  mbd 的 propertyValues |
| applyPropertyValues                  | 根据 mbd 的 propertyValues 进行依赖注入（即xml中 `<property name ref|value/>`） |

有测试代码如下所示

```java
// 测试如果对同一属性进行的 @Autowired 注入、AUTOWIRE_BY_NAME、精确指定注入名称, 优先级是怎样的
public class TestInjection {
    public static void main(String[] args) {
        GenericApplicationContext context = new GenericApplicationContext();
        AnnotationConfigUtils.registerAnnotationConfigProcessors(context.getDefaultListableBeanFactory());
        context.registerBean("bean1", Bean1.class, bd -> {
            // 优先级最高的：精确指定注入 bean 的名称 <property name="bean3" ref="bean2"/>
            bd.getPropertyValues().add("bean3", new RuntimeBeanReference("bean2"));
            // 优先级次之的：通过 AUTOWIRE_BY_NAME 匹配
            ((RootBeanDefinition) bd).setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_NAME);
        });
        context.registerBean("bean2", Bean2.class);
        context.registerBean("bean3", Bean3.class);
        context.registerBean("bean4", Bean4.class);

        context.refresh();
    }

    static class Bean1 {
        MyInterface bean;

        // 有三个bean实现了MyInterface 因此需要使用Qualifier指定注入对象
        // 优先级最低的：@Autowired 匹配
        @Autowired @Qualifier("bean4")
        public void setBean3(MyInterface bean) {
            System.out.println(bean);
            this.bean = bean;
        }
    }

    interface MyInterface {
    }

    static class Bean2 implements MyInterface {
    }

    static class Bean3 implements MyInterface {
    }

    static class Bean4 implements MyInterface {
    }
}
```

若注释掉

```java
// 优先级最高的：精确指定注入 bean 的名称 <property name="bean3" ref="bean2"/>
bd.getPropertyValues().add("bean3", new RuntimeBeanReference("bean2"));
// 优先级次之的：通过 AUTOWIRE_BY_NAME 匹配
((RootBeanDefinition) bd).setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_NAME);
```

则使用默认优先级，此时输出

```
day04.bean.TestInjection$Bean4@28f2a10f

进程已结束,退出代码0
```

可见自动装配Bean4，遵循`@Qualifier("bean4")`指定

放开`AUTOWIRE_BY_NAME`，则根据名称进行匹配。由于方法名含有Bean3，因此bean3被装配，此时输出

```java
day04.bean.TestInjection$Bean3@5acf93bb

进程已结束,退出代码0
```

可见装配Bean3

放开全部注释，在注入bean时指定优先级，则输出

```
day04.bean.TestInjection$Bean2@6e0f5f7f

进程已结束,退出代码0
```

满足我们指定的优先级标准

#### 2.5.3 初始化

| **要点**              | **总结**                                                     |
| --------------------- | ------------------------------------------------------------ |
| 内置 Aware 接口的装配 | 包括 BeanNameAware，BeanFactoryAware 等                      |
| 扩展 Aware 接口的装配 | 由 ApplicationContextAwareProcessor 解析，执行时机在  postProcessBeforeInitialization |
| @PostConstruct        | 由 CommonAnnotationBeanPostProcessor 解析，执行时机在  postProcessBeforeInitialization |
| InitializingBean      | 通过接口回调执行初始化                                       |
| initMethod            | 根据 BeanDefinition 得到的初始化方法执行初始化，即 `<bean init-method>` 或 @Bean(initMethod) |
| 创建 aop 代理         | 由 AnnotationAwareAspectJAutoProxyCreator 创建，执行时机在  postProcessAfterInitialization |

样例代码如下

```java
public class TestInitialization {

    public static void main(String[] args) {
        GenericApplicationContext context = new GenericApplicationContext();
        context.registerBean(CommonAnnotationBeanPostProcessor.class);
        // <bean init-method="initMethod">
        context.registerBean("bean1", Bean1.class, bd -> bd.setInitMethodName("initMethod"));
        context.refresh();
    }

    static class Bean1 implements InitializingBean, BeanFactoryAware {

        @Override
        public void afterPropertiesSet() throws Exception {
            System.out.println(1);
        }

        @PostConstruct
        public void init() {
            System.out.println(2);java
        }

        public void initMethod() {
            System.out.println(3);
        }

        @Override
        public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
            System.out.println(4);
        }
    }
}
```

输出情况如下

```java
4
2
1
3

进程已结束,退出代码0
```

可见各方法执行顺序为`setBeanFactory` -> `@PostConstruct` -> `afterPropertiesSet()` -> `initMethod()`

先执行`BeanFactoryAware`再执行注解初始化，接口初始化，最后执行自定义方法

#### 2.5.4 注册可销毁 bean

在这一步判断并登记可销毁 bean

* 判断依据
  * 如果实现了 DisposableBean 或 AutoCloseable 接口，则为可销毁 bean
  * 如果自定义了 destroyMethod，则为可销毁 bean
  * 如果采用 @Bean 没有指定 destroyMethod，则采用自动推断方式获取销毁方法名（close，shutdown）
  * 如果有 @PreDestroy 标注的方法
* 存储位置
  * singleton scope 的可销毁 bean 会存储于 beanFactory 的成员当中
  * 自定义 scope 的可销毁 bean 会存储于对应的域对象当中
  * prototype scope 不会存储，需要自己找到此对象销毁
* 存储时都会封装为 DisposableBeanAdapter 类型对销毁方法的调用进行适配 -> 适配器模式



### 2.6 类型转换处理

* 如果 getBean 的 requiredType 参数与实际得到的对象类型不同，会尝试进行类型转换



### 2.7 销毁 bean

* 销毁时机
  * singleton bean 的销毁在 ApplicationContext.close 时，此时会找到所有 DisposableBean 的名字，逐一销毁
  * 自定义 scope bean 的销毁在作用域对象生命周期结束时
  * prototype bean 的销毁可以通过自己手动调用 AutowireCapableBeanFactory.destroyBean 方法执行销毁
* 同一 bean 中不同形式销毁方法的调用次序
  * 优先后处理器销毁，即 @PreDestroy
  * 其次 DisposableBean 接口销毁
  * 最后 destroyMethod 销毁（包括自定义名称，推断名称，AutoCloseable 接口 多选一）



## 3. Spring bean 循环依赖

**要求**

* 掌握单例 set 方式循环依赖的原理
* 掌握其它循环依赖的解决方法

**循环依赖的产生**

* 首先要明白，bean 的创建要遵循一定的步骤，必须是创建、注入、初始化三步，这些顺序不能乱

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903085238916.png" alt="image-20210903085238916" style="zoom:50%;" />

* set 方法（包括成员变量）的循环依赖如图所示

  * 可以在【a 创建】和【a set 注入 b】之间加入 b 的整个流程来解决
  * 【b set 注入 a】 时可以成功，因为之前 a 的实例已经创建完毕

  * a 的顺序，及 b 的顺序都能得到保障

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903085454603.png" alt="image-20210903085454603" style="zoom: 33%;" />

* 构造方法的循环依赖如图所示，显然无法用前面的方法解决

<img src="D:\0_大学\2023.2\Java自学\Java面试专题-资料\day04-框架篇\讲义\img\image-20210903085906315.png" alt="image-20210903085906315" style="zoom: 50%;" />

**构造循环依赖的解决**

* 思路1
  * a 注入 b 的代理对象，这样能够保证 a 的流程走通
  * 后续需要用到 b 的真实对象时，可以通过代理间接访问

<img src="D:\0_大学\2023.2\Java自学\Java面试专题-资料\day04-框架篇\讲义\img\image-20210903091627659.png" alt="image-20210903091627659" style="zoom: 50%;" />

* 思路2
  * a 注入 b 的工厂对象，让 b 的实例创建被推迟，这样能够保证 a 的流程先走通
  * 后续需要用到 b 的真实对象时，再通过 ObjectFactory 工厂间接访问

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903091743366.png" alt="image-20210903091743366" style="zoom:50%;" />

* 示例1：用 @Lazy 为构造方法参数生成代理

```java
public class App60_1 {

    static class A {
        private static final Logger log = LoggerFactory.getLogger("A");
        private B b;

        public A(@Lazy B b) {
            log.debug("A(B b) {}", b.getClass());
            this.b = b;
        }

        @PostConstruct
        public void init() {
            log.debug("init()");
        }
    }

    static class B {
        private static final Logger log = LoggerFactory.getLogger("B");
        private A a;

        public B(A a) {
            log.debug("B({})", a);
            this.a = a;
        }

        @PostConstruct
        public void init() {
            log.debug("init()");
        }
    }

    public static void main(String[] args) {
        GenericApplicationContext context = new GenericApplicationContext();
        context.registerBean("a", A.class);
        context.registerBean("b", B.class);
        AnnotationConfigUtils.registerAnnotationConfigProcessors(context.getDefaultListableBeanFactory());
        context.refresh();
        System.out.println();
    }
}
```

* 示例2：用 ObjectProvider 延迟依赖对象的创建

```java
public class App60_2 {

    static class A {
        private static final Logger log = LoggerFactory.getLogger("A");
        private ObjectProvider<B> b;

        public A(ObjectProvider<B> b) {
            log.debug("A({})", b);
            this.b = b;
        }

        @PostConstruct
        public void init() {
            log.debug("init()");
        }
    }

    static class B {
        private static final Logger log = LoggerFactory.getLogger("B");
        private A a;

        public B(A a) {
            log.debug("B({})", a);
            this.a = a;
        }

        @PostConstruct
        public void init() {
            log.debug("init()");
        }
    }

    public static void main(String[] args) {
        GenericApplicationContext context = new GenericApplicationContext();
        context.registerBean("a", A.class);
        context.registerBean("b", B.class);
        AnnotationConfigUtils.registerAnnotationConfigProcessors(context.getDefaultListableBeanFactory());
        context.refresh();

        System.out.println(context.getBean(A.class).b.getObject());
        System.out.println(context.getBean(B.class));
    }
}
```

* 示例3：用 @Scope 产生代理

```java
public class App60_3 {

    public static void main(String[] args) {
        GenericApplicationContext context = new GenericApplicationContext();
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(context.getDefaultListableBeanFactory());
        scanner.scan("com.itheima.app60.sub");
        context.refresh();
        System.out.println();
    }
}
```



```java
@Component
class A {
    private static final Logger log = LoggerFactory.getLogger("A");
    private B b;

    public A(B b) {
        log.debug("A(B b) {}", b.getClass());
        this.b = b;
    }

    @PostConstruct
    public void init() {
        log.debug("init()");
    }
}
```



```java
@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component
class B {
    private static final Logger log = LoggerFactory.getLogger("B");
    private A a;

    public B(A a) {
        log.debug("B({})", a);
        this.a = a;
    }

    @PostConstruct
    public void init() {
        log.debug("init()");
    }
}
```



* 示例4：用 Provider 接口解决，原理上与 ObjectProvider 一样，Provider 接口是独立的 jar 包，需要加入依赖

```xml
<dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
    <version>1</version>
</dependency>
```



```java
public class App60_4 {

    static class A {
        private static final Logger log = LoggerFactory.getLogger("A");
        private Provider<B> b;

        public A(Provider<B> b) {
            log.debug("A({}})", b);
            this.b = b;
        }

        @PostConstruct
        public void init() {
            log.debug("init()");
        }
    }

    static class B {
        private static final Logger log = LoggerFactory.getLogger("B");
        private A a;

        public B(A a) {
            log.debug("B({}})", a);
            this.a = a;
        }

        @PostConstruct
        public void init() {
            log.debug("init()");
        }
    }

    public static void main(String[] args) {
        GenericApplicationContext context = new GenericApplicationContext();
        context.registerBean("a", A.class);
        context.registerBean("b", B.class);
        AnnotationConfigUtils.registerAnnotationConfigProcessors(context.getDefaultListableBeanFactory());
        context.refresh();

        System.out.println(context.getBean(A.class).b.get());
        System.out.println(context.getBean(B.class));
    }
}
```



### 解决 set 循环依赖的原理

**一级缓存**

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903100752165.png" alt="image-20210903100752165" style="zoom:80%;" />

作用是保证单例对象仅被创建一次

* 第一次走 `getBean("a")` 流程后，最后会将成品 a 放入 singletonObjects 一级缓存
* 后续再走 `getBean("a")` 流程时，先从一级缓存中找，这时已经有成品 a，就无需再次创建

**一级缓存与循环依赖**

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903100914140.png" alt="image-20210903100914140" style="zoom:80%;" />

一级缓存无法解决循环依赖问题，分析如下

* 无论是获取 bean a 还是获取 bean b，走的方法都是同一个 getBean 方法，假设先走 `getBean("a")`
* 当 a 的实例对象创建，接下来执行 `a.setB()` 时，需要走 `getBean("b")` 流程，红色箭头 1
* 当 b 的实例对象创建，接下来执行 `b.setA()` 时，又回到了 `getBean("a")` 的流程，红色箭头 2
* 但此时 singletonObjects 一级缓存内没有成品的 a，陷入了死循环

**二级缓存**

<img src="D:\0_大学\2023.2\Java自学\Java面试专题-资料\day04-框架篇\讲义\img\image-20210903101849924.png" alt="image-20210903101849924" style="zoom:80%;" />

解决思路如下：

* 再增加一个 singletonFactories 缓存
* 在依赖注入前，即 `a.setB()` 以及 `b.setA()` 将 a 及 b 的半成品对象（未完成依赖注入和初始化）放入此缓存
* 执行依赖注入时，先看看 singletonFactories 缓存中是否有半成品的对象，如果有拿来注入，顺利走完流程

对于上面的图

* `a = new A()` 执行之后就会把这个半成品的 a 放入 singletonFactories 缓存，即 `factories.put(a)`
* 接下来执行 `a.setB()`，走入 `getBean("b")` 流程，红色箭头 3
* 这回再执行到 `b.setA()` 时，需要一个 a 对象，有没有呢？有！
* `factories.get()` 在 singletonFactories  缓存中就可以找到，红色箭头 4 和 5
* b 的流程能够顺利走完，将 b 成品放入 singletonObject 一级缓存，返回到 a 的依赖注入流程，红色箭头 6

**二级缓存与创建代理**

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903103030877.png" alt="image-20210903103030877" style="zoom:80%;" />

二级缓存无法正确处理循环依赖并且包含有代理创建的场景，分析如下

* spring 默认要求，在 `a.init` 完成之后才能创建代理 `pa = proxy(a)`
* 由于 a 的代理创建时机靠后，在执行 `factories.put(a)` 向 singletonFactories 中放入的还是原始对象
* 接下来箭头 3、4、5 这几步 b 对象拿到和注入的都是原始对象

**三级缓存**

![image-20210903103628639](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903103628639.png)

简单分析的话，只需要将代理的创建时机放在依赖注入之前即可，但 spring 仍然希望代理的创建时机在 init 之后，只有出现循环依赖时，才会将代理的创建时机提前。所以解决思路稍显复杂：

* 图中 `factories.put(fa)` 放入的既不是原始对象，也不是代理对象而是工厂对象 fa
* 当检查出发生循环依赖时，fa 的产品就是代理 pa，没有发生循环依赖，fa 的产品是原始对象 a
* 假设出现了循环依赖，拿到了 singletonFactories 中的工厂对象，通过在依赖注入前获得了 pa，红色箭头 5
* 这回 `b.setA()` 注入的就是代理对象，保证了正确性，红色箭头 7
* 还需要把 pa 存入新加的 earlySingletonObjects 缓存，红色箭头 6
* `a.init` 完成后，无需二次创建代理，从哪儿找到 pa 呢？earlySingletonObjects 已经缓存，蓝色箭头 9

当成品对象产生，放入 singletonObject 后，singletonFactories 和 earlySingletonObjects 就中的对象就没有用处，清除即可



## 4. Spring 事务失效

**要求**

* 掌握事务失效的八种场景

**基础类**

总体配置类

```java
@Configuration
@PropertySource("classpath:jdbc.properties")
@EnableTransactionManagement
@EnableAspectJAutoProxy(exposeProxy = true)
@ComponentScan("day04.tx.app.service")
@MapperScan("day04.tx.app.mapper") // 配合mybatis
public class AppConfig {

    // 配置数据源
    @ConfigurationProperties("jdbc")
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }

    @Bean
    public DataSourceInitializer dataSourceInitializer(DataSource dataSource, DatabasePopulator populator) {
        DataSourceInitializer dataSourceInitializer = new DataSourceInitializer();
        dataSourceInitializer.setDataSource(dataSource);
        dataSourceInitializer.setDatabasePopulator(populator);
        return dataSourceInitializer;
    }

    // 初始化器 要求每次执行时都调用account.sql
    @Bean
    public DatabasePopulator databasePopulator() {
        return new ResourceDatabasePopulator(new ClassPathResource("account.sql"));
    }

    @Bean // 配合mybatis
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(dataSource);
        return factoryBean;
    }

    @Bean // 启用事务
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean
    // 只要 beanFactory 的 allowBeanDefinitionOverriding==true, 即使系统的 @Bean 定义没
    // @ConditionalOnMissingBean 条件，也会被我们的同名 @Bean 覆盖掉
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource(false);
    }

}
```



### 4.1 抛出检查异常导致事务不能正确回滚

```java
public class TestService1 {
    public static void main(String[] args) throws FileNotFoundException {
        GenericApplicationContext context = new GenericApplicationContext();
        AnnotationConfigUtils.registerAnnotationConfigProcessors(context.getDefaultListableBeanFactory());
        ConfigurationPropertiesBindingPostProcessor.register(context.getDefaultListableBeanFactory());
        context.registerBean(AppConfig.class);
        context.refresh();

        Service1 bean = context.getBean(Service1.class);
        bean.transfer(1, 2, 500);
    }
}
```

被调用的`transfer()`方法如下

```java
@Service
public class Service1 {

    @Autowired
    private AccountMapper accountMapper;

    @Transactional
    public void transfer(int from, int to, int amount) throws FileNotFoundException {
        int fromBalance = accountMapper.findBalanceBy(from);
        if (fromBalance - amount >= 0) {
            accountMapper.update(from, -1 * amount);
            new FileInputStream("aaa"); // 构造FileNotFoundException
            accountMapper.update(to, amount);
        }
    }
}
```

在上述情况下出现FileNotFoundException事务不回滚

输出如下

```java
[INFO] 16:44:19.787 [main] - HikariPool-1 - Starting... 
[INFO] 16:44:20.018 [main] - HikariPool-1 - Start completed. 
[DEBUG] 16:44:20.106 [main] - Creating new transaction with name [day04.tx.app.service.Service1.transfer]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,-java.lang.Exception 
[DEBUG] 16:44:20.107 [main] - Acquired Connection [HikariProxyConnection@1938259481 wrapping com.mysql.cj.jdbc.ConnectionImpl@7b208b45] for JDBC transaction 
[DEBUG] 16:44:20.108 [main] - Switching JDBC Connection [HikariProxyConnection@1938259481 wrapping com.mysql.cj.jdbc.ConnectionImpl@7b208b45] to manual commit 
[DEBUG] 16:44:20.133 [main] - ==>  Preparing: select balance from account where accountNo=? for update 
[DEBUG] 16:44:20.159 [main] - ==> Parameters: 1(Integer) 
[DEBUG] 16:44:20.210 [main] - <==      Total: 1 
[DEBUG] 16:44:20.215 [main] - ==>  Preparing: update account set balance=balance+? where accountNo=? 
[DEBUG] 16:44:20.216 [main] - ==> Parameters: -500(Integer), 1(Integer) 
[DEBUG] 16:44:20.222 [main] - <==    Updates: 1 
[DEBUG] 16:44:20.223 [main] - Initiating transaction rollback 
[DEBUG] 16:44:20.227 [main] - Releasing JDBC Connection [HikariProxyConnection@1938259481 wrapping com.mysql.cj.jdbc.ConnectionImpl@7b208b45] after transaction 
Exception in thread "main" java.io.FileNotFoundException: aaa (系统找不到指定的文件。)
	at java.base/java.io.FileInputStream.open0(Native Method)
	...

进程已结束,退出代码1
```

未发生回滚，Account中显示的情况如下

![image-20230329164601098](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230329164601098.png)

正常发生回滚时日志应输入如下内容

```
[DEBUG] 16:44:20.224 [main] - Rolling back JDBC transaction on Connection [HikariProxyConnection@1938259481 wrapping com.mysql.cj.jdbc.ConnectionImpl@7b208b45] 
```

包含`Rolling back JDBC transaction`

* 原因：Spring 默认只会回滚非检查异常(即RuntimeException与Error，其他Exception不回滚)

* 解法：配置 rollbackFor 属性
  * ```java
    @Transactional(rollbackFor = Exception.class)
    ```




### 4.2 业务方法内自己 try-catch 异常导致事务不能正确回滚

```java
@Service
public class Service2 {

    @Autowired
    private AccountMapper accountMapper;

    @Transactional(rollbackFor = Exception.class)
    public void transfer(int from, int to, int amount)  {
        try {
            int fromBalance = accountMapper.findBalanceBy(from);
            if (fromBalance - amount >= 0) {
                accountMapper.update(from, -1 * amount);
                new FileInputStream("aaa");
                accountMapper.update(to, amount);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

输出如下

```java
[INFO] 16:50:32.730 [main] - HikariPool-1 - Starting... 
[INFO] 16:50:32.912 [main] - HikariPool-1 - Start completed. 
[DEBUG] 16:50:32.991 [main] - Creating new transaction with name [day04.tx.app.service.Service2.transfer]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,-java.lang.Exception 
[DEBUG] 16:50:32.991 [main] - Acquired Connection [HikariProxyConnection@1938259481 wrapping com.mysql.cj.jdbc.ConnectionImpl@7b208b45] for JDBC transaction 
[DEBUG] 16:50:32.993 [main] - Switching JDBC Connection [HikariProxyConnection@1938259481 wrapping com.mysql.cj.jdbc.ConnectionImpl@7b208b45] to manual commit 
[DEBUG] 16:50:33.017 [main] - ==>  Preparing: select balance from account where accountNo=? for update 
[DEBUG] 16:50:33.067 [main] - ==> Parameters: 1(Integer) 
[DEBUG] 16:50:33.120 [main] - <==      Total: 1 
[DEBUG] 16:50:33.129 [main] - ==>  Preparing: update account set balance=balance+? where accountNo=? 
[DEBUG] 16:50:33.129 [main] - ==> Parameters: -500(Integer), 1(Integer) 
[DEBUG] 16:50:33.133 [main] - <==    Updates: 1 
[DEBUG] 16:50:33.136 [main] - Initiating transaction commit 

直接提交并释放连接 未回滚
[DEBUG] 16:50:33.136 [main] - Committing JDBC transaction on Connection [HikariProxyConnection@1938259481 wrapping com.mysql.cj.jdbc.ConnectionImpl@7b208b45] 
[DEBUG] 16:50:33.140 [main] - Releasing JDBC Connection [HikariProxyConnection@1938259481 wrapping com.mysql.cj.jdbc.ConnectionImpl@7b208b45] after transaction 
java.io.FileNotFoundException: aaa (系统找不到指定的文件。)
```

我们看到在发生异常之前的第一次update操作已经被`Committing JDBC transaction on Connection`，日志中并没有回滚操作

* 原因：**事务通知只有捉到了目标抛出的异常，才能进行后续的回滚处理**，如果目标自己处理掉异常，事务通知无法知悉

* 解法1：异常**原样抛出**
  * 在 catch 块添加 `throw new RuntimeException(e);`

* 解法2：手动设置 TransactionStatus.setRollbackOnly()
  * 在 catch 块添加 `TransactionInterceptor.currentTransactionStatus().setRollbackOnly();`



### 4.3 aop 切面顺序导致导致事务不能正确回滚

```java
@Service
public class Service3 {

    @Autowired
    private AccountMapper accountMapper;

    @Transactional(rollbackFor = Exception.class)
    public void transfer(int from, int to, int amount) throws FileNotFoundException {
        int fromBalance = accountMapper.findBalanceBy(from);
        if (fromBalance - amount >= 0) {
            accountMapper.update(from, -1 * amount);
            new FileInputStream("aaa");
            accountMapper.update(to, amount);
        }
    }
}
```

调用方法中有一切面，如下

```java
@Aspect
public class MyAspect {
    @Around("execution(* transfer(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        LoggerUtils.get().debug("log:{}", pjp.getTarget());
        try {
            return pjp.proceed();
        } catch (Throwable e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

此时输出如下

```java
[INFO] 17:06:30.668 [main] - HikariPool-1 - Starting... 
[INFO] 17:06:30.850 [main] - HikariPool-1 - Start completed. 

先进入了事务切面
[DEBUG] 17:06:30.914 [main] - Creating new transaction with name [day04.tx.app.service.Service3.transfer]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,-java.lang.Exception 
[DEBUG] 17:06:30.914 [main] - Acquired Connection [HikariProxyConnection@2036704540 wrapping com.mysql.cj.jdbc.ConnectionImpl@3eee3e2b] for JDBC transaction 
[DEBUG] 17:06:30.915 [main] - Switching JDBC Connection [HikariProxyConnection@2036704540 wrapping com.mysql.cj.jdbc.ConnectionImpl@3eee3e2b] to manual commit 

然后再进入了自定义切面
[DEBUG] 17:06:30.917 [main] - log:day04.tx.app.service.Service3@cc239ba 

最里层: 目标方法
[DEBUG] 17:06:30.936 [main] - ==>  Preparing: select balance from account where accountNo=? for update 
[DEBUG] 17:06:30.963 [main] - ==> Parameters: 1(Integer) 
[DEBUG] 17:06:31.006 [main] - <==      Total: 1 
[DEBUG] 17:06:31.011 [main] - ==>  Preparing: update account set balance=balance+? where accountNo=? 
[DEBUG] 17:06:31.012 [main] - ==> Parameters: -500(Integer), 1(Integer) 
[DEBUG] 17:06:31.013 [main] - <==    Updates: 1 
[DEBUG] 17:06:31.016 [main] - Initiating transaction commit 

直接提交 未发生回滚
[DEBUG] 17:06:31.017 [main] - Committing JDBC transaction on Connection [HikariProxyConnection@2036704540 wrapping com.mysql.cj.jdbc.ConnectionImpl@3eee3e2b] 
java.io.FileNotFoundException: aaa (系统找不到指定的文件。)
```

由于AOP切面中抓住了异常并进行了如下操作

```java
e.printStackTrace();
return null;
```

因此外层事务切面不能捕获到异常，也就无法发生回滚。

* 原因：事务切面优先级最低，但如果自定义的切面优先级和他一样，则还是自定义切面在内层，这时若自定义切面没有正确抛出异常…

* 解法1、2：同情况2 中的解法:1、2

  ```java
  TransactionInterceptor.currentTransactionStatus().setRollbackOnly();
  ```

  上述catch块中的代码将在ThreadLocal中通知发生异常，因此最外层的事务切面也能正常获得异常信息。

* 解法3：调整切面顺序，在 MyAspect 上添加 `@Order(Ordered.LOWEST_PRECEDENCE - 1)` （不推荐）

  配置类中的

  ```java
  @EnableTransactionManagement
  ```

  决定了事务的优先级，源码如下

  ```java
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Import({TransactionManagementConfigurationSelector.class})
  public @interface EnableTransactionManagement {
      boolean proxyTargetClass() default false;
  
      AdviceMode mode() default AdviceMode.PROXY;
  
      int order() default Integer.MAX_VALUE;
  }
  ```

  由于为反编译版本，此处的`Integer.MAX_VALUE`实际上是常量`Ordered.LOWEST_PRECEDENCE`,即最低优先级(数字越大优先级越低)



### 4.4 非 public 方法导致的事务失效

```java
@Service
public class Service4 {

    @Autowired
    private AccountMapper accountMapper;

    @Transactional
    void transfer(int from, int to, int amount) throws FileNotFoundException {
        int fromBalance = accountMapper.findBalanceBy(from);
        if (fromBalance - amount >= 0) {
            accountMapper.update(from, -1 * amount);
            accountMapper.update(to, amount);
        }
    }
}
```

日志输出如下

```java
[INFO] 21:22:48.120 [main] - HikariPool-1 - Starting... 
[INFO] 21:22:48.334 [main] - HikariPool-1 - Start completed. 
[DEBUG] 21:22:48.417 [main] - ==>  Preparing: select balance from account where accountNo=? for update 
[DEBUG] 21:22:48.442 [main] - ==> Parameters: 1(Integer) 
[DEBUG] 21:22:48.483 [main] - <==      Total: 1 
[DEBUG] 21:22:48.489 [main] - ==>  Preparing: update account set balance=balance+? where accountNo=? 
[DEBUG] 21:22:48.489 [main] - ==> Parameters: -500(Integer), 1(Integer) 
[DEBUG] 21:22:48.506 [main] - <==    Updates: 1 
[DEBUG] 21:22:48.507 [main] - ==>  Preparing: update account set balance=balance+? where accountNo=? 
[DEBUG] 21:22:48.507 [main] - ==> Parameters: 500(Integer), 2(Integer) 
[DEBUG] 21:22:48.510 [main] - <==    Updates: 1 
```

没有有关事务的输出

* 原因：Spring 为方法创建代理、添加事务通知、前提条件都是该方法是 public 的

* 解法1：改为 public 方法
* 解法2：在配置类中添加 bean 配置如下（不推荐）

```java
@Bean
public TransactionAttributeSource transactionAttributeSource() {
    return new AnnotationTransactionAttributeSource(false);
}
```



### 4.5 父子容器导致的事务失效

```java
package day04.tx.app.service;

// ...

@Service
public class Service5 {

    @Autowired
    private AccountMapper accountMapper;

    @Transactional(rollbackFor = Exception.class)
    public void transfer(int from, int to, int amount) throws FileNotFoundException {
        int fromBalance = accountMapper.findBalanceBy(from);
        if (fromBalance - amount >= 0) {
            accountMapper.update(from, -1 * amount);
            accountMapper.update(to, amount);
        }
    }
}
```

控制器类

```java
package day04.tx.app.controller;

// ...

@Controller
public class AccountController {

    @Autowired
    public Service5 service;

    public void transfer(int from, int to, int amount) throws FileNotFoundException {
        service.transfer(from, to, amount);
    }
}
```

该情况下日志输出不包含transactional的内容

App 配置类 —— 父容器

```java
@Configuration
@ComponentScan("day04.tx.app.service")
@EnableTransactionManagement
// ...
public class AppConfig {
    // ... 有事务相关配置
}
```

Web 配置类 —— 子容器

```java
@Configuration
@ComponentScan("day04.tx.app")
// ...
public class WebConfig {
    // ... 无事务配置
}
```

现在配置了父子容器，WebConfig 对应子容器，AppConfig 对应父容器，发现事务依然失效

子容器并没有配置事务管理`@EnableTransactionManagement`，没有开启事务。

又根据就近原则

```java
@Autowired
public Service5 service;
```

注入的实际上是子容器的Service5.因此失去了事务控制。

* 原因：子容器扫描范围过大，把未加事务配置的 service 扫描进来

* 解法1：各扫描各的，不要图简便(配置扫描包的时候需要精确设置)

* 解法2：不要用父子容器，所有 bean 放在同一容器



### 4.6 调用本类方法导致传播行为失效

```java
@Service
public class Service6 {

    // Propagation.REQUIRED 默认传播行为 若没有事务则新建 若有事务则加入已有
    // foo方法调用bar方法 我们期望foo和bar方法是两个事务
    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void foo() throws FileNotFoundException {
        LoggerUtils.get().debug("foo");
        bar();
    }

    // Propagation.REQUIRES_NEW 无论是否已有事务都新建一个
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void bar() throws FileNotFoundException {
        LoggerUtils.get().debug("bar");
    }
}
```

输出如下

```java
[INFO] 14:24:09.692 [main] - HikariPool-1 - Starting... 
[INFO] 14:24:09.884 [main] - HikariPool-1 - Start completed. 
-- foo方法的CGLIB代理
class day04.tx.app.service.Service6$$EnhancerBySpringCGLIB$$de77784
[DEBUG] 14:24:09.962 [main] - Creating new transaction with name [day04.tx.app.service.Service6.foo]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,-java.lang.Exception 
[DEBUG] 14:24:09.962 [main] - Acquired Connection [HikariProxyConnection@293870357 wrapping com.mysql.cj.jdbc.ConnectionImpl@73877e19] for JDBC transaction 
[DEBUG] 14:24:09.964 [main] - Switching JDBC Connection [HikariProxyConnection@293870357 wrapping com.mysql.cj.jdbc.ConnectionImpl@73877e19] to manual commit 
[DEBUG] 14:24:09.970 [main] - foo 
[DEBUG] 14:24:09.971 [main] - bar 
[DEBUG] 14:24:09.971 [main] - Initiating transaction commit 
[DEBUG] 14:24:09.971 [main] - Committing JDBC transaction on Connection [HikariProxyConnection@293870357 wrapping com.mysql.cj.jdbc.ConnectionImpl@73877e19] 
[DEBUG] 14:24:09.971 [main] - Releasing JDBC Connection [HikariProxyConnection@293870357 wrapping com.mysql.cj.jdbc.ConnectionImpl@73877e19] after transaction 

进程已结束,退出代码0
```

我们看到只开启了一个事务

* 原因：本类方法调用不经过代理，因此无法增强

* 解法1：依赖注入自己（代理）来调用

* 解法2：通过 AopContext 拿到代理对象，来调用

* 解法3：通过 CTW，LTW 实现功能增强

解法1 循环依赖 不推荐

```java
@Service
public class Service6 {

	@Autowired
	private Service6 proxy; // 本质上是一种循环依赖 不推荐

    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void foo() throws FileNotFoundException {
        LoggerUtils.get().debug("foo");
		System.out.println(proxy.getClass());
		proxy.bar();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void bar() throws FileNotFoundException {
        LoggerUtils.get().debug("bar");
    }
}
```

解法2，还需要在 AppConfig 上添加 `@EnableAspectJAutoProxy(exposeProxy = true)`

```java
@Service
public class Service6 {
    
    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void foo() throws FileNotFoundException {
        LoggerUtils.get().debug("foo");
        ((Service6) AopContext.currentProxy()).bar();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void bar() throws FileNotFoundException {
        LoggerUtils.get().debug("bar");
    }
}
```

此时输出为

```java
[INFO] 14:31:59.752 [main] - HikariPool-1 - Starting... 
[INFO] 14:31:59.976 [main] - HikariPool-1 - Start completed. 
class day04.tx.app.service.Service6$$EnhancerBySpringCGLIB$$de77784
[DEBUG] 14:32:00.071 [main] - Creating new transaction with name [day04.tx.app.service.Service6.foo]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,-java.lang.Exception 
[DEBUG] 14:32:00.072 [main] - Acquired Connection [HikariProxyConnection@293870357 wrapping com.mysql.cj.jdbc.ConnectionImpl@73877e19] for JDBC transaction 
[DEBUG] 14:32:00.074 [main] - Switching JDBC Connection [HikariProxyConnection@293870357 wrapping com.mysql.cj.jdbc.ConnectionImpl@73877e19] to manual commit 
[DEBUG] 14:32:00.083 [main] - foo 
[DEBUG] 14:32:00.083 [main] - Suspending current transaction, creating new transaction with name [day04.tx.app.service.Service6.bar] 
[DEBUG] 14:32:00.087 [main] - Acquired Connection [HikariProxyConnection@1120072844 wrapping com.mysql.cj.jdbc.ConnectionImpl@3005db4a] for JDBC transaction 
[DEBUG] 14:32:00.087 [main] - Switching JDBC Connection [HikariProxyConnection@1120072844 wrapping com.mysql.cj.jdbc.ConnectionImpl@3005db4a] to manual commit 
[DEBUG] 14:32:00.088 [main] - bar 
[DEBUG] 14:32:00.088 [main] - Initiating transaction commit 
[DEBUG] 14:32:00.088 [main] - Committing JDBC transaction on Connection [HikariProxyConnection@1120072844 wrapping com.mysql.cj.jdbc.ConnectionImpl@3005db4a] 
[DEBUG] 14:32:00.088 [main] - Releasing JDBC Connection [HikariProxyConnection@1120072844 wrapping com.mysql.cj.jdbc.ConnectionImpl@3005db4a] after transaction 
[DEBUG] 14:32:00.089 [main] - Resuming suspended transaction after completion of inner transaction 
[DEBUG] 14:32:00.089 [main] - Initiating transaction commit 
[DEBUG] 14:32:00.089 [main] - Committing JDBC transaction on Connection [HikariProxyConnection@293870357 wrapping com.mysql.cj.jdbc.ConnectionImpl@73877e19] 
[DEBUG] 14:32:00.089 [main] - Releasing JDBC Connection [HikariProxyConnection@293870357 wrapping com.mysql.cj.jdbc.ConnectionImpl@73877e19] after transaction 
```

可见两个事务均被正常执行



### 4.7 @Transactional 没有保证原子行为

原子性：即一个操作或者多个操作，要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行

```java
@Service
public class Service7 {

    private static final Logger logger = LoggerFactory.getLogger(Service7.class);

    @Autowired
    private AccountMapper accountMapper;

    @Transactional(rollbackFor = Exception.class)
    public void transfer(int from, int to, int amount) {
        int fromBalance = accountMapper.findBalanceBy(from);
        logger.debug("更新前查询余额为: {}", fromBalance);
        if (fromBalance - amount >= 0) {
            accountMapper.update(from, -1 * amount);
            accountMapper.update(to, amount);
        }
    }

    public int findBalance(int accountNo) {
        return accountMapper.findBalanceBy(accountNo);
    }
}
```

上面的代码实际上是有 bug 的，假设 from 余额为 1000，两个线程都来转账 1000，可能会出现扣减为负数的情况

* 原因：事务的原子性仅涵盖 insert、update、delete、select … for update 语句，select 方法并不阻塞

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903120436365.png" alt="image-20210903120436365" style="zoom: 50%;" />

* 如上图所示，红色线程和蓝色线程的查询都发生在扣减之前，都以为自己有足够的余额做扣减



### 4.8 补充: @Transactional 方法导致的 synchronized 失效

针对上面的问题（4.7），能否在方法上加 synchronized 锁来解决呢？

```java
@Service
public class Service7 {

    private static final Logger logger = LoggerFactory.getLogger(Service7.class);

    @Autowired
    private AccountMapper accountMapper;

    @Transactional(rollbackFor = Exception.class)
    public synchronized void transfer(int from, int to, int amount) {
        int fromBalance = accountMapper.findBalanceBy(from);
        logger.debug("更新前查询余额为: {}", fromBalance);
        if (fromBalance - amount >= 0) {
            accountMapper.update(from, -1 * amount);
            accountMapper.update(to, amount);
        }
    }

    public int findBalance(int accountNo) {
        return accountMapper.findBalanceBy(accountNo);
    }
}
```

答案是不行，原因如下：

* synchronized 保证的**仅是目标方法的原子性**，环绕目标方法的**还有 commit 等操作**，它们并**未处于 sync 块内**
* 可以参考下图发现，蓝色线程的查询只要在红色线程提交之前执行，那么依然会查询到有 1000 足够余额来转账

![image-20210903120800185](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903120800185.png)

* 解法1：synchronized 范围应扩大至代理方法调用

  ```java
  Object lock = new Object();
  
  CountDownLatch latch = new CountDownLatch(2);
  new MyThread(() -> {
      synchronized (lock) {
          bean.transfer(1, 2, 1000);
      }
      latch.countDown();
  }, "t1", "boldMagenta").start();
  ```

  取消`transfer(int from, int to, int amount)`方法上的sync，将其加在调用的

* 解法2：使用 select … for update 替换 select

  ```java
  @Select("select balance from account where accountNo=#{accountNo} for update")
  int findBalanceBy(int accountNo);
  ```

  再加上`for update`后可以在数据库层面保证操作原子性，是一种行级锁/排它锁，在数据库层面仅允许一个select操作。



## 5. Spring MVC 执行流程

**要求**

* 掌握 Spring MVC 的执行流程
* 了解 Spring MVC 的重要组件的作用

**概要**

我把整个流程分成三个阶段

* 准备阶段
* 匹配阶段
* 执行阶段

### 准备阶段

1. 在 Web 容器(tomcat/SpringBoot)第一次用到 DispatcherServlet 的时候，会创建其对象并执行 init 方法

2. init 方法内会创建 Spring Web 容器，并调用容器 refresh 方法

3. refresh 过程中会创建并初始化 SpringMVC 中的重要组件， 例如 MultipartResolver，HandlerMapping，HandlerAdapter，HandlerExceptionResolver、ViewResolver 等

4. 容器初始化后，会将上一步初始化好的重要组件，赋值给 DispatcherServlet 的成员变量，留待后用

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903140657163.png" alt="image-20210903140657163" style="zoom: 80%;" />

### 匹配阶段

1. 用户发送的请求统一到达前端控制器 DispatcherServlet

2. DispatcherServlet 遍历所有 HandlerMapping ，找到与路径匹配的处理器

   ① HandlerMapping 有多个，每个 HandlerMapping 会返回不同的处理器对象，谁先匹配，返回谁的处理器。其中能识别 @RequestMapping 的优先级最高

   ② 对应 @RequestMapping 的处理器是 HandlerMethod，它包含了控制器对象和控制器方法信息

   ③ 其中路径与处理器的映射关系在 HandlerMapping 初始化时就会建立好

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903141017502.png" alt="image-20210903141017502" style="zoom:80%;" />

3. 将 HandlerMethod 连同匹配到的拦截器，生成调用链对象 HandlerExecutionChain 返回

<img src="D:\0_大学\2023.2\Java自学\Java面试专题-资料\day04-框架篇\讲义\img\image-20210903141124911.png" alt="image-20210903141124911" style="zoom:80%;" />

4. 遍历HandlerAdapter 处理器适配器，找到能处理 HandlerMethod 的适配器对象，开始调用

<img src="D:\0_大学\2023.2\Java自学\Java面试专题-资料\day04-框架篇\讲义\img\image-20210903141204799.png" alt="image-20210903141204799" style="zoom:80%;" />

### 调用阶段

1. 执行拦截器 preHandle

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903141445870.png" alt="image-20210903141445870" style="zoom: 67%;" />

2. 由 HandlerAdapter 调用 HandlerMethod

   ① 调用前处理不同类型的参数

   ② 调用后处理不同类型的返回值

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903141658199.png" alt="image-20210903141658199" style="zoom:67%;" />

3. 第 2 步没有异常

   ① 返回 ModelAndView

   ② 执行拦截器 postHandle 方法

   ③ 解析视图，得到 View 对象，进行视图渲染

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903141749830.png" alt="image-20210903141749830" style="zoom:67%;" />

4. 第 2 步有异常，进入 HandlerExceptionResolver 异常处理流程

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20210903141844185.png" alt="image-20210903141844185" style="zoom:67%;" />

5. 最后都会执行拦截器的 afterCompletion 方法

6. 如果控制器方法标注了 @ResponseBody 注解，则在第 2 步，就会生成 json 结果，并标记 ModelAndView 已处理，这样就不会执行第 3 步的视图渲染



## 6. Spring 注解

**要求**

* 掌握 Spring 常见注解

> ***提示***
>
> * 注解的详细列表请参考：面试题-spring-注解.xmind
> * 下面列出了视频中重点提及的注解，考虑到大部分注解同学们已经比较熟悉了，仅对个别的作简要说明



**事务注解**

* @EnableTransactionManagement，会额外加载 4 个 bean
  * BeanFactoryTransactionAttributeSourceAdvisor 事务切面类
  * TransactionAttributeSource 用来解析事务属性
  * TransactionInterceptor 事务拦截器
  * TransactionalEventListenerFactory 事务监听器工厂
* @Transactional

**核心**

* @Order

**切面**

* @EnableAspectJAutoProxy
  * 会加载 AnnotationAwareAspectJAutoProxyCreator，它是一个 bean 后处理器，用来创建代理
  * 如果没有配置 @EnableAspectJAutoProxy，又需要用到代理（如事务）则会使用 InfrastructureAdvisorAutoProxyCreator 这个 bean 后处理器

**组件扫描与配置类**

* @Component

* @Controller

* @Service

* @Repository

* @ComponentScan

* @Conditional 

* @Configuration

  ```java
  public class TestConfiguration {
      public static void main(String[] args) {
          // 创建Spring容器
          GenericApplicationContext context = new GenericApplicationContext();
          // bean后工厂处理器 用于解析@Bean与@Configuration注解
          AnnotationConfigUtils
              .registerAnnotationConfigProcessors(context.getDefaultListableBeanFactory());
          // 注册config
          context.registerBean("myConfig", MyConfig.class);
          // 调用refresh进行初始化操作
          context.refresh();
  //        System.out.println(context.getBean(MyConfig.class).getClass());
      }
  
      @Configuration
      @MapperScan("aaa")
      // 注意点1: 配置类其实相当于一个工厂, 标注 @Bean 注解的方法相当于工厂方法
      static class MyConfig {
          // 注意点2: @Bean 不支持方法重载, 如果有多个重载方法, 仅有一个能入选为工厂方法
          // 注意点3: @Configuration 默认会为标注的类生成代理, 其目的是保证 @Bean 方法相互调用时, 仍然能保证其单例特性
          @Bean
          public Bean1 bean1() {
              System.out.println("bean1()");
              System.out.println(bean2());
              System.out.println(bean2());
              System.out.println(bean2());
              return new Bean1();
          }
  
          @Bean
          public Bean1 bean1(@Value("${java.class.version}") String a) {
              System.out.println("bean1(" + a + ")");
              return new Bean1();
          }
  
  		// 如果有多个重载方法 参数越多方法优先级越高
          @Bean
          public Bean1 bean1(@Value("${java.class.version}") String a, @Value("${JAVA_HOME}") String b) {
              System.out.println("bean1(" + a + ", " + b + ")");
              return new Bean1();
          }
  
          @Bean
          public Bean2 bean2() {
              System.out.println("bean2()");
              return new Bean2();
          }
  
          // 注意点4: @Configuration 中如果含有 BeanFactory 后处理器, 则实例工厂方法会导致 MyConfig 提前创建, 造成其依赖注入失败
          // 解决方法是该用静态工厂方法或直接为 @Bean 的方法参数依赖注入, 针对 MapperScanner 可以改用注解方式
          @Value("${java.class.version}")
          private String version;
  
          // 这个bean后处理器是以成员方法的形式出现的 需要先有MyConfig对象才能创建扫描器
          // 根据refresh流程 bean后处理器的创建在第五步invokeBeanFactoryPostProcessors
          // 但是配置类应当在第十一步finishBeanFactoryInitialization被创建(进行bean的增强)
          // 因此调用该方法是MyConfig并未被创建 因此无法正常执行依赖注入
          // 修改方法 1. 将该方法改为static
          @Bean
          public static MapperScannerConfigurer configurer() {
              MapperScannerConfigurer scanner = new MapperScannerConfigurer();
              scanner.setBasePackage("aaa");
              return scanner;
          }
  
          @Bean
          public Bean3 bean3() {
              System.out.println("bean3() " + version);
              return new Bean3();
          }
      }
  
      static class Bean1 {
  
      }
  
      static class Bean2 {
  
      }
  
      static class Bean3 {
  
      }
  }
  ```

  * 配置类其实相当于一个工厂, 标注 @Bean 注解的方法相当于工厂方法
  * @Bean 不支持方法重载, 如果有多个重载方法, 仅有一个能入选为工厂方法
  * @Configuration 默认会为标注的类生成代理, 其目的是保证 @Bean 方法相互调用时, 仍然能保证其单例特性
  * @Configuration 中如果含有 BeanFactory 后处理器, 则实例工厂方法会导致 MyConfig 提前创建, 造成其依赖注入失败，解决方法是改用静态工厂方法或直接为 @Bean 的方法参数依赖注入, 针对 Mapper 扫描可以改用注解方式

* @Bean

* @Import 

  ```java
  public class TestDeferredImport {
  
      public static void main(String[] args) {
          GenericApplicationContext context = new GenericApplicationContext();
          DefaultListableBeanFactory beanFactory = context.getDefaultListableBeanFactory();
          beanFactory.setAllowBeanDefinitionOverriding(false); // 不允许同名定义覆盖
          AnnotationConfigUtils.registerAnnotationConfigProcessors(beanFactory);
          context.registerBean(MyConfig.class);
          context.refresh();
  
          System.out.println(context.getBean(MyBean.class));
      }
  
      // 1. 同一配置类中, @Import 先解析  @Bean 后解析
      // 2. 同名定义, 默认后面解析的会覆盖前面解析的
      // 3. 不允许覆盖的情况下, 如何能够让 MyConfig(主配置类) 的配置优先? (虽然覆盖方式能解决)
      // 4. DeferredImportSelector 最后工作, 可以简单认为先解析 @Bean, 再 Import
      @Configuration
      // @Import(Bean1.class) // 1. 引入单个bean
      // @Import(OtherConfig.class) // 2. 引入一个配置类 会将配置类以及配置类中的bean都注入到容器中
      @Import(MySelector.class) // 3. 通过selector引入多个类
      // @Import(MyRegistry.class) // 4. 通过beanDefinition注册器 注册bean加入容器
      static class MyConfig { // 主配置 - 程序员编写的
          @Bean
          public MyBean myBean() {
              return new Bean1();
          }
      }
  
      static class MySelector implements DeferredImportSelector {
  
          @Override
          public String[] selectImports(AnnotationMetadata importingClassMetadata) {
              return new String[]{OtherConfig.class.getName()};
          }
      }
      
      static class MyRegister implements ImportBeanDefinitionRegistrar {
  
          @Override
          public void registerBeanDefinitions(
              AnnotationMetadata importingClassMetadata, 
              BeanDefinitionRegistry registry) {
              registry.registerBeanDefinition("myBean", new RootBeanDefinition(Bean2.class));
          }
      }
  
      @Configuration
      static class OtherConfig { // 从属配置 - 自动配置
          @Bean
          @ConditionalOnMissingBean
          public MyBean myBean() {
              return new Bean2();
          }
      }
  
      interface MyBean {
  
      }
  
      static class Bean1 implements MyBean {
  
      }
  
      static class Bean2 implements MyBean {
  
      }
  
  }
  ```

  * 四种用法

    ① 引入单个 bean

    ② 引入一个配置类

    ③ 通过 Selector 引入多个类

    ④ 通过 beanDefinition 注册器

  * 解析规则

    * 同一配置类中, @Import 先解析  @Bean 后解析
    * 同名定义, 默认后面解析的会覆盖前面解析的
    * 不允许覆盖的情况下, 如何能够让 MyConfig(主配置类) 的配置优先? (虽然覆盖方式能解决)
    * 采用 DeferredImportSelector，因为它最后工作, 可以简单认为先解析 @Bean, 再 Import

* @Lazy

  * 加在类上，表示此类延迟实例化、初始化
  * 加在方法参数上，此参数会以代理方式注入

* @PropertySource

**依赖注入**

* @Autowired
* @Qualifier
* @Value

**mvc mapping**

* @RequestMapping，可以派生多个注解如 @GetMapping 等

**mvc rest**

* @RequestBody
* @ResponseBody，组合 @Controller =>  @RestController
* @ResponseStatus

**mvc 统一处理**

* @ControllerAdvice，组合 @ResponseBody => @RestControllerAdvice
* @ExceptionHandler

**mvc 参数**

* @PathVariable

**mvc ajax**

* @CrossOrigin

**boot auto**

* @SpringBootApplication
  * @SpringBootConfiguration
  * @EnableAutoConfiguration

* @EnableAutoConfiguration
* @SpringBootConfiguration 整个程序只能有一个被标注@SpringBootConfiguration的类
  * @Configuration


**boot condition**

* @ConditionalOnClass，classpath 下存在某个 class 时，条件才成立
* @ConditionalOnMissingBean，beanFactory 内不存在某个 bean 时，条件才成立
* @ConditionalOnProperty，配置文件中存在某个 property（键、值）时，条件才成立

**boot properties**

* @ConfigurationProperties，会将当前 bean 的属性与配置文件中的键值进行绑定
* @EnableConfigurationProperties，会添加两个较为重要的 bean
  * ConfigurationPropertiesBindingPostProcessor，bean 后处理器，在 bean 初始化前调用下面的 binder
  * ConfigurationPropertiesBinder，真正执行绑定操作



## 7. SpringBoot 自动配置原理

**要求**

* 掌握 SpringBoot 自动配置原理

### 自动配置原理

@SpringBootConfiguration 是一个组合注解，由 @ComponentScan、@EnableAutoConfiguration 和 @SpringBootConfiguration 组成

1. @SpringBootConfiguration 与普通 @Configuration 相比，唯一区别是前者要求整个 app 中只出现一次
2. @ComponentScan
   * excludeFilters - 用来在组件扫描时进行排除，也会排除自动配置类

3. @EnableAutoConfiguration 也是一个组合注解，由下面注解组成
   * @AutoConfigurationPackage – 用来记住扫描的起始包
   * @Import(AutoConfigurationImportSelector.class) 用来加载 `META-INF/spring.factories` 中的自动配置类

### 为什么不使用 @Import 直接引入自动配置类

有两个原因：

1. 让主配置类和自动配置类变成了强耦合，主配置类不应该知道有哪些从属配置
2. 直接用 `@Import(自动配置类.class)`，引入的配置解析优先级较高，自动配置类的解析应该在主配置没提供时作为默认配置

因此，采用了 `@Import(AutoConfigurationImportSelector.class)`

* 由 `AutoConfigurationImportSelector.class` 去读取 `META-INF/spring.factories` 中的自动配置类，实现了弱耦合。
* 另外 `AutoConfigurationImportSelector.class` 实现了 DeferredImportSelector 接口，让自动配置的解析晚于主配置的解析



## 8. Spring 中的设计模式

**要求**

* 掌握 Spring 中常见的设计模式

### 8.1 Spring 中的 Singleton

请大家区分 singleton pattern 与 Spring 中的 singleton bean

* 根据单例模式的目的 *Ensure a class only has one instance, and provide a global point of access to it* 
* 显然 Spring 中的 singleton bean 并非实现了单例模式，**singleton bean 只能保证每个容器内，相同 id 的 bean 单实例**
* 当然 Spring 中也用到了单例模式，例如
  * org.springframework.transaction.TransactionDefinition#withDefaults
  * org.springframework.aop.TruePointcut#INSTANCE
  * org.springframework.aop.interceptor.ExposeInvocationInterceptor#ADVISOR
  * org.springframework.core.annotation.AnnotationAwareOrderComparator#INSTANCE
  * org.springframework.core.OrderComparator#INSTANCE

### 8.2 Spring 中的 Builder

定义 *Separate the construction of a complex object from its representation so that the same construction process can create different representations* 

它的主要亮点有三处：

1. 较为灵活的构建产品对象

2. 在不执行最后 build 方法前，产品对象都不可用

3. 构建过程采用链式调用，看起来比较爽

Spring 中体现 Builder 模式的地方：

* org.springframework.beans.factory.support.BeanDefinitionBuilder

* org.springframework.web.util.UriComponentsBuilder

* org.springframework.http.ResponseEntity.HeadersBuilder

* org.springframework.http.ResponseEntity.BodyBuilder

### 8.3 Spring 中的 Factory Method

定义 *Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses* 

根据上面的定义，Spring 中的 ApplicationContext 与 BeanFactory 中的 getBean 都可以视为工厂方法，它隐藏了 bean （产品）的创建过程和具体实现

Spring 中其它工厂：

* org.springframework.beans.factory.FactoryBean

* @Bean 标注的静态方法及实例方法

* ObjectFactory 及 ObjectProvider

前两种工厂主要封装第三方的 bean 的创建过程，后两种工厂可以推迟 bean 创建，解决循环依赖及单例注入多例等问题

### 8.4 Spring 中的 Adapter

定义 *Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces* 

典型的实现有两处：

* org.springframework.web.servlet.HandlerAdapter – 因为控制器实现有各种各样，比如有
  * 大家熟悉的 @RequestMapping 标注的控制器实现
  * 传统的基于 Controller 接口（不是 @Controller注解啊）的实现
  * 较新的基于 RouterFunction 接口的实现
  * 它们的处理方法都不一样，为了统一调用，必须适配为 HandlerAdapter 接口
* org.springframework.beans.factory.support.DisposableBeanAdapter – 因为销毁方法多种多样，因此都要适配为 DisposableBean 来统一调用销毁方法 

### 8.5 Spring 中的 Composite

定义 *Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly* 

典型实现有：

* org.springframework.web.method.support.HandlerMethodArgumentResolverComposite
* org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite
* org.springframework.web.servlet.handler.HandlerExceptionResolverComposite
* org.springframework.web.servlet.view.ViewResolverComposite

composite 对象的作用是，将分散的调用集中起来，统一调用入口，它的特征是，与具体干活的实现实现同一个接口，当调用 composite 对象的接口方法时，其实是委托具体干活的实现来完成

### 8.6 Spring 中的 Decorator

定义 *Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality* 

典型实现：

* org.springframework.web.util.ContentCachingRequestWrapper

### 8.7 Spring 中的 Proxy

定义 *Provide a surrogate or placeholder for another object to control access to it* 

装饰器模式注重的是功能增强，避免子类继承方式进行功能扩展，而代理模式更注重控制目标的访问

典型实现：

* org.springframework.aop.framework.JdkDynamicAopProxy
* org.springframework.aop.framework.ObjenesisCglibAopProxy

### 8.8 Spring 中的 Chain of Responsibility

定义 *Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it* 

典型实现：

* org.springframework.web.servlet.HandlerInterceptor

### 8.9 Spring 中的 Observer

定义 *Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically* 

典型实现：

* org.springframework.context.ApplicationListener
* org.springframework.context.event.ApplicationEventMulticaster
* org.springframework.context.ApplicationEvent

### 8.10 Spring 中的 Strategy

定义 *Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it* 

典型实现：

* org.springframework.beans.factory.support.InstantiationStrategy
* org.springframework.core.annotation.MergedAnnotations.SearchStrategy
* org.springframework.boot.autoconfigure.condition.SearchStrategy

### 8.11 Spring 中的 Template Method

定义 *Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure* 

典型实现：

* 大部分以 Template 命名的类，如 JdbcTemplate，TransactionTemplate
* 很多以 Abstract 命名的类，如 AbstractApplicationContext



## 9. 循环依赖

**要求**

* 要完全理解循环依赖，需要理解代理对象的创建时机
* 掌握ProxyFactory创建代理的过程，理解Advisor，Advice，PointCut与Aspect
* 掌握AnnotationAwareAspectJAutoProxyCreater筛选Advisor合格者并创建代理的过程

