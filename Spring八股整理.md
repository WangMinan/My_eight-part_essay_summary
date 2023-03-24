<div align="center">
    <h1>
        🥬Spring八股整理
    </h1>
</div>



## IoC(IoC与DI)

参考[IoC源码解读](https://www.cnblogs.com/ITtangtang/p/3978349.html)

**Ioc—Inversion of Control，即“控制反转”**，不是什么技术，而是一种设计思想。在Java开发中，Ioc意味着将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制。如何理解好Ioc呢？理解好Ioc的关键是要明确“谁控制谁，控制什么，为何是反转，哪些方面反转了”，那我们来深入分析一下：

 

●**谁控制谁，控制什么：**

传统Java SE程序设计，我们直接在对象内部**通过new进行创建对象**，是程序主动去创建依赖对象；

而IoC是有**专门一个容器来创建这些对象**，即由Ioc容器来控制对象的创建；谁控制谁？当然是IoC 容器控制了对象；控制什么？那就是主要控制了外部资源获取（不只是对象包括比如文件等）。

 

●**为何是反转，哪些方面反转了：**

有反转就有正转，传统应用程序是由我们自己在对象中主动控制去直接获取依赖对象，也就是正转；而反转则是由容器来帮忙创建及注入依赖对象；为何是反转？因为由容器帮我们查找及注入依赖对象，对象只是被动的接受依赖对象，所以是反转；哪些方面反转了？依赖对象的获取被反转了。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/1368782-20180426150619347-288835784.jpg)





### Spring IoC的体系结构

#### BeanFactory

Spring Bean的创建是典型的工厂模式，这一系列的Bean工厂，也即IOC容器为开发者管理对象间的依赖关系提供了很多便利和基础服务，在Spring中有许多的IoC容器的实现供用户选择和使用，其相互关系如下：

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172219470349285.x-png)

其中`BeanFactory`作为最顶层的一个接口类，它定义了IoC容器的基本功能规范，`BeanFactory`有三个子类：`ListableBeanFactory`、`HierarchicalBeanFactory`和`AutowireCapableBeanFactory`。

但是从上图中我们可以发现最终的默认实现类是 `DefaultListableBeanFactory`，他实现了所有的接口。

那为何要定义这么多层次的接口呢？

查阅这些接口的源码和说明发现，每个接口都有他使用的场合，它主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。例如 `ListableBeanFactory` 接口表示这些 Bean 是**可列表的**，而 `HierarchicalBeanFactory` 表示的是这些 Bean 是**有继承关系的**，也就是每个Bean 有可能有父 Bean。`AutowireCapableBeanFactory` 接口**定义 Bean 的自动装配规则**。

这四个接口共同定义了 Bean 的集合、Bean 之间的关系、以及 Bean 行为.

**最基本的IoC容器接口BeanFactory**

```java
public interface BeanFactory {    
    //对FactoryBean的转义定义，因为如果使用bean的名字检索FactoryBean得到的对象是工厂生成的对象，    
    //如果需要得到工厂本身，需要转义           
    String FACTORY_BEAN_PREFIX = "&"; 
    //根据bean的名字，获取在IoC容器中得到bean实例    
    Object getBean(String name) throws BeansException;    

    //根据bean的名字和Class类型来得到bean实例，增加了类型安全验证机制。    
    Object getBean(String name, Class requiredType) throws BeansException;    

    //提供对bean的检索，看看是否在IoC容器有这个名字的bean    
    boolean containsBean(String name);    

    //根据bean名字得到bean实例，并同时判断这个bean是不是单例    
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;    

    //得到bean实例的Class类型    
    Class getType(String name) throws NoSuchBeanDefinitionException;    

    //得到bean的别名，如果根据别名检索，那么其原名也会被检索出来    
    String[] getAliases(String name);    
}
```

在BeanFactory里**只对IOC容器的基本行为作了定义**，根本**不关心你的bean是如何定义怎样加载的**。正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象的，这个基本的接口不关心。

而要知道工厂是如何产生对象的，我们**需要看具体的IoC容器实现**，spring提供了许多IoC容器的实现。比如XmlBeanFactory，ClasspathXmlApplicationContext等。

+ XmlBeanFactory就是针对最基本的ioc容器的实现，这个IOC容器可以读取XML文件定义的BeanDefinition.
+ ApplicationContext是Spring提供的一个高级的IoC容器，读取Java代码中对Bean的定义。它除了能够提供IoC容器的基本功能外，还有以下特点：
  + 支持信息源，可以实现国际化。（实现MessageSource接口）
  + 访问资源。(实现ResourcePatternResolver接口)
  + 支持应用事件。(实现ApplicationEventPublisher接口)

#### BeanDefinition

SpringIOC容器管理了我们定义的各种Bean对象及其相互的关系，Bean对象在Spring实现中是以BeanDefinition来描述的，其继承体系如下：

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172221137214635.x-png)

Bean 的解析过程非常复杂，功能被分的很细，因为这里需要被扩展的地方很多，必须保证有足够的灵活性，以应对可能的变化。Bean 的解析主要就是对 Spring 配置文件的解析。这个解析过程主要通过下图中的类完成：

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172221565186760.x-png)



### Ioc容器的初始化(源码)

IoC容器的初始化包括`BeanDefinition`的**Resource定位、载入和注册**这三个基本的过程。我们以`ApplicationContext`为例讲解，`ApplicationContext`系列容器也许是我们最熟悉的，因为web项目中使用的`XmlWebApplicationContext`就属于这个继承体系，还有`ClasspathXmlApplicationContext`等，其继承体系如下图所示：

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172222438935854.x-png)

`ApplicationContext`允许上下文嵌套，通过保持父上下文可以维持一个上下文体系。对于bean的查找可以在这个上下文体系中发生，首先检查当前上下文，其次是父上下文，逐级向上，这样为不同的Spring应用提供了一个共享的bean定义环境。

#### XmlBeanFactory

```java
//根据Xml配置文件创建Resource资源对象，该对象中包含了BeanDefinition的信息
ClassPathResource resource = new ClassPathResource("application-context.xml");
//创建DefaultListableBeanFactory
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
//创建XmlBeanDefinitionReader读取器，用于载入BeanDefinition。之所以需要BeanFactory作为参数，是因为会将读取的信息回调配置给factory
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
//XmlBeanDefinitionReader执行载入BeanDefinition的方法，最后会完成Bean的载入和注册。完成后Bean就成功的放置到IOC容器当中，以后我们就可以从中取得Bean来使用
reader.loadBeanDefinitions(resource);
```

#### FileSystemXmlApplicationContext

##### 1. 定义

```java
ApplicationContext =new FileSystemXmlApplicationContext(xmlPath);
```

##### 2. 设置资源加载器和资源定位

```java
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) 
            throws BeansException {
    // 首先，调用父类容器的构造方法(super(parent)方法)为容器设置好Bean资源加载器。
    super(parent);
    // 调用父类AbstractRefreshableConfigApplicationContext的setConfigLocations(configLocations)方法设置Bean定义资源文件的定位路径。
    setConfigLocations(configLocations);  
    if (refresh) {  
        refresh();  
    }  
}

//处理单个资源文件路径为一个字符串的情况  
public void setConfigLocation(String location) {  
   //String CONFIG_LOCATION_DELIMITERS = ",; /t/n";  
   //即多个资源文件路径之间用” ,; /t/n”分隔，解析成数组形式  
    setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));  
}  
//解析Bean定义资源文件的路径，处理多个资源文件字符串数组  
public void setConfigLocations(String[] locations) {  
    if (locations != null) {  
        Assert.noNullElements(locations, "Config locations must not be null");  
        this.configLocations = new String[locations.length];  
        for (int i = 0; i < locations.length; i++) {  
            // resolvePath为同一个类中将字符串解析为路径的方法  
            this.configLocations[i] = resolvePath(locations[i]).trim();  
        }  
    }  
    else {  
        this.configLocations = null;  
    }  
}
```

通过这两个方法的源码我们可以看出，我们既可以使用一个字符串来配置多个Spring Bean定义资源文件，也可以使用字符串数组，即下面两种方式都是可以的：

+ ClasspathResource res = new ClasspathResource(“a.xml,b.xml,……”);

  多个资源文件路径之间可以是用” ,; /t/n”等分隔。

+ ClasspathResource res = new ClasspathResource(newString[]{“a.xml”,”b.xml”,……});

至此，Spring IoC容器在初始化时**将配置的Bean定义资源文件定位为Spring封装的Resource**。

##### 3. `AbstractApplicationContext`的`refresh`函数载入Bean定义过程

Spring IoC容器对Bean定义资源的载入是**从`refresh()`函数开始的**，`refresh()`是一个模板方法

`refresh()`方法的作用是：

在创建IoC容器前，如果已经有容器存在，则需要把已有的容器销毁和关闭，以**保证在refresh之后使用的是新建立起来的IoC容器**。refresh的作用类似于对IoC容器的**重启**，在新建立好的容器中对容器进行初始化，对Bean定义资源进行载入

`FileSystemXmlApplicationContext`通过调用其父类`AbstractApplicationContext`的`refresh()`函数启动整个IoC容器对Bean定义的载入过程：

<span id="jump">refresh源码</span>

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
       //调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
       prepareRefresh();
       //告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从
       //子类的refreshBeanFactory()方法启动
       ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
       //为BeanFactory配置容器特性，例如类加载器、事件处理器等
       prepareBeanFactory(beanFactory);
       try {
           //为容器的某些子类指定特殊的BeanPost事件处理器
           postProcessBeanFactory(beanFactory);
           //调用所有注册的BeanFactoryPostProcessor的Bean
           invokeBeanFactoryPostProcessors(beanFactory);
           //为BeanFactory注册BeanPost事件处理器.
           //BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
           registerBeanPostProcessors(beanFactory);
           //初始化信息源，和国际化相关.
           initMessageSource();
           //初始化容器事件传播器.
           initApplicationEventMulticaster();
           //调用子类的某些特殊Bean初始化方法
           onRefresh();
           //为事件传播器注册事件监听器.
           registerListeners();
           //初始化所有剩余的单态Bean.
           finishBeanFactoryInitialization(beanFactory);
           //初始化容器的生命周期事件处理器，并发布容器的生命周期事件
           finishRefresh();
       }
       catch (BeansException ex) {
           //销毁以创建的单态Bean
           destroyBeans();
           //取消refresh操作，重置容器的同步标识.
           cancelRefresh(ex);
           throw ex;
       }
   }
}
```

`refresh()`方法主要为IoC容器Bean的生命周期管理提供条件，Spring IoC容器载入Bean定义资源文件从其子类容器的`refreshBeanFactory()`方法启动，所以整个`refresh()`中`ConfigurableListableBeanFactory beanFactory =obtainFreshBeanFactory();`这句以后代码的都是注册容器的信息源和生命周期事件，载入过程就是从这句代码启动。

`AbstractApplicationContext`的`obtainFreshBeanFactory()`方法调用子类容器的`refreshBeanFactory()`方法，启动容器载入Bean定义资源文件的过程，代码如下：

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {  
    //这里使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法，具体实现调用子类容器的refreshBeanFactory()方法
 	refreshBeanFactory();  
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();  
    if (logger.isDebugEnabled()) {  
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);  
    }  
    return beanFactory;  
}
```

`AbstractApplicationContext`子类的`refreshBeanFactory()`方法：

`AbstractApplicationContext`类中只抽象定义了`refreshBeanFactory()`方法，容器真正调用的是其子类`AbstractRefreshableApplicationContext`实现的  `refreshBeanFactory()`方法，方法的源码如下：

```java
protected final void refreshBeanFactory() throws BeansException {
   if (hasBeanFactory()) {//如果已经有容器，销毁容器中的bean，关闭容器
       destroyBeans();
       closeBeanFactory();
   }
   try {
        //创建IoC容器
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
       //对IoC容器进行定制化，如设置启动参数，开启注解的自动装配等
       customizeBeanFactory(beanFactory);
       //调用载入Bean定义的方法，主要这里又使用了一个委派模式，在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
       loadBeanDefinitions(beanFactory);
       synchronized (this.beanFactoryMonitor) {
           this.beanFactory = beanFactory;
       }
   }
   catch (IOException ex) {
       throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}
```

在这个方法中，先判断BeanFactory是否存在，如果存在则先销毁beans并关闭beanFactory，接着创建`DefaultListableBeanFactory`，并调用`loadBeanDefinitions(beanFactory)`装载bean定义。

##### 4. `AbstractRefreshableApplicationContext`子类的`loadBeanDefinitions`方法

`AbstractRefreshableApplicationContext`中只定义了抽象的`loadBeanDefinitions`方法，容器真正调用的是其子类`AbstractXmlApplicationContext`对该方法的实现，`AbstractXmlApplicationContext`的主要源码如下：

`loadBeanDefinitions`方法同样是抽象方法，是由其子类实现的，也即在`AbstractXmlApplicationContext`中。

