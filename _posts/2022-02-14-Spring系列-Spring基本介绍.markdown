---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring基础介绍
subtitle: 
date:   2022-02-14 09:32:20 +0800
description: Spring基础介绍包括Spring基础接口使用介绍
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---


## Spring简介

Spring 是一个开源框架，主要优势之一就是其分层架构，分层架构允许您可以自由选择使用哪一个组件，同时为 J2EE 应用程序开发提供集成的框架。任何 Java 应用都可以从 Spring 中受益。

## Spring模块

Spring包含很多模块，如下图所示：

![spring module](https://s2.loli.net/2022/02/11/8krDpJWoS64FmHN.png)



但是其中最重要的主要是三个：

- spring-core

  > 主要是一些核心工具，例如：类解析(asm)，动态代理(cglib)，转换器(conversion)，资源加载(resource)等。

  ![spring core](https://s2.loli.net/2022/02/14/HoI3KuQRCfY4cVZ.png)

- spring-beans

  > 主要是与bean有关：BeanFactory，BeanDefinition，BeanFactoryPostProcessor，BeanPostProcessor等。

  ![spring beans](https://s2.loli.net/2022/02/14/AoaC8Rv4YfzXKyk.png)

- spring-contenxt

  > 主要与上下文有关，用于扩展BeanFactory的功能：ApplicationContext，weaving，cache，event，jmx，jndi，i18n等。
  
  ![spring context](https://s2.loli.net/2022/02/14/JCl5QTW6equxBRv.png)

## Spring两个核心

### IOC

IOC主要需要理解什么是控制反转和注入。

控制反转是一种思想，可以理解为本来按照我们之前的编程习惯，在需要引用其它对象的地方，我们都要手动传入该对象实例或直接new一个该对象实例，但是现在我们不需要关心对象之前的依赖关系了，只需要保证对象实例在Spring中存在，spring会自动帮我们注入该实例，也就是本来需要我们控制的事交给了Spring控制。

依赖注入是一种实现，我们使用Spring编程的时候，只需要对该字段使用一种的注入方式，然后将该字段类实例成为一个 bean，Spring就会自动在指定的位置给字段赋予正确的对象或值。

> 依赖注入的三种方式：set方式注入，构造器注入，注解注入。

### AOP

AOP是一种编程范式，旨在通过允许横切关注点的分离，提高模块化。我们可以使用aop的方式将通用逻辑或者与业务无关代码抽取出来，放到一个指定的地方。这样在保持功能完整的同时还能保持逻辑的清晰，而且还使代码变得简洁，不必在代码中充斥这大量的重复代码。

AOP分为静态代理和动态代理，静态代理的代表是AspectJ，动态代理的代表是Spring AOP。

Spring AOP中的动态代理主要有两种实现方式：JDK动态代理和CGLIB动态代理。

在如下场景使用：**日志记录，性能统计，安全控制，事务处理，异常处理**等。

> 这里需要注意的是，Spring AOP中的AspectJ语法，并不是Spring中使用了AspectJ，仅仅是Spring AOP使用了AspectJ的语法，底层还是使用的动态代理，并不是AspectJ的实现。
>
> AspectJ可以做Spring AOP干不了的事情，它是AOP编程的完全解决方案，Spring AOP则致力于解决企业级开发中最普遍的AOP（方法织入）。

## Spring容器

### BeanFactory

BeanFactory是Spring容器最顶层父类接口，它定义了容器最基本的功能。如下：

```java
public interface BeanFactory {

  // 如果bean的名字是&符号开始，spring就会认为这是要获取FactoryBean而不是bean
  String FACTORY_BEAN_PREFIX = "&";

  Object getBean(String name) throws BeansException;

  <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;

  Object getBean(String name, Object... args) throws BeansException;

  <T> T getBean(Class<T> requiredType) throws BeansException;

  <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

  boolean containsBean(String name);

  boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

  boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

  boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

  boolean isTypeMatch(String name, @Nullable Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

  @Nullable
  Class<?> getType(String name) throws NoSuchBeanDefinitionException;

  String[] getAliases(String name);
}
```

DefaultListableBeanFactory 是Spring IoC容器最核心的实现，包含了基本IOC容器所具有的所有重要功能，是一个完整的IOC容器。BeanFactory 有三个子类接口：

- ListableBeanFactory 

  > 可以查询bean列表，以及根据条件获取相应的bean列表信息

```java
public interface ListableBeanFactory extends BeanFactory {

  boolean containsBeanDefinition(String beanName);

  int getBeanDefinitionCount();

  String[] getBeanDefinitionNames();

  String[] getBeanNamesForType(ResolvableType type);

  String[] getBeanNamesForType(@Nullable Class<?> type);

  String[] getBeanNamesForType(@Nullable Class<?> type, boolean includeNonSingletons, boolean allowEagerInit);

  <T> Map<String, T> getBeansOfType(@Nullable Class<T> type) throws BeansException;

  <T> Map<String, T> getBeansOfType(@Nullable Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
        throws BeansException;

  String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType);

  Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) throws BeansException;

  @Nullable
  <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
        throws NoSuchBeanDefinitionException;
}
```

- HierarchicalBeanFactory

  > 使BeanFactory可以拥有父BeanFactory，使得BeanFactory有了层级关系

```java
public interface HierarchicalBeanFactory extends BeanFactory {
  @Nullable 
  BeanFactory getParentBeanFactory();

  boolean containsLocalBean(String name);
}
```

- AutowireCapableBeanFactory

  > AutowireCapableBeanFactory的作用：让Spring管理的Bean去装配和填充那些不被Spring托管的Bean，为第三方框架赋能。一般应用开发者不会使用这个接口,所以像`ApplicationContext`这样的外观实现类不会实现这个接口。

```java
public interface AutowireCapableBeanFactory extends BeanFactory {
  // 不装配
  int AUTOWIRE_NO = 0;
  // 依据名称装配
  int AUTOWIRE_BY_NAME = 1;
  // 依据类型装配
  int AUTOWIRE_BY_TYPE = 2;
  // 构造方法装配
  int AUTOWIRE_CONSTRUCTOR = 3;
  /** @deprecated */
  @Deprecated
  int AUTOWIRE_AUTODETECT = 4;
  // 该属性是一种约定俗成的用法：以类全限定名+.ORIGINAL 作为Bean Name，用于告诉Spring，在初始化的时候，
  // 需要返回原始给定实例，而别返回代理对象
  String ORIGINAL_INSTANCE_SUFFIX = ".ORIGINAL";
  
  // 用给定的class创建一个Bean实例，完整经历一个Bean创建过程的生命周期节点回调，但不执行传统的autowiring
  // (传统的wutowiring就是不是使用注解的方式)
  <T> T createBean(Class<T> beanClass) throws BeansException;

  void autowireBean(Object existingBean) throws BeansException;

  Object configureBean(Object existingBean, String beanName) throws BeansException;

  Object createBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;

  Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;

  void autowireBeanProperties(Object existingBean, int autowireMode, boolean dependencyCheck)
        throws BeansException;

  void applyBeanPropertyValues(Object existingBean, String beanName) throws BeansException;

  Object initializeBean(Object existingBean, String beanName) throws BeansException;

  Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
        throws BeansException;

  Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
        throws BeansException;

  void destroyBean(Object existingBean);

  <T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType) throws BeansException;

  @Nullable
  Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName) throws BeansException;

  @Nullable
  Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException;
}
```

### ApplicationContext

> ApplicationContext可以看做是升级版的BeanFactory，可以看到它不仅继承了ListableBeanFactory，HierarchicalBeanFactory基本BeanFactory接口，还继承了环境变量，国际化，事件通知，资源解析等接口，进一步扩展了基本容器的功能。

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
    @Nullable
    String getId();

    String getApplicationName();

    String getDisplayName();

    long getStartupDate();

    @Nullable
    ApplicationContext getParent();
    // Spring不推荐我们在应用中直接使用AutowireCapableBeanFactory，但还是为我们提供了获取接口
    AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;
}
```

我们的项目可能运行于不同的环境，可能是普通Java项目，可能是Web项目，可能使用xml配置，可能使用注解配置，所以Spring给我们提供了很多不同的ApplicationContext实现类。例如：

- FileSystemXmlApplicationContext：该类可以从文件系统绝对路径加载配置文件
- ClassPathXmlApplicationContext：该类可以从classpath下加载配置文件
- XmlWebApplicationContext：该类可以在在web项目下使用，使用xml配置
- AnnotationConfigApplicationContext：该类可以在SpringBoot普通Java项目下使用，使用注解配置
- AnnotationConfigEmbeddedWebApplicationContext：该类可以在SpringBoot Web项目中使用，使用注解配置

## Spring基础接口

### Resource

Resource 接口是 Spring 资源访问策略的抽象，它本身并不提供任何资源访问实现，具体的资源访问由该接口的实现类完成——每个实现类代表一种资源访问策略。

```java
public interface Resource extends InputStreamSource {

  boolean exists();

  default boolean isReadable() {
        return true;
    }

  default boolean isOpen() {
        return false;
    }

  default boolean isFile() {
        return false;
    }

  URL getURL() throws IOException;

  URI getURI() throws IOException;

  File getFile() throws IOException;

  default ReadableByteChannel readableChannel() throws IOException {
        return Channels.newChannel(getInputStream());
    }

  long contentLength() throws IOException;

  long lastModified() throws IOException;

  Resource createRelative(String relativePath) throws IOException;

  String getFilename();

  String getDescription();
}
```

Resource一般包括这些实现类：

- UrlResource: 访问网络资源的实现类
- ClassPathResource: 访问类加载路径下的资源实现类
- FileSystemResource：访问文件系统资源实现类
- ServletContextResource：ServletContext资源的Resource实现，它解释相关Web应用程序根目录中的相对路径
- InputStreamResource：给定的输入流(InputStream)的Resource实现类
- ByteArrayResource：字节数组的Resource实现类

### ResourceLoader/ResourcePatternResolver

ResourceLoader接口旨在由可以返回(即加载)Resource实例的对象实现。所有的应用程序上下文都实现了ResourceLoader接口。因此，所有的应用程序上下文都可能会获取Resource实例。

```java
public interface ResourceLoader {
  
  String CLASSPATH_URL_PREFIX = "classpath:";

  Resource getResource(String location);

  @Nullable
  ClassLoader getClassLoader();

}

public interface ResourcePatternResolver extends ResourceLoader {

  // 新增了一种新的协议前缀 "classpath*:"，该协议前缀由其子类负责实现。
  String CLASSPATH_ALL_URL_PREFIX = "classpath*:";
	
  // 增加 #getResources(String locationPattern) 方法 以支持根据路径匹配模式返回多个 Resource 实例
  Resource[] getResources(String locationPattern) throws IOException;
}
```

ResourceLoader一般包括下面这些实现类：

- DefaultResourceLoader：getResource()单文件加载
- PathMatchingResourcePatternResolver：getResources()多文件加载

### BeanDefinition

BeanDefinition用于承载定义Bean的相关信息，Spring会根据XML或注解配置的Bean信息创建相应的BeanDefinition保存起来。

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
  
  String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

  String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

  int ROLE_APPLICATION = 0;

  int ROLE_SUPPORT = 1;

  int ROLE_INFRASTRUCTURE = 2;

  void setParentName(@Nullable String parentName);

  @Nullable
  String getParentName();

  void setBeanClassName(@Nullable String beanClassName);

  @Nullable
  String getBeanClassName();

  void setScope(@Nullable String scope);

  @Nullable
  String getScope();

  void setLazyInit(boolean lazyInit);

  boolean isLazyInit();

  void setDependsOn(@Nullable String... dependsOn);

  @Nullable
  String[] getDependsOn();

  void setAutowireCandidate(boolean autowireCandidate);

  boolean isAutowireCandidate();

  void setPrimary(boolean primary);

  boolean isPrimary();

  void setFactoryBeanName(@Nullable String factoryBeanName);

  @Nullable
  String getFactoryBeanName();

  void setFactoryMethodName(@Nullable String factoryMethodName);

  @Nullable
  String getFactoryMethodName();

  ConstructorArgumentValues getConstructorArgumentValues();

  default boolean hasConstructorArgumentValues() {
    return !getConstructorArgumentValues().isEmpty();
  }

  MutablePropertyValues getPropertyValues();

  default boolean hasPropertyValues() {
    return !getPropertyValues().isEmpty();
  }

  boolean isSingleton();

  boolean isPrototype();

  boolean isAbstract();

  int getRole();

  @Nullable
  String getDescription();

  @Nullable
  String getResourceDescription();

  @Nullable
  BeanDefinition getOriginatingBeanDefinition();
}
```

BeanDefinition一般包括下面实现类：

- RootBeanDefinition：该实现类对应了一般的元素标签

- ChildBeanDefinition：该实现类可以让子 BeanDefinition 定义拥有从父 BeanDefinition 那里继承配置的能力

- GenericBeanDefinition：该实现类可以动态设置父 Bean，同时兼具 RootBeanDefinition 和 ChildBeanDefinition 的功能

- AnnotatedBeanDefinition：该实现类为注解类型 BeanDefinition，拥有获取注解元数据和方法元数据的能力

- AnnotatedGenericBeanDefinition：使用了 @Configuration 注解标记配置类会解析为 AnnotatedGenericBeanDefinition

### BeanDefinitionReader

BeanDefinitionReader的作用是读取 Spring 配置文件中的内容，将其转换为 IoC 容器内部的数据，即BeanDefinition。

```java
public interface BeanDefinitionReader {

  BeanDefinitionRegistry getRegistry();

  @Nullable
  ResourceLoader getResourceLoader();

  @Nullable
  ClassLoader getBeanClassLoader();

  BeanNameGenerator getBeanNameGenerator();

  int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;

  int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;

  int loadBeanDefinitions(String location) throws BeanDefinitionStoreException;

  int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;
}
```

BeanDefinition一般包括下面实现类：

- XmlBeanDefinitionReader：读取 XML 文件定义的 BeanDefinition
- PropertiesBeanDefinitionReader：可以从属性文件，Resource，Property 对象等读取 BeanDefinition
- GroovyBeanDefinitionReader：可以读取 Groovy 语言定义的 Bean

### BeanDefinitionRegister

BeanDefinitionRegister是bean定义注册器，它的作用就是利用BeanDefinitionReader把我们的资源加载成一个个BeanDefinition并注册到beanDefinitionMap中。

```java
public interface BeanDefinitionRegistry extends AliasRegistry {

  void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException;

  void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

  BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

  boolean containsBeanDefinition(String beanName);

  String[] getBeanDefinitionNames();

  int getBeanDefinitionCount();

  boolean isBeanNameInUse(String beanName);
}
```

由于该接口是一个特别基础的接口，所以BeanFactory以及ApplicationContext的实现类都会实现该接口，例如：

- DefaultListableBeanFactory
- AnnotationConfigApplicationContext
- SimpleBeanDefinitionRegistry

### BeanFactory/ApplicationContext

上面已介绍。

### ApplicationEventPublisher

Spring 专门提供了一套事件机制的接口，方便我们运用。

```java
@FunctionalInterface
public interface ApplicationEventPublisher {

  default void publishEvent(ApplicationEvent event) {
    publishEvent((Object) event);
  }

  void publishEvent(Object event);
}
```

该接口数据ApplicationContext中扩展的时间通知机制，所以所有的ApplicationContext实现类都可以使用。

### BeanFactoryPostProcessor

BeanFactoryPostProcessor可以对bean的定义（配置元数据）进行处理。也就是说，Spring IoC容器允许BeanFactoryPostProcessor在容器实际实例化任何其它的bean之前读取配置元数据，并有可能修改它。如果你愿意，你可以配置多个BeanFactoryPostProcessor。你还能通过设置'order'属性来控制BeanFactoryPostProcessor的执行次序。

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {

  void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

依据于BeanFactoryPostProcessor扩展出来的接口有：

- BeanDefinitionRegistryPostProcessor：添加了自定义注册BeanDefinition的功能

  ```java
  public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
  }
  ```

### BeanPostProcessor

与BeanFactoryPostProcessor不同的是，BeanPostProcessor修改的不是bean的元数据，而是修改在实例化以及初始化bean前后对bean的生成进行一些修改。

```java
public interface BeanPostProcessor {  
  
  // 实例化、依赖注入完毕，在调用显示的初始化之前完成一些定制的初始化任务  
  Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;  
  
  // 实例化、依赖注入、初始化完毕时执行  
  Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;  
}
```

依据于BeanPostProcessor扩展出来的接口有：

- DestructionAwareBeanPostProcessor：用于在销毁bean实例前进行前置处理

  ```java
  public interface DestructionAwareBeanPostProcessor extends BeanPostProcessor {
    void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
  
      default boolean requiresDestruction(Object bean) {
        return true;
      }
  }
  ```

- MergedBeanDefinitionPostProcessor：该处理器会对bean定义做一些处理，比如解析出来需要注入的注解元素信息，便于InstantiationAwareBeanPostProcessor处理器的后续属性注入

  ```java
  public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {
    void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
  
    default void resetBeanDefinition(String beanName) {
    }
  }
  ```

- InstantiationAwareBeanPostProcessor：用于对注解属性/方法进行注入

  ```java
  public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
      return null;
    }
  
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
      return true;
    }
  
    // 新增接口
    @Nullable
      default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        return null;
      }
  
      @Deprecated
      @Nullable
      default PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        return pvs;
      }
  }
  ```

- SmartInstantiationAwareBeanPostProcessor：用于自动推断构造方法，以及解决循环依赖问题

  ```java
  public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {
    @Nullable
    default Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException {
      return null;
    }
  
    @Nullable
    default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException {
      return null;
    }
  
    default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
      return bean;
    }
  }
  ```

## Aware

Aware接口可以让bean在创建过程中感知到外界信息，不同的Aware接口有不同的感知能力。

```java
public interface Aware {
}
```

常见的Aware扩展接口如下（接口都要实现一个setXXX的方法）：
- BeanNameAware：可以感知beanName
- BeanFactoryAware：可以感知beanFactory
- BeanClassLoaderAware：可以感知beanClassLoader
- EnvironmentAware：可以感知Environment
- ServletContextAware：可以感知servletContext
- ApplicationContextAware：可以感知ApplicationContext
- EmbeddedValueResolverAware：可以感知EmbeddedValueResolve

## InitializingBean

可以在applyBeanPostProcessorsBeforeInitialization()方法执行之后执行一些资源初始化逻辑。

```java
public interface InitializingBean {
  void afterPropertiesSet() throws Exception;
}
```

## DisposableBean

可以在销毁bean之前执行一些资源清理逻辑。

```java
public interface DisposableBean {
  void destroy() throws Exception;
}
```

