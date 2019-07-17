# Spring

[TOC]



# 1、BeanFactory 和 FactoryBean？

**共同点**：

​         都是接口

**区别**：

​      BeanFactory 以Factory结尾，表示它是一个工厂类，用于管理Bean的一个工厂

​             在Spring中，所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的。

​      但对FactoryBean而言，这个Bean不是简单的Bean，而是一个能生产或者修饰对象生成的工厂Bean,

​             它的实现与设计模式中的工厂模式和修饰器模式类似。

 

## 1、 BeanFactory

​     BeanFactory定义了IOC容器的最基本形式，并提供了IOC容器应遵守的的最基本的接口，也就是Spring IOC所遵守的最底层和最基本的编程规范。

​     它的职责包括：实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。

​    在Spring代码中，BeanFactory只是个接口，并不是IOC容器的具体实现，

​          但是Spring容器给出了很多种实现，如 DefaultListableBeanFactory、XmlBeanFactory、ApplicationContext等，都是附加了某种功能的实现。

 

```
package org.springframework.beans.factory;

import org.springframework.beans.BeansException;
import org.springframework.core.ResolvableType;

public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";

    Object getBean(String var1) throws BeansException;

    <T> T getBean(String var1, Class<T> var2) throws BeansException;

    <T> T getBean(Class<T> var1) throws BeansException;

    Object getBean(String var1, Object... var2) throws BeansException;

    <T> T getBean(Class<T> var1, Object... var2) throws BeansException;

    boolean containsBean(String var1);

    boolean isSingleton(String var1) throws NoSuchBeanDefinitionException;

    boolean isPrototype(String var1) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String var1, ResolvableType var2) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String var1, Class<?> var2) throws NoSuchBeanDefinitionException;

    Class<?> getType(String var1) throws NoSuchBeanDefinitionException;

    String[] getAliases(String var1);
}
```



## 2、FactoryBean

​      一般情况下，Spring通过反射机制利用<bean>的class属性指定实现类实例化Bean，在某些情况下，实例化Bean过程比较复杂，

如果按照传统的方式，则需要在<bean>中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。

Spring为此提供了一个org.springframework.bean.factory.FactoryBean的工厂类接口，用户可以通过实现该接口定制实例化Bean的逻辑。
      FactoryBean接口对于Spring框架来说占用重要的地位，Spring自身就提供了70多个FactoryBean的实现。

它们隐藏了实例化一些复杂Bean的细节，给上层应用带来了便利。从Spring3.0开始，FactoryBean开始支持泛型，即接口声明改为FactoryBean<T>的形式

 

```
package org.springframework.beans.factory;

public interface FactoryBean<T> {
    T getObject() throws Exception;

    Class<?> getObjectType();

    boolean isSingleton();
}
```



在该接口中还定义了以下3个方法：

T getObject()：

​          返回由FactoryBean创建的Bean实例，如果isSingleton()返回true，则该实例会放到Spring容器中单实例缓存池中；
boolean isSingleton()：

​          返回由FactoryBean创建的Bean实例的作用域是singleton还是prototype；
Class<T> getObjectType()：

​          返回FactoryBean创建的Bean类型。
当配置文件中<bean>的class属性配置的实现类是FactoryBean时，通过getBean()方法返回的不是FactoryBean本身，

​         而是FactoryBean#getObject()方法所返回的对象，相当于FactoryBean#getObject()代理了getBean()方法。


例：如果使用传统方式配置下面Car的<bean>时，Car的每个属性分别对应一个<property>元素标签。

 



```
    package  com.baobaotao.factorybean;  
    public   class  Car  {  
        private   int maxSpeed ;  
        private  String brand ;  
        private   double price ;  
        public   int  getMaxSpeed ()   {  
            return   this . maxSpeed ;  
        }  
        public   void  setMaxSpeed ( int  maxSpeed )   {  
            this . maxSpeed  = maxSpeed;  
        }  
        public  String getBrand ()   {  
            return   this . brand ;  
        }  
        public   void  setBrand ( String brand )   {  
            this . brand  = brand;  
        }  
        public   double  getPrice ()   {  
            return   this . price ;  
        }  
        public   void  setPrice ( double  price )   {  
            this . price  = price;  
       }  
}   
```



​     如果用FactoryBean的方式实现就灵活点，下例通过逗号分割符的方式一次性的为Car的所有属性指定配置值： 



```
package  com.baobaotao.factorybean;  
import  org.springframework.beans.factory.FactoryBean;  
public   class  CarFactoryBean  implements  FactoryBean<Car>  {  
    private  String carInfo ;  
    public  Car getObject ()   throws  Exception  {  
        Car car =  new  Car () ;  
        String []  infos =  carInfo .split ( "," ) ;  
        car.setBrand ( infos [ 0 ]) ;  
        car.setMaxSpeed ( Integer. valueOf ( infos [ 1 ])) ;  
        car.setPrice ( Double. valueOf ( infos [ 2 ])) ;  
        return  car;  
    }  
    public  Class<Car> getObjectType ()   {  
        return  Car. class ;  
    }  
    public   boolean  isSingleton ()   {  
        return   false ;  
    }  
    public  String getCarInfo ()   {  
        return   this . carInfo ;  
    }  
  
    // 接受逗号分割符设置属性信息  
    public   void  setCarInfo ( String carInfo )   {  
        this . carInfo  = carInfo;  
    }  
}   
```



 有了这个CarFactoryBean后，就可以在配置文件中使用下面这种自定义的配置方式配置CarBean了：

<bean id="car"class="com.baobaotao.factorybean.CarFactoryBean" *P:carInfo="法拉利,400,2000000"/>*


当调用getBean("car")时，Spring通过反射机制发现CarFactoryBean实现了FactoryBean的接口，这时Spring容器就调用接口方法CarFactoryBean#getObject()方法返回。

如果希望获取CarFactoryBean的实例，则需要在使用getBean(beanName)方法时在beanName前显示的加上"&"前缀：如getBean("&car");

 

## 3 Others：

   3.1 Spring中共有两种bean，一种为普通bean，另一种则为工厂bean。

#  2、Spring IOC 的理解，其初始化过程？

IoC容器是什么？
IoC文英全称Inversion of Control，即控制反转，我么可以这么理解IoC容器：
　　“把某些业务对象的的控制权交给一个平台或者框架来同一管理，这个同一管理的平台可以称为IoC容器。”

我们刚开始学习spring的时候会经常看到的类似下面的这代码：

```
ApplicationContext appContext = new ClassPathXmlApplicationContext("cjj/models/beans.xml");
Person p = (Person)appContext.getBean("person");
```

上面代码中，在创建ApplicationContext实例对象过程中会创建一个spring容器，该容器会读取配置文件"cjj/models/beans.xml",并统一管理由该文件中定义好的所有bean实例对象，如果要获取某个bean实例，使用getBean方法就行了。例如我们只需要将Person提前配置在beans.xml文件中（可以理解为注入），之后我们可以不需使用new   Person()的方式创建实例，而是通过容器来获取Person实例，这就相当于将Person的控制权交由spring容器了，差不多这就是控制反转的概念。

那在创建IoC容器时经历了哪些呢？为此，先来了解下Spring中IoC容器分类，继而根据一个具体的容器来讲解IoC容器初始化的过程。
Spring中有两个主要的容器系列：

1. 实现BeanFactory接口的简单容器；
2. 实现ApplicationContext接口的高级容器。

ApplicationContext比较复杂，它不但继承了BeanFactory的大部分属性，还继承其它可扩展接口，扩展的了许多高级的属性，其接口定义如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public interface ApplicationContext extends EnvironmentCapable, 　　　　　　　　　　　　　　　　　　ListableBeanFactory,    //继承于BeanFactory　　　　　　　　　　　　　　　　　　HierarchicalBeanFactory,//继承于BeanFactory
　　　　　　　　　　　　　　　　　　MessageSource,            //　　　　　　　　　　　　　　　　　　ApplicationEventPublisher,//　　　　　　　　　　　　　　　　　　ResourcePatternResolver   //继承ResourceLoader，用于获取resource对象
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

在BeanFactory子类中有一个***DefaultListableBeanFactory***类，它包含了基本Spirng   IoC容器所具有的重要功能，开发时不论是使用BeanFactory系列还是ApplicationContext系列来创建容器基本都会使用到DefaultListableBeanFactory类，可以这么说，在spring中实际上把它当成默认的IoC容器来使用。下文在源码实例分析时你将会看到这个类。

（注：文章有点长，需要细心阅读，不同版本的spring中源码可能不同，但逻辑几乎是一样的，如果可以建议还是看源码 ^_^）

 

回到本文正题上来，关于Spirng IoC容器的初始化过程在《Spirng技术内幕：深入解析Spring架构与设计原理》一书中有明确的指出，IoC容器的初始化过程可以分为三步：

1. **Resource定位（Bean的定义文件定位）**
2. **将Resource定位好的资源载入到BeanDefinition**
3. **将BeanDefiniton注册到容器中**

 

- ## **第一步 Resource定位**

Resource是Sping中用于封装I/O操作的接口。正如前面所见，在创建spring容器时，通常要访问XML配置文件，除此之外还可以通过访问文件类型、二进制流等方式访问资源，还有当需要网络上的资源时可以通过访问URL，Spring把这些文件统称为Resource，Resource的体系结构如下:

![img](https://images2015.cnblogs.com/blog/1072097/201612/1072097-20161202134243834-1191621438.png)

常用的resource资源类型如下：
　　**FileSystemResource：**以文件的绝对路径方式进行访问资源，效果类似于Java中的File;
　　**ClassPathResourcee：**以类路径的方式访问资源，效果类似于this.getClass().getResource("/").getPath();
　　**ServletContextResource：**web应用根目录的方式访问资源，效果类似于request.getServletContext().getRealPath("");
　　**UrlResource：**访问网络资源的实现类。例如file: http: ftp:等前缀的资源对象;
　　**ByteArrayResource:** 访问字节数组资源的实现类。

那如何获取上图中对应的各种Resource对象呢？
Spring提供了ResourceLoader接口用于实现不同的Resource加载策略，该接口的实例对象中可以获取一个resource对象，也就是说将不同Resource实例的创建交给ResourceLoader的实现类来处理。ResourceLoader接口中只定义了两个方法：

```
Resource getResource(String location); //通过提供的资源location参数获取Resource实例ClassLoader getClassLoader(); // 获取ClassLoader,通过ClassLoader可将资源载入JVM
```

注：ApplicationContext的所有实现类都实现RecourceLoader接口，因此可以直接调用getResource（参数）获取Resoure对象**。**不同的ApplicatonContext实现类使用getResource方法取得的资源类型不同，例如：FileSystemXmlApplicationContext.getResource获取的就是FileSystemResource实例；ClassPathXmlApplicationContext.gerResource获取的就是ClassPathResource实例；XmlWebApplicationContext.getResource获取的就是ServletContextResource实例，另外像不需要通过xml直接使用注解@Configuation方式加载资源的AnnotationConfigApplicationContext等等。

在资源定位过程完成以后，就为资源文件中的bean的载入创造了I/O操作的条件，如何读取资源中的数据将会在下一步介绍的BeanDefinition的载入过程中描述。

 

- ## **第二步 通过返回的resource对象，进行BeanDefinition的载入**

1.什么是BeanDefinition? BeanDefinition与Resource的联系呢？

　　官方文档中对BeanDefinition的解释如下：
　　A  BeanDefinition describes a bean instance, which has property values,  constructor argument values, and further information supplied by  concrete implementations.
它们之间的联系从官方文档描述的一句话：Load bean definitions from the specified resource中可见一斑。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
　　/**
     * Load bean definitions from the specified resource.
     * @param resource the resource descriptor
     * @return the number of bean definitions found
     * @throws BeanDefinitionStoreException in case of loading or parsing errors
     */
    int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　总之，BeanDefinition相当于一个数据结构，这个数据结构的生成过程是根据定位的resource资源对象中的bean而来的，这些bean在Spirng   IoC容器内部表示成了的BeanDefintion这样的数据结构，IoC容器对bean的管理和依赖注入的实现都是通过操作BeanDefinition来进行的。

2.如何将BeanDefinition载入到容器？
	 　　在Spring中配置文件主要格式是XML，对于用来读取XML型资源文件来进行初始化的IoC  容器而言，该类容器会使用到AbstractXmlApplicationContext类，该类定义了一个名为loadBeanDefinitions(DefaultListableBeanFactory  beanFactory) 的方法用于获取BeanDefinition：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// 该方法属于AbstractXmlApplicationContect类
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    this.initBeanDefinitionReader(beanDefinitionReader);
    // 用于获取BeanDefinition
    this.loadBeanDefinitions(beanDefinitionReader);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

此方法在具体执行过程中首先会new一个与容器对应的BeanDefinitionReader型实例对象，然后将生成的BeanDefintionReader实例作为参数传入loadBeanDefintions(XmlBeanDefinitionReader)，继续往下执行载入BeanDefintion的过程。例如AbstractXmlApplicationContext有两个实现类：FileSystemXmlApplicationContext、ClassPathXmlApplicationContext，这些容器在调用此方法时会创建一个**XmlBeanDefinitionReader类**对象专门用来载入所有的BeanDefinition。

下面以XmlBeanDefinitionReader对象载入BeanDefinition为例，使用源码说明载入BeanDefinition的过程：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// 该方法属于AbstractXmlApplicationContect类protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        Resource[] configResources = getConfigResources();//获取所有定位到的resource资源位置（用户定义）
        if (configResources != null) {
            reader.loadBeanDefinitions(configResources);//载入resources
        }
        String[] configLocations = getConfigLocations();//获取所有本地配置文件的位置（容器自身）
        if (configLocations != null) {
            reader.loadBeanDefinitions(configLocations);//载入resources
        }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

通过上面代码将用户定义的资源以及容器本身需要的资源全部加载到reader中，reader.loadBeanDefinitions方法的源码如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// 该方法属于AbstractBeanDefinitionReader类, 父接口BeanDefinitionReader
@Override
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int counter = 0;
    for (Resource resource : resources) {
        // 将所有资源全部加载，交给AbstractBeanDefinitionReader的实现子类处理这些resource
        counter += loadBeanDefinitions(resource);
    }
    return counter;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

BeanDefinitionReader接口定义了 int loadBeanDefinitions（Resource resource）方法：

```
int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;
int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;
```

 ![img](https://images2018.cnblogs.com/blog/1072097/201805/1072097-20180508110631385-1333213476.png)

XmlBeanDefinitionReader 类实现了BeanDefinitionReader接口中的loadBeanDefinitions(Resource)方法，其继承关系如上图所示。XmlBeanDefinitionReader类中几主要对加载的所有resource开始进行处理，大致过程是，先将resource包装为EncodeResource类型，然后处理，为生成BeanDefinition对象做准备，其主要几个方法的源码如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
        // 包装resource为EncodeResource类型
        return loadBeanDefinitions(new EncodedResource(resource)); 
    }

    // 加载包装后的EncodeResource资源
    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        Assert.notNull(encodedResource, "EncodedResource must not be null");
        if (logger.isInfoEnabled()) {
            logger.info("Loading XML bean definitions from " + encodedResource.getResource());
        }

        try {
            // 通过resource对象得到XML文件内容输入流，并为IO的InputSource做准备
            InputStream inputStream = encodedResource.getResource().getInputStream();
            try {
                // Create a new input source with a byte stream.
                InputSource inputSource = new InputSource(inputStream);
                if (encodedResource.getEncoding() != null) {
                    inputSource.setEncoding(encodedResource.getEncoding());
                }
                // 开始准备 load bean definitions from the specified XML file
                return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
            }
            finally {
                inputStream.close();
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "IOException parsing XML document from " + encodedResource.getResource(), ex);
        }
    }

    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
            throws BeanDefinitionStoreException {
        try {
            // 获取指定资源的验证模式
            int validationMode = getValidationModeForResource(resource);

            // 从资源对象中加载DocumentL对象，大致过程为：将resource资源文件的内容读入到document中
            // DocumentLoader在容器读取XML文件过程中有着举足轻重的作用！
            // XmlBeanDefinitionReader实例化时会创建一个DefaultDocumentLoader型的私有属性,继而调用loadDocument方法
            // inputSource--要加载的文档的输入源
            Document doc = this.documentLoader.loadDocument(
                    inputSource, this.entityResolver, this.errorHandler, validationMode, this.namespaceAware);
            
            // 将document文件的bean封装成BeanDefinition，并注册到容器
            return registerBeanDefinitions(doc, resource);
        }
        catch ...(略)
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

DefaultDocumentLoader大致了解即可，感兴趣可继续深究，其源码如下：（看完收起，便于阅读下文）

![img](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)  View Code

上面代码分析到了registerBeanDefinitions(doc, resource)这一步，也就是准备将Document中的Bean按照Spring bean语义进行解析并转化为BeanDefinition类型，这个方法的具体过程如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
/**
 * 属于XmlBeanDefinitionReader类
 * Register the bean definitions contained in the given DOM document.
 * @param doc the DOM document
 * @param resource
 * @return the number of bean definitions found
 * @throws BeanDefinitionStoreException
 */
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 获取到DefaultBeanDefinitionDocumentReader实例
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    // 获取容器中bean的数量
    int countBefore = getRegistry().getBeanDefinitionCount();
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

通过 XmlBeanDefinitionReader 类中的私有属性 documentReaderClass 可以获得一个 DefaultBeanDefinitionDocumentReader 实例对象： 

```
private Class<?> documentReaderClass = DefaultBeanDefinitionDocumentReader.class;
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {   return BeanDefinitionDocumentReader.class.cast(BeanUtils.instantiateClass(this.documentReaderClass));}
```

**DefaultBeanDefinitionDocumentReader**实现了**BeanDefinitionDocumentReader接口**，它的registerBeanDefinitions方法定义如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;

    logger.debug("Loading bean definitions");
    // 获取doc的root节点，通过该节点能够访问所有的子节点
    Element root = doc.getDocumentElement();
    // 处理beanDefinition的过程委托给BeanDefinitionParserDelegate实例对象来完成
    BeanDefinitionParserDelegate delegate = createHelper(readerContext, root);

    // Default implementation is empty.
    // Subclasses can override this method to convert custom elements into standard Spring bean definitions
    preProcessXml(root);
    // 核心方法，代理
    parseBeanDefinitions(root, delegate);
    postProcessXml(root);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

上面出现的**BeanDefinitionParserDelegate类**非常非常重要（需要了解代理技术，如JDK动态代理、cglib动态代理等）。Spirng  BeanDefinition的解析就是在这个代理类下完成的，此类包含了各种对符合Spring  Bean语义规则的处理，比如<bean></bean>、<import></import>、<alias><alias/>等的检测。

parseBeanDefinitions(root, delegate)方法如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
   protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            // 遍历所有节点，做对应解析工作
            // 如遍历到<import>标签节点就调用importBeanDefinitionResource(ele)方法对应处理
            // 遍历到<bean>标签就调用processBeanDefinition(ele,delegate)方法对应处理
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    if (delegate.isDefaultNamespace(ele)) {
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        //对应用户自定义节点处理方法
                        delegate.parseCustomElement(ele); 
                    }
                }
            }
        }
        else {
            delegate.parseCustomElement(root);
        }
    }

    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        // 解析<import>标签
        if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            importBeanDefinitionResource(ele);
        }
        // 解析<alias>标签
        else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            processAliasRegistration(ele);
        }
        // 解析<bean>标签,最常用，过程最复杂
        else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            processBeanDefinition(ele, delegate);
        }
        // 解析<beans>标签
        else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
            // recurse
            doRegisterBeanDefinitions(ele);
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