```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {
    ……
    //实现父类抽象的载入Bean定义方法
    @Override
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        //创建XmlBeanDefinitionReader，即创建Bean读取器，并通过回调设置到容器中去，容  器使用该读取器读取Bean定义资源
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        //为Bean读取器设置Spring资源加载器，AbstractXmlApplicationContext的
        //祖先父类AbstractApplicationContext继承DefaultResourceLoader，因此，容器本身也是一个资源加载器
       beanDefinitionReader.setResourceLoader(this);
       //为Bean读取器设置SAX xml解析器
       beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
       //当Bean读取器读取Bean定义的Xml资源文件时，启用Xml的校验机制
       initBeanDefinitionReader(beanDefinitionReader);
       //Bean读取器真正实现加载的方法
       loadBeanDefinitions(beanDefinitionReader);
   }
   //Xml Bean读取器加载Bean定义资源
   protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
       //获取Bean定义资源的定位
       Resource[] configResources = getConfigResources();
       if (configResources != null) {
           //Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位
           //的Bean定义资源
           reader.loadBeanDefinitions(configResources);
       }
       //如果子类中获取的Bean定义资源定位为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源
       String[] configLocations = getConfigLocations();
       if (configLocations != null) {
           //Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位
           //的Bean定义资源
           reader.loadBeanDefinitions(configLocations);
       }
   }
   //这里又使用了一个委托模式，调用子类的获取Bean定义资源定位的方法
   //该方法在ClassPathXmlApplicationContext中进行实现，对于我们
   //举例分析源码的FileSystemXmlApplicationContext没有使用该方法
   protected Resource[] getConfigResources() {
       return null;
   }   ……
}
```

Xml Bean读取器(`XmlBeanDefinitionReader`)调用其父类`AbstractBeanDefinitionReader`的 `reader.loadBeanDefinitions`方法读取Bean定义资源。

由于我们使用`FileSystemXmlApplicationContext`作为例子分析，因此`getConfigResources`的返回值为null，因此程序执行`reader.loadBeanDefinitions(configLocations)`分支。

##### 5. AbstractBeanDefinitionReader读取Bean定义资源 

 可以到org.springframework.beans.factory.support看一下BeanDefinitionReader的结构

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172239426289723.x-png)

在其抽象父类AbstractBeanDefinitionReader中定义了载入过程

`AbstractBeanDefinitionReader`的`loadBeanDefinitions`方法源码如下：

```java
//重载方法，调用下面的loadBeanDefinitions(String, Set<Resource>);方法  
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {  
    return loadBeanDefinitions(location, null);  
}

public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {  
    //获取在IoC容器初始化过程中设置的资源加载器  
    ResourceLoader resourceLoader = getResourceLoader();  
    if (resourceLoader == null) {  
        throw new BeanDefinitionStoreException(  
            "Cannot import bean definitions from location [" + location + "]: no ResourceLoader available"); 
    }  
    if (resourceLoader instanceof ResourcePatternResolver) {  
        try {  
            //将指定位置的Bean定义资源文件解析为Spring IoC容器封装的资源  
            //加载多个指定位置的Bean定义资源文件  
            Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);  
            //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能  
            int loadCount = loadBeanDefinitions(resources);  
            if (actualResources != null) {  
                for (Resource resource : resources) {  
                    actualResources.add(resource);  
                }  
            }  
            if (logger.isDebugEnabled()) {  
                logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");  
            }  
            return loadCount;  
        }  
        catch (IOException ex) {  
            throw new BeanDefinitionStoreException(  
                "Could not resolve bean definition resource pattern [" + location + "]", ex);  
       }  
   	}  
    else {  
        //将指定位置的Bean定义资源文件解析为Spring IoC容器封装的资源  
        //加载单个指定位置的Bean定义资源文件  
        Resource resource = resourceLoader.getResource(location);  
        //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能  
        int loadCount = loadBeanDefinitions(resource);  
        if (actualResources != null) {  
            actualResources.add(resource);  
        }  
        if (logger.isDebugEnabled()) {  
           logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");  
        }  
        return loadCount;  
    }  
}  

//重载方法，调用loadBeanDefinitions(String);  
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {  
    Assert.notNull(locations, "Location array must not be null");  
    int counter = 0;  
    for (String location : locations) {  
        counter += loadBeanDefinitions(location);  
    }  
    return counter;  
}
```

**补充:Java的三点运算符**

>  java类型后面跟三个点是代表可以接受多个实际参数，这里的多个指的是不限个数，可以是一个、两个、三个甚至更多。java中类型后面加三个点是java1.5之后出现的新的内容,使用在函数的形参上，相当于一个数组，调用函数时传递多少了实际参数，都可以存储到这个形参上，需要注意的是，使用这个形参必须放在最后一位形参位置上，否则会报错。

`loadBeanDefinitions(Resource...resources)`方法和上面分析的3个方法类似，同样也是调用`XmlBeanDefinitionReader`的`loadBeanDefinitions`方法。

从对`AbstractBeanDefinitionReader`的`loadBeanDefinitions`方法源码分析可以看出该方法做了以下两件事：

+ 首先，调用资源加载器的获取资源方法`resourceLoader.getResource(location)`，获取到要加载的资源。
+ 其次，真正执行加载功能是其子类`XmlBeanDefinitionReader`的`loadBeanDefinitions`方法。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172241209099766.x-png)

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172241310182544.x-png)

看到第8、16行(`if (resourceLoader == null) {  `开始)，结合上面的ResourceLoader与ApplicationContext的继承关系图，可以知道此时调用的是`DefaultResourceLoader`中的`getSource()`方法定位Resource，因为`FileSystemXmlApplicationContext`本身就是`DefaultResourceLoader`的实现类，所以此时又回到了`FileSystemXmlApplicationContext`中来。

##### 6. 资源加载器获取要读入的资源

`XmlBeanDefinitionReader`通过调用其父类`DefaultResourceLoader`的`getResource`方法获取要加载的资源，其源码如下

```java
//获取Resource的具体实现方法
public Resource getResource(String location) {
   Assert.notNull(location, "Location must not be null");
   //如果是类路径的方式，那需要使用ClassPathResource 来得到bean文件的资源对象
   if (location.startsWith(CLASSPATH_URL_PREFIX)) {
       return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
   }
    try {
     // 如果是URL 方式，使用UrlResource作为bean文件的资源对象
    URL url = new URL(location);
    return new UrlResource(url);
   } catch (MalformedURLException ex) {
        ...
   }
   //如果既不是classpath标识，又不是URL标识的Resource定位，则调用
   //容器本身的getResourceByPath方法获取Resource
   return getResourceByPath(location);
}
```

`FileSystemXmlApplicationContext`容器提供了`getResourceByPath`方法的实现，就是为了处理既不是classpath标识，又不是URL标识的Resource定位这种情况。

```java
protected Resource getResourceByPath(String path) {    
   if (path != null && path.startsWith("/")) {    
        path = path.substring(1);    
    }  
    //这里使用文件系统资源对象来定义bean 文件
    return new FileSystemResource(path);  
}
```

这样代码就回到了`FileSystemXmlApplicationContext`中来，他提供了`FileSystemResource` 来完成从文件系统得到配置文件的资源定义。

这样，就可以从文件系统路径上对IoC 配置文件进行加载——当然我们可以按照这个逻辑从任何地方加载，在Spring中我们看到它提供 的各种资源抽象，比如`ClassPathResource`, `URLResource`, `FileSystemResource` 等来供我们使用。上面我们看到的是定位Resource 的一个过程，而这只是加载过程的一部分.

##### 7. XmlBeanDefinitionReader加载Bean定义资源

Bean定义的Resource得到了

继续回到`XmlBeanDefinitionReader`的`loadBeanDefinitions(Resource …)`方法看到代表bean文件的资源定义以后的载入过程。

通过源码分析，载入Bean定义资源文件的最后一步是将Bean定义资源转换为`Document`对象，该过程由`documentLoader`实现

##### 8. DocumentLoader将Bean定义资源转换为Document对象

```java
//使用标准的JAXP将载入的Bean定义资源转换成document对象
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
       ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
   //创建文件解析器工厂
   DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
   if (logger.isDebugEnabled()) {
       logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
   }
   //创建文档解析器
   DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
   //解析Spring的Bean定义资源
   return builder.parse(inputSource);
}
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
       throws ParserConfigurationException {
   //创建文档解析工厂
   DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
   factory.setNamespaceAware(namespaceAware);
   //设置解析XML的校验
   if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
       factory.setValidating(true);
       if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
           factory.setNamespaceAware(true);
           try {
               factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
           }
           catch (IllegalArgumentException ex) {
               ParserConfigurationException pcex = new ParserConfigurationException(
                       "Unable to validate using XSD: Your JAXP provider [" + factory +
                       "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
                       "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
               pcex.initCause(ex);
               throw pcex;
           }
       }
   }
   return factory;
}
```

该解析过程调用JavaEE标准的JAXP标准进行处理。



**至此Spring IoC容器根据定位的Bean定义资源文件，将其加载读入并转换成为`Document`对象过程完成。**

**接下来我们要继续分析Spring IoC容器将载入的Bean定义资源文件转换为`Document`对象之后，是如何将其解析为Spring IoC管理的Bean对象并将其注册到容器中的。**



##### 9. XmlBeanDefinitionReader解析载入的Bean定义资源文件

`XmlBeanDefinitionReader`类中的`doLoadBeanDefinitions`方法是从特定XML文件中实际载入Bean定义资源的方法，该方法在载入Bean定义资源之后将其转换为Document对象，接下来调用`registerBeanDefinitions`启动Spring IoC容器对Bean定义的解析过程，`registerBeanDefinitions`方法源码如下：

```java
//按照Spring的Bean语义要求将Bean定义资源解析并转换为容器内部数据结构  
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException{  
    //得到BeanDefinitionDocumentReader来对xml格式的BeanDefinition解析  
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();  
    //获得容器中注册的Bean数量  
    int countBefore = getRegistry().getBeanDefinitionCount();  
    //解析过程入口，这里使用了委派模式，BeanDefinitionDocumentReader只是个接口，
    //具体的解析实现过程有实现类DefaultBeanDefinitionDocumentReader完成  
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));  
    //统计解析的Bean数量  
    return getRegistry().getBeanDefinitionCount() - countBefore;  
} 

//创建BeanDefinitionDocumentReader对象，解析Document对象  
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {  
    return BeanDefinitionDocumentReader.class.cast(BeanUtils.instantiateClass(this.documentReaderClass));  
}
```

Bean定义资源的载入解析分为以下两个过程：

+ 首先，通过调用XML解析器将Bean定义资源文件转换得到Document对象，但是这些Document对象并没有按照Spring的Bean规则进行解析。这一步是载入的过程

+ 其次，在完成通用的XML解析之后，按照Spring的Bean规则对Document对象进行解析。

按照Spring的Bean规则对Document对象解析的过程是在接口`BeanDefinitionDocumentReader`的实现类`DefaultBeanDefinitionDocumentReader`中实现的。

##### 10. DefaultBeanDefinitionDocumentReader对Bean定义的Document对象解析

`BeanDefinitionDocumentReader`接口通过`registerBeanDefinitions`方法调用其实现类`DefaultBeanDefinitionDocumentReader`对Document对象进行解析

整个源码就是XML解析的过程 不放了

通过上述Spring IoC容器对载入的Bean定义Document解析可以看出，我们使用Spring时，在Spring配置文件中可以使用`<Import>`元素来导入IoC容器所需要的其他资源，Spring IoC容器在解析时会首先将指定导入的资源加载进容器中。使用`<Ailas>`别名时，Spring IoC容器首先将别名元素所定义的别名注册到容器中。

对于既不是`<Import>`元素，又不是`<Alias>`元素的元素，即Spring配置文件中普通的`<Bean>`元素的解析由`BeanDefinitionParserDelegate`类的`parseBeanDefinitionElement`方法来实现。

##### 11. BeanDefinitionParserDelegate解析Bean定义资源文件中的`<Bean>`元素

Bean定义资源文件中的`<Import>`和`<Alias>`元素解析在`DefaultBeanDefinitionDocumentReader`中已经完成，对Bean定义资源文件中使用最多的`<Bean>`元素交由`BeanDefinitionParserDelegate`来解析

同样还是XML解析

只要使用过Spring，对Spring配置文件比较熟悉的人，通过对上述源码的分析，就会明白我们在Spring配置文件中`<Bean>`元素的中配置的属性就是通过该方法解析和设置到Bean中去的。

注意：在解析`<Bean>`元素过程中没有创建和实例化Bean对象，只是创建了Bean对象的定义类BeanDefinition，将`<Bean>`元素中的配置信息设置到BeanDefinition中作为记录，当依赖注入时才使用这些记录信息创建和实例化具体的Bean对象。

上面方法中一些对一些配置如元信息(meta)、qualifier等的解析，我们在Spring中配置时使用的也不多，我们在使用Spring的`<Bean>`元素时，配置最多的是`<property>`属性.

`<Bean>`元素中`<property>`元素的相关配置是如何处理的：

+ ref被封装为指向依赖对象一个引用。
+ value配置都会封装成一个字符串类型的对象。
+ ref和value都通过“解析的数据类型`属性值.setSource(extractSource(ele))`方法将属性值/引用与所引用的属性关联起来。

在方法的最后对于`<property>`元素的子元素通过`parsePropertySubElement` 方法解析.



**经过对Spring Bean定义资源文件转换的Document对象中的元素层层解析，Spring IoC现在已经将XML形式定义的Bean定义资源文件转换为Spring IoC所识别的数据结构——BeanDefinition，它是Bean定义资源文件中配置的POJO对象在Spring IoC容器中的映射，我们可以通过AbstractBeanDefinition为入口，用IoC容器进行索引、查询和操作。**