这里针对常用的<bean>标签中的方法做简单介绍，其他标签的加载方式类似：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    /**
     * Process the given bean element, parsing the bean definition
     * and registering it with the registry.
     */
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        // 该对象持有beanDefinition的name和alias，可以使用该对象完成beanDefinition向容器的注册
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                // 注册最终被修饰的bean实例，下文注册beanDefinition到容器会讲解该方法
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }
            // Send registration event.
            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

parseBeanDefinitionElement(Element ele)方法会调用parseBeanDefinitionElement(ele, null)方法，并将值返回**BeanDefinitionHolder类**对象，这个方法将会对给定的<bean>标签进行解析，如果在解析<bean>标签的过程中出现错误则返回null。

需要强调一下的是parseBeanDefinitionElement(ele, null)方法中产生了一个抽象类型的BeanDefinition实例，这也是我们首次看到直接定义BeanDefinition的地方，这个方法里面会将<bean>标签中的内容解析到BeanDefinition中，之后再对BeanDefinition进行包装，将它与beanName,Alias等封装到BeanDefinitionHolder 对象中，该部分源码如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
        return parseBeanDefinitionElement(ele, null);
    }

    public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
        String id = ele.getAttribute(ID_ATTRIBUTE);
        String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
            ...(略)
        String beanName = id;
           ...（略）
        // 从上面按过程走来，首次看到直接定义BeanDefinition ！！！
        // 该方法会对<bean>节点以及其所有子节点如<property>、<List>、<Set>等做出解析，具体过程本文不做分析（太多太长）
        AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
        if (beanDefinition != null) {
            if (!StringUtils.hasText(beanName)) {
                ...(略)
            }
            String[] aliasesArray = StringUtils.toStringArray(aliases);
            return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
        }

        return null;
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 