**通过Spring IoC容器对Bean定义资源的解析后，IoC容器大致完成了管理Bean对象的准备工作，即初始化过程，但是最为重要的依赖注入还没有发生，现在在IoC容器中BeanDefinition存储的只是一些静态信息，接下来需要向容器注册Bean定义信息才能全部完成IoC容器的初始化过程**



##### 12. 解析过后的BeanDefinition在IoC容器中的注册

让我们继续跟踪程序的执行顺序，接下来会到我们第3步中分析`DefaultBeanDefinitionDocumentReader`对Bean定义转换的Document对象解析的流程中，在其`parseDefaultElement`方法中完成对Document对象的解析后得到封装`BeanDefinition`的`BeanDefinitionHold`对象，然后调用`BeanDefinitionReaderUtils`的`registerBeanDefinition`方法向IoC容器注册解析的Bean，BeanDefinitionReaderUtils的注册的源码如下

```java
//将解析的BeanDefinitionHold注册到容器中 
public static void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) throws BeanDefinitionStoreException {  
    //获取解析的BeanDefinition的名称
     String beanName = definitionHolder.getBeanName();  
    //向IoC容器注册BeanDefinition 
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());  
    //如果解析的BeanDefinition有别名，向容器为其注册别名  
     String[] aliases = definitionHolder.getAliases();  
    if (aliases != null) {  
        for (String aliase : aliases) {  
            registry.registerAlias(beanName, aliase);  
        }  
    }  
}
```

当调用`BeanDefinitionReaderUtils`向IoC容器注册解析的`BeanDefinition`时，真正完成注册功能的是`DefaultListableBeanFactory`。

##### 13. DefaultListableBeanFactory向IoC容器注册解析后的BeanDefinition

DefaultListableBeanFactory中使用一个HashMap的集合对象存放IoC容器中注册解析的BeanDefinition，向IoC容器注册的主要源码如下：

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172308561595278.x-png)

```java
//存储注册的俄BeanDefinition  
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();

//向IoC容器注册解析的BeanDefiniton  
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)  
	throws BeanDefinitionStoreException {  
    Assert.hasText(beanName, "Bean name must not be empty");  
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");  
  	//校验解析的BeanDefiniton  
   	if (beanDefinition instanceof AbstractBeanDefinition) {  
        try {  
            ((AbstractBeanDefinition) beanDefinition).validate();  
        }  
        catch (BeanDefinitionValidationException ex) {  
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,  
                    "Validation of bean definition failed", ex);  
        }  
    }  
    //注册的过程中需要线程同步，以保证数据的一致性  
    synchronized (this.beanDefinitionMap) {  
        Object oldBeanDefinition = this.beanDefinitionMap.get(beanName);  
        //检查是否有同名的BeanDefinition已经在IoC容器中注册，如果已经注册，  
        //并且不允许覆盖已注册的Bean，则抛出注册失败异常  
        if (oldBeanDefinition != null) {  
            if (!this.allowBeanDefinitionOverriding) {  
               throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,  
                        "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +  
                        "': There is already [" + oldBeanDefinition + "] bound.");  
            }  
            else {//如果允许覆盖，则同名的Bean，后注册的覆盖先注册的  
                if (this.logger.isInfoEnabled()) {  
                    this.logger.info("Overriding bean definition for bean '" + beanName +  
                            "': replacing [" + oldBeanDefinition + "] with [" + beanDefinition + "]");  
                }  
            }  
        }  
        //IoC容器中没有已经注册同名的Bean，按正常注册流程注册  
        else {  
            this.beanDefinitionNames.add(beanName);  
            this.frozenBeanDefinitionNames = null;  
        }  
        this.beanDefinitionMap.put(beanName, beanDefinition);  
        //重置所有已经注册过的BeanDefinition的缓存  
        resetBeanDefinition(beanName);  
    }  
}
```



**至此，Bean定义资源文件中配置的Bean被解析过后，已经注册到IoC容器中，被容器管理起来，真正完成了IoC容器初始化所做的全部工作。现 在IoC容器中已经建立了整个Bean的配置信息，这些BeanDefinition信息已经可以使用，并且可以被检索，IoC容器的作用就是对这些注册的Bean定义信息进行处理和维护。这些的注册的Bean定义信息是IoC容器控制反转的基础，正是有了这些注册的数据，容器才可以进行依赖注入。**



#### IoC初始化流程总结

+ 初始化的入口在容器实现中的 refresh()调用来完成
+ 对 bean 定义载入 IoC 容器使用的方法是 `loadBeanDefinition`,其中的大致过程如下：
  + 通过 `ResourceLoader` 来完成资源文件位置的定位，`DefaultResourceLoader` 是默认的实现，同时上下文本身就给出了 `ResourceLoader` 的实现，可以从类路径，文件系统, URL 等方式来定为资源位置。
    + 如果是 `XmlBeanFactory`作为 IOC 容器，那么需要为它指定 bean 定义的资源，也就是说 bean 定义文件时通过抽象成 Resource 来被 IOC 容器处理的，容器通过 BeanDefinitionReader来完成定义信息的解析和 Bean 信息的注册,往往使用的是`XmlBeanDefinitionReader` 来解析 bean 的 xml 定义文件——实际的处理过程是委托给 `BeanDefinitionParserDelegate` 来完成的，从而得到 bean 的定义信息，这些信息在 Spring 中使用 BeanDefinition 对象来表示——这个名字可以让我们想到`loadBeanDefinition`,`RegisterBeanDefinition` 这些相关的方法——他们都是为处理 BeanDefinitin 服务的。
    + 容器解析得到 `BeanDefinitionIoC` 以后，需要把它在IoC 容器中注册，这由 IoC 实现 `BeanDefinitionRegistry` 接口来实现。注册过程就是在 IoC 容器内部维护的一个`HashMap` 来保存得到的 `BeanDefinition` 的过程。这个 `HashMap` 是 IoC 容器持有 bean 信息的场所，以后对 bean 的操作都是围绕这个`HashMap` 来实现的.

+ 然后我们就可以通过 `BeanFactory` 和 `ApplicationContext` 来享受到 Spring IoC 的服务了,在使用 IoC 容器的时候，我们注意到除了少量粘合代码，绝大多数以正确 IoC 风格编写的应用程序代码完全不用关心如何到达工厂，因为容器将把这些对象与容器管理的其他对象钩在一起。基本的策略是把工厂放到已知的地方，最好是放在对预期使用的上下文有意义的地方，以及代码将实际需要访问工厂的地方。 Spring 本身提供了对声明式载入 web 应用程序用法的应用程序上下文,并将其存储在ServletContext 中的框架实现。

**在使用 Spring IOC 容器的时候我们还需要区别两个概念:**

+ `Beanfactory` 和 `Factory bean`

  其中 `BeanFactory` 指的是 IoC 容器的编程抽象，比如 `ApplicationContext`， `XmlBeanFactory` 等，这些都是 IOC 容器的具体表现，需要使用什么样的容器由客户决定,但 Spring 为我们提供了丰富的选择。 

  FactoryBean 只是一个可以在 IoC而容器中被管理的一个 bean,是对各种处理过程和资源使用的抽象,Factory bean 在需要时产生另一个对象，而不返回 FactoryBean本身,我们可以把它看成是一个抽象工厂，对它的调用返回的是工厂生产的产品。所有的 Factory bean 都实现特殊的`org.springframework.beans.factory.FactoryBean` 接口，当使用容器中 factory bean 的时候，该容器不会返回 factory bean 本身,而是返回其生成的对象。

  Spring 包括了大部分的通用资源和服务访问抽象的 Factory bean 的实现，其中包括:对 JNDI 查询的处理，对代理对象的处理，对事务性代理的处理，对 RMI 代理的处理等，这些我们都可以看成是具体的工厂,看成是Spring 为我们建立好的工厂。也就是说 Spring 通过使用抽象工厂模式为我们准备了一系列工厂来生产一些特定的对象,免除我们手工重复的工作，我们要使用时只需要在 IOC 容器里配置好就能很方便的使用了



### 补充知识：Java动态代理与CGLIB代理

#### Java动态代理

jdk为代理提供了两个类

```java
java.lang.reflect.Proxy
java.lang.reflect.InvocationHandler
```

jdk自带的代理使用上面有个限制，**只能为接口创建代理类**，如果需要给具体的类创建代理类，需要用后面要说的cglib

##### 一些方法

这是jdk动态代理中主要的一个类，里面有一些静态方法会经常用到

+ getProxyClass

  ```java
  // 为指定的接口创建代理类，返回代理类的Class对象
  public static Class<?> getProxyClass(ClassLoader loader, Class<?>... interfaces)
  ```

  loader：定义代理类的类加载器

  interfaces：指定需要实现的接口列表，创建的代理默认会按顺序实现interfaces指定的接口

+ newProxyInstance

  ```java
  // 创建代理类的实例对象
  public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
  ```

  这个方法先为指定的接口创建代理类，然后会生成代理类的一个实例，最后一个参数比较特殊，是InvocationHandler类型的，这个是个接口

  ```java
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
  ```

  上面方法会返回一个代理对象，当调用代理对象的任何方法的时候，会就被`InvocationHandler`接口的`invoke`方法处理。

+ isProxy

  ```java
  // 判断指定的类是否是一个代理类
  public static boolean isProxyClass(Class<?> cl)
  ```

+ getInvocationHandler

  ```java
  // 获取代理对象的InvocationHandler对象
  public static InvocationHandler getInvocationHandler(Object proxy) throws IllegalArgumentException
  ```

##### 创建JDK代理的两种方式

###### 方法1

1.调用Proxy.getProxyClass方法获取代理类的Class对象
2.使用InvocationHandler接口创建代理类的处理器
3.通过代理类和InvocationHandler创建代理对象
4.上面已经创建好代理对象了，接着我们就可以使用代理对象了

**原始接口**

```java
public interface IService {
    void m1();
    void m2();
    void m3();
}
```

**代理对象**

```java
@Test
public void m1() throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
    // 1. 获取接口对应的代理类
    Class<IService> proxyClass = (Class<IService>) Proxy.getProxyClass(IService.class.getClassLoader(), IService.class);
    // 2. 创建代理类的处理器
    InvocationHandler invocationHandler = new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("我是InvocationHandler，被调用的方法是：" + method.getName());
            return null;
        }
    };
    // 3. 创建代理实例
    IService proxyService = proxyClass.getConstructor(InvocationHandler.class).newInstance(invocationHandler);
    // 4. 调用代理的方法
    proxyService.m1();
    proxyService.m2();
    proxyService.m3();
}
```

**运行输出**

```
我是InvocationHandler，被调用的方法是：m1
我是InvocationHandler，被调用的方法是：m2
我是InvocationHandler，被调用的方法是：m3
```

印证了

> 上面方法会返回一个代理对象，当调用代理对象的任何方法的时候，会就被`InvocationHandler`接口的`invoke`方法处理。



###### 方法2

1.使用InvocationHandler接口创建代理类的处理器
2.使用Proxy类的静态方法newProxyInstance直接创建代理对象
3.使用代理对象

**代理对象**

```java
@Test
public void m2() throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
    // 1. 创建代理类的处理器
    InvocationHandler invocationHandler = new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("我是InvocationHandler，被调用的方法是：" + method.getName());
            return null;
        }
    };
    // 2. 创建代理实例 跳过了方法1中的ProxyClass 直接使用静态方法创建
    IService proxyService = (IService) Proxy.newProxyInstance(IService.class.getClassLoader(), new Class[]{IService.class}, invocationHandler);
    // 3. 调用代理的方法
    proxyService.m1();
    proxyService.m2();
    proxyService.m3();
}
```

**运行输出**

```
我是InvocationHandler，被调用的方法是：m1
我是InvocationHandler，被调用的方法是：m2
我是InvocationHandler，被调用的方法是：m3
```

##### 使用请注意

1. jdk中的Proxy只能为接口生成代理类，如果你想给某个类创建代理类，那么Proxy是无能为力的，此时需要我们用到下面要说的cglib了。
2. Proxy类中提供的几个常用的静态方法大家需要掌握
3. 通过Proxy创建代理对象，当调用代理对象任意方法时候，会被InvocationHandler接口中的invoke方法进行处理，这个接口内容是关键

#### cglib代理

jdk动态代理只能为接口创建代理，使用上有局限性。实际的场景中我们的类不一定有接口，此时如果我们**想为普通的类也实现代理功能**，我们就需要用到cglib来实现了。

cglib是一个强大、高性能的字节码生成库，它用于在运行时**扩展Java类和实现接口**；本质上它是通过动态的生成一个子类去覆盖所要代理的类（非final修饰的类和方法）。Enhancer可能是CGLIB中最常用的一个类，和jdk中的Proxy不同的是，Enhancer既能够代理普通的class，也能够代理接口。Enhancer**创建一个被代理对象的子类，并且拦截所有的方法调用**（包括从Object中继承的toString和hashCode方法）。**Enhancer不能够拦截final方法**，例如Object.getClass()方法，这是由于Java final方法语义决定的。基于同样的道理，Enhancer**也不能对final类进行代理操作**。

CGLIB作为一个开源项目，其代码托管在github，地址为：https://github.com/cglib/cglib

##### cglib组成结构

![图片](https://cdn.jsdelivr.net/gh/WangMinan/Pics/640)

CGLIB底层使用了ASM（一个短小精悍的字节码操作框架）来操作字节码生成新的类。除了CGLIB库外，脚本语言（如Groovy和BeanShell）也使用ASM生成字节码。ASM使用类似SAX的解析器来实现高性能。我们不鼓励直接使用ASM，因为它需要对Java字节码的格式足够的了解。

Spring已将第三方cglib jar包中所有的类集成到Spring自己的jar包中，本系列内容都是和Spring相关的，为了方便，我们直接使用spring内部已集成的来讲解。

##### 5个案例

###### 1. 拦截所有方法（MethodInterceptor）

创建一个具体的类

```java
public class Service1 {
    public void m1() {
        System.out.println("我是m1方法");
    }

    public void m2() {
        System.out.println("我是m2方法");
    }
}
```

为其创建代理并打印日志

```java
public class CglibTest {

    @Test
    public void test1() {
        //使用Enhancer来给某个类创建代理类，步骤
        //1.创建Enhancer对象
        Enhancer enhancer = new Enhancer();
        //2.通过setSuperclass来设置父类型，即需要给哪个类创建代理类
        enhancer.setSuperclass(Service1.class);
        /*3.设置回调，需实现org.springframework.cglib.proxy.Callback接口，
        此处我们使用的是org.springframework.cglib.proxy.MethodInterceptor，也是一个接口，实现了Callback接口，
        当调用代理对象的任何方法的时候，都会被MethodInterceptor接口的invoke方法处理*/
        enhancer.setCallback(new MethodInterceptor() {
            /**
             * 代理对象方法拦截器
             * @param o 代理对象
             * @param method 被代理的类的方法，即Service1中的方法
             * @param objects 调用方法传递的参数
             * @param methodProxy 方法代理对象
             * @return
             * @throws Throwable
             */
            @Override
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) 					throws Throwable {
                System.out.println("调用方法:" + method);
                //可以调用MethodProxy的invokeSuper调用被代理类的方法
                Object result = methodProxy.invokeSuper(o, objects);
                return result;
            }
        });
        //4.获取代理对象,调用enhancer.create方法获取代理对象，这个方法返回的是Object类型的，所以需要强转一下
        Service1 proxy = (Service1) enhancer.create();
        //5.调用代理对象的方法
        proxy.m1();
        proxy.m2();
    }
}
```

上面代码中的注释很详细，列出了给指定的类创建代理的具体步骤，整个过程中主要用到了Enhancer类和`MethodInterceptor`接口。

`enhancer.setSuperclass`用来设置代理类的父类，即需要给哪个类创建代理类，此处是Service1

`enhancer.setCallback`传递的是`MethodInterceptor`接口类型的参数，`MethodInterceptor`接口有个`intercept`方法，这个方法会拦截代理对象所有的方法调用。

还有一个重点是`Object result = methodProxy.invokeSuper(o, objects);`可以调用被代理类，也就是Service1类中的具体的方法，从方法名称的意思可以看出是调用父类，实际对某个类创建代理，cglib底层通过修改字节码的方式为Service1类创建了一个子类。

运行输出

```
调用方法:public void com.javacode2018.lesson001.demo17.Service1.m1()
我是m1方法
调用方法:public void com.javacode2018.lesson001.demo17.Service1.m2()
我是m2方法
```



###### 2. 拦截所有方法(MethodInterceptor)

使用一个与上一题类似的类

```java
public class Service2 {
    public void m1() {
        System.out.println("我是m1方法");
        this.m2(); //@1 方法m1调用了m2
    }

    public void m2() {
        System.out.println("我是m2方法");
    }
}
```

为其创建代理

```java
@Test
public void test2() {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(Service2.class);
    enhancer.setCallback(new MethodInterceptor() {
        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            System.out.println("调用方法:" + method);
            Object result = methodProxy.invokeSuper(o, objects);
            return result;
        }
    });
    Service2 proxy = (Service2) enhancer.create();
    proxy.m1(); //@1
}
```

只调用了m1方法，但输出结果包含了m1和m2的执行

```
调用方法:public void com.javacode2018.lesson001.demo17.Service2.m1()
我是m1方法
调用方法:public void com.javacode2018.lesson001.demo17.Service2.m2()
我是m2方法
```

从输出中可以看出m1和m2方法都被拦截器处理了，而m2方法是在Service1的m1方法中调用的，也被拦截处理了。



###### 3. 拦截所有方法并返回固定值(FixedValue)

当调用某个类的任何方法的时候，都希望返回一个固定的值，此时可以使用`FixedValue`接口，如下：

```java
enhancer.setCallback(new FixedValue() {
    @Override
    public Object loadObject() throws Exception {
        return "路人甲";
    }
});
```

上面创建的代理对象，调用其任意方法返回的都是"路人甲"。

还是使用之前的service进行代理操作

```java
public class Service3 {
    public String m1() {
        System.out.println("我是m1方法");
        return "hello:m1";
    }

    public String m2() {
        System.out.println("我是m2方法");
        return "hello:m2";
    }
}
```

对其添加代理的代码如下

```java
@Test
public void test3() {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(Service3.class);
    enhancer.setCallback(new FixedValue() {
        @Override
        public Object loadObject() throws Exception {
            return "路人甲";
        }
    });
    Service3 proxy = (Service3) enhancer.create();
    System.out.println(proxy.m1());//@1
    System.out.println(proxy.m2()); //@2
    System.out.println(proxy.toString());//@3
}
```

得到的结果如下

```
路人甲
路人甲
路人甲
```

输出部分均被拦截



###### 4. 直接放行，不做任何操作(NoOp.INSTANCE)

对Service3进行操作

```java
@Test
public void test6() {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(Service3.class);
    enhancer.setCallback(NoOp.INSTANCE);
    Service3 proxy = (Service3) enhancer.create();
    System.out.println(proxy.m1());
    System.out.println(proxy.m2());
}
```

`Callback`接口下面有个子接口`org.springframework.cglib.proxy.NoOp`，将这个作为Callback的时候，被调用的方法会直接放行，像没有任何代理一样。

运行输出

```
我是m1方法
hello:m1
我是m2方法
hello:m2
```



###### 5. 不同的方法使用不同的拦截器(CallbackFilter)

创建一个新的Service

```java
public class Service4 {
    public void insert1() {
        System.out.println("我是insert1");
    }

    public void insert2() {
        System.out.println("我是insert2");
    }

    public String get1() {
        System.out.println("我是get1");
        return "get1";
    }

    public String get2() {
        System.out.println("我是get2");
        return "get2";
    }
}
```

**需求，给这个类创建一个代理需要实现下面的功能：**

1. 以insert开头的方法需要统计方法耗时
2. 以get开头的的方法直接返回固定字符串欢迎和【路人甲java】一起学spring！

下来看代码，然后再解释：

```java
@Test
public void test4() {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(Service4.class);
    //创建2个Callback
    Callback[] callbacks = {
            //这个用来拦截所有insert开头的方法
            new MethodInterceptor() {
                @Override
                public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) 					throws Throwable {
                    long starTime = System.nanoTime();
                    Object result = methodProxy.invokeSuper(o, objects);
                    long endTime = System.nanoTime();
                    System.out.println(method + "，耗时(纳秒):" + (endTime - starTime));
                    return result;
                }
            },
            //下面这个用来拦截所有get开头的方法，返回固定值的
            new FixedValue() {
                @Override
                public Object loadObject() throws Exception {
                    return "路人甲Java";
                }
            }
    };
    enhancer.setCallbackFilter(new CallbackFilter() {
        @Override
        public int accept(Method method) {
            return 0;
        }
    });
    //调用enhancer的setCallbacks传递Callback数组
    enhancer.setCallbacks(callbacks);
    /**
     * 设置过滤器CallbackFilter
     * CallbackFilter用来判断调用方法的时候使用callbacks数组中的哪个Callback来处理当前方法
     * 返回的是callbacks数组的下标
     */
    enhancer.setCallbackFilter(new CallbackFilter() {
        @Override
        public int accept(Method method) {
            //获取当前调用的方法的名称
            String methodName = method.getName();
            /*
             * 方法名称以insert开头，
             * 返回callbacks中的第1个Callback对象来处理当前方法，
             * 否则使用第二个Callback处理被调用的方法
             */
            return methodName.startsWith("insert") ? 0 : 1;
        }
    });
    Service4 proxy = (Service4) enhancer.create();
    System.out.println("---------------");
    proxy.insert1();
    System.out.println("---------------");
    proxy.insert2();
    System.out.println("---------------");
    System.out.println(proxy.get1());
    System.out.println("---------------");
    System.out.println(proxy.get2());
}
```

输出情况

```
---------------
我是insert1
public void com.javacode2018.lesson001.demo17.Service4.insert1()，耗时(纳秒):15396100
---------------
我是insert2
public void com.javacode2018.lesson001.demo17.Service4.insert2()，耗时(纳秒):66200
---------------
路人甲Java
---------------
路人甲Java
```

由于需求中要对不同的方法做不同的处理，所以需要有2个Callback对象，当调用代理对象的方法的时候，具体会走哪个Callback呢，此时会通过`CallbackFilter`中的`accept`来判断，这个方法返回`callbacks数组的索引`。

可以对拦截方法进行优化，代码如下

```java
@Test
public void test5() {
    Enhancer enhancer = new Enhancer();
    //创建2个Callback
    //这个用来拦截所有insert开头的方法
    Callback costTimeCallback = (MethodInterceptor) (Object o, Method method, Object[] objects, MethodProxy methodProxy) -> {
        long starTime = System.nanoTime();
        Object result = methodProxy.invokeSuper(o, objects);
        long endTime = System.nanoTime();
        System.out.println(method + "，耗时(纳秒):" + (endTime - starTime));
        return result;
    };
    //下面这个用来拦截所有get开头的方法，返回固定值的
    Callback fixdValueCallback = (FixedValue) () -> "路人甲Java";
    CallbackHelper callbackHelper = new CallbackHelper(Service4.class, null) {
        @Override
        protected Object getCallback(Method method) {
            // 判断由哪一个Callback进行拦截
            return method.getName().startsWith("insert") ? costTimeCallback : fixdValueCallback;
        }
    };
    enhancer.setSuperclass(Service4.class);
    //调用enhancer的setCallbacks传递Callback数组
    enhancer.setCallbacks(callbackHelper.getCallbacks());
    /*
     * 设置CallbackFilter,用来判断某个方法具体走哪个Callback
     */
    enhancer.setCallbackFilter(callbackHelper);
    Service4 proxy = (Service4) enhancer.create();
    System.out.println("---------------");
    proxy.insert1();
    System.out.println("---------------");
    proxy.insert2();
    System.out.println("---------------");
    System.out.println(proxy.get1());
    System.out.println("---------------");
    System.out.println(proxy.get2());
}
```



#### Java动态代理和CGLIB的区别

+ Java动态代理只能够对接口进行代理，不能对普通的类进行代理（因为所有生成的代理类的父类为Proxy，Java类继承机制不允许多重继承）；CGLIB能够代理普通类；

+ Java动态代理使用Java原生的反射API进行操作，在生成类上比较高效；CGLIB使用ASM框架直接对字节码进行操作，在类的执行过程中比较高效



### IoC容器的依赖注入(DI)

DI——Dependency Injection，即**依赖注入**。

是组件之间依赖关系由容器在运行期决定，形象的说，即由容器动态地将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统

带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台。通过依赖注入机制，我们只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现。

**IoC 和 DI 有什么关系呢？**
其实它们是同一个概念的不同角度描述，由于控制反转概念比较含糊（可能只是理解为容器控制对象这一个层面，很难让人想到谁来维护对象关系），所以2004年大师级人物Martin Fowler又给出了一个新的名字：“依赖注入”。相对IoC 而言，**依赖注入**明确描述了被注入对象依赖IoC容器配置依赖对象。

#### 依赖注入发生的时间

当Spring IoC容器完成了Bean定义资源的定位、载入和解析注册以后，IoC容器中已经管理类Bean定义的相关数据，但是此时IoC容器还没有对所管理的Bean进行依赖注入，依赖注入在以下两种情况发生：

+ 用户**第一次通过getBean方法向IoC容索要Bean时**，IoC容器触发依赖注入。

+ 当用户在Bean定义资源中为`<Bean>`元素配置了`lazy-init`属性，即让容器**在解析注册Bean定义时进行预实例化**，触发依赖注入。

BeanFactory接口定义了Spring IoC容器的基本功能规范，是Spring IoC容器所应遵守的最底层和最基本的编程规范。BeanFactory接口中定义了几个getBean方法，就是用户向IoC容器索取管理的Bean的方法，我们通过分析其子类的具体实现，理解Spring IoC容器在用户索取Bean时如何完成依赖注入。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/172312283627249.x-png)

在BeanFactory中我们看到getBean（String…）函数，它的具体实现在AbstractBeanFactory中

#### AbstractBeanFactory通过getBean向IoC容器获取被管理的Bean

通过对向IoC容器获取Bean方法的分析，我们可以看到在Spring中

+ 如果Bean定义的单态模式(Singleton)，则容器在创建之前先从缓存中查找，以确保整个容器中只存在一个实例对象。
+ 如果Bean定义的是原型模式(Prototype)，则容器每次都会创建一个新的实例对象。