- ## **第三步，将BeanDefiniton注册到容器中**

 　　最终Bean配置会被解析成BeanDefinition并与beanName,Alias一同封装到**BeanDefinitionHolder类**中， 之后beanFactory.registerBeanDefinition(beanName, bdHolder.getBeanDefinition())，注册到**DefaultListableBeanFactory**.beanDefinitionMap中。之后客户端如果要获取Bean对象，Spring容器会根据注册的BeanDefinition信息进行实例化。

 BeanDefinitionReaderUtils类：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static void registerBeanDefinition(
            BeanDefinitionHolder bdHolder, BeanDefinitionRegistry beanFactory) throws BeansException {

        // Register bean definition under primary name.
        String beanName = bdHolder.getBeanName();　　　　　// 注册beanDefinition!!!
        beanFactory.registerBeanDefinition(beanName, bdHolder.getBeanDefinition());

        // 如果有别名的话也注册进去，Register aliases for bean name, if any.
        String[] aliases = bdHolder.getAliases();
        if (aliases != null) {
            for (int i = 0; i < aliases.length; i++) {
                beanFactory.registerAlias(beanName, aliases[i]);
            }
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**DefaultListableBeanFactory**实现了上面调用BeanDefinitionRegistry接口的  registerBeanDefinition( beanName,   bdHolder.getBeanDefinition())方法，这一部分的主要逻辑是向DefaultListableBeanFactory对象的**beanDefinitionMap**中存放beanDefinition，当初始化容器进行bean初始化时，在bean的生命周期分析里必然会在这个beanDefinitionMap中获取beanDefition实例，有机会成文分析一下bean的生命周期，到时可以分析一下如何使用这个beanDefinitionMap。

registerBeanDefinition( beanName,  bdHolder.getBeanDefinition() )方法具体方法如下：

```
    /** Map of bean definition objects, keyed by bean name */
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException {

        Assert.hasText(beanName, "Bean name must not be empty");
        Assert.notNull(beanDefinition, "Bean definition must not be null");

        if (beanDefinition instanceof AbstractBeanDefinition) {
            try {
                ((AbstractBeanDefinition) beanDefinition).validate();
            }
            catch (BeanDefinitionValidationException ex) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                        "Validation of bean definition failed", ex);
            }
        }
　　　　　　　　 // beanDefinitionMap是个ConcurrentHashMap类型数据，用于存放beanDefinition,它的key值是beanName
        Object oldBeanDefinition = this.beanDefinitionMap.get(beanName);
        if (oldBeanDefinition != null) {
            if (!this.allowBeanDefinitionOverriding) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                        "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
                        "': there's already [" + oldBeanDefinition + "] bound");
            }
            else {
                if (logger.isInfoEnabled()) {
                    logger.info("Overriding bean definition for bean '" + beanName +
                            "': replacing [" + oldBeanDefinition + "] with [" + beanDefinition + "]");
                }
            }
        }
        else {
            this.beanDefinitionNames.add(beanName);  
        }　　　　　
        // 将获取到的BeanDefinition放入Map中，容器操作使用bean时通过这个HashMap找到具体的BeanDefinition
        this.beanDefinitionMap.put(beanName, beanDefinition);        
        // Remove corresponding bean from singleton cache, if any.
        // Shouldn't usually be necessary, rather just meant for overriding
        // a context's default beans (e.g. the default StaticMessageSource
        // in a StaticApplicationContext).
        removeSingleton(beanName);
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　

 本文可看做本人看书笔记学习，其中大部分参考《Spirng技术内幕：深入解析Spring架构与设计原理》以及结合网上博客等仓促而作，作此文只希望自己的技术不断提高，同时记录自己学习过程中的点点滴滴，其中肯定有许多不足之处，望谅解与指出。

#  3、BeanFactory 和 ApplicationContext？

## 一. spring容器理解

spring容器可以理解为生产对象（OBJECT）的地方，在这里容器不只是帮我们创建了对象那么简单，它负责了对象的整个生命周期--创建、装配、销毁。而这里对象的创建管理的控制权都交给了Spring容器，所以这是一种控制权的反转，称为IOC容器，而这里IOC容器不只是Spring才有，很多框架也都有该技术。

## 二. BeanFactory和ApplicationContext之间的关系

- BeanFactory和ApplicationContext是Spring的两大核心接口，而其中ApplicationContext是BeanFactory的子接口。它们都可以当做Spring的容器，Spring容器是生成Bean实例的工厂，并管理容器中的Bean。在基于Spring的Java EE应用中，所有的组件都被当成Bean处理，包括数据源，Hibernate的SessionFactory、事务管理器等。
- 生活中我们一般会把生产产品的地方称为工厂，而在这里bean对象的地方官方取名为BeanFactory，直译Bean工厂（com.springframework.beans.factory.BeanFactory），我们一般称BeanFactory为IoC容器，而称ApplicationContext为应用上下文。
- Spring的核心是容器，而容器并不唯一，框架本身就提供了很多个容器的实现，大概分为两种类型：
   一种是不常用的BeanFactory，这是最简单的容器，只能提供基本的DI功能；
   一种就是继承了BeanFactory后派生而来的ApplicationContext(应用上下文)，它能提供更多企业级的服务，例如解析配置文本信息等等，这也是ApplicationContext实例对象最常见的应用场景。

## 三. BeanFactory详情介绍

Spring容器最基本的接口就是BeanFactory。BeanFactory负责配置、创建、管理Bean，它有一个子接口ApplicationContext，也被称为Spring上下文，容器同时还管理着Bean和Bean之间的依赖关系。
 **spring Ioc容器的实现，从根源上是beanfactory，但真正可以作为一个可以独立使用的ioc容器还是DefaultListableBeanFactory，因此可以这么说， DefaultListableBeanFactory 是整个spring ioc的始祖。**
 



![img](https:////upload-images.jianshu.io/upload_images/12234310-6bf928fc2231465a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/753)



> #### 接口介绍：
>
> **1.BeanFactory接口：**
>   是Spring bean容器的根接口，提供获取bean，是否包含bean,是否单例与原型，获取bean类型，bean 别名的方法 。它最主要的方法就是getBean(String beanName)。
>  **2.BeanFactory的三个子接口：**
>   * HierarchicalBeanFactory：提供父容器的访问功能
>   * ListableBeanFactory：提供了批量获取Bean的方法
>   * AutowireCapableBeanFactory：在BeanFactory基础上实现对已存在实例的管理
>  **3.ConfigurableBeanFactory：**
>  主要单例bean的注册，生成实例，以及统计单例bean
>  **4.ConfigurableListableBeanFactory：**
>  继承了上述的所有接口，增加了其他功能：比如类加载器,类型转化,属性编辑器,BeanPostProcessor,作用域,bean定义,处理bean依赖关系, bean如何销毁…
>  **5.实现类DefaultListableBeanFactory详细介绍：**
>  实现了ConfigurableListableBeanFactory，实现上述BeanFactory所有功能。它还可以注册BeanDefinition
>  接口详细介绍请参考:[揭秘BeanFactory](https://blog.csdn.net/u011179993/article/details/51636742)

## 四. ApplicationContext介绍

如果说BeanFactory是Sping的心脏，那么ApplicationContext就是完整的身躯了。





![img](https:////upload-images.jianshu.io/upload_images/12234310-a14ad5a594b524fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

ApplicationContext结构图.png



![img](https:////upload-images.jianshu.io/upload_images/12234310-be1edded652cea7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

ApplicationContext类结构树.png

|     ApplicationContext常用实现类      |                             作用                             |
| :-----------------------------------: | :----------------------------------------------------------: |
|  AnnotationConfigApplicationContext   | 从一个或多个基于java的配置类中加载上下文定义，适用于java注解的方式。 |
|    ClassPathXmlApplicationContext     | 从类路径下的一个或多个xml配置文件中加载上下文定义，适用于xml配置的方式。 |
|    FileSystemXmlApplicationContext    | 从文件系统下的一个或多个xml配置文件中加载上下文定义，也就是说系统盘符中加载xml配置文件。 |
| AnnotationConfigWebApplicationContext |            专门为web应用准备的，适用于注解方式。             |
|       XmlWebApplicationContext        | 从web应用下的一个或多个xml配置文件加载上下文定义，适用于xml配置方式。 |

Spring具有非常大的灵活性，它提供了三种主要的装配机制：

- 1. 在XMl中进行显示配置
- 1. 在Java中进行显示配置
- 1. 隐式的bean发现机制和自动装配
      *组件扫描（component scanning）：Spring会自动发现应用上下文中所创建的bean。
      *自动装配（autowiring）：Spring自动满足bean之间的依赖。

（使用的优先性: 3>2>1）尽可能地使用自动配置的机制，显示配置越少越好。当必须使用显示配置bean的时候（如：有些源码不是由你来维护的，而当你需要为这些代码配置bean的时候），推荐使用类型安全比XML更加强大的JavaConfig。最后只有当你想要使用便利的XML命名空间，并且在JavaConfig中没有同样的实现时，才使用XML。

代码示例：

1. 通过xml文件将配置加载到IOC容器中

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
     <!--若没写id，则默认为com.test.Man#0,#0为一个计数形式-->
    <bean id="man" class="com.test.Man"></bean>
</beans>
public class Test {
    public static void main(String[] args) {
        //加载项目中的spring配置文件到容器
        //ApplicationContext context = new ClassPathXmlApplicationContext("resouces/applicationContext.xml");
        //加载系统盘中的配置文件到容器
        ApplicationContext context = new FileSystemXmlApplicationContext("E:/Spring/applicationContext.xml");
        //从容器中获取对象实例
        Man man = context.getBean(Man.class);
        man.driveCar();
    }
}
```

1. 通过java注解的方式将配置加载到IOC容器

```
//同xml一样描述bean以及bean之间的依赖关系
@Configuration
public class ManConfig {
    @Bean
    public Man man() {
        return new Man(car());
    }
    @Bean
    public Car car() {
        return new QQCar();
    }
}
public class Test {
    public static void main(String[] args) {
        //从java注解的配置中加载配置到容器
        ApplicationContext context = new AnnotationConfigApplicationContext(ManConfig.class);
        //从容器中获取对象实例
        Man man = context.getBean(Man.class);
        man.driveCar();
    }
}
```

1. 隐式的bean发现机制和自动装配

```
/**
 * 这是一个游戏光盘的实现
 */
//这个简单的注解表明该类回作为组件类，并告知Spring要为这个创建bean。
@Component
public class GameDisc implements Disc{
    @Override
    public void play() {
        System.out.println("我是马里奥游戏光盘。");
    }
}
```

不过，组件扫描默认是不启用的。我们还需要显示配置一下Spring，从而命令它去寻找@Component注解的类，并为其创建bean。

```
@Configuration
@ComponentScan
public class DiscConfig {
}
```

我们在DiscConfig上加了一个@ComponentScan注解表示在Spring中开启了组件扫描，默认扫描与配置类相同的包，就可以扫描到这个GameDisc的Bean了。这就是Spring的自动装配机制。

------

**除了提供BeanFactory所支持的所有功能外ApplicationContext还有额外的功能**

- 默认初始化所有的Singleton，也可以通过配置取消预初始化。
- 继承MessageSource，因此支持国际化。
- 资源访问，比如访问URL和文件。
- 事件机制。
- 同时加载多个配置文件。
- 以声明式方式启动并创建Spring容器。

> 注：由于ApplicationContext会预先初始化所有的Singleton Bean，于是在系统创建前期会有较大的系统开销，但一旦ApplicationContext初始化完成，程序后面获取Singleton Bean实例时候将有较好的性能。也可以为bean设置lazy-init属性为true，即Spring容器将不会预先初始化该bean。



#  4、Spring Bean 的生命周期，如何被管理的？

Spring框架中，一旦把一个Bean纳入Spring   IOC容器之中，这个Bean的生命周期就会交由容器进行管理，一般担当管理角色的是BeanFactory或者ApplicationContext,认识一下Bean的生命周期活动，对更好的利用它有很大的帮助：

下面以BeanFactory为例，说明一个Bean的生命周期活动

- Bean的建立， 由BeanFactory读取Bean定义文件，并生成各个实例
- Setter注入，执行Bean的属性依赖注入
- BeanNameAware的setBeanName(), 如果实现该接口，则执行其setBeanName方法
- BeanFactoryAware的setBeanFactory()，如果实现该接口，则执行其setBeanFactory方法
- BeanPostProcessor的processBeforeInitialization()，如果有关联的processor，则在Bean初始化之前都会执行这个实例的processBeforeInitialization()方法
- InitializingBean的afterPropertiesSet()，如果实现了该接口，则执行其afterPropertiesSet()方法
- Bean定义文件中定义init-method
- BeanPostProcessors的processAfterInitialization()，如果有关联的processor，则在Bean初始化之前都会执行这个实例的processAfterInitialization()方法
- DisposableBean的destroy()，在容器关闭时，如果Bean类实现了该接口，则执行它的destroy()方法
- Bean定义文件中定义destroy-method，在容器关闭时，可以在Bean定义文件中使用“destory-method”定义的方法

如果使用ApplicationContext来维护一个Bean的生命周期，则基本上与上边的流程相同，只不过在执行BeanNameAware的setBeanName()后，若有Bean类实现了org.springframework.context.ApplicationContextAware接口，则执行其setApplicationContext()方法，然后再进行BeanPostProcessors的processBeforeInitialization()
实际上，ApplicationContext除了向BeanFactory那样维护容器外，还提供了更加丰富的框架功能，如Bean的消息，事件处理机制等。

在这里一用仓颉的一幅图说明流程： 转载自 https://www.cnblogs.com/xrq730/p/6363055.html

![img](https://img2018.cnblogs.com/blog/1082754/201810/1082754-20181030104208937-1907545744.png)

以下是自己测试时打印的日志信息，可以看下加载顺序： 



```
 1 /**
 2  * @ClassName: MySpringBean
 3  * @Description: my spring bean to test
 4  * @author: daniel.zhao
 5  * @date: 2018年10月26日 上午10:12:37
 6  */
 7 public class MySpringBean implements BeanNameAware, BeanFactoryAware, InitializingBean, ApplicationContextAware {
 8 
 9     private ApplicationContext applicationContext;
10 
11     private static final Logger logger = LoggerFactory.getLogger(MySpringBean.class);
12 
13     public MySpringBean() {
14         logger.info("new MySpringBean......");
15     }
16 
17     @Override
18     public void setApplicationContext(ApplicationContext context) throws BeansException {
19         logger.info("ApplicationContextAware-setApplicationContext......");
20         this.applicationContext = context;
21     }
22 
23     @Override
24     public void afterPropertiesSet() throws Exception {
25         logger.info("InitializingBean-afterPropertiesSet......");
26     }
27 
28     @Override
29     public void setBeanFactory(BeanFactory bf) throws BeansException {
30         logger.info("BeanFactoryAware-setBeanFactory......");
31     }
32 
33     @Override
34     public void setBeanName(String name) {
35         logger.info("BeanNameAware-setBeanName......");
36     }
37 
38     public void init() {
39         logger.info("init-method......");
40     }
41 }
```



```
/**
 * @ClassName: MySpringBeanPostProcessor
 * @author: daniel.zhao
 * @date: 2018年10月26日 上午10:40:21
 */
@Component
public class MySpringBeanPostProcessor implements BeanPostProcessor {

    private static final Logger logger = LoggerFactory.getLogger(MySpringBeanPostProcessor.class);

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof MySpringBean) {
            logger.info("BeanPostProcessor-postProcessAfterInitialization......");
        }
        return bean;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof MySpringBean) {
            logger.info("BeanPostProcessor-postProcessBeforeInitialization......");
        }
        return bean;
    }

}
```



```
2018-10-26 10:49:48.768  INFO 5732 --- [           main] com.daniel.bean.MySpringBean             : BeanNameAware-setBeanName......2018-10-26 10:49:48.769  INFO 5732 --- [           main] com.daniel.bean.MySpringBean             : BeanFactoryAware-setBeanFactory......2018-10-26 10:49:48.769  INFO 5732 --- [           main] com.daniel.bean.MySpringBean             : ApplicationContextAware-setApplicationContext......2018-10-26 10:49:48.770  INFO 5732 --- [           main] c.daniel.bean.MySpringBeanPostProcessor  : BeanPostProcessor-postProcessBeforeInitialization......2018-10-26 10:49:48.770  INFO 5732 --- [           main] com.daniel.bean.MySpringBean             : InitializingBean-afterPropertiesSet......2018-10-26 10:49:48.770  INFO 5732 --- [           main] com.daniel.bean.MySpringBean             : init-method......2018-10-26 10:49:48.771  INFO 5732 --- [           main] c.daniel.bean.MySpringBeanPostProcessor  : BeanPostProcessor-postProcessAfterInitialization......
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

#  5、Spring Bean 的加载过程是怎样的？

## **代码入口**

之前写文章都会啰啰嗦嗦一大堆再开始，进入【Spring源码分析】这个板块就直接切入正题了。

很多朋友可能想看Spring源码，但是不知道应当如何入手去看，这个可以理解：Java开发者通常从事的都是Java   Web的工作，对于程序员来说，一个Web项目用到Spring，只是配置一下配置文件而已，Spring的加载过程相对是不太透明的，不太好去找加载的代码入口。

下面有很简单的一段代码可以作为Spring代码加载的入口：

```
 1 ApplicationContext ac = new ClassPathXmlApplicationContext("spring.xml");
 2 ac.getBean(XXX.class);
```

ClassPathXmlApplicationContext用于加载CLASSPATH下的Spring配置文件，可以看到，第二行就已经可以获取到Bean的实例了，那么必然第一行就已经完成了对所有Bean实例的加载，因此可以通过ClassPathXmlApplicationContext作为入口。为了后面便于代码阅读，先给出一下ClassPathXmlApplicationContext这个类的继承关系：![img](https://images2015.cnblogs.com/blog/801753/201702/801753-20170201125310058-568989522.png)

大致的继承关系是如上图所示的，由于版面的关系，没有继续画下去了，左下角的ApplicationContext应当还有一层继承关系，比较关键的一点是它是BeanFactory的子接口。

最后声明一下，本文使用的Spring版本为3.0.7，比较老，使用这个版本纯粹是因为公司使用而已。

 

## **ClassPathXmlApplicationContext存储内容**

为了更理解ApplicationContext，拿一个实例ClassPathXmlApplicationContext举例，看一下里面存储的内容，加深对ApplicationContext的认识，以表格形式展现：

| **对象名**                  | **类  型**                     | **作  用**                                                   | **归属类**                                  |
| --------------------------- | ------------------------------ | ------------------------------------------------------------ | ------------------------------------------- |
| configResources             | Resource[]                     | 配置文件资源对象数组                                         | ClassPathXmlApplicationContext              |
| configLocations             | String[]                       | 配置文件字符串数组，存储配置文件路径                         | AbstractRefreshableConfigApplicationContext |
| beanFactory                 | DefaultListableBeanFactory     | 上下文使用的Bean工厂                                         | AbstractRefreshableApplicationContext       |
| beanFactoryMonitor          | Object                         | Bean工厂使用的同步监视器                                     | AbstractRefreshableApplicationContext       |
| id                          | String                         | 上下文使用的唯一Id，标识此ApplicationContext                 | AbstractApplicationContext                  |
| parent                      | ApplicationContext             | 父级ApplicationContext                                       | AbstractApplicationContext                  |
| beanFactoryPostProcessors   | List<BeanFactoryPostProcessor> | 存储BeanFactoryPostProcessor接口，Spring提供的一个扩展点     | AbstractApplicationContext                  |
| startupShutdownMonitor      | Object                         | refresh方法和destory方法公用的一个监视器，避免两个方法同时执行 | AbstractApplicationContext                  |
| shutdownHook                | Thread                         | Spring提供的一个钩子，JVM停止执行时会运行Thread里面的方法    | AbstractApplicationContext                  |
| resourcePatternResolver     | ResourcePatternResolver        | 上下文使用的资源格式解析器                                   | AbstractApplicationContext                  |
| lifecycleProcessor          | LifecycleProcessor             | 用于管理Bean生命周期的生命周期处理器接口                     | AbstractApplicationContext                  |
| messageSource               | MessageSource                  | 用于实现国际化的一个接口                                     | AbstractApplicationContext                  |
| applicationEventMulticaster | ApplicationEventMulticaster    | Spring提供的事件管理机制中的事件多播器接口                   | AbstractApplicationContext                  |
| applicationListeners        | Set<ApplicationListener>       | Spring提供的事件管理机制中的应用监听器                       | AbstractApplicationContext                  |

 

## **ClassPathXmlApplicationContext构造函数**

看下ClassPathXmlApplicationContext的构造函数：

```
 1 public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
 2     this(new String[] {configLocation}, true, null);
 3 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
2         throws BeansException {
3 
4     super(parent);
5     setConfigLocations(configLocations);
6     if (refresh) {
7         refresh();
8     }
9 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

从第二段代码看，总共就做了三件事：

　　1、super(parent)

　　　　没什么太大的作用，设置一下父级ApplicationContext，这里是null

　　2、setConfigLocations(configLocations)

　　　　代码就不贴了，一看就知道，里面做了两件事情：

　　　　（1）将指定的Spring配置文件的路径存储到本地

　　　　（2）解析Spring配置文件路径中的${PlaceHolder}占位符，替换为系统变量中PlaceHolder对应的Value值，System本身就自带一些系统变量比如class.path、os.name、user.dir等，也可以通过System.setProperty()方法设置自己需要的系统变量

　　3、refresh()

　　　　这个就是整个Spring  Bean加载的核心了，它是ClassPathXmlApplicationContext的父类AbstractApplicationContext的一个方法，顾名思义，用于刷新整个Spring上下文信息，定义了整个Spring上下文加载的流程。

 

## **refresh方法**

上面已经说了，refresh()方法是整个Spring Bean加载的核心，因此看一下整个refresh()方法的定义：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 public void refresh() throws BeansException, IllegalStateException {
 2         synchronized (this.startupShutdownMonitor) {
 3             // Prepare this context for refreshing.
 4             prepareRefresh();
 5 
 6             // Tell the subclass to refresh the internal bean factory.
 7             ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
 8 
 9             // Prepare the bean factory for use in this context.
10             prepareBeanFactory(beanFactory);
11 
12             try {
13                 // Allows post-processing of the bean factory in context subclasses.
14                 postProcessBeanFactory(beanFactory);
15 
16                 // Invoke factory processors registered as beans in the context.
17                 invokeBeanFactoryPostProcessors(beanFactory);
18 
19                 // Register bean processors that intercept bean creation.
20                 registerBeanPostProcessors(beanFactory);
21 
22                 // Initialize message source for this context.
23                 initMessageSource();
24 
25                 // Initialize event multicaster for this context.
26                 initApplicationEventMulticaster();
27 
28                 // Initialize other special beans in specific context subclasses.
29                 onRefresh();
30 
31                 // Check for listener beans and register them.
32                 registerListeners();
33 
34                 // Instantiate all remaining (non-lazy-init) singletons.
35                 finishBeanFactoryInitialization(beanFactory);
36 
37                 // Last step: publish corresponding event.
38                 finishRefresh();
39             }
40 
41             catch (BeansException ex) {
42                 // Destroy already created singletons to avoid dangling resources.
43                 destroyBeans();
44 
45                 // Reset 'active' flag.
46                 cancelRefresh(ex);
47 
48                 // Propagate exception to caller.
49                 throw ex;
50             }
51         }
52     }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

每个子方法的功能之后一点一点再分析，首先refresh()方法有几点是值得我们学习的：

　　1、方法是加锁的，这么做的原因是避免多线程同时刷新Spring上下文

　　2、尽管加锁可以看到是针对整个方法体的，但是没有在方法前加synchronized关键字，而使用了对象锁startUpShutdownMonitor，这样做有两个好处：

　　　　（1）refresh()方法和close()方法都使用了startUpShutdownMonitor对象锁加锁，这就保证了**在调用refresh()方法的时候无法调用close()方法**，反之亦然，避免了冲突

　　　　（2）另外一个好处不在这个方法中体现，但是提一下，使用对象锁可以减小了同步的范围，只对不能并发的代码块进行加锁，提高了整体代码运行的效率

　　3、**方法里面使用了每个子方法定义了整个refresh()方法的流程，使得整个方法流程清晰易懂**。这点是非常值得学习的，一个方法里面几十行甚至上百行代码写在一起，在我看来会有三个显著的问题：

　　　　（1）扩展性降低。反过来讲，假使把流程定义为方法，子类可以继承父类，可以根据需要重写方法

　　　　（2）代码可读性差。很简单的道理，看代码的人是愿意看一段500行的代码，还是愿意看10段50行的代码？

　　　　（3）代码可维护性差。这点和上面的类似但又有不同，可维护性差的意思是，一段几百行的代码，功能点不明确，不易后人修改，可能会导致“牵一发而动全身”

 

## **prepareRefresh方法**

下面挨个看refresh方法中的子方法，首先是prepareRefresh方法，看一下源码：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 /**
 2  * Prepare this context for refreshing, setting its startup date and
 3  * active flag.
 4  */
 5 protected void prepareRefresh() {
 6     this.startupDate = System.currentTimeMillis();
 7         synchronized (this.activeMonitor) {
 8         this.active = true;
 9     }
10 
11     if (logger.isInfoEnabled()) {
12         logger.info("Refreshing " + this);
13     }
14 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

这个方法功能比较简单，顾名思义，准备刷新Spring上下文，其功能注释上写了：

1、设置一下刷新Spring上下文的开始时间

2、将active标识位设置为true

另外可以注意一下12行这句日志，这句日志打印了真正加载Spring上下文的Java类。

 

## **obtainFreshBeanFactory方法**

obtainFreshBeanFactory方法的作用是获取刷新Spring上下文的Bean工厂，其代码实现为：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
2     refreshBeanFactory();
3     ConfigurableListableBeanFactory beanFactory = getBeanFactory();
4     if (logger.isDebugEnabled()) {
5         logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
6     }
7     return beanFactory;
8 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

其核心是第二行的refreshBeanFactory方法，这是一个抽象方法，有AbstractRefreshableApplicationContext和GenericApplicationContext这两个子类实现了这个方法，看一下上面ClassPathXmlApplicationContext的继承关系图即知，调用的应当是AbstractRefreshableApplicationContext中实现的refreshBeanFactory，其源码为：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 protected final void refreshBeanFactory() throws BeansException {
 2     if (hasBeanFactory()) {
 3         destroyBeans();
 4         closeBeanFactory();
 5     }
 6     try {
 7         DefaultListableBeanFactory beanFactory = createBeanFactory();
 8         beanFactory.setSerializationId(getId());
 9         customizeBeanFactory(beanFactory);
10         loadBeanDefinitions(beanFactory);
11         synchronized (this.beanFactoryMonitor) {
12             this.beanFactory = beanFactory;
13         }
14     }
15     catch (IOException ex) {
16         throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
17     }
18 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

这段代码的核心是第7行，这行点出了**DefaultListableBeanFactory**这个类，这个类是构造Bean的核心类，这个类的功能会在下一篇文章中详细解读，首先给出DefaultListableBeanFactory的继承关系图：

![img](https://images2015.cnblogs.com/blog/801753/201702/801753-20170202113720339-1992987568.png)

AbstractAutowireCapableBeanFactory这个类的继承层次比较深，版面有限，就没有继续画下去了，本图基本上清楚地展示了DefaultListableBeanFactory的层次结构。

为了更清晰地说明DefaultListableBeanFactory的作用，列举一下DefaultListableBeanFactory中存储的一些重要对象及对象中的内容，DefaultListableBeanFactory基本就是操作这些对象，以表格形式说明：

| **对象名**                    | **类  型**                      | **作    用**                                        | **归属类**                         |
| ----------------------------- | ------------------------------- | --------------------------------------------------- | ---------------------------------- |
| aliasMap                      | Map<String, String>             | 存储Bean名称->Bean别名映射关系                      | SimpleAliasRegistry                |
| **singletonObjects**          | **Map<String, Object>**         | **存储单例Bean名称->单例Bean实现映射关系**          | **DefaultSingletonBeanRegistry**   |
| singletonFactories            | Map<String, ObjectFactory>      | 存储Bean名称->ObjectFactory实现映射关系             | DefaultSingletonBeanRegistry       |
| earlySingletonObjects         | Map<String, Object>             | 存储Bean名称->预加载Bean实现映射关系                | DefaultSingletonBeanRegistry       |
| registeredSingletons          | Set<String>                     | 存储注册过的Bean名                                  | DefaultSingletonBeanRegistry       |
| singletonsCurrentlyInCreation | Set<String>                     | 存储当前正在创建的Bean名                            | DefaultSingletonBeanRegistry       |
| disposableBeans               | Map<String, Object>             | 存储Bean名称->Disposable接口实现Bean实现映射关系    | DefaultSingletonBeanRegistry       |
| factoryBeanObjectCache        | Map<String, Object>             | 存储Bean名称->FactoryBean接口Bean实现映射关系       | FactoryBeanRegistrySupport         |
| propertyEditorRegistrars      | Set<PropertyEditorRegistrar>    | 存储PropertyEditorRegistrar接口实现集合             | AbstractBeanFactory                |
| embeddedValueResolvers        | List<StringValueResolver>       | 存储StringValueResolver（字符串解析器）接口实现列表 | AbstractBeanFactory                |
| beanPostProcessors            | List<BeanPostProcessor>         | 存储 BeanPostProcessor接口实现列表                  | AbstractBeanFactory                |
| mergedBeanDefinitions         | Map<String, RootBeanDefinition> | 存储Bean名称->合并过的根Bean定义映射关系            | AbstractBeanFactory                |
| alreadyCreated                | Set<String>                     | 存储至少被创建过一次的Bean名集合                    | AbstractBeanFactory                |
| ignoredDependencyInterfaces   | Set<Class>                      | 存储不自动装配的接口Class对象集合                   | AbstractAutowireCapableBeanFactory |
| resolvableDependencies        | Map<Class, Object>              | 存储修正过的依赖映射关系                            | DefaultListableBeanFactory         |
| beanDefinitionMap             | Map<String, BeanDefinition>     | 存储Bean名称-->Bean定义映射关系                     | DefaultListableBeanFactory         |
| beanDefinitionNames           | List<String>                    | 存储Bean定义名称列表                                | DefaultListableBeanFactory         |

#  6、如果要你实现Spring IOC，你会注意哪些问题？

百度的面试官问，如果让你自己设计一个IOC,和AOP，如何设计，

我把IOC的过程答出来了，但是明显不对， 
(1) IOC  利用了反射，自己有个id,classtype,hashmap，所有的功能都在hashmap中，然后利用反射的Class.forName把把classtype转化成类，然后利用反射的setFieldValue()从hashMap中把属性和方法取出来，注入进去。最终把类创建出来，

(2)AOP是动态代理，其实底层也是反射；

## 一、如何自己实现Spring的IOC的功能

我们都知道，IOC是利用了反射机制，接下来就让我们自己写个Spring 来看看Spring 到底是怎么运行的吧！ **(百度二面：如何自己实现Spring的 IOC依赖反转的功能)**
首先，我们定义一个Bean类，这个类用来存放一个Bean拥有的属性 



```
/* Bean Id */  
    private String id;  
    /* Bean Class */  
    private String type;  //这是类名称
    /* Bean Property */  
    private Map<String, Object> properties = new HashMap<String, Object>();  
```



 一个Bean包括id,type,和Properties。 type 就是Class类名称



```
<bean id="personService" class="cn.itcast.service.OrderFactory" factory-method="createOrder"/>

public class OrderFactory {
    // 注意这里的这个方法是 static 的！
    public static OrderServiceBean createOrder(){   
        return new OrderServiceBean();
    }
}
```



接下来Spring 就开始加载我们的配置文件了，将我们配置的信息保存在一个HashMap中，HashMap的key就是Bean 的 Id  ，HasMap  的value是这个Bean，只有这样我们才能通过context.getBean("animal")这个方法获得Animal这个类。我们都知道Spirng可以注入基本类型，而且可以注入像List，Map这样的类型，接下来就让我们以Map为例看看Spring是怎么保存的吧 

Map配置可以像下面的   



```
<bean id="test" class="Test">  
        <property name="testMap">  
            <map>  
                <entry key="a">  
                    <value>1</value>  
                </entry>  
                <entry key="b">  
                    <value>2</value>  
                </entry>  
            </map>  
        </property>  
    </bean> 
```



 Spring是怎样保存上面的配置呢？，代码如下：  



```
if (beanProperty.element("map") != null) {  
                    Map<String, Object> propertiesMap = new HashMap<String, Object>();  
                    Element propertiesListMap = (Element) beanProperty  
                            .elements().get(0);  
                    Iterator<?> propertiesIterator = propertiesListMap  
                            .elements().iterator();  
                    while (propertiesIterator.hasNext()) {  
                        Element vet = (Element) propertiesIterator.next();  
                        if (vet.getName().equals("entry")) {  
                            String key = vet.attributeValue("key");  
                            Iterator<?> valuesIterator = vet.elements()  
                                    .iterator();  
                            while (valuesIterator.hasNext()) {  
                                Element value = (Element) valuesIterator.next();  
                                if (value.getName().equals("value")) {  
                                    propertiesMap.put(key, value.getText());  
                                }  
                                if (value.getName().equals("ref")) {  
                                    propertiesMap.put(key, new String[] { value  
                                            .attributeValue("bean") });  
                                }  
                            }  
                        }  
                    }  
                    bean.getProperties().put(name, propertiesMap);  
                }
```



 **接下来就进入最核心部分了**，让我们看看Spring 到底是怎么依赖注入的吧，其实依赖注入的思想也很简单，它是通过反射机制实现的，在实例化一个类时，它通过反射调用类中set方法将事先保存在HashMap中的类属性注入到类中。让我们看看具体它是怎么做的吧。 

首先实例化一个类，像这样  



```
public static Object newInstance(String className) {  
        Class<?> cls = null;  
        Object obj = null;  
        try {  
            cls = Class.forName(className);  
            obj = cls.newInstance();  
        } catch (ClassNotFoundException e) {  
            throw new RuntimeException(e);  
        } catch (InstantiationException e) {  
            throw new RuntimeException(e);  
        } catch (IllegalAccessException e) {  
            throw new RuntimeException(e);  
        }  
        return obj;  
    }  
```



 接着它将这个类的依赖注入进去，像这样   



```
public static void setProperty(Object obj, String name, String value) {  
        Class<? extends Object> clazz = obj.getClass();  
        try {  
            String methodName = returnSetMthodName(name);  
            Method[] ms = clazz.getMethods();  
            for (Method m : ms) {  
                if (m.getName().equals(methodName)) {  
                    if (m.getParameterTypes().length == 1) {  
                        Class<?> clazzParameterType = m.getParameterTypes()[0];  
                        setFieldValue(clazzParameterType.getName(), value, m,  
                                obj);  
                        break;  
                    }  
                }  
            }  
        } catch (SecurityException e) {  
            throw new RuntimeException(e);  
        } catch (IllegalArgumentException e) {  
            throw new RuntimeException(e);  
        } catch (IllegalAccessException e) {  
            throw new RuntimeException(e);  
        } catch (InvocationTargetException e) {  
            throw new RuntimeException(e);  
        }  
}  
```



最后它将这个类的实例返回给我们，我们就可以用了。我们还是以Map为例看看它是怎么做的，我写的代码里面是创建一个HashMap并把该HashMap注入到需要注入的类中，像这样， 

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (value instanceof Map) {  
                Iterator<?> entryIterator = ((Map<?, ?>) value).entrySet()  
                        .iterator();  
                Map<String, Object> map = new HashMap<String, Object>();  
                while (entryIterator.hasNext()) {  
                    Entry<?, ?> entryMap = (Entry<?, ?>) entryIterator.next();  
                    if (entryMap.getValue() instanceof String[]) {  
                        map.put((String) entryMap.getKey(),  
                                getBean(((String[]) entryMap.getValue())[0]));  
                    }  
                }  
                BeanProcesser.setProperty(obj, property, map);  
            } 
```



好了，这样我们就可以用Spring 给我们创建的类了，是不是也不是很难啊？当然Spring能做到的远不止这些，这个示例程序仅仅提供了Spring最核心的依赖注入功能中的一部分。  

 参考：[Spring IOC 原理](http://blog.csdn.net/it_man/article/details/4402245)：

## 二、IOC原理

让我们来看下IoC容器到底是如何工作。在此我们以xml配置方式来分析一下：

**一、准备配置文件**：就像前边Hello World配置文件一样，在配置文件中声明Bean定义也就是为Bean配置元数据。

**二、由****IoC****容器进行解析元数据：** IoC容器的Bean Reader读取并解析配置文件，根据定义生成BeanDefinition配置元数据对象，IoC容器根据BeanDefinition进行实例化、配置及组装Bean。

**三、实例化****IoC****容器：**由客户端实例化容器，获取需要的Bean。 

实例化bean 有三种方式：类构造器实例化、静态工厂方法实例化及实例工厂方法实例化

参考：[Spring实例化Bean的三种方式及Bean的类型](https://www.cnblogs.com/aspirant/p/9209268.html)

IoC（Inversion of Control）  

  (1). IoC（Inversion of  Control）是指容器控制程序对象之间的关系，而不是传统实现中，由程序代码直接操控。控制权由应用代码中转到了外部容器，控制权的转移是所谓反转。  对于Spring而言，就是由Spring来控制对象的生命周期和对象之间的关系；IoC还有另外一个名字——“依赖注入（Dependency  Injection）”。从名字上理解，所谓依赖注入，即组件之间的依赖关系由容器在运行期决定，即由容器动态地将某种依赖关系注入到组件之中。  

(2). 在Spring的工作方式中，所有的类都会在spring容器中登记，告诉spring这是个什么东西，你需要什么东西，然后spring会在系统运行到适当的时候，把你要的东西主动给你，同时也把你交给其他需要你的东西。**所有的类的创建、销毁都由 spring来控制，也就是说控制对象生存周期的不再是引用它的对象**，而是spring。对于某个具体的对象而言，以前是它控制其他对象，现在是所有对象都被spring控制，所以这叫控制反转。

(3). 在系统运行中，动态的向某个对象提供它所需要的其他对象。  

(4). **依赖注入的思想是通过反射机制实现的，在实例化一个类时，它通过反射调用类中set方法将事先保存在HashMap中的类属性(至于是什么样的HashMap后面会提到)注入到类中。 总而言之，在传统的对象创建方式中，通常由调用者来创建被调用者的实例，而在Spring中创建被调用者的工作由Spring来完成，然后注入调用者，即所谓的依赖注入or控制反转。 注入方式有两种：依赖注入和设置注入**； IoC的优点：降低了组件之间的耦合，降低了业务对象之间替换的复杂性，使之能够灵活的管理对象。

 

控制反转/依赖注入

IOC（DI）：Java如果对象调用对象，对象间的耦合度高了。而IOC的思想是：Spring容器来实现这些相互依赖对象的创建、协调工作。对象只需要关系业务逻辑本身就可以了。从这方面来说，对象如何得到他的协作对象的责任被反转了（IOC、DI）。DI其实就是IOC的另外一种说法。DI  就是：获得依赖对象的方式反转了。 

IoC与DI

　　首先想说说IoC（Inversion of  Control，控制倒转）所谓IoC，对于spring框架来说，就是由spring来负责控制对象的生命周期和对象间的关系。这是什么意思呢，举个简单的例子，我们是如何找女朋友的？常见的情况是，我们到处去看哪里有长得漂亮身材又好的mm，然后打听她们的兴趣爱好、qq号、电话号、ip号、iq号………，想办法认识她们，投其所好送其所要，然后嘿嘿……这个过程是复杂深奥的，我们必须自己设计和面对每个环节。传统的程序开发也是如此，在一个对象中，如果要使用另外的对象，就必须得到它（自己new一个，或者从JNDI中查询一个），使用完之后还要将对象销毁（比如Connection等），对象始终会和其他的接口或类藕合起来。

　那么IoC是如何做的呢？有点像通过婚介找女朋友，在我和女朋友之间引入了一个第三者：婚姻介绍所。婚介管理了很多男男女女的资料，我可以向婚介提出一个列表，告诉它我想找个什么样的女朋友，比如长得像李嘉欣，身材像林熙雷，唱歌像周杰伦，速度像卡洛斯，技术像齐达内之类的，然后婚介就会按照我们的要求，提供一个mm，我们只需要去和她谈恋爱、结婚就行了。简单明了，如果婚介给我们的人选不符合要求，我们就会抛出异常。整个过程不再由我自己控制，而是有婚介这样一个类似容器的机构来控制。Spring所倡导的开发方式就是如此，所有的类都会在spring容器中登记，告诉spring你是个什么东西，你需要什么东西，然后spring会在系统运行到适当的时候，把你要的东西主动给你，同时也把你交给其他需要你的东西。所有的类的创建、销毁都由   spring来控制，也就是说控制对象生存周期的不再是引用它的对象，而是spring。对于某个具体的对象而言，以前是它控制其他对象，现在是所有对象都被spring控制，所以这叫控制反转。如果你还不明白的话，我决定放弃。

IoC的一个重点是在系统运行中，动态的向某个对象提供它所需要的其他对象。这一点是通过DI（Dependency  Injection，依赖注入）来实现的。比如对象A需要操作数据库，以前我们总是要在A中自己编写代码来获得一个Connection对象，有了  spring我们就只需要告诉spring，A中需要一个Connection，至于这个Connection怎么构造，何时构造，A不需要知道。在系统运行时，spring会在适当的时候制造一个Connection，然后像打针一样，注射到A当中，这样就完成了对各个对象之间关系的控制。A需要依赖  Connection才能正常运行，而这个Connection是由spring注入到A中的，依赖注入的名字就这么来的。那么DI是如何实现的呢？  Java  1.3之后一个重要特征是反射（reflection），它允许程序在运行的时候动态的生成对象、执行对象的方法、改变对象的属性，spring就是通过反射来实现注入的。关于反射的相关资料请查阅java  doc。

　理解了IoC和DI的概念后，一切都将变得简单明了，剩下的工作只是在spring的框架中堆积木而已。

**下面来让大家了解一下Spring到底是怎么运行的。**  



```
public static void main(String[] args) {  
        ApplicationContext context = new FileSystemXmlApplicationContext(  
                "applicationContext.xml");  
        Animal animal = (Animal) context.getBean("animal");  
        animal.say();  
    }  
```



 这段代码你一定很熟悉吧，不过还是让我们分析一下它吧，首先是applicationContext.xml 

```
<bean id="animal" class="phz.springframework.test.Cat">  
        <property name="name" value="kitty" />  
    </bean> 
```

  他有一个类phz.springframework.test.Cat   



```
public class Cat implements Animal {  
    private String name;  
    public void say() {  
        System.out.println("I am " + name + "!");  
    }  
    public void setName(String name) {  
        this.name = name;  
    }  
}  
```



 实现了phz.springframework.test.Animal接口  

```
public interface Animal {  
    public void say();  
}  
```

 

很明显上面的代码输出I am kitty! 那么到底Spring是如何做到的呢？ 

##  

 

AOP（Aspect Oriented Programming）

(1**). AOP面向方面编程基于IoC，是对OOP的有益补充；**

(2). AOP利用一种称为“横切”的技术，剖解开封装的对象内部，并将那些影响了  多个类的公共行为封装到一个可重用模块，并将其名为“Aspect”，即方面。所谓“方面”，简单地说，就是将那些与业务无关，却为业务模块所共同调用的  逻辑或责任封装起来，比如日志记录，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可操作性和可维护性。

(3). AOP代表的是一个横向的关  系，将“对象”比作一个空心的圆柱体，其中封装的是对象的属性和行为；则面向方面编程的方法，就是将这个圆柱体以切面形式剖开，选择性的提供业务逻辑。而  剖开的切面，也就是所谓的“方面”了。然后它又以巧夺天功的妙手将这些剖开的切面复原，不留痕迹，但完成了效果。

(4). 实现AOP的技术，主要分为两大类：**一是采用动态代理技术**，利用截取消息的方式，对该消息进行装饰，以取代原有对象行为的执行；二是采用静态织入的方式，引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。

(5). Spring实现AOP：JDK动态代理和CGLIB代理 JDK动态代理：**其代理对象必须是某个接口的实现，它是通过在运行期间创建一个接口的实现类来完成对目标对象的代理；其核心的两个类是InvocationHandler和Proxy**。   CGLIB代理：实现原理类似于JDK动态代理，只是它在运行期间生成的代理对象是针对目标类扩展的子类。CGLIB是高效的代码生成包，底层是依靠ASM（开源的java字节码编辑类库）操作字节码实现的，性能比JDK强；需要引入包asm.jar和cglib.jar。      使用AspectJ注入式切面和@AspectJ注解驱动的切面实际上底层也是通过动态代理实现的。

如果需要了解具体的动态代理参考：[深入理解Java反射+动态代理](https://www.cnblogs.com/aspirant/p/9036805.html)

(6). AOP使用场景：                     

Authentication 权限检查        

Caching 缓存        

Context passing 内容传递        

Error handling 错误处理        

Lazy loading　延迟加载        

Debugging　　调试      

logging, tracing, profiling and monitoring　日志记录，跟踪，优化，校准        

Performance optimization　性能优化，效率检查        

Persistence　　持久化        

Resource pooling　资源池        

Synchronization　同步        

Transactions 事务管理    

另外Filter的实现和struts2的拦截器的实现都是AOP思想的体现。  

#  7、如果要你实现Spring AOP，请问怎么实现？

在学习Spring框架的历程中，最重要的是要理解Spring的IOC和AOP了，不但要学会怎么用，最好是知道它是怎么实现的，通过这个国庆假期，好好地过了一下spring的AOP的皮毛，故记录一下学习心得。

## 一、为什么需要AOP

假如我们应用中有n个业务逻辑组件，每个业务逻辑组件又有m个方法，那现在我们的应用就一共包含了n*m个方法，我会抱怨方法太多。。。现在，我有这样一个需求，每个方法都增加一个通用的功能，常见的如：事务处理，日志，权限控制。。。最容易想到的方法，先定义一个额外的方法，实现该功能，然后再每个需要实现这个功能的地方去调用这个额外的方法。这种做法的好处和坏处分别是。
好处：可以动态地添加和删除在切面上的逻辑而不影响原来的执行代码。
坏处：一旦要修改，就要打开所有调用到的地方去修改。
好，现在我们用AOP的方式可以实现在不修改源方法代码的前提下，可以统一为原多个方法增加横切性质的“通用处理”。

## 二、什么是AOP

都说AOP好用，那现在我们来谈谈什么是AOP。

AOP（Aspect-OrientedProgramming，面向方面编程），可以说是OOP（Object-Oriented Programing，面向对象编程）的补充和完善。本文提供Spring官方文档出处：Aspect Oriented Programming with Spring

从官方文档上摘抄的解释就是：面向方面编程（AOP）是面向对象编程（OOP）补充的另一种提供思考程序结构补充。在OOP中模块化的关键单元是类，而在AOP模块的单位是一个方面。面对关注点，如事务管理跨越多个类型和对象切模块化。（这些关注经常被称为在AOP文学横切关注点。）

相关概念（只需做个大概的了解就好）----来自于官方文档直译（本人英文水平有限。。。）：

Aspect：这横切多个对象关心的模块化。事务管理是企业Java应用程序的横切关注点的一个很好的例子。在Spring AOP中，切面可以使用类（基于模式）或@Aspect注解（@AspectJ风格）注解普通班实施。
Join point：程序在执行过程中的一个点，如方法的执行或异常的处理。在Spring AOP中，一个连接点总是代表一个方法的执行。
Advice：在切面的某个特定的动作连接点。不同类型的意见，包括 "around," "before" and "after"的advice。 （通知的类型将在下面讨论）。许多AOP框架，包括Spring都是以拦截器作为通知模型，去维护一条围绕着一个连接点的拦截器链。
Pointcut：匹配连接点的断言。通知是跟一个切入点表达式，并在运行在切入点匹配的连接点相关联（例如，一个方法的执行要有一个确定的名字）。切入点表达式作为匹配的连接点的概念是重要的对AOP和Spring缺省使用AspectJ切入点表达式语言。
Introduction：声明代表的类型的额外的方法或字段。 Spring允许引入新的接口（以及一个对应的实现）到任何被代理的对象。例如，你可以使用引入来使一个bean实现IsModified接口，以便简化缓存。 （介绍被誉为AspectJ的社会类型间的声明。）
Target object：对象由一个或多个方面被建议。也被称作被通知对象。既然Spring AOP是通过运行时代理实现的，这个对象永远是一个被代理对象。
AOP proxy：AOP框架，以实现切面契约（例如通知方法执行等等）创建的对象。在Spring中，AOP代理可以是JDK动态代理或者CGLIB代理。
Weaving：与连接其他应用程序类型或对象方面来创建一个被通知的对象。这是可以做到在编译时（使用AspectJ编译器，例如），加载时间，或在运行时。 Spring AOP中，像其他纯Java AOP框架，在运行时进行编织。

以上概念个人感觉在做实验的时候就差不多理解了，不需要咬文嚼字地区啃。

那问题来了，AOP是在什么时候去改我们的代码的？即给我们加上额外的横切性质的"通用处理"的？

两个时机：

1.在编译java源代码的时候 ----编译时增强
2.在运行时动态地修改类 ----运行时增强（动态代理）

我们的Spring的AOP的实现原理就是基于动态代理。（之后我会尝试一下跟随源码看看Spring在AOP方面做了哪些事情）

## 三、Spring AOP的3种实现方式

对于框架的学习，我觉得得先会用，然后再深入原理。关于Spring AOP的实现我在这里划分成3个方式（以日志管理为例）废话不多说，直接上代码了。（以下代码是基于我之前所写的SSM框架整合的例子，如果有需要可查看我之前的博客）

配置之前注意配置文件要加上命名空间：xmlns:aop="http://www.springframework.org/schema/aop"

### 1.基于xml配置的实现

spring-mvc.xml

     	<!-- 使用xml配置aop -->
    	<!-- 强制使用cglib代理，如果不设置，将默认使用jdk的代理，但是jdk的代理是基于接口的 -->
    	<aop:config proxy-target-class="true" />  
    	<aop:config>
    	<!--定义切面-->
     	<aop:aspect id="logAspect" ref="logInterceptor">
     	<!-- 定义切入点 (配置在com.gray.user.controller下所有的类在调用之前都会被拦截)-->
      	<aop:pointcut expression="execution(* com.gray.user.controller.*.*(..))" id="logPointCut"/>
      	<!--方法执行之前被调用执行的-->
      	<aop:before method="before" pointcut-ref="logPointCut"/><!--一个切入点的引用-->
     	<aop:after method="after" pointcut-ref="logPointCut"/><!--一个切入点的引用-->
     	</aop:aspect>
    	</aop:config>

LogInterceptor.java

    package com.gray.interceptor;
     
    import org.springframework.stereotype.Component;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
     
    @Component
    public class LogInterceptor {
    	private final Logger logger = LoggerFactory.getLogger(LogInterceptor.class);
    	public void before(){
    		logger.info("login start!");
    	}
    	
    	public void after(){
    		logger.info("login end!");
    	}
    }

在这里我没有配bean是因为我在之前的配置文件里面写了自动扫描组件的配置了
要加入日志管理逻辑的地方

    @RequestMapping("/dologin.do") //url
    public String dologin(User user, Model model){
    	logger.info("login ....");
    	String info = loginUser(user);
    	if (!"SUCC".equals(info)) {
    		model.addAttribute("failMsg", "用户不存在或密码错误！");
    		return "/jsp/fail";
    	}else{
    		model.addAttribute("successMsg", "登陆成功！");//返回到页面说夹带的参数
    		model.addAttribute("name", user.getUsername());
    		return "/jsp/success";//返回的页面
    	}
      }

结果截图：

![](https://img-blog.csdn.net/20161007234356025?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### 2.基于注解的实现

spring-mvc.xml

        <aop:aspectj-autoproxy proxy-target-class="true">
        </aop:aspectj-autoproxy>

LogInterceptor.java

    @Aspect
    @Component
    public class LogInterceptor {
    	private final Logger logger = LoggerFactory.getLogger(LogInterceptor.class);
    	@Before(value = "execution(* com.gray.user.controller.*.*(..))")
    	public void before(){
    		logger.info("login start!");
    	}
    	@After(value = "execution(* com.gray.user.controller.*.*(..))")
    	public void after(){
    		logger.info("login end!");
    	}
    }

要加入逻辑的地方同上。

结果截图：

![](https://img-blog.csdn.net/20161007234418697?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### 3.基于自定义注解的实现

基于注解，所以spring-mvc.xml也是和上面的一样的。

LogInterceptor.java（这里我只加入前置日志）

    package com.gray.interceptor;
     
    import java.lang.reflect.Method;
     
    import org.aspectj.lang.JoinPoint;
    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Before;
    import org.aspectj.lang.annotation.Pointcut;
    import org.aspectj.lang.reflect.MethodSignature;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.stereotype.Component;
     
    import com.gray.annotation.Log;
     
    @Aspect
    @Component
    public class LogInterceptor {
    	private final Logger logger = LoggerFactory.getLogger(LogInterceptor.class);
     
        @Pointcut("@annotation(com.gray.annotation.Log)")    
        public void controllerAspect() {
        	
        }
        @Before("controllerAspect()")
        public void before(JoinPoint joinPoint){
    		logger.info(getOper(joinPoint));
    	}
    	private String getOper(JoinPoint joinPoint) {
    		MethodSignature methodName = (MethodSignature)joinPoint.getSignature();
    		Method method = methodName.getMethod();
    		return method.getAnnotation(Log.class).oper();
    	}
    }

同时，加入逻辑的地方需要加入Log注解

    @RequestMapping("/dologin.do") //url
    @Log(oper="user login")
    public String dologin(User user, Model model){
    	logger.info("login ....");
    	String info = loginUser(user);
    	if (!"SUCC".equals(info)) {
    		model.addAttribute("failMsg", "用户不存在或密码错误！");
    		return "/jsp/fail";
    	}else{
    		model.addAttribute("successMsg", "登陆成功！");//返回到页面说夹带的参数
    		model.addAttribute("name", user.getUsername());
    		return "/jsp/success";//返回的页面
    	}
      }

结果截图：

![](https://img-blog.csdn.net/20161007234434923?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


以上就是我总结的SpringAOP的3种实现方式，对于这我还有点话要说，常见的基于注解的方式除了before和after，还有around和AfterThrowing等，在做第三种方式实验的时候还遇到这种错误：error at ::0 can't find referenced pointcut。

我的解决办法是：因为我的jdk是1.7，与原来在pom.xml中配的那个aspectjweaver的jar包版本不匹配，所以我需要升级，由1.5.4改成1.7.4。

此外多说一句，对于日志，权限等业务逻辑很多人其实也喜欢用拦截器来实现的。

#  8、Spring 是如何管理事务的，事务管理机制？

Spring事务机制主要包括**声明式事务和编程式事务**，此处侧重讲解声明式事务，编程式事务在实际开发中得不到广泛使用，仅供学习参考。

Spring声明式事务让我们从复杂的事务处理中得到解脱。使得我们**再也无需要去处理获得连接、关闭连接、事务提交和回滚等这些操作**。再也无需要我们在与事务相关的方法中处理大量的try…catch…finally代码。我们在使用Spring声明式事务时，有一个非常重要的概念就是事务属性。**事务属性通常由事务的传播行为，事务的隔离级别，事务的超时值和事务只读标志组成**。我们在进行事务划分时，需要进行事务定义，也就是配置事务的属性。

**下面分别详细讲解，事务的四种属性，仅供诸位学习参考：**

Spring在TransactionDefinition接口中定义这些属性,以供PlatfromTransactionManager使用, **PlatfromTransactionManager是Spring事务管理的核心接口**。

```
public interface TransactionDefinition { 
  int getPropagationBehavior(); //返回事务的传播行为。 
  int getIsolationLevel(); //返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据。 
  int getTimeout(); //返回事务必须在多少秒内完成。 
  boolean isReadOnly(); //事务是否只读，事务管理器能够根据这个返回值进行优化，确保事务是只读的。 
}
```

## 1、TransactionDefinition接口中定义五个隔离级别：**

> **ISOLATION_DEFAULT 这是一个PlatfromTransactionManager默认的隔离级别**，使用数据库默认的事务隔离级别.另外四个与JDBC的隔离级别相对应；
>
> **ISOLATION_READ_UNCOMMITTED 这是事务最低的隔离级别**，它充许别外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读。
>
> **ISOLATION_READ_COMMITTED 保证一个事务修改的数据提交后才能被另外一个事务读取**。另外一个事务不能读取该事务未提交的数据。这种事务隔离级别可以避免脏读出现，但是可能会出现不可重复读和幻像读。
>
> **ISOLATION_REPEATABLE_READ 这种事务隔离级别可以防止脏读，不可重复读**。但是可能出现幻像读。它除了保证一个事务不能读取另一个事务未提交的数据外，还保证了避免下面的情况产生(不可重复读)。
>
> **ISOLATION_SERIALIZABLE 这是花费最高代价但是最可靠的事务隔离级别。**事务被处理为顺序执行。除了防止脏读，不可重复读外，还避免了幻像读。

## 2、**在TransactionDefinition接口中定义了七个事务传播行为：**

（1）**PROPAGATION_REQUIRED 如果存在一个事务，则支持当前事务**。如果没有事务则开启一个新的事务。

```
// 事务属性 PROPAGATION_REQUIRED 
methodA { 
   …… 
   methodB();
   …… 
}
// 事务属性 PROPAGATION_REQUIRED 
methodB { 
   …… 
}
```

使用Spring声明式事务，spring使用AOP来支持声明式事务，会根据事务属性，自动在方法调用之前决定是否开启一个事务，并在方法执行之后决定事务提交或回滚事务。

**单独调用methodB方法：**

```
main { 
   metodB(); 
}
相当于
Main {
   Connection con=null; 
   try{ 
     con = getConnection(); 
     con.setAutoCommit(false); 
     //方法调用
     methodB(); 
     //提交事务
     con.commit(); 
   } Catch(RuntimeException ex) { 
     //回滚事务
     con.rollback();   
   } finally { 
     //释放资源
     closeCon(); 
   } 
}
```

Spring保证在methodB方法中所有的调用都获得到一个相同的连接。在调用methodB时，**没有一个存在的事务，所以获得一个新的连接，开启了一个新的事务**。

**单独调用MethodA时，在MethodA内又会调用MethodB。执行效果相当于：**

```
Main { 
   Connection con = null; 
   try { 
     con = getConnection(); 
     methodA(); 
     con.commit(); 
   } catch(RuntimeException ex) { 
     con.rollback(); 
   } finally { 
     closeCon(); 
   }  
}
```

调用MethodA时，环境中没有事务，所以开启一个新的事务.**当在MethodA中调用MethodB时，环境中已经有了一个事务，所以methodB就加入当前事务**。

（2）**PROPAGATION_SUPPORTS 如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。**但是对于事务同步的事务管理器，PROPAGATION_SUPPORTS与不使用事务有少许不同。

```
// 事务属性 PROPAGATION_REQUIRED 
methodA() {   
   methodB(); 
}
// 事务属性 PROPAGATION_SUPPORTS 
methodB() {
   ……
}
```

单纯的调用methodB时，methodB方法是非事务的执行的。**当调用methdA时,methodB则加入了methodA的事务中执行**。

（3）**PROPAGATION_MANDATORY 如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常**。

```
// 事务属性 PROPAGATION_REQUIRED
methodA() {
   methodB();
}
//事务属性 PROPAGATION_MANDATORY     
methodB() {     
   ……
}
```

当单独调用methodB时，**因为当前没有一个活动的事务，则会抛出异常throw new IllegalTransactionStateException("Transaction propagation 'mandatory' but no existing transaction found");** 当调用methodA时，methodB则加入到methodA的事务中执行。

（4）**PROPAGATION_REQUIRES_NEW 总是开启一个新的事务**。如果一个事务已经存在，则将这个存在的事务挂起。

```
//事务属性 PROPAGATION_REQUIRED 
methodA() {    
   doSomeThingA(); 
   methodB(); 
   doSomeThingB(); 
}
//事务属性 PROPAGATION_REQUIRES_NEW 
methodB() {
   …… 
}
main() {
   methodA(); 
}
```

相当于：

```
main() {
   TransactionManager tm = null; 
   try {   
     // 获得一个JTA事务管理器     
     tm = getTransactionManager();
     tm.begin(); 
     // 开启一个新的事务
     Transaction ts1 = tm.getTransaction();
     doSomeThing();     
     tm.suspend(); // 挂起当前事务     
     try {
       tm.begin();// 重新开启第二个事务       
       Transaction ts2 = tm.getTransaction();       
       methodB();       
       ts2.commit();// 提交第二个事务    
     } catch(RunTimeException ex) {
        ts2.rollback(); // 回滚第二个事务   
     } finally {      
        // 释放资源
     }
     // methodB执行完后，复恢第一个事务     
     tm.resume(ts1); 
     doSomeThingB();     
     ts1.commit();// 提交第一个事务 
   } catch(RunTimeException ex) {
     ts1.rollback();// 回滚第一个事务 
   } finally {
     //释放资源 
   } 
}
```

在这里，我把**ts1称为外层事务，ts2称为内层事务**。从上面的代码可以看出，**ts2与ts1是两个独立的事务，互不相干。Ts2是否成功并不依赖于ts1**。如果methodA方法在调用methodB方法后的doSomeThingB方法失败了，而methodB方法所做的结果依然被提交。而除了methodB之外的其它代码导致的结果却被回滚了。**使用PROPAGATION_REQUIRES_NEW,需要使用JtaTransactionManager作为事务管理器**。

（5）**PROPAGATION_NOT_SUPPORTED 总是非事务地执行，并挂起任何存在的事务**。使用PROPAGATION_NOT_SUPPORTED,也需要使用JtaTransactionManager作为事务管理器。

（6）**PROPAGATION_NEVER 总是非事务地执行，如果存在一个活动事务，则抛出异常；**

（7）**PROPAGATION_NESTED如果一个活动的事务存在，则运行在一个嵌套的事务中. 如果没有活动事务, 则按TransactionDefinition.PROPAGATION_REQUIRED 属性执行**。这是一个嵌套事务,使用JDBC 3.0驱动时,仅仅支持DataSourceTransactionManager作为事务管理器。需要JDBC 驱动的java.sql.Savepoint类。有一些JTA的事务管理器实现可能也提供了同样的功能。使用PROPAGATION_NESTED，还需要把PlatformTransactionManager的nestedTransactionAllowed属性设为true;而nestedTransactionAllowed属性值默认为false;

```
// 事务属性 PROPAGATION_REQUIRED 
methodA() {
   doSomeThingA();
   methodB();
   doSomeThingB(); 
}
//事务属性 PROPAGATION_NESTED 
methodB() {
   …… 
}
```

**如果单独调用methodB方法，则按REQUIRED属性执行**。如果调用methodA方法，相当于下面的效果：

```
main() { 
    Connection con = null; 
    Savepoint savepoint = null; 
    try{
      con = getConnection();
      con.setAutoCommit(false);
      doSomeThingA();
      savepoint = con2.setSavepoint();
      try{
          methodB();
      } catch(RuntimeException ex) {
          con.rollback(savepoint);
      } finally {
          //释放资源   
      }
      doSomeThingB();
      con.commit(); 
    } catch(RuntimeException ex) {
      con.rollback(); 
    } finally {
      //释放资源
    } 
}
```

当methodB方法调用之前，调用setSavepoint方法，保存当前的状态到savepoint。如果methodB方法调用失败，则恢复到之前保存的状态。但是需要注意的是，这时的事务并没有进行提交，如果后续的代码(doSomeThingB()方法)调用失败，则回滚包括methodB方法的所有操作。

**嵌套事务一个非常重要的概念就是内层事务依赖于外层事务。外层事务失败时，会回滚内层事务所做的动作。而内层事务操作失败并不会引起外层事务的回滚**。

**PROPAGATION_NESTED 与PROPAGATION_REQUIRES_NEW的区别:**

> 它们非常类似,都像一个嵌套事务，如果不存在一个活动的事务，都会开启一个新的事务。**使用PROPAGATION_REQUIRES_NEW时，内层事务与外层事务就像两个独立的事务一样**，一旦内层事务进行了提交后，外层事务不能对其进行回滚。两个事务互不影响。两个事务不是一个真正的嵌套事务。同时它需要JTA事务管理器的支持。
>
> 使用PROPAGATION_NESTED时，**外层事务的回滚可以引起内层事务的回滚。而内层事务的异常并不会导致外层事务的回滚，它是一个真正的嵌套事务**。DataSourceTransactionManager使用savepoint支持PROPAGATION_NESTED时，需要JDBC 3.0以上驱动及1.4以上的JDK版本支持。其它的JTA TrasactionManager实现可能有不同的支持方式。
>
> **PROPAGATION_REQUIRES_NEW 启动一个新的, 不依赖于环境的 "内部" 事务**。这个事务将被完全 commited 或 rolled back 而不依赖于外部事务, 它拥有自己的隔离范围, 自己的锁, 等等. 当内部事务开始执行时, 外部事务将被挂起, 内务事务结束时, 外部事务将继续执行。
>
> 另一方面, PROPAGATION_NESTED 开始一个 "嵌套的" 事务,  它是已经存在事务的一个真正的子事务. **潜套事务开始执行时, 它将取得一个 savepoint. 如果这个嵌套事务失败, 我们将回滚到此 savepoint**. 潜套事务是外部事务的一部分, 只有外部事务结束后它才会被提交。
>
> 由此可见, PROPAGATION_REQUIRES_NEW 和 PROPAGATION_NESTED 的最大区别在于, **PROPAGATION_REQUIRES_NEW 完全是一个新的事务, 而 PROPAGATION_NESTED 则是外部事务的子事务, 如果外部事务 commit, 潜套事务也会被 commit, 这个规则同样适用于 roll back**。PROPAGATION_REQUIRED应该是我们首先的事务传播行为。它能够满足我们大多数的事务需求。

#  9、Spring 的不同事务传播行为有哪些，干什么用的？

1、PROPAGATION_REQUIRED：如果当前没有事务，就创建一个新事务，如果当前存在事务，就加入该事务，该设置是最常用的设置。

2、PROPAGATION_SUPPORTS：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就以非事务执行。‘

3、PROPAGATION_MANDATORY：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就抛出异常。

4、PROPAGATION_REQUIRES_NEW：创建新事务，无论当前存不存在事务，都创建新事务。

5、PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。

6、PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。

7、PROPAGATION_NESTED：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。

罗列了7种spring的事务传播行为，我们具体来看看它的实现。在这里，我们使用spring annotation注解实现事务。

**实现事务的类BusinessServiceImpl**

    package com.aop;
     
    import org.springframework.aop.support.AopUtils;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Isolation;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;
     
    import com.entity.Student;
     
    @Service("businessSerivce")
    public class BusinessServiceImpl implements IBaseService {
     
    	@Autowired
    	IStudentDao studentDao;
     
    	@Autowired
    	IBaseServiceB baseServiceb;
     
    	@Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.DEFAULT, rollbackFor = Exception.class)
    	public String doA() throws Exception {
    		Student st = new Student();
    		st.setId(1);
    		st.setSex("girl");
    		st.setUsername("zx");
    		studentDao.insertStudent(st);
     
    		System.out.println(baseServiceb);
    		System.out.println("是否是代理调用，AopUtils.isAopProxy(baseServiceb) : " + AopUtils.isAopProxy(baseServiceb));
    		System.out
    				.println("是否是cglib类代理调用，AopUtils.isCglibProxy(baseServiceb) : " + AopUtils.isCglibProxy(baseServiceb));
    		System.out.println("是否是jdk动态接口代理调用，AopUtils.isJdkDynamicProxy(baseServiceb) : "
    				+ AopUtils.isJdkDynamicProxy(baseServiceb));
    		
    		//使用代理调用方法doB()
    		baseServiceb.doB();
    		int i = 1 / 0;// 抛出异常，doB()的事务事务回滚
    		return "success";
    	}
     
    	// @Transactional(propagation = Propagation.REQUIRES_NEW, isolation =
    	// Isolation.DEFAULT, rollbackFor = Exception.class)
    	// public String doB() throws Exception {
    	// Student st = new Student();
    	// st.setId(2);
    	// st.setSex("girl");
    	// st.setUsername("zx2");
    	// studentDao.insertStudent(st);
    	//
    	// return "success";
    	// }
     
    }

**实现事务的类BusinessServiceImpl**

    package com.aop;
     
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Isolation;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;
     
    import com.entity.Student;
     
    @Service("businessSerivceB")
    public class BusinessServiceImplB implements IBaseServiceB {
      
    	@Autowired
    	IStudentDao studentDao;
    	
    	@Transactional(propagation = Propagation.REQUIRES_NEW,isolation=Isolation.DEFAULT,rollbackFor=Exception.class)  
    	public String doB() throws Exception {
    		Student st = new Student();
    		st.setId(2);
    		st.setSex("girl");
    		st.setUsername("zx2");
    		studentDao.insertStudent(st);
    		return "success";
    	}
    	 
    }

**测试类**：

    package com.aop;
     
    import org.springframework.beans.factory.BeanFactory;
    import org.springframework.context.support.ClassPathXmlApplicationContext;
     
    import com.thread.service.IBaseFacadeService;


​     
    //@RunWith(SpringJUnit4ClassRunner.class)
    //@ContextConfiguration(locations = { "classpath:/spring-ibatis.xml", "classpath:/spring-jdbctemplate.xml" })
    public class TestStudentDao {
     
    	public static void main(String[] args) {
    		  try {
    		  BeanFactory factory = new ClassPathXmlApplicationContext("spring-jdbctemplate.xml");
    		  IBaseService service = (IBaseService) factory.getBean("businessSerivce");
    		
    			  service.doA();
    		} catch (Exception e) {
    			// TODO Auto-generated catch block
    			e.printStackTrace();
    		}
    	}
    }


测试结果：

om.aop.BusinessServiceImplB@14b75fd9
是否是代理调用，AopUtils.isAopProxy(baseServiceb) : true
是否是cglib类代理调用，AopUtils.isCglibProxy(baseServiceb) : true
是否是jdk动态接口代理调用，AopUtils.isJdkDynamicProxy(baseServiceb) : false
java.lang.ArithmeticException: / by zero

![](https://img-blog.csdn.net/20161019162051999?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


从测试结果可以看到，我们在数据库中成功插入了doB()方法中要插入的数据，而doA()中插入的数据被回滚了，这是因为我们在doB()方法中加入ROPAGATION_REQUIRES_NEW传播行为，doB()创建了属于自己的事务，挂起了doA()中的事务，所以事务B提交了，而事务A因为执行异常插入的数据被回滚了。

如果我们将BusinessServiceImplB做下修改，改为第一种传播行为ROPAGATION_REQUIRED

    package com.aop;
     
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Isolation;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;
     
    import com.entity.Student;
     
    @Service("businessSerivceB")
    public class BusinessServiceImplB implements IBaseServiceB {
      
    	@Autowired
    	IStudentDao studentDao;
    	
    	@Transactional(propagation = Propagation.REQUIRED,isolation=Isolation.DEFAULT,rollbackFor=Exception.class)  
    	public String doB() throws Exception {
    		Student st = new Student();
    		st.setId(2);
    		st.setSex("girl");
    		st.setUsername("zx2");
    		studentDao.insertStudent(st);
    		return "success";
    	}
    	 
    }


测试结果：

com.aop.BusinessServiceImplB@97c87a4
是否是代理调用，AopUtils.isAopProxy(baseServiceb) : true
是否是cglib类代理调用，AopUtils.isCglibProxy(baseServiceb) : true
是否是jdk动态接口代理调用，AopUtils.isJdkDynamicProxy(baseServiceb) : false
java.lang.ArithmeticException: / by zero

![](https://img-blog.csdn.net/20161019162807457?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


从结果可以看到，我们没有成功插入数据，这是为什么呢？因为我们使用的是第一种传播行为PROPAGATION_REQUIRED ，doB()方法被加入到doA()的事务中，doA()执行时抛出了异常，因为doB()和doA()同属于一个事务，则执行操作被一起回滚了。其实在doB()中我们不加入注解，也等同PROPAGATION_REQUIRED的效果。

接下来我们再来看看第五种传播行为PROPAGATION_NOT_SUPPORTED，我们同样修改B类，如下

    package com.aop;
     
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Isolation;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;
     
    import com.entity.Student;
     
    @Service("businessSerivceB")
    public class BusinessServiceImplB implements IBaseServiceB {
      
    	@Autowired
    	IStudentDao studentDao;
    	
    	@Transactional(propagation = Propagation.NOT_SUPPORTED,isolation=Isolation.DEFAULT,rollbackFor=Exception.class)  
    	public String doB() throws Exception {
    		Student st = new Student();
    		st.setId(2);
    		st.setSex("girl");
    		st.setUsername("zx2");
    		studentDao.insertStudent(st);
    		return "success";
    	}
    	 
    }

测试结果：

![](https://img-blog.csdn.net/20161019163439643?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


可以看到doB()方法成功插入了数据。doA()方法中插入的数据被回滚了。这是因为传播行为PROPAGATION_NOT_SUPPORTED是doB()以非事务执行的，并且提交了。所以当doA()的事务被回滚时，doB()的操作没有被回滚。

其他的传播行为就不一一列举了，机制是差不多的。

注意：在上面的实例中，我没有把doB()方法放在类BusinessServiceImpl，而是放在BusinessServiceImplB中，这是因为spring通过扫描所有含有注解的@Trasation的方法，使用aop形成事务增强advise。但是加入增强时是通过代理对象调用方法的形式加入的，如果将doB()方法放在doA()方法直接调用时，在调用doB()方法的时候是通过当前对象来调用doB()方法的，而不是通过代理来调用的doB()方法，这个时候doB()方法上加的事务注解就失效了不起作用。在Spring事务传播行为在内部方法不起作用讲到。

spring配置文件：

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
    	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
    	xmlns:tx="http://www.springframework.org/schema/tx" xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    	xmlns:context="http://www.springframework.org/schema/context"
    	xmlns:aop="http://www.springframework.org/schema/aop"
    	xsi:schemaLocation="
                http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd 
                http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd 
                http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
            	http://www.springframework.org/schema/tx  http://www.springframework.org/schema/tx/spring-tx-3.0.xsd
            	http://www.springframework.org/schema/jdbc http://www.springframework.org/schema/jdbc/spring-jdbc-3.0.xsd"
            	default-lazy-init="false">
     
    	<context:component-scan base-package="com.aop"/>
    	<aop:aspectj-autoproxy proxy-target-class="true"/>
    	<!-- 数据 -->
    	<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource">
    		<property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    		<property name="url" value="jdbc:mysql://localhost:3306/lpWeb"/>
    		<property name="username" value="root"/>
    		<property name="password" value="root123"/>
    	</bean>
    	
    	<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    	 	<property name = "dataSource" ref="dataSource"/>  
    	</bean>
      
    	 <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    		<property name="dataSource" ref="dataSource"/>
    	</bean> 
    	
    	<tx:annotation-driven transaction-manager="transactionManager" />  
    </beans>



#  10、Spring 中用到了那些设计模式？

spring中用到了哪些设计模式？（顺丰）

 

spring中常用的设计模式达到九种，我们举例说明：



## **第一种：简单工厂**

又叫做静态工厂方法（StaticFactory Method）模式，但不属于23种GOF设计模式之一。 
简单工厂模式的实质是由一个工厂类根据传入的参数，动态决定应该创建哪一个产品类。 
spring中的BeanFactory就是简单工厂模式的体现，根据传入一个唯一的标识来获得bean对象，但是否是在传入参数后创建还是传入参数前创建这个要根据具体情况来定。如下配置，就是在 HelloItxxz 类中创建一个 itxxzBean。

```
<beans>

​    <bean id="singletonBean" class="com.itxxz.HelloItxxz">

​        <constructor-arg>

​            <value>Hello! 这是singletonBean!value>

​        </constructor-arg>

   </ bean>

 

​    <bean id="itxxzBean" class="com.itxxz.HelloItxxz"

​        singleton="false">

​        <constructor-arg>

​            <value>Hello! 这是itxxzBean! value>

​        </constructor-arg>

​    </bean>

 

</beans>
```



## **第二种：工厂方法（Factory Method****）**



通常由应用程序直接使用new创建新的对象，为了将对象的创建和使用相分离，采用工厂模式,即应用程序将对象的创建及初始化职责交给工厂对象。

一般情况下,应用程序有自己的工厂对象来创建bean.如果将应用程序自己的工厂对象交给Spring管理,那么Spring管理的就不是普通的bean,而是工厂Bean。

螃蟹就以工厂方法中的静态方法为例讲解一下： 

```
import java.util.Random;

public class StaticFactoryBean {

​      public static Integer createRandom() {

​           return new Integer(new Random().nextInt());

​       }

}
```

建一个config.xm配置文件，将其纳入Spring容器来管理,需要通过factory-method指定静态方法名称

```
<bean id="random"

class="example.chapter3.StaticFactoryBean" factory-method="createRandom" //createRandom方法必须是static的,才能找到 scope="prototype"

/>
```

测试:

```
public static void main(String[] args) {
       //调用getBean()时,返回随机数.如果没有指定factory-method,会返回StaticFactoryBean的实例,即返回工厂Bean的实例       
       XmlBeanFactory factory = new XmlBeanFactory(new  ClassPathResource("config.xml"));        System.out.println("我是IT学习者创建的实例:"+factory.getBean("random").toString());

}
```



## **第三种：单例模式（Singleton****）**



当我们试图从Spring容器中取得某个类的实例时，默认情况下，Spring会才用单例模式进行创建。

如果我不想使用默认的单例模式，每次请求我都希望获得一个新的对象怎么办呢？很简单，将scope属性值设置为prototype(原型)就可以了

```
<bean id="date" class="java.util.Date" scope="prototype"/>
```

通过以上配置信息，Spring就会每次给客户端返回一个新的对象实例。
那么Spring对单例的底层实现，到底是饿汉式单例还是懒汉式单例呢？

Spring框架对单例的支持是采用单例注册表的方式进行实现的



## **第四种：适配器（Adapter****）**

在Spring的Aop中，使用的Advice（通知）来增强被代理类的功能。Spring实现这一AOP功能的原理就使用代理模式（1、JDK动态代理。2、CGLib字节码生成技术代理。）对类进行方法级别的切面增强，即，生成被代理类的代理类，  并在代理类的方法前，设置拦截器，通过执行拦截器重的内容增强了代理方法的功能，实现的面向切面编程。

**Adapter类接口**：Target

```
public interface AdvisorAdapter {

 

boolean supportsAdvice(Advice advice);

 

​      MethodInterceptor getInterceptor(Advisor advisor);

 

} **MethodBeforeAdviceAdapter类**，Adapter

class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

 

​      public boolean supportsAdvice(Advice advice) {

​            return (advice instanceof MethodBeforeAdvice);

​      }

 

​      public MethodInterceptor getInterceptor(Advisor advisor) {

​            MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();

​      return new MethodBeforeAdviceInterceptor(advice);

​      }

 

}
```



## **第五种：包装器（Decorator****）**

在我们的项目中遇到这样一个问题：我们的项目需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访问不同的数据库。我们以往在spring和hibernate框架中总是配置一个数据源，因而sessionFactory的dataSource属性总是指向这个数据源并且恒定不变，所有DAO在使用sessionFactory的时候都是通过这个数据源访问数据库。但是现在，由于项目的需要，我们的DAO在访问sessionFactory的时候都不得不在多个数据源中不断切换，问题就出现了：如何让sessionFactory在执行数据持久化的时候，根据客户的需求能够动态切换不同的数据源？我们能不能在spring的框架下通过少量修改得到解决？是否有什么设计模式可以利用呢？ 
首先想到在spring的applicationContext中配置所有的dataSource。这些dataSource可能是各种不同类型的，比如不同的数据库：Oracle、SQL   Server、MySQL等，也可能是不同的数据源：比如apache 提供的org.apache.commons.dbcp.BasicDataSource、spring提供的org.springframework.jndi.JndiObjectFactoryBean等。然后sessionFactory根据客户的每次请求，将dataSource属性设置成不同的数据源，以到达切换数据源的目的。
spring中用到的包装器模式在类名上有两种表现：一种是类名中含有Wrapper，另一种是类名中含有Decorator。基本上都是动态地给一个对象添加一些额外的职责。 

## **第六种：代理（Proxy****）**

为其他对象提供一种代理以控制对这个对象的访问。  从结构上来看和Decorator模式类似，但Proxy是控制，更像是一种对功能的限制，而Decorator是增加职责。 
spring的Proxy模式在aop中有体现，比如JdkDynamicAopProxy和Cglib2AopProxy。 

## **第七种：观察者（Observer****）**

定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。
spring中Observer模式常用的地方是listener的实现。如ApplicationListener。 

## **第八种：策略（Strategy****）**

定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换。本模式使得算法可独立于使用它的客户而变化。 
spring中在实例化对象的时候用到Strategy模式
在SimpleInstantiationStrategy中有如下代码说明了策略模式的使用情况： 
![点击查看原始大小图片](http://dl.iteye.com/upload/attachment/356172/4a3dfd16-5d4f-3c32-ae11-d7f5d774c2dd.jpg) 

## **第九种：模板方法（Template Method****）**

定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。Template Method使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。
Template  Method模式一般是需要继承的。这里想要探讨另一种对Template  Method的理解。spring中的JdbcTemplate，在用这个类时并不想去继承这个类，因为这个类的方法太多，但是我们还是想用到JdbcTemplate已有的稳定的、公用的数据库连接，那么我们怎么办呢？我们可以把变化的东西抽出来作为一个参数传入JdbcTemplate的方法中。但是变化的东西是一段代码，而且这段代码会用到JdbcTemplate中的变量。怎么办？那我们就用回调对象吧。在这个回调对象中定义一个操纵JdbcTemplate中变量的方法，我们去实现这个方法，就把变化的东西集中到这里了。然后我们再传入这个回调对象到JdbcTemplate，从而完成了调用。这可能是Template  Method不需要继承的另一种实现方式吧。 

以下是一个具体的例子： 
JdbcTemplate中的execute方法 
![点击查看原始大小图片](http://dl.iteye.com/upload/attachment/356951/a1266e58-6822-3178-b0d2-2d3825945099.jpg) 
JdbcTemplate执行execute方法 
![点击查看原始大小图片](http://dl.iteye.com/upload/attachment/356949/17993bf3-35de-38cd-abb7-86d6a98ba127.jpg)

 

#  11、Spring MVC 的工作原理？

## 组件

### **1、前端控制器DispatcherServlet（不需要工程师开发）,由框架提供**

作用：接收请求，响应结果，相当于转发器，中央处理器。有了dispatcherServlet减少了其它组件之间的耦合度。
用户请求到达前端控制器，它就相当于mvc模式中的c，dispatcherServlet是整个流程控制的中心，由它调用其它组件处理用户的请求，dispatcherServlet的存在降低了组件之间的耦合性。

### **2、处理器映射器HandlerMapping(不需要工程师开发),由框架提供**

作用：根据请求的url查找Handler
HandlerMapping负责根据用户请求找到Handler即处理器，springmvc提供了不同的映射器实现不同的映射方式，例如：配置文件方式，实现接口方式，注解方式等。

### **3、处理器适配器HandlerAdapter**

作用：按照特定规则（HandlerAdapter要求的规则）去执行Handler
通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。

### **4、处理器Handler(需要工程师开发)**

**注意：编写Handler时按照HandlerAdapter的要求去做，这样适配器才可以去正确执行Handler**
Handler 是继DispatcherServlet前端控制器的后端控制器，在DispatcherServlet的控制下Handler对具体的用户请求进行处理。
由于Handler涉及到具体的用户业务请求，所以一般情况需要工程师根据业务需求开发Handler。

### **5、视图解析器View resolver(不需要工程师开发),由框架提供**

作用：进行视图解析，根据逻辑视图名解析成真正的视图（view）
View  Resolver负责将处理结果生成View视图，View  Resolver首先根据逻辑视图名解析成物理视图名即具体的页面地址，再生成View视图对象，最后对View进行渲染将处理结果通过页面展示给用户。  springmvc框架提供了很多的View视图类型，包括：jstlView、freemarkerView、pdfView等。
一般情况下需要通过页面标签或页面模版技术将模型数据通过页面展示给用户，需要由工程师根据业务需求开发具体的页面。

### **6、视图View(需要工程师开发jsp...)**

View是一个接口，实现类支持不同的View类型（jsp、freemarker、pdf...）

## **核心架构的具体流程步骤如下：**

1、首先用户发送请求——>DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他的解析器进行处理，作为统一访问点，进行全局的流程控制；
2、DispatcherServlet——>HandlerMapping，  HandlerMapping 将会把请求映射为HandlerExecutionChain 对象（包含一个Handler  处理器（页面控制器）对象、多个HandlerInterceptor 拦截器）对象，通过这种策略模式，很容易添加新的映射策略；
3、DispatcherServlet——>HandlerAdapter，HandlerAdapter 将会把处理器包装为适配器，从而支持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；
4、HandlerAdapter——>处理器功能处理方法的调用，HandlerAdapter 将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理；并返回一个ModelAndView 对象（包含模型数据、逻辑视图名）；
5、ModelAndView的逻辑视图名——> ViewResolver， ViewResolver 将把逻辑视图名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；
6、View——>渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术；
7、返回控制权给DispatcherServlet，由DispatcherServlet返回响应给用户，到此一个流程结束。

下边两个组件通常情况下需要开发：

Handler：处理器，即后端控制器用controller表示。

View：视图，即展示给用户的界面，视图中通常需要标签语言展示模型数据。

 

**在将SpringMVC之前我们先来看一下什么是MVC模式**

MVC：MVC是一种设计模式

MVC的原理图：

![img](https://images2015.cnblogs.com/blog/249993/201702/249993-20170207135959401-404841652.png)

**分析：**

M-Model 模型（完成业务逻辑：有javaBean构成，service+dao+entity）

V-View 视图（做界面的展示  jsp，html……）

C-Controller 控制器（接收请求—>调用模型—>根据结果派发页面）

 

**springMVC是什么：** 

　　springMVC是一个MVC的开源框架，springMVC=struts2+spring，springMVC就相当于是Struts2加上sring的整合，但是这里有一个疑惑就是，springMVC和spring是什么样的关系呢？这个在百度百科上有一个很好的解释：意思是说，springMVC是spring的一个后续产品，其实就是spring在原有基础上，又提供了web应用的MVC模块，可以简单的把springMVC理解为是spring的一个模块（类似AOP，IOC这样的模块），网络上经常会说springMVC和spring无缝集成，其实springMVC就是spring的一个子模块，所以根本不需要同spring进行整合。

**SpringMVC的原理图：**

**![img](https://images2015.cnblogs.com/blog/249993/201702/249993-20170207140151791-1932120070.png)**

**看到这个图大家可能会有很多的疑惑，现在我们来看一下这个图的步骤：（可以对比MVC的原理图进行理解）**

第一步:用户发起请求到前端控制器（DispatcherServlet）

第二步：前端控制器请求处理器映射器（HandlerMappering）去查找处理器（Handle）：通过xml配置或者注解进行查找

第三步：找到以后处理器映射器（HandlerMappering）像前端控制器返回执行链（HandlerExecutionChain）

第四步：前端控制器（DispatcherServlet）调用处理器适配器（HandlerAdapter）去执行处理器（Handler）

第五步：处理器适配器去执行Handler

第六步：Handler执行完给处理器适配器返回ModelAndView

第七步：处理器适配器向前端控制器返回ModelAndView

第八步：前端控制器请求视图解析器（ViewResolver）去进行视图解析

第九步：视图解析器像前端控制器返回View

第十步：前端控制器对视图进行渲染

第十一步：前端控制器向用户响应结果

**看到这些步骤我相信大家很感觉非常的乱，这是正常的，但是这里主要是要大家理解springMVC中的几个组件：**

前端控制器（DispatcherServlet）：接收请求，响应结果，相当于电脑的CPU。

处理器映射器（HandlerMapping）：根据URL去查找处理器

处理器（Handler）：（需要程序员去写代码处理逻辑的）

处理器适配器（HandlerAdapter）：会把处理器包装成适配器，这样就可以支持多种类型的处理器，类比笔记本的适配器（适配器模式的应用）

视图解析器（ViewResovler）：进行视图解析，多返回的字符串，进行处理，可以解析成对应的页面

#  12、Spring 循环注入的原理？

## 什么是循环依赖

循环依赖就是循环引用，就是两个或多个Bean相互之间的持有对方，比如A引用B，B引用C，C引用A，则它们最终反映为一个环。

spring 中循环依赖注入分三种情况

1. 构造器循环依赖
2. setter方法循环注入
    2.1 setter方法注入 单例模式(scope=singleton)
    2.2 setter方法注入 非单例模式

------

我们首先创造3个互相依赖的bean类

**A.java**

```
public class A {
    private B b;
    
    public A(){}
    public A(B b){ this.b = b; }

    public B getB() { return b; }
    public void setB(B b) { this.b = b; }

    public void hello(){ b.doHello(); }
    
    public void doHello(){
        System.out.println("I am A");
    }
}
```

**B.java**

```
public class B {
    private C c;
    
    public B(){}
    public B(C c){ this.c = c; }

    public C getC() { return c; }
    public void setC(C c) { this.c = c; }
    
    public void hello(){ c.doHello(); }
    
    public void doHello(){
        System.out.println("I am B");
    }
}
```

**C.java**

```
public class C {
    private A a;
    
    public C(){}
    public C(A a){ this.a = a; }

    public A getA() { return a; }
    public void setA(A a) { this.a = a; }
    
    public void hello(){ a.doHello(); }
    
    public void doHello(){
        System.out.println("I am C");
    }
}
```

**执行类SpringMain.java**

```
public class SpringMain {
    public static void main(String[] args) {
        ApplicationContext ac = new ClassPathXmlApplicationContext("bean-circle.xml");
        A a = A.class.cast(ac.getBean("a"));
        a.hello();
    }
}
```

## 1. 构造器循环依赖

表示通过构造器注入构成的循环依赖，此依赖是无法解决的，只能抛出BeanCurrentlyInCreationException异常表示循环依赖。

- 在创建A类时，构造器需要B类，那将去创建B，
- 在创建B类时又发现需要C类，则又去创建C，
- 最后在创建C时发现又需要A；从而形成一个环，没办法创建。

Spring容器将每一个正在创建的Bean 标识符放在一个“当前创建Bean池”中，Bean标识符在创建过程中将一直保持在这个池中，因此如果在创建Bean过程中发现自己已经在“当前创建Bean池”里时将抛出BeanCurrentlyInCreationException异常表示循环依赖；而对于创建完毕的Bean将从“当前创建Bean池”中清除掉。

```
<bean id="a" class="cn.com.infcn.test.A">
    <constructor-arg ref="b" />
</bean>
<bean id="b" class="cn.com.infcn.test.B">
    <constructor-arg ref="c" />
</bean>
<bean id="c" class="cn.com.infcn.test.C">
    <constructor-arg ref="a" />
</bean>
```

执行SpringMain.main()方法报错





![img](https:////upload-images.jianshu.io/upload_images/2843224-8d4d969ce5fd6a3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)



## 2. setter方法循环注入

setter循环依赖：表示通过setter注入方式构成的循环依赖。
 对于setter注入造成的依赖是通过Spring容器提前暴露刚完成构造器注入但未完成其他步骤（如setter注入）的Bean来完成的，而且只能解决单例作用域的Bean循环依赖。

### 2.1 setter方法注入 单例模式 (scope="singleton")

具体步骤如下：

1. Spring容器创建单例“A” Bean，首先根据无参构造器创建Bean，并暴露一个“ObjectFactory ”用于返回一个提前暴露一个创建中的Bean，并将“A” 标识符放到“当前创建Bean池”；然后进行setter注入“B”；
2. Spring容器创建单例“B” Bean，首先根据无参构造器创建Bean，并暴露一个“ObjectFactory”用于返回一个提前暴露一个创建中的Bean，并将“B” 标识符放到“当前创建Bean池”，然后进行setter注入“C”；
3. Spring容器创建单例“C” Bean，首先根据无参构造器创建Bean，并暴露一个“ObjectFactory ”用于返回一个提前暴露一个创建中的Bean，并将“C” 标识符放到“当前创建Bean池”，然后进行setter注入“A”；进行注入“A”时由于提前暴露了“ObjectFactory”工厂从而使用它返回提前暴露一个创建中的Bean；
4. 最后在依赖注入“B”和“A”，完成setter注入。

```
<bean id="a" class="cn.com.infcn.test.A">
    <property name="b" ref="b"></property>
</bean>
<bean id="b" class="cn.com.infcn.test.B">
    <property name="c" ref="c"></property>
</bean>
<bean id="c" class="cn.com.infcn.test.C">
    <property name="a" ref="a"></property>
</bean>
```

执行SpringMain.main()方法打印如下：

```
I am B
```

### 2.2 非单例 setter 循环注入(scope=“prototype”)

对于“prototype”作用域Bean，Spring容器无法完成依赖注入，因为“prototype”作用域的Bean，Spring容器不进行缓存，因此无法提前暴露一个创建中的Bean。

```
<bean id="a" class="cn.com.infcn.test.A" scope="prototype">
    <property name="b" ref="b"></property>
</bean>
<bean id="b" class="cn.com.infcn.test.B" scope="prototype">
    <property name="c" ref="c"></property>
</bean>
<bean id="c" class="cn.com.infcn.test.C" scope="prototype">
    <property name="a" ref="a"></property>
</bean>
```

执行SpringMain.main()方法报错





![img](https:////upload-images.jianshu.io/upload_images/2843224-d96e2ffc845fc69c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)



## 模拟 Spring 单例 setter 循环依赖实现

**创建一个ObjectFactory.java**

```
public class ObjectFactory<T> {

    private String className;
    private T t;

    public ObjectFactory(String className, T t) {
        this.className = className;
        this.t = t;
    }

    public T getObject() {
        //如果该bean使用了代理，则返回代理后的bean，否则直接返回bean
        return t;
    }
}
```

**模拟实现类**

```
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class Main {
    // 单例Bean的缓存池
    public static final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
    //单例Bean在创建之初过早的暴露出去的Factory，为什么采用工厂方式，是因为有些Bean是需要被代理的，总不能把代理前的暴露出去那就毫无意义了。
    public static final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
    //执行了工厂方法生产出来的Bean，总不能每次判断是否解决了循环依赖都要执行下工厂方法吧，故而缓存起来。
    public static final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
    
    public static void main(String[] args) {
        A a = (A)getA();
        a.hello();
        
        a = (A)getA();
        a.hello();
        
        B b = (B)getB();
        b.hello();
        
        C c = (C)getC();
        c.hello();
    }
    
    //模拟 spring中 applicationContext.getBean("a")
    public static Object getA(){
        String beanName = "A";
        Object singletonObject = getSingleton(beanName);
        if(singletonObject == null){
            A bean = new A();
            singletonFactories.put(beanName, new ObjectFactory<A>(beanName, bean));
            bean.setB((B)getB());
            addSingleton("A", bean);
            return bean;
        }
        return singletonObject;
    }
    
    //模拟 spring中 applicationContext.getBean("b")
    public static Object getB(){
        String beanName = "B";
        Object singletonObject = getSingleton(beanName);
        if(singletonObject == null){
            B bean = new B();
            singletonFactories.put(beanName, new ObjectFactory<B>(beanName, bean));
            bean.setC((C)getC());
            addSingleton(beanName, bean);
            return bean;
        }
        return singletonObject;
    }
    
    //模拟 spring中 applicationContext.getBean("c")
    public static Object getC(){
        String beanName = "C";
        Object singletonObject = getSingleton(beanName);
        if(singletonObject == null){
            C bean = new C();
            singletonFactories.put(beanName, new ObjectFactory<C>(beanName, bean));
            bean.setA((A)getA());
            addSingleton(beanName, bean);
            return bean;
        }
        return singletonObject;
    }
    
    public static void addSingleton(String beanName, Object singletonObject){
        singletonObjects.put(beanName, singletonObject);
        earlySingletonObjects.remove(beanName);
        singletonFactories.remove(beanName);
    }
    
    public static Object getSingleton(String beanName){
        Object singletonObject = singletonObjects.get(beanName);
        if(singletonObject==null){
            synchronized (singletonObjects) {
                singletonObject = earlySingletonObjects.get(beanName);
                if (singletonObject == null) {
                    ObjectFactory<?> singletonFactory = singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        earlySingletonObjects.put(beanName, singletonObject);
                        singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }
}
```

- singletonObjects：单例Bean的缓存池
- singletonFactories：单例Bean在创建之初过早的暴露出去的Factory，为什么采用工厂方式，是因为有些Bean是需要被代理的，总不能把代理前的暴露出去那就毫无意义了。
- earlySingletonObjects：执行了工厂方法生产出来的Bean，总不能每次判断是否解决了循环依赖都要执行下工厂方法吧，故而缓存起来。

getA()方法、 getB()方法、 getC()方法 是为了模拟applicationContext.getBean() 方法获取bean实例的。因为这里省略了xml配置文件，就把getBean() 方法拆分了三个方法。

这里的ObjectFactory有什么用呢，为什么不直接保留bean 实例对象呢？
 spring源码中是这样实现的如下代码：





![img](https:////upload-images.jianshu.io/upload_images/2843224-8d8a4c4ec115980f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/528)





![img](https:////upload-images.jianshu.io/upload_images/2843224-ccda140530ef9ebf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)



从源码中可以看出，这个ObjectFactory的作用是：如果bean配置了代理，则返回代理后的bean。



#  13、Spring AOP的理解，各个术语，他们是怎么相互工作的？

## **AOP术语以及解释**

### **1）连接点（Joinpoint）**

​    程序执行的某个特定位置：如类开始初始化前、类初始化后、类某个方法调用前、调用后、方法抛出异常后。一个类或一段程序代码拥有一些具有边界性质的特定点，这些点中的特定点就称为“连接点”。Spring仅支持方法的连接点，即仅能在方法调用前、方法调用后、方法抛出异常时以及方法调用前后这些程序执行点织入通知。连接点由两个信息确定：第一是用方法表示的程序执行点；第二是用相对点表示的方位。

###     **2）切点（Pointcut）**

​    每个程序类都拥有多个连接点，如一个拥有两个方法的类，这两个方法都是连接点，即连接点是程序类中客观存在的事物。AOP通过“切点”定位特定的连接点。连接点相当于数据库中的记录，而切点相当于查询条件。切点和连接点不是一对一的关系，一个切点可以匹配多个连接点。

​    **连接点是一个比较空泛的概念，就是定义了哪一些地方是可以切入的，也就是所有允许你通知的地方。**

​    **切点就是定义了通知被应用的位置 （配合通知的方位信息，可以确定具体连接点）**

###     **3）通知（Advice）**

​    通知是织入到目标类连接点上的一段程序代码，在Spring中，通知除用于描述一段程序代码外，还拥有另一个和连接点相关的信息，这便是执行点的方位。结合执行点方位信息和切点信息，我们就可以找到特定的连接点。

Spring切面可以应用5种类型的通知：

​        前置通知（Before）：在目标方法被调用之前调用通知功能；

​        后置通知（After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么；

​        返回通知（After-returning）：在目标方法成功执行之后调用通知；

​        异常通知（After-throwing）：在目标方法抛出异常后调用通知；

​        环绕通知（Around）：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定义的行为。

​        **通知就定义了，需要做什么，以及在某个连接点的什么时候做。 上面的切点定义了在哪里做。**

###     **4）目标对象（Target）**

​    通知逻辑的织入目标类。如果没有AOP，目标业务类需要自己实现所有逻辑，而在AOP的帮助下，目标业务类只实现那些非横切逻辑的程序逻辑，而性能监视和事务管理等这些横切逻辑则可以使用AOP动态织入到特定的连接点上。

###     **5）引介（Introduction）**

​    引介是一种特殊的通知，它为类添加一些属性和方法。这样，即使一个业务类原本没有实现某个接口，通过AOP的引介功能，我们可以动态地为该业务类添加接口的实现逻辑，让业务类成为这个接口的实现类。  

​    **允许向现有类中添加方法和属性（通知中定义的）**

###     **6）织入（Weaving）**

​    织入是将通知添加对目标类具体连接点上的过程。AOP像一台织布机，将目标类、通知或引介通过AOP这台织布机天衣无缝地编织到一起。根据不同的实现技术，AOP有三种织入的方式：

​    a、编译期织入，这要求使用特殊的Java编译器。

​    b、类装载期织入，这要求使用特殊的类装载器。

​    c、动态代理织入，在运行期为目标类添加通知生成子类的方式。

​    **把切面应用到目标对象来创建新的代理对象的过程，Spring采用动态代理织入，而AspectJ采用编译期织入和类装载期织入。**

###     **7）代理（Proxy）**

​    一个类被AOP织入通知后，就产出了一个结果类，它是融合了原类和通知逻辑的代理类。根据不同的代理方式，代理类既可能是和原类具有相同接口的类，也可能就是原类的子类，所以我们可以采用调用原类相同的方式调用代理类。

###     **8）切面（Aspect）**

​    切面由切点和通知组成，它既包括了横切逻辑的定义，也包括了连接点的定义，Spring AOP就是负责实施切面的框架，它将切面所定义的横切逻辑织入到切面所指定的连接点中。

​    **切点的通知的结合，切面知道所有它需要做的事：何时/何处/做什么**

#  14、Spring 如何保证 Controller 并发的安全？

Spring 多线程请求过来调用的Controller对象都是一个，而不是一个请求过来就创建一个Controller对象。

**并发的安全？**
 原因就在于Controller对象是单例的，那么如果不小心在类中定义了类变量，那么这个类变量是被所有请求共享的，这可能会造成多个请求修改该变量的值，出现与预期结果不符合的异常

**那有没有办法让Controller不以单例而以每次请求都重新创建的形式存在呢？**
 答案是当然可以，只需要在类上添加注解@Scope("prototype")即可，这样每次请求调用的类都是重新生成的（每次生成会影响效率）

虽然这样可以解决问题，但增加了时间成本，总让人不爽，还有其他方法么？答案是肯定的！

使用**ThreadLocal**来保存类变量，将类变量保存在线程的变量域中，让不同的请求隔离开来。