除此之外，Bean定义还可以扩展为指定其生命周期范围。

上面的源码只是定义了根据Bean定义的模式，采取的不同创建Bean实例对象的策略，具体的Bean实例对象的创建过程由实现了`ObejctFactory`接口的匿名内部类的createBean方法完成，ObejctFactory使用委派模式，具体的Bean实例创建过程交由其实现类AbstractAutowireCapableBeanFactory完成，我们继续分析AbstractAutowireCapableBeanFactory的createBean方法的源码，理解其创建Bean实例的具体实现过程。

通过对方法源码的分析，我们看到**具体的依赖注入实现在以下两个过程中**：

+ createBeanInstance：**生成Bean所包含的java对象实例**。

  在createBeanInstance方法中，根据指定的初始化策略，使用静态工厂、工厂方法或者容器的自动装配特性生成java实例对象。

  对使用工厂方法和自动装配特性的Bean的实例化相当比较清楚，**调用相应的工厂方法或者参数匹配的构造方法即可完成实例化对象的工作**，但是对于我们最常使用的**默认无参构造方法就需要使用相应的初始化策略(JDK的反射机制或者CGLIB)来进行初始化了**，在方法`getInstantiationStrategy().instantiate`中就具体实现类使用初始策略实例化对象。

  + 在使用默认的无参构造方法创建Bean的实例化对象时，方法`getInstantiationStrategy().instantiate`调用了`SimpleInstantiationStrategy`类中的实例化Bean的方法.

    我们看到了如果Bean有方法被覆盖了，则使用JDK的反射机制进行实例化，否则，使用CGLIB进行实例化。

    `instantiateWithMethodInjection`方法调用`SimpleInstantiationStrategy`的子类`CglibSubclassingInstantiationStrategy`使用CGLIB来进行初始化

    CGLIB是一个常用的字节码生成器的类库，它提供了一系列API实现java字节码的生成和转换功能。我们在学习JDK的动态代理时都知道，JDK的动态代理只能针对接口，如果一个类没有实现任何接口，要对其进行动态代理只能使用CGLIB。

+ populateBean ：**对Bean属性的依赖注入进行处理**。

  分析上述代码，我们可以看出，对属性的注入过程分以下两种情况：

  + 属性值类型**不需要转换**时，不需要解析属性值，直接准备进行依赖注入。

  + 属性值**需要进行类型转换时**，如对其他对象的引用等，首先需要解析属性值，然后对解析后的属性值进行依赖注入。

  对属性值的解析是在`BeanDefinitionValueResolver`类中的`resolveValueIfNecessary`方法中进行的，对属性值的依赖注入是通过`bw.setPropertyValue`s方法实现的，在分析属性值的依赖注入之前，我们先分析一下对属性值的解析过程。

  **属性值的解析**

  当容器在对属性进行依赖注入时，如果发现属性值需要进行类型转换，如属性值是容器中另一个Bean实例对象的引用，则容器首先**需要根据属性值解析出所引用的对象，然后才能将该引用对象注入到目标实例对象的属性上去**，对属性进行解析的由`resolveValueIfNecessary`方法实现

  **注入**

  属性值解析完成后就可以进行依赖注入了，依赖注入的过程就是Bean对象实例设置到它所依赖的Bean对象属性上去，在第7步中我们已经说过，依赖注入是通过`bw.setPropertyValues`方法实现的，该方法也使用了委托模式，在`BeanWrapper`接口中至少定义了方法声明，依赖注入的具体实现交由其实现类`BeanWrapperImpl`来完成.

  Spring IoC容器通过`BeanWrapperImpl`将属性值注入到Bean实例的过程如下：

  + 对于**集合类型的属性**，将其属性值解析为目标类型的集合后直接赋值给属性。

  + 对于**非集合类型的属性**，大量使用了**JDK的反射和内省机制**，通过属性的getter方法(reader method)获取指定属性注入以前的值，同时调用属性的setter方法(writer method)为属性设置注入后的值。看到这里相信很多人都明白了Spring的setter注入原理。

  

**至此Spring IoC容器对Bean定义资源文件的定位，载入、解析和依赖注入已经全部分析完毕，现在Spring IoC容器中管理了一系列靠依赖关系联系起来的Bean，程序不需要应用自己手动创建所需的对象，Spring IoC容器会在我们使用的时候自动为我们创建，并且为我们注入好相关的依赖，这就是Spring核心功能的控制反转和依赖注入的相关功能。**



### IoC容器的高级特性

Spring IoC容器还有一些高级特性，如使用`lazy-init`属性对Bean预初始化、FactoryBean产生或者修饰Bean对象的生成、IoC容器初始化Bean过程中使用`BeanPostProcessor`后置处理器对Bean声明周期事件管理和IoC容器的autowiring自动装配功能等。

#### Spring IoC容器的lazy-init属性实现预实例化

通过前面我们对IoC容器的实现和工作原理分析，我们知道IoC容器的初始化过程就是对Bean定义资源的定位、载入和注册，此时容器对Bean的依赖注入并没有发生，**依赖注入主要是在应用程序第一次向容器索取Bean时，通过getBean方法的调用完成**。

**当Bean定义资源的`<Bean>`元素中配置了lazy-init属性时，容器将会在初始化的时候对所配置的Bean进行预实例化，Bean的依赖注入在容器初始化的时候就已经完成。**这样，当应用程序第一次向容器索取被管理的Bean时，就不用再初始化和对Bean进行依赖注入了，直接从容器中获取已经完成依赖注入的现成Bean，可以提高应用第一次向容器获取Bean的性能。

##### 1. refresh()

先从IoC容器的初始会过程开始，通过前面文章分析，我们知道IoC容器读入已经定位的Bean定义资源是从refresh方法开始的.refresh的源码已经给出了。

[点击查看源码](#jump)

在refresh方法中`ConfigurableListableBeanFactorybeanFactory = obtainFreshBeanFactory();`启动了Bean定义资源的载入、注册过程，而`finishBeanFactoryInitialization`方法是对注册后的Bean定义中的预实例化(lazy-init=false，Spring默认就是预实例化，即为true)的Bean进行处理的地方。



##### 2. finishBeanFactoryInitialization处理预实例化Bean

当Bean定义资源被载入IoC容器之后，容器将Bean定义资源解析为容器内部的数据结构BeanDefinition注册到容器中，`AbstractApplicationContext类中的finishBeanFactoryInitialization`方法对配置了预实例化属性的Bean进行预初始化过程

```java
//对配置了lazy-init属性的Bean进行预实例化处理  
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {  
    //这是Spring3以后新加的代码，为容器指定一个转换服务(ConversionService)  
    //在对某些Bean属性进行转换时使用  
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&  
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {  
        beanFactory.setConversionService(  
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));  
    }  
    //为了类型匹配，停止使用临时的类加载器  
    beanFactory.setTempClassLoader(null);  
    //缓存容器中所有注册的BeanDefinition元数据，以防被修改  
    beanFactory.freezeConfiguration();  
    //对配置了lazy-init属性的单态模式Bean进行预实例化处理  
    beanFactory.preInstantiateSingletons();  
}
```

`ConfigurableListableBeanFactory`是一个接口，其`preInstantiateSingletons`方法由其子类`DefaultListableBeanFactory`提供。



##### 3. DefaultListableBeanFactory对配置lazy-init属性单态Bean的预实例化

通过对lazy-init处理源码的分析，我们可以看出，如果设置了lazy-init属性，则容器在完成Bean定义的注册之后，**会通过getBean方法，触发对指定Bean的初始化和依赖注入过程**，这样当应用第一次向容器索取所需的Bean时，容器不再需要对Bean进行初始化和依赖注入，直接从已经完成实例化和依赖注入的Bean中取一个线程的Bean，这样就提高了第一次获取Bean的性能。



#### FactoryBean的实现

在Spring中，有两个很容易混淆的类：**BeanFactory**和**FactoryBean**。

+ **BeanFactory**：**Bean工厂，是一个工厂(Factory)**，我们Spring IoC容器的最顶层接口就是这个BeanFactory，它的作用是管理Bean，即实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。

+ **FactoryBean**：**工厂Bean，是一个Bean**，**作用是产生其他bean实例**。通常情况下，这种bean没有什么特别的要求，仅需要提供一个工厂方法，该方法用来返回其他bean实例。通常情况下，bean无须自己实现工厂模式，Spring容器担任工厂角色；但少数情况下，容器中的bean本身就是工厂，其作用是产生其它bean实例。

当用户使用容器本身时，可以使用转义字符”&”来得到FactoryBean本身，以区别通过FactoryBean产生的实例对象和FactoryBean对象本身。在BeanFactory中通过如下代码定义了该转义字符：

```java
String FACTORY_BEAN_PREFIX = "&";
```

如果`myJndiObject`是一个`FactoryBean`，则使用`&myJndiObject`得到的是myJndiObject**对象**，而**不是**myJndiObject**产生出来的对象**。



##### 1. FactoryBean的源码如下

```java
//工厂Bean，用于产生其他对象  
public interface FactoryBean<T> {  
   	//获取容器管理的对象实例  
    T getObject() throws Exception;  
    //获取Bean工厂创建的对象的类型  
    Class<?> getObjectType();  
    //Bean工厂创建的对象是否是单态模式，如果是单态模式，则整个容器中只有一个实例  
   	//对象，每次请求都返回同一个实例对象  
    boolean isSingleton();  
}
```



##### 2. AbstractBeanFactory的getBean方法调用FactoryBean

在前面我们分析Spring Ioc容器实例化Bean并进行依赖注入过程的源码时，提到在getBean方法触发容器实例化Bean的时候会调用`AbstractBeanFactory`的doGetBean方法来进行实例化的过程.

在上面获取给定Bean的实例对象的getObjectForBeanInstance方法中，会调用FactoryBeanRegistrySupport类的getObjectFromFactoryBean方法，该方法实现了Bean工厂生产Bean实例对象。

Dereference(解引用)：一个在C/C++中应用比较多的术语，在C++中，”*”是解引用符号，而”&”是引用符号，解引用是指变量指向的是所引用对象的本身数据，而不是引用对象的内存地址。



##### 3. AbstractBeanFactory生产Bean实例对象

BeanFactory接口调用其实现类的getObject方法来实现创建Bean实例对象的功能



##### 4. 工厂Bean的实现类getObject方法创建Bean实例对象

FactoryBean的实现类有非常多，比如：Proxy、RMI、JNDI、ServletContextFactoryBean等等，FactoryBean接口为Spring容器提供了一个很好的封装机制，具体的getObject有不同的实现类根据不同的实现策略来具体提供，我们分析一个最简单的AnnotationTestFactoryBean的实现源码

```java
public class AnnotationTestBeanFactory implements FactoryBean<IJmxTestBean> {  
    private final FactoryCreatedAnnotationTestBean instance = new FactoryCreatedAnnotationTestBean();  
    public AnnotationTestBeanFactory() {  
        this.instance.setName("FACTORY");  
    }  
    //AnnotationTestBeanFactory产生Bean实例对象的实现  
    public IJmxTestBean getObject() throws Exception {  
        return this.instance;  
    }  
    public Class<? extends IJmxTestBean> getObjectType() {  
        return FactoryCreatedAnnotationTestBean.class;  
    }  
    public boolean isSingleton() {  
        return true;  
    }  
}
```

其他的Proxy，RMI，JNDI等等，都是根据相应的策略提供getObject的实现。

#### Spring IoC容器autowiring实现原理

Spring IoC容器提供了两种管理Bean依赖关系的方式：

+ 显式管理：通过BeanDefinition的属性值和构造方法实现Bean依赖关系管理。

+ autowiring：Spring IoC容器的依赖**自动装配**功能，不需要对Bean属性的依赖关系做显式的声明，只需要在配置好autowiring属性，IoC容器会自动使用反射查找属性的类型和名称，然后基于属性的类型或者名称来自动匹配容器中管理的Bean，从而自动地完成依赖注入。

通过对autowiring自动装配特性的理解，我们知道容器对Bean的自动装配发生在容器对Bean依赖注入的过程中。在前面对Spring IoC容器的依赖注入过程源码分析中，我们已经知道了容器对Bean实例对象的属性注入的处理发生在`AbstractAutoWireCapableBeanFactory`类中的`populateBean`方法中.



##### 1. AbstractAutoWireCapableBeanFactory对Bean实例进行属性依赖注入

应用第一次通过getBean方法(配置了lazy-init预实例化属性的除外)向IoC容器索取Bean时，容器创建Bean实例对象，并且对Bean实例对象进行属性依赖注入，AbstractAutoWireCapableBeanFactory的populateBean方法就是实现Bean属性依赖注入的功能

```java
protected void populateBean(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw) {  
    //获取Bean定义的属性值，并对属性值进行处理  
    PropertyValues pvs = mbd.getPropertyValues();  
    ……  
    //对依赖注入处理，首先处理autowiring自动装配的依赖注入  
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||  
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {  
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);  
        //根据Bean名称进行autowiring自动装配处理  
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {  
            autowireByName(beanName, mbd, bw, newPvs);  
        }  
        //根据Bean类型进行autowiring自动装配处理  
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {  
            autowireByType(beanName, mbd, bw, newPvs);  
        }  
    }  
    //对非autowiring的属性进行依赖注入处理  
     ……  
}
```

可以从上面的代码中看出主要通过**名称(处理时若未指定Bean名称则默认按照类名首字母小写)和类型**进行自动装配，具体细节如下



##### 2. Spring IoC容器根据Bean名称或者类型进行autowiring自动依赖注入：

```java
//根据名称对属性进行自动依赖注入  
protected void autowireByName(  
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {  
     //对Bean对象中非简单属性(不是简单继承的对象，如8中原始类型，字符串，URL等//都是简单属性)进行处理  
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);  
    for (String propertyName : propertyNames) {  
        //如果Spring IoC容器中包含指定名称的Bean  
        if (containsBean(propertyName)) {  
             //调用getBean方法向IoC容器索取指定名称的Bean实例，迭代触发属性的//初始化和依赖注入  
            Object bean = getBean(propertyName);  
            //为指定名称的属性赋予属性值  
            pvs.add(propertyName, bean);  
            //指定名称属性注册依赖Bean名称，进行属性依赖注入  
            registerDependentBean(propertyName, beanName);  
            if (logger.isDebugEnabled()) {  
                logger.debug("Added autowiring by name from bean name '" + beanName +  
                        "' via property '" + propertyName + "' to bean named '" + propertyName + "'");  
            }  
        }  
        else {  
            if (logger.isTraceEnabled()) {  
                logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +  
                        "' by name: no matching bean found");  
            }  
        }  
    }  
}

//根据类型对属性进行自动依赖注入  
protected void autowireByType(  
    	String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {  
    //获取用户定义的类型转换器  
    TypeConverter converter = getCustomTypeConverter();  
    if (converter == null) {  
        converter = bw;  
    }  
    //存放解析的要注入的属性  
    Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);  
      //对Bean对象中非简单属性(不是简单继承的对象，如8中原始类型，字符  
     //URL等都是简单属性)进行处理  
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);  
    for (String propertyName : propertyNames) {  
        try {  
            //获取指定属性名称的属性描述器  
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);  
            //不对Object类型的属性进行autowiring自动依赖注入  
            if (!Object.class.equals(pd.getPropertyType())) {  
                //获取属性的setter方法  
                MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);  
                //检查指定类型是否可以被转换为目标对象的类型  
                boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());  
                //创建一个要被注入的依赖描述  
                DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);  
                //根据容器的Bean定义解析依赖关系，返回所有要被注入的Bean对象  
                Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);  
               if (autowiredArgument != null) {  
                    //为属性赋值所引用的对象  
                    pvs.add(propertyName, autowiredArgument);  
                }  
                for (String autowiredBeanName : autowiredBeanNames) {  
                    //指定名称属性注册依赖Bean名称，进行属性依赖注入  
                    registerDependentBean(autowiredBeanName, beanName);  
                    if (logger.isDebugEnabled()) {  
                        logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +  
                                propertyName + "' to bean named '" + autowiredBeanName + "'");  
                    }  
                }  
                //释放已自动注入的属性  
                autowiredBeanNames.clear();  
            }  
        }  
        catch (BeansException ex) {  
           throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);  
        }  
    }  
}
```

通过上面的源码分析，我们可以看出来通过属性名进行自动依赖注入的相对比通过属性类型进行自动依赖注入要稍微简单一些，但是真正实现属性注入的是`DefaultSingletonBeanRegistry`类的`registerDependentBean`方法。



##### 3. DefaultSingletonBeanRegistry的registerDependentBean方法对属性注入

```java
//为指定的Bean注入依赖的Bean  
public void registerDependentBean(String beanName, String dependentBeanName) {  
    //处理Bean名称，将别名转换为规范的Bean名称  
    String canonicalName = canonicalName(beanName);  
    //多线程同步，保证容器内数据的一致性  
    //先从容器中：bean名称-->全部依赖Bean名称集合找查找给定名称Bean的依赖Bean  
    synchronized (this.dependentBeanMap) {  
        //获取给定名称Bean的所有依赖Bean名称  
        Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);  
        if (dependentBeans == null) {  
            //为Bean设置依赖Bean信息  
            dependentBeans = new LinkedHashSet<String>(8);  
            this.dependentBeanMap.put(canonicalName, dependentBeans);  
        }  
        //向容器中：bean名称-->全部依赖Bean名称集合添加Bean的依赖信息  
        //即，将Bean所依赖的Bean添加到容器的集合中  
        dependentBeans.add(dependentBeanName);  
    }  
  	//从容器中：bean名称-->指定名称Bean的依赖Bean集合找查找给定名称  
 	//Bean的依赖Bean  
    synchronized (this.dependenciesForBeanMap) {  
        Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(dependentBeanName);  
        if (dependenciesForBean == null) {  
            dependenciesForBean = new LinkedHashSet<String>(8);  
            this.dependenciesForBeanMap.put(dependentBeanName, dependenciesForBean);  
        }  
        //向容器中：bean名称-->指定Bean的依赖Bean名称集合添加Bean的依赖信息  
        //即，将Bean所依赖的Bean添加到容器的集合中  
        dependenciesForBean.add(canonicalName);  
    }  
}
```

通过对autowiring的源码分析，我们可以看出，autowiring的实现过程：

+ 对Bean的属性迭代调用getBean方法，完成依赖Bean的初始化和依赖注入。
+ 将依赖Bean的属性引用设置到被依赖的Bean属性上。
+ 将依赖Bean的名称和被依赖Bean的名称存储在IoC容器的集合中。

Spring IoC容器的autowiring属性自动依赖注入是一个很方便的特性，可以简化开发时的配置，但是凡是都有两面性，自动属性依赖注入也有不足，首先，Bean的依赖关系在配置文件中无法很清楚地看出来，对于维护造成一定困难。其次，由于自动依赖注入是Spring容器自动执行的，容器是不会智能判断的，如果配置不当，将会带来无法预料的后果，所以自动依赖注入特性在使用时还是综合考虑。



## AOP

AOP（Aspect Oriented Programming面向切面编程）

参考了[面试今日头条，被虐的不行了。。。 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MTkxMDQ4MQ==&mid=2648934876&idx=1&sn=7794b50e658e0ec3e0aff6cf5ed4aa2e&chksm=886211e2bf1598f4e0e636170a4b36a5a5edd8811c8b7c30d61135cb114b0ce506a6fa84df0b&token=1672930952&lang=zh_CN&scene=21#wechat_redirect)

更详细的源码解释可以看同一个博主的[AOP 核心源码、原理详解 - china_coding - 博客园 (cnblogs.com)](https://www.cnblogs.com/konglxblog/p/15409195.html)

### 一些概念

+ **目标对象(target)**

  目标对象指将要被增强的对象，即包含主业务逻辑的类对象。

+ **连接点(JoinPoint)**

  程序执行过程中明确的点，如方法的调用或特定的异常被抛出。

  连接点由两个信息确定：

  + 方法(表示程序执行点，即在哪个目标方法)
  + 相对点(表示方位，即目标方法的什么位置，比如调用前，后等)

  简单来说，连接点就是被拦截到的程序执行点，因为Spring只支持方法类型的连接点，所以在Spring中连接点就是被拦截到的方法。

+ **代理对象(Proxy)**

  AOP中会通过代理的方式，对目标对象生成一个代理对象，代理对象中会加入需要增强功能，通过代理对象来间接的方式目标对象，起到增强目标对象的效果。

+ **通知(Advice)**

  需要在目标对象中增强的功能，如上面说的：业务方法前验证用户的功能、方法执行之后打印方法的执行日志。

  通知中有2个重要的信息：**方法的什么地方**，**执行什么操作**，这2个信息通过通知来指定。

  方法的什么地方？之前、之后、包裹目标方法、方法抛出异常后等。

+ **切入点(Pointcut)**

  用来指定需要将通知使用到哪些地方，比如需要用在哪些类的哪些方法上，切入点就是做这个配置的。

+ **切面（Aspect）**

  通知（Advice）和切入点（Pointcut）的组合。切面来定义在哪些地方（Pointcut）执行什么操作（Advice）。

+ **顾问（Advisor)**

  Advisor 也是通知（Advice）和切入点（Pointcut）的组合。，Advice 是要增强的逻辑，而增强的逻辑要在什么地方执行是通过Pointcut来指定的，所以 Advice 必需与 Pointcut 组合在一起，这就诞生了 Advisor 这个类，spring Aop中提供了一个Advisor接口将Pointcut 与 Advice 的组合起来。

  Advisor有好几个称呼：顾问、通知器。

其中这4个：连接点(JoinPoint)、通知(advise)、切入点(pointcut)、顾问（advisor)，在spring中都定义了接口和类来表示这些对象，下面我们一个个来看一下。

#### JoinPoint

```java
package org.aopalliance.intercept;

public interface Joinpoint {

    /**
     * 转到拦截器链中的下一个拦截器
     */
    Object proceed() throws Throwable;

    /**
     * 返回保存当前连接点静态部分【的对象】，这里一般指被代理的目标对象
     */
    Object getThis();

    /**
     * 返回此静态连接点  一般就为当前的Method(至少目前的唯一实现是MethodInvocation,所以连接点得静态部分肯定就是本方法)
     */
    AccessibleObject getStaticPart();

}
```

几个重要的子接口和实现类如下图所示

![640](https://cdn.jsdelivr.net/gh/WangMinan/Pics/640.png)

##### Invocation接口

```java
package org.aopalliance.intercept;

/**
 * 此接口表示程序中的调用
 * 调用是一个连接点，可以被拦截器拦截。
 */
public interface Invocation extends Joinpoint {

    /**
     * 将参数作为数组对象获取。可以更改此数组中的元素值以更改参数。
     * 通常用来获取调用目标方法的参数
     */
    Object[] getArguments();
}
```

##### MethodInvocation接口

```java
package org.aopalliance.intercept;

import java.lang.reflect.Method;

/**
 * 方法调用的描述，在方法调用时提供给拦截器。
 * 方法调用是一个连接点，可以被方法拦截器拦截。
 */
public interface MethodInvocation extends Invocation {

    /**
     * 返回正在被调用得方法~~~  返回的是当前Method对象。
     * 此时，效果同父类的AccessibleObject getStaticPart() 这个方法
     */
    Method getMethod();
}
```

##### ProxyMethodInvocation接口

表示代理方法的调用

```csharp
public interface ProxyMethodInvocation extends MethodInvocation {

    /**
     * 获取被调用的代理对象
     */
    Object getProxy();

    /**
     * 克隆一个方法调用器MethodInvocation
     */
    MethodInvocation invocableClone();

    /**
     * 克隆一个方法调用器MethodInvocation，并为方法调用器指定参数
     */
    MethodInvocation invocableClone(Object... arguments);

    /**
     * 设置要用于此链中任何通知的后续调用的参数。
     */
    void setArguments(Object... arguments);

    /**
     * 添加一些扩展用户属性，这些属性不在AOP框架内使用。它们只是作为调用对象的一部分保留，用于特殊的拦截器。
     */
    void setUserAttribute(String key, @Nullable Object value);

    /**
     * 根据key获取对应的用户属性
     */
    @Nullable
    Object getUserAttribute(String key);

}
```

通俗点理解：连接点表示方法的调用过程，内部包含了方法调用过程中的所有信息，比如被调用的方法、目标、代理对象、执行拦截器链等信息。

上面定义都是一些接口，最终有2个实现。

+ ReflectiveMethodInvocation

  当代理对象是采用jdk动态代理创建的，通过代理对象来访问目标对象的方法的时，最终过程是由ReflectiveMethodInvocation来处理的，内部会通过递归调用方法拦截器，最终会调用到目标方法。

+ CglibMethodInvocation

  功能和上面的类似，当代理对象是采用cglib创建的，通过代理对象来访问目标对象的方法的时，最终过程是由CglibMethodInvocation来处理的，内部会通过递归调用方法拦截器，最终会调用到目标方法。



#### Advice

顶层advice接口中没有定义任何方法

```java
package org.aopalliance.aop;

public interface Advice {
}
```

存在4个子接口

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/1369022-20211014234038616-143351400.png)



##### MethodBeforeAdvice接口

方法执行前通知，需要在目标方法执行前执行一些逻辑的，可以通过这个实现。

通俗点说：**需要在目标方法执行之前增强一些逻辑，可以通过这个接口来实现。before方法：在调用给定方法之前回调**。

```java
package org.springframework.aop;

public interface MethodBeforeAdvice extends BeforeAdvice {

    /**
     * 调用目标方法之前会先调用这个before方法
     * method：需要执行的目标方法
     * args：目标方法的参数
     * target：目标对象
     */
    void before(Method method, Object[] args, @Nullable Object target) throws Throwable;
}
```

如同

```java
public Object invoke(){
    调用MethodBeforeAdvice#before方法
    return 调用目标方法;
}
```



##### AfterReturningAdvice接口

方法执行后通知，需要在目标方法执行之后执行增强一些逻辑的，可以通过这个实现。

**不过需要注意一点：目标方法正常执行后，才会回调这个接口，当目标方法有异常，那么这通知会被跳过。**

```java
package org.springframework.aop;

public interface AfterReturningAdvice extends AfterAdvice {

    /**
     * 目标方法执行之后会回调这个方法
     * method：需要执行的目标方法
     * args：目标方法的参数
     * target：目标对象
     */
    void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;
}
```

如同

```java
public Object invoke(){
    Object retVal = 调用目标方法;
    调用AfterReturningAdvice#afterReturning方法
    return retVal;
}
```



##### ThrowsAdvice接口

```java
package org.springframework.aop;

public interface ThrowsAdvice extends AfterAdvice {
}
```

此接口上没有任何方法，因为方法由反射调用，实现类必须实现以下形式的方法，前3个参数是可选的，最后一个参数为需要匹配的异常的类型。

```java
void afterThrowing([Method, args, target], ThrowableSubclass);
```

有效方法的一些例子如下：

```java
public void afterThrowing(Exception ex)
public void afterThrowing(RemoteException)
public void afterThrowing(Method method, Object[] args, Object target, Exception ex)
public void afterThrowing(Method method, Object[] args, Object target, ServletException ex)
```



##### MethodInterceptor接口

方法拦截器，这个接口最强大，可以实现上面3种类型的通知，上面3种通知**最终都通过适配模式将其转换为MethodInterceptor方式去执行**。

```java
package org.aopalliance.intercept;

@FunctionalInterface
public interface MethodInterceptor extends Interceptor {

    /**
     * 拦截目标方法的执行，可以在这个方法内部实现需要增强的逻辑，以及主动调用目标方法
     */
    Object invoke(MethodInvocation invocation) throws Throwable;
}
```

使用方式如：

```java
public class TracingInterceptor implements MethodInterceptor {
    Object invoke(MethodInvocation i) throws Throwable {
        System.out.println("method "+i.getMethod()+" is called on "+ i.getThis()+" with args "+i.getArguments());
        Object ret=i.proceed();//转到拦截器链中的下一个拦截器
        System.out.println("method "+i.getMethod()+" returns "+ret);
        return ret;
    }
}
```



##### 拦截器链

一个目标方法中可以添加很多Advice，这些Advice最终都会被转换为`MethodInterceptor`类型的方法拦截器。转化的过程通过下面的通知包装器来实现。一共有三个Interceptor.

最终会有多个`MethodInterceptor`，这些`MethodInterceptor`会组成一个方法调用链。

Aop内部会给目标对象创建一个代理，代理对象中会放入这些`MethodInterceptor`会组成一个方法调用链，当调用代理对象的方法的时候，会按顺序执行这些方法调用链，一个个执行，最后会通过反射再去调用目标方法，进而对目标方法进行增强。

![640-3](https://cdn.jsdelivr.net/gh/WangMinan/Pics/640-3.png)



##### 通知包装器

**负责将各种非MethodInterceptor类型的通知(Advice)包装为MethodInterceptor类型。**

**刚才有说过：Aop中所有的Advice最终都会转换为MethodInterceptor类型的，组成一个方法调用链，然后执行**

3个包装器类

- MethodBeforeAdviceInterceptor
- AfterReturningAdviceInterceptor
- ThrowsAdviceInterceptor



##### MethodBeforeAdviceInterceptor类

这个类实现了`MethodInterceptor`接口，负责将`MethodBeforeAdvice`方法前置通知包装为`MethodInterceptor`类型，创建这个类型的对象的时候需要传递一个`MethodBeforeAdvice`类型的参数，重点是`invoke`方法

```java
package org.springframework.aop.framework.adapter;

@SuppressWarnings("serial")
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {

    private final MethodBeforeAdvice advice;

    public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
        Assert.notNull(advice, "Advice must not be null");
        this.advice = advice;
    }


    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        //负责调用前置通知的方法
        this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
        //继续执行方法调用链
        return mi.proceed();
    }

}
```



##### AfterReturningAdviceInterceptor类

这个类实现了`MethodInterceptor`接口，负责将`AfterReturningAdvice`方法后置通知包装为`MethodInterceptor`类型，创建这个类型的对象的时候需要传递一个`AfterReturningAdvice`类型的参数，重点是`invoke`方法

```java
public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {

    private final AfterReturningAdvice advice;

    public AfterReturningAdviceInterceptor(AfterReturningAdvice advice) {
        Assert.notNull(advice, "Advice must not be null");
        this.advice = advice;
    }


    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        //先执行方法调用链,可以获取目标方法的执行结果
        Object retVal = mi.proceed();
        //执行后置通知
        this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
        //返回结果
        return retVal;
    }

}
```



##### ThrowsAdviceInterceptor类

这个类实现了`MethodInterceptor`接口，负责将`ThrowsAdvice`异常通知包装为`MethodInterceptor`类型，创建这个类型的对象的时候需要传递一个`Object`类型的参数，通常这个参数是`ThrowsAdvice`类型的，重点是`invoke`方法

```java
package org.springframework.aop.framework.adapter;

public class ThrowsAdviceInterceptor implements MethodInterceptor, AfterAdvice {

    private static final String AFTER_THROWING = "afterThrowing";

    private final Object throwsAdvice;

    //创建ThrowsAdviceInterceptor
    public ThrowsAdviceInterceptor(Object throwsAdvice) {
        Assert.notNull(throwsAdvice, "Advice must not be null");
        this.throwsAdvice = throwsAdvice;
        //获取异常通知中定义的所有方法（public、默认的、protected、private）
        Method[] methods = throwsAdvice.getClass().getMethods();
        //轮询methods
        for (Method method : methods) {
            //方法名称为afterThrowing && 方法参数为1或者4
            if (method.getName().equals(AFTER_THROWING) &&
                    (method.getParameterCount() == 1 || method.getParameterCount() == 4)) {
                //获取方法的最后一个参数类型
                Class<?> throwableParam = method.getParameterTypes()[method.getParameterCount() - 1];
                //判断方法参数类型是不是Throwable类型的
                if (Throwable.class.isAssignableFrom(throwableParam)) {
                    // 缓存异常处理方法到map中（异常类型->异常处理方法）
                    this.exceptionHandlerMap.put(throwableParam, method);
                }
            }
        }
        //如果exceptionHandlerMap，抛出异常，所以最少要有一个异常处理方法
        if (this.exceptionHandlerMap.isEmpty()) {
            throw new IllegalArgumentException(
                "At least one handler method must be found in class [" + throwsAdvice.getClass() + "]");
        }
    }


    /**
     * 获取异常通知中自定义的处理异常方法的数量
     */
    public int getHandlerMethodCount() {
        return this.exceptionHandlerMap.size();
    }


    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        try {
            //调用通知链
            return mi.proceed();
        }
        catch (Throwable ex) {
            //获取异常通知中自定义的处理异常的方法
            Method handlerMethod = getExceptionHandler(ex);
            //当处理的方法不为空
            if (handlerMethod != null) {
                //调用异常处理方法
                invokeHandlerMethod(mi, ex, handlerMethod);
            }
            //继续向外抛出异常
            throw ex; //@1
        }
    }

    /**
     * 获取throwsAdvice中处理exception参数指定的异常的方法
     */
    @Nullable
    private Method getExceptionHandler(Throwable exception) {
        //获取异常类型
        Class<?> exceptionClass = exception.getClass();
        //从缓存中获取异常类型对应的方法
        Method handler = this.exceptionHandlerMap.get(exceptionClass);
        //来一个循环，查询处理方法，循环条件：方法为空 && 异常类型!=Throwable
        while (handler == null && exceptionClass != Throwable.class) {
            //获取异常的父类型
            exceptionClass = exceptionClass.getSuperclass();
            //从缓存中查找异常对应的处理方法
            handler = this.exceptionHandlerMap.get(exceptionClass);
        }
        //将查找结果返回
        return handler;
    }

    //通过反射调用异常通知中的异常方法
    private void invokeHandlerMethod(MethodInvocation mi, Throwable ex, Method method) throws Throwable {
        //构建方法请求参数
        Object[] handlerArgs;
        //若只有1个参数，参数为：异常对象
        if (method.getParameterCount() == 1) {
            handlerArgs = new Object[] {ex};
        }
        else {
            //4个参数（方法、方法请求参数、目标对象、异常对象）
            handlerArgs = new Object[] {mi.getMethod(), mi.getArguments(), mi.getThis(), ex};
        }
        try {
            //通过反射调用异常通知中的方法
            method.invoke(this.throwsAdvice, handlerArgs);
        }
        catch (InvocationTargetException targetEx) {
            throw targetEx.getTargetException();
        }
    }

}
```

从上面可以看出，异常通知，自定义处理异常的方法有几个特点

1. 方法名称必须为`afterThrowing`
2. 方法参数必须1个或4个，最后一个参数是`Throwable`类型或其子类型
3. 可以在异常处理中记录一些异常信息，这个还是比较有用的，但是注意一点目标方法抛出的异常最后还是会向外继续抛出`@1`



##### 实际案例

**光讲源码，大家看着枯燥乏味，来点案例。**

先来一个类，用来模拟用户资金操作：充值、提现、查询资金余额；提现的时候余额不足的时候，会抛出异常。

```csharp
//模拟资金操作
public class FundsService {
    //账户余额
    private double balance = 1000;

    //模拟提现
    double recharge(String userName, double price) {
        System.out.println(String.format("%s提现%s", userName, price));
        balance += price;
        return balance;
    }

    //模拟提现
    double cashOut(String userName, double price) {
        if (balance < price) {
            throw new RuntimeException("余额不足!");
        }
        System.out.println(String.format("%s提现%s", userName, price));
        balance -= price;
        return balance;
    }

    //获取余额
    double getBalance(String userName) {
        return balance;
    }
}
```



###### 案例1

**前置通知拦截非法访问**

资金操作的所有方法都需要验证用户名，当用户名不是“路人”的时候，直接抛出非法访问异常。

```java
import org.junit.Test;
import org.springframework.aop.MethodBeforeAdvice;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.lang.Nullable;

import java.lang.reflect.Method;

public class AopTest4 {

    @Test
    public void test1() {
        //代理工厂
        ProxyFactory proxyFactory = new ProxyFactory(new FundsService());
        //添加一个方法前置通知，判断用户名不是“路人”的时候，抛出非法访问异常
        proxyFactory.addAdvice(new MethodBeforeAdvice() {
            @Override
            public void before(Method method, Object[] args, @Nullable Object target) throws Throwable {
                String userName = (String) args[0];
                //如果不是路人的时候，抛出非法访问异常
                if (!"路人".equals(userName)) {
                    throw new RuntimeException(String.format("[%s]非法访问!", userName));
                }
            }
        });
        //通过代理工厂创建代理
        FundsService proxy = (FundsService) proxyFactory.getProxy();
        //调用代理的方法
        proxy.recharge("路人", 100);
        proxy.recharge("张学友", 100);
    }
}
```

运行输出

```
路人提现100.0

java.lang.RuntimeException: [张学友]非法访问!

    at com.javacode2018.aop.demo4.AopTest4$1.before(AopTest4.java:25)
    at org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor.invoke(MethodBeforeAdviceInterceptor.java:55)
```



###### 案例2

**通过异常通知记录异常**

通过异常通知来捕获所有方法的运行，发现异常之后，通知开发修复bug。

```java
public static class SendMsgThrowsAdvice implements ThrowsAdvice {
    //注意方法名称必须为afterThrowing
    public void afterThrowing(Method method, Object[] args, Object target, RuntimeException e) {
        //监控到异常后发送消息通知开发者
        System.out.println("异常警报：");
        System.out.println(String.format("method:[%s]，args:[%s]", method.toGenericString(), 						Arrays.stream(args).collect(Collectors.toList())));
        System.out.println(e.getMessage());
        System.out.println("请尽快修复bug！");
    }
}

@Test
public void test2() {
    //代理工厂
    ProxyFactory proxyFactory = new ProxyFactory(new FundsService());
    //添加一个异常通知，发现异常之后发送消息给开发者尽快修复bug
    proxyFactory.addAdvice(new SendMsgThrowsAdvice());
    //通过代理工厂创建代理
    FundsService proxy = (FundsService) proxyFactory.getProxy();
    //调用代理的方法
    proxy.cashOut("路人", 2000);
}
```

运行输出

```
异常警报：
method:[double com.javacode2018.aop.demo4.FundsService.cashOut(java.lang.String,double)]，args:[[路人, 2000.0]]
余额不足!
请尽快修复bug！

java.lang.RuntimeException: 余额不足!
	at com.javacode2018.aop.demo4.FundsService.cashOut(FundsService.java:18)
```



#### PointCut

通知（Advice）用来指定**需要增强的逻辑**

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/1369022-20211014234115350-834073284.png)

哪些类的哪些方法中需要使用这些通知呢？这个就是通过切入点来配置的，切入点在Spring中对应了一个接口

##### PointCut接口

```java
package org.springframework.aop;

public interface Pointcut {

    /**
     * 类过滤器, 可以知道哪些类需要拦截
     */
    ClassFilter getClassFilter();

    /**
     * 方法匹配器, 可以知道哪些方法需要拦截
     */
    MethodMatcher getMethodMatcher();

    /**
     * 匹配所有对象的 Pointcut，内部的2个过滤器默认都会返回true
     */
    Pointcut TRUE = TruePointcut.INSTANCE;
}
```



##### ClassFilter接口

比较简单，用来过滤类的

```java
@FunctionalInterface
public interface ClassFilter {

    /**
     * 用来判断目标类型是否匹配
     */
    boolean matches(Class<?> clazz);
}
```



##### MethodMatcher接口

用来过滤方法的。

```java
public interface MethodMatcher {

    /**
     * 执行静态检查给定方法是否匹配
     * @param method 目标方法
     * @param targetClass 目标对象类型
     */
    boolean matches(Method method, Class<?> targetClass);

    /**
     * 是否是动态匹配，即是否每次执行目标方法的时候都去验证一下
     */
    boolean isRuntime();

    /**
     * 动态匹配验证的方法，比第一个matches方法多了一个参数args，这个参数是调用目标方法传入的参数
     */
    boolean matches(Method method, Class<?> targetClass, Object... args);


    /**
     * 匹配所有方法，这个内部的2个matches方法任何时候都返回true
     */
    MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}
```

为什么需要2个maches方法？什么是动态匹配？

比如下面一个类

```java
public class UserService{
    public void work(String userName){
        System.out.print(userName+",开始工作了!");
    }
}
```

work方法表示当前用户的工作方法，内部可以实现一些工作的逻辑。

我们希望通过aop对这个类进行增强，调用这个方法的时候，当传入的用户名是`路人的粉丝的`的时候，需要先进行问候，其他用户的时候，无需问候，将这个问题的代码可以放在`MethodBeforeAdvice`中实现，这种情况就是当参数满足一定的条件了，才会使用这个通知，不满足的时候，通知无效，此时就可以使用上面的动态匹配来实现，

`MethodMatcher`类中3个参数的matches方法可以用来对目标方法的参数做校验。

来看一下`MethodMatcher`过滤的整个过程

1. 调用matches(Method method, Class<?> targetClass)方法，验证方法是否匹配
2. isRuntime方法是否为true，如果为false，则以第一步的结果为准，否则继续向下
3. 调用matches(Method method, Class<?> targetClass, Object... args)方法继续验证，这个方法多了一个参数，可以对目标方法传入的参数进行校验。

通过上面的过程，大家可以看出来，如果`isRuntime`为false的时候，只需要对方法名称进行校验，当目标方法调用多次的时候，实际上第一步的验证结果是一样的，所以如果`isRuntime`为false的情况，可以将验证结果放在缓存中，提升效率，而spring内部就是这么做的，`isRuntime`为false的时候，需要每次都进行校验，效率会低一些，不过对性能的影响基本上可以忽略。



#### Advisor

通知定义了需要做什么，切入点定义了在哪些类的哪些方法中执行通知，那么需要将他们2个组合起来才有效啊。

顾问（Advisor）就是做这个事情的。

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/1369022-20211014234133432-1182028968.png)

##### Advisor接口

```java
package org.springframework.aop;

import org.aopalliance.aop.Advice;

/**
 * 包含AOP通知（在joinpoint处执行的操作）和确定通知适用性的过滤器（如切入点[PointCut]）的基本接口。
 * 这个接口不是供Spring用户使用的，而是为了支持不同类型的建议的通用性。
 */
public interface Advisor {
    /**
     * 返回引用的通知
     */
    Advice getAdvice();
}
```

上面这个接口通常不会直接使用，这个接口有2个子接口，通常我们会和这2个子接口来打交道，下面看一下这2个子接口。

![640-4](https://cdn.jsdelivr.net/gh/WangMinan/Pics/640-4.png)



##### PointcutAdvisor接口

通过名字就能看出来，这个和Pointcut有关，内部有个方法用来获取`Pointcut`，AOP使用到的大部分Advisor都属于这种类型的。

在目标方法中实现各种增强功能基本上都是通过PointcutAdvisor来实现的。

```java
package org.springframework.aop;

/**
 * 切入点类型的Advisor
 */
public interface PointcutAdvisor extends Advisor {

    /**
     * 获取顾问中使用的切入点
     */
    Pointcut getPointcut();
}
```

##### IntroductionAdvisor接口

一个Java类，没有实现A接口，在不修改Java类的情况下，使其具备A接口的功能。可以通过IntroductionAdvisor给目标类引入更多接口的功能。



### 案例演示

定义一个`UserService`类

```java
public class UserService {
    public void work(String userName) {
        System.out.println(userName + ",正在和路人甲java一起学Spring Aop，欢迎大家一起来！");
    }
}
```

需要通过AOP实现一些需求，来增强work方法的功能

#### 案例1

需求：在work方法执行之前，打印一句：你好：userName

我们使用硬编码，不借助SpringAOP的情况来编写代码，得到的代码如下所示

```java
@Test
public void test1() {
    //定义目标对象
    UserService target = new UserService();
    //创建pointcut，用来拦截UserService中的work方法
    Pointcut pointcut = new Pointcut() {
        @Override
        public ClassFilter getClassFilter() {
            //判断是否是UserService类型的
            return clazz -> UserService.class.isAssignableFrom(clazz);
        }

        @Override
        public MethodMatcher getMethodMatcher() {
            return new MethodMatcher() {
                @Override
                public boolean matches(Method method, Class<?> targetClass) {
                    //判断方法名称是否是work
                    return "work".equals(method.getName());
                }

                @Override
                public boolean isRuntime() {
                    return false;
                }

                @Override
                public boolean matches(Method method, Class<?> targetClass, Object... args) {
                    return false;
                }
            };
        }
    };
    //创建通知，此处需要在方法之前执行操作，所以需要用到MethodBeforeAdvice类型的通知
    MethodBeforeAdvice advice = (method, args, target1) -> System.out.println("你好:" + args[0]);

    //创建Advisor，将pointcut和advice组装起来
    DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, advice);

    //通过spring提供的代理创建工厂来创建代理
    ProxyFactory proxyFactory = new ProxyFactory();
    //为工厂指定目标对象
    proxyFactory.setTarget(target);
    //调用addAdvisor方法，为目标添加增强的功能，即添加Advisor，可以为目标添加很多个Advisor
    proxyFactory.addAdvisor(advisor);
    //通过工厂提供的方法来生成代理对象
    UserService userServiceProxy = (UserService) proxyFactory.getProxy();

    //调用代理的work方法
    userServiceProxy.work("路人");
}
```

输出结果为

```
你好:路人
路人,正在和路人甲java一起学Spring Aop，欢迎大家一起来！
```



#### 案例2

需求：统计一下work方法的耗时，将耗时输出

```java
@Test
public void test2() {
    //定义目标对象
    UserService target = new UserService();
    //创建pointcut，用来拦截UserService中的work方法
    Pointcut pointcut = new Pointcut() {
        @Override
        public ClassFilter getClassFilter() {
            //判断是否是UserService类型的
            return clazz -> UserService.class.isAssignableFrom(clazz);
        }

        @Override
        public MethodMatcher getMethodMatcher() {
            return new MethodMatcher() {
                @Override
                public boolean matches(Method method, Class<?> targetClass) {
                    //判断方法名称是否是work
                    return "work".equals(method.getName());
                }

                @Override
                public boolean isRuntime() {
                    return false;
                }

                @Override
                public boolean matches(Method method, Class<?> targetClass, Object... args) {
                    return false;
                }
            };
        }
    };
    
    //创建通知，需要拦截方法的执行，所以需要用到MethodInterceptor类型的通知
    MethodInterceptor advice = new MethodInterceptor() {
        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            System.out.println("准备调用:" + invocation.getMethod());
            long starTime = System.nanoTime();
            Object result = invocation.proceed();
            long endTime = System.nanoTime();
            System.out.println(invocation.getMethod() + "，调用结束！");
            System.out.println("耗时(纳秒):" + (endTime - starTime));
            return result;
        }
    };

    //创建Advisor，将pointcut和advice组装起来
    DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, advice);

    //通过spring提供的代理创建工厂来创建代理
    ProxyFactory proxyFactory = new ProxyFactory();
    //为工厂指定目标对象
    proxyFactory.setTarget(target);
    //调用addAdvisor方法，为目标添加增强的功能，即添加Advisor，可以为目标添加很多个Advisor
    proxyFactory.addAdvisor(advisor);
    //通过工厂提供的方法来生成代理对象
    UserService userServiceProxy = (UserService) proxyFactory.getProxy();

    //调用代理的work方法
    userServiceProxy.work("路人");
}
```

输出

```
准备调用:public void com.javacode2018.aop.demo3.UserService.work(java.lang.String)
路人,正在和路人甲java一起学Spring Aop，欢迎大家一起来！
public void com.javacode2018.aop.demo3.UserService.work(java.lang.String)，调用结束！
耗时(纳秒):9526200
```



## 面试题：Bean的生存周期

Bean生命周期一般有下面的四个阶段：

- Bean的定义
- Bean的初始化
- Bean的生存期
- Bean的销毁

![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/8069900-5170a2a4e69f4b3c.png)



### Bean的定义过程

+ 资源定位，就是Spring根据我们定义的注解(@Component)，找到相应的类。
+ 找到了资源就开始解析，并将定义的信息保存起来，此时，并没有初始化bean,这点需要注意。
+ 然后将bean的定义发布到SpringIoC的容器中，此时,SpringIoC的容器中还是没有Bean的生成。**只是定义的信息**。



### Bean的初始化

经过Bean的定义，初始化，Spring会继续完成Bean的实例和化和依赖注入，这样从IoC容器中就可以得到一个依赖注入完成的Bean。下图是初始化图的示例：

`@Configuration` 代表是一个 Java 配置文件 ， Spring会根据它来生成 IoC 容器去装配 Bean。

`@Bean` 代表将 configBean方法返回的 POJO 装配到 IoC 容器中, name为Bean 的名称，如果没有配置它，则会将方法名称作为 Bean 的名称保存到 Spring IoC 容器中 。![Bean的初始化](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20200418204525950.jpg)



### Bean的生存期

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20200418204600618.jpg)

**一个范例**

![](https://cdn.jsdelivr.net/gh/WangMinan/Pics/person_fruit.jpg)



父接口

```java
public interface Person {
    void service();

    // 设置水果类型
    void eatFruit(Fruit fruit);
}  

public interface Fruit {
	void use();
}
```

定义Person和Fruit的实现类Children和Apple，并将Apple类注入到Children中，在Children中加入生命周期需要的接口

```java
@Component
public class Apple implements Fruit {
    @Override
    public void use() {
    	System.out.println(this.getClass().getSimpleName()+"这个苹果很甜");
    }
}

@Component
public class Children implements Person, BeanNameAware, BeanFactoryAware, ApplicationContextAware, InitializingBean, DisposableBean {

    Fruit fruit = null;

    // 注入水果类
    public Children(@Autowired Fruit fruit){
    	this.fruit = fruit;
    }
    
    @Override
    public void service() {
    	fruit.use();
    }

    @Override
    public void eatFruit(Fruit fruit) {
    	this.fruit = fruit;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException   {
    	System.out.println("this"+this.getClass().getSimpleName()+"调用BeanFactory的setBeanFactory方法");
    }

    @Override
    public void setBeanName(String name) {
    	System.out.println("this"+this.getClass().getSimpleName()+"调用setBeanName的方法");
    }

    @Override
    public void destroy() throws Exception {
    	System.out.println("this"+this.getClass().getSimpleName()+"调用DisposableBean的方法");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
    	System.out.println("this"+this.getClass().getSimpleName()+"调用Initializing的afterPropertiesSet()的方法");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    	System.out.println("this"+this.getClass().getSimpleName()+"调用Application的setApplicationContext的方法");
    }

    @PostConstruct
    public void init(){
	    System.out.println(this.getClass().getSimpleName()+"注解@PostConstruct的自定义的初始化方法");
    }

    @PreDestroy
    public void destory1(){
    	System.out.println(this.getClass().getSimpleName()+"调用@PreDestory的自定义销毁的方法");
    }
}
```

测试类

```java
@SpringBootApplication
 public class BeanprocessorApplication {
     public static void main(String[] args) {
         SpringApplication.run(BeanprocessorApplication.class, args);
         /*
          * AnnotationConfigApplicationContext 基于Java编码的高级容器
          * 通过构造函数参数指定配置类
          */
         AnnotationConfigApplicationContext applicationContext = new 				AnnotationConfigApplicationContext(BeanConfig.class);
         Person person = applicationContext.getBean(Children.class);
         System.out.println("===========初始化完成==========");
         person.service();
         applicationContext.close();
     }
}
```

输出结果如下

```
-- 初始化Children类
thisChildren调用setBeanName的方法
thisChildren调用BeanFactory的setBeanFactory方法
thisChildren调用Application的setApplicationContext的方法
Children注解@PostConstruct的自定义的初始化方法
thisChildren调用Initializing的afterPropertiesSet()的方法

2020-04-18 17:28:38.902  INFO 9784 --- [           main] c.b.BeanprocessorApplication             : Started BeanprocessorApplication in 0.682 seconds (JVM running for 1.29)
-- 初始化注入的apple
thisChildren调用setBeanName的方法
thisChildren调用BeanFactory的setBeanFactory方法
thisChildren调用Application的setApplicationContext的方法
Children注解@PostConstruct的自定义的初始化方法
thisChildren调用Initializing的afterPropertiesSet()的方法

===========初始化完成==========
Apple这个苹果很甜 -- Apple覆写的Use方法被调用


Children调用@PreDestory的自定义销毁的方法
thisChildren调用DisposableBean的方法
Children调用@PreDestory的自定义销毁的方法
thisChildren调用DisposableBean的方法
```

从测试结果来看，Bean被初始化了两次，这是因为在初始化Children这个类时，还初始化了注入的Apple这个类。