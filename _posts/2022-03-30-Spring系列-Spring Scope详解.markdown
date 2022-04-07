---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring Scope详解
subtitle: 永远不要后退，退到最后是无路可退。
date:   2022-03-30 14:08:20 +0800
description: Spring Scope @Scope singleton prototype
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

在Spring中，默认有两种Scope：singleton和prototype。其它框架也会依据自己的需求扩展Scope，当然我们也可以在我们的程序中对Scope进行自定义扩展，这一节我们先看一下Scope的基本使用以及实现原理。

## Scope有两种配置方式

### XML配置

```xml
<bean id="Test" class="com.demo.Test" scope="prototype">
```

### 注解配置

在类上配置：

```java
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class Test {
}
```

在方法上配置

```java
@Configuration
public Config {
  @Bean
  @Scope("prototype")
  public Test test() {
    return new Test();
  }
}
```

## Scope代理

我们可以看到在上面的注解配置中的，在类上配置的注解中配置了一个proxyMode属性，这个属性用来控制是否需要进行Scope代理，以及通过JDK动态代理还是CGLIB动态代理。

proxyMode属性的取值有四种：

- ScopedProxyMode.DEFAULT: 默认取值，同ScopedProxyMode.NO，不进行代理；
- ScopedProxyMode.NO：不进行代理；
- ScopedProxyMode.INTERFACES：使用JDK动态代理；
- ScopedProxyMode.TARGET_CLASS：使用CGLIB动态代理；

## Scope代理有什么用?

我们知道，在Spring中单例Bean只会实例化一次，之后再次获取该单例Bean，都是直接从缓存中获取之前创建的Bean实例。而像指定Scope为prototype的Bean，每次通过getBean()获取都会重新创建一个新的Bean，这样一看，好像Scope不使用代理也可以满足，那要Scope有什么用呢？

我们思考这样一个场景，我需要在一个单例Bean A中注入一个多例Bean B，此时A中持有的B实例是多例的吗？显然不是的，因为在初始化单例Bean A的时候Spring只会注入一次B，虽然B是在创建的时候刚创建的，但是注入了之后，A中的B实例就不会在改变了，但是我们将B改为多例的目的就是A中每次使用B都是重新创建一个新的Bean实例，此时就不满足我们的需求了，我们可以怎么做呢？

我们有几种方式可以解决这个问题：

1. 每次都从BeanFactory中获取B实例

   ```java
   public class A implements ApplicationContextAware {
   
     private ApplicationContext applicationContext; 
   
     @Override
     public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
       applicationContext = applicationContext;
     }
     // 每次重新获取
     public B getB() {
       return (B)applicationContext.getBean("b");
     }
   }
   ```

   ```java
   @Component
   public class A implements BeanFactoryAware {
   
     private BeanFactory beanFactory;
   
     @Override
     public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
       beanFactory = beanFactory;
     }
     // 每次重新获取
     public B getB() {
       return (B)beanFactory.getBean("b");
     }
   }
   ```

2. 使用@Lookup注解

   ```java
   @Component
   public class A {
   
     // @Lookup注解会将getB()方法实现改掉，其会根据返回值去beanFactory中获取B.class类型的实例，实际上同第一种方式原理一样
     @Lookup
     public B getB() {
       return null;
     }
   }
   ```

3. 使用Scope代理

   ```java
   @Component
   public class A {
     @Autowired
     private B b;
   }
   @Component
   @Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
   public class B {}
   ```

## 为什么Scope代理能解决这个问题呢？

下面我们就从源码的角度详细讲一下(以注解@Scope为例)。首先我们看一下@Scope注解的定义：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Scope {

  @AliasFor("scopeName")
  String value() default "";

  @AliasFor("value")
  String scopeName() default "";

  ScopedProxyMode proxyMode() default ScopedProxyMode.DEFAULT;
}
```

@Target注解中的值表名@Scope注解即可以标注在类上也可以标注在方法上，标注在方法上时，方法上还需要有@Bean标识，说明该方法是要注册一个Bean。

如果标注在类上，在类扫描的时候会对Scope进行处理，例如在ClassPathBeanDefinitionScanner的doScan()方法中：

```java
// ClassPathBeanDefinitionScanner.java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
  Assert.notEmpty(basePackages, "At least one base package must be specified");
  Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
  // 循环扫描所有的包路径
  for (String basePackage : basePackages) {
    Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
    // 处理扫描到的BeanDefinition
    for (BeanDefinition candidate : candidates) {
      // 解析BeanDefinition中的Scope元数据
      ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
      candidate.setScope(scopeMetadata.getScopeName());
      String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
      if (candidate instanceof AbstractBeanDefinition) {
        postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
      }
      if (candidate instanceof AnnotatedBeanDefinition) {
        AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
      }
      if (checkCandidate(beanName, candidate)) {
        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
        // 处理Scope代理
        definitionHolder =
          AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
        beanDefinitions.add(definitionHolder);
        registerBeanDefinition(definitionHolder, this.registry);
      }
    }
  }
  return beanDefinitions;
}
```

如果是在配置类中引入的或者是标注在配置类中的方法上，会在ConfigurationClassBeanDefinitionReader类中处理@Scope注解配置，例如如果标注在@Bean方法上，会在loadBeanDefinitionsForBeanMethod()中处理：

   > 调用路径：
   >
   > -> ConfigurationClassPostProcessor.processConfigBeanDefinitions()
   >
   > -> ConfigurationClassBeanDefinitionReader.loadBeanDefinitions()
   >
   > -> ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForConfigurationClass()
   >
   > -> ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod()

```java
private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
  ConfigurationClass configClass = beanMethod.getConfigurationClass();
  MethodMetadata metadata = beanMethod.getMetadata();
  String methodName = metadata.getMethodName();
  // ...
  ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata, beanName);
  beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));
  
  // 处理@Scope注解
  ScopedProxyMode proxyMode = ScopedProxyMode.NO;
  AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
  if (attributes != null) {
    beanDef.setScope(attributes.getString("value"));
    proxyMode = attributes.getEnum("proxyMode");
    if (proxyMode == ScopedProxyMode.DEFAULT) {
      proxyMode = ScopedProxyMode.NO;
    }
  }

  BeanDefinition beanDefToRegister = beanDef;
  if (proxyMode != ScopedProxyMode.NO) {
    // 如果需要代理，创建Scope代理
    BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
      new BeanDefinitionHolder(beanDef, beanName), this.registry,
      proxyMode == ScopedProxyMode.TARGET_CLASS);
    beanDefToRegister = new ConfigurationClassBeanDefinition(
      (RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata, beanName);
  }
  this.registry.registerBeanDefinition(beanName, beanDefToRegister);
}
```

无论是哪种方式，最终都是调用到ScopedProxyUtils工具类进行处理。该类会将目标Bean改个名字进行注册，并返回代理BeanDefinitionHolder。

```java
// ScopedProxyUtils.java
public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
                                                     BeanDefinitionRegistry registry, boolean proxyTargetClass) {
  String originalBeanName = definition.getBeanName();
  BeanDefinition targetDefinition = definition.getBeanDefinition();
  // 获取目标beanName，因为要给Scope标识的类进行代理，所以原始的beanName实际上就是代理类了，而原始目标类的名称需要换一个名字，而这个名字就是在原始beanName之前加上
  // scopedTarget. 前缀
  String targetBeanName = getTargetBeanName(originalBeanName);

  // 创建一个代理类的BeanDefinition，类型为ScopedProxyFactoryBean
  RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
  // 将原始BeanDefinition的一些信息设置到代理BeanDefinition中
  proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
  proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
  proxyDefinition.setSource(definition.getSource());
  proxyDefinition.setRole(targetDefinition.getRole());
  // 设置代理类代理的目标beanName, 也就是ScopedTarget.beanName
  proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
  if (proxyTargetClass) {
    targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
  } else {
    proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
  }

  proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
  proxyDefinition.setPrimary(targetDefinition.isPrimary());
  if (targetDefinition instanceof AbstractBeanDefinition) {
    proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
  }

  // 将目标Bean设置为不能作为其他Bean的注入候选者，因为该Bean已经被它的代理类取代了
  targetDefinition.setAutowireCandidate(false);
  targetDefinition.setPrimary(false);

  // 注册目标BeanDefinition
  registry.registerBeanDefinition(targetBeanName, targetDefinition);

  // 返回代理BeanDefinitionHolder
  return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
}
```

因为对Scope进行代理的类型为ScopedProxyFactoryBean，所以我们看一下该代理类是怎么进行代理实现的。

   1. 该类实现了ProxyConfig接口，说明该类可以进行一些代理配置；
   2. 该类实现了FactoryBean接口，说明是一个FactoryBean，通过FactoryBean获取真正的Bean是通过getObject()方法；
   3. 该类实现了BeanFactoryAware接口，那么在初始化该类的时候，会回调setBeanFactory()方法；
   4. 该类实现了AopInfrastructureBean接口，说明类可以进行引介增强。

```java
public class ScopedProxyFactoryBean extends ProxyConfig
  implements FactoryBean<Object>, BeanFactoryAware, AopInfrastructureBean {
 
  private final SimpleBeanTargetSource scopedTargetSource = new SimpleBeanTargetSource();

  @Nullable
  private String targetBeanName;

  @Nullable
  private Object proxy;

  public ScopedProxyFactoryBean() {
    setProxyTargetClass(true);
  }

  public void setTargetBeanName(String targetBeanName) {
    this.targetBeanName = targetBeanName;
    this.scopedTargetSource.setTargetBeanName(targetBeanName);
  }

  // 在初始化设置beanFactory的时候，生成Scope代理类
  @Override
  public void setBeanFactory(BeanFactory beanFactory) {
    if (!(beanFactory instanceof ConfigurableBeanFactory)) {
      throw new IllegalStateException("Not running in a ConfigurableBeanFactory: " + beanFactory);
    }
    ConfigurableBeanFactory cbf = (ConfigurableBeanFactory) beanFactory;

    this.scopedTargetSource.setBeanFactory(beanFactory);
		
    ProxyFactory pf = new ProxyFactory();
    pf.copyFrom(this);
    // 设置代理类代理的targetSource，之前我们说过Spring动态代理都会将代理类放到TargetSource中，这些方便进行对目标类进行动态替换，增强扩展性
    pf.setTargetSource(this.scopedTargetSource);

    Assert.notNull(this.targetBeanName, "Property 'targetBeanName' is required");
    Class<?> beanType = beanFactory.getType(this.targetBeanName);
    if (beanType == null) {
      throw new IllegalStateException("Cannot create scoped proxy for bean '" + this.targetBeanName +
                                      "': Target type could not be determined at the time of proxy creation.");
    }
    // 设置目标类实现的接口
    if (!isProxyTargetClass() || beanType.isInterface() || Modifier.isPrivate(beanType.getModifiers())) {
      pf.setInterfaces(ClassUtils.getAllInterfacesForClass(beanType, cbf.getBeanClassLoader()));
    }

    // 添加一个DelegatingIntroductionInterceptor拦截器，用于引介增强，对目标类添加了ScopedObject接口的功能
    ScopedObject scopedObject = new DefaultScopedObject(cbf, this.scopedTargetSource.getTargetBeanName());
    pf.addAdvice(new DelegatingIntroductionInterceptor(scopedObject));

    // 添加AopInfrastructureBean接口，标识该代理类不受Aop动态代理的影响
    pf.addInterface(AopInfrastructureBean.class);
    // 生成代理类
    this.proxy = pf.getProxy(cbf.getBeanClassLoader());
  }

  // 返回代理类
  @Override
  public Object getObject() {
    if (this.proxy == null) {
      throw new FactoryBeanNotInitializedException();
    }
    return this.proxy;
  }

  @Override
  public Class<?> getObjectType() {
    if (this.proxy != null) {
      return this.proxy.getClass();
    }
    return this.scopedTargetSource.getTargetClass();
  }

  @Override
  public boolean isSingleton() {
    return true;
  }
}
```

ProxyFactory生成代理类的逻辑我们已经在[Spring系列-Spring Aop实现原理分析](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/23/Spring%E7%B3%BB%E5%88%97-Spring%E4%BA%8B%E5%8A%A1%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.html)中讲过了，不清楚的童鞋可以去看一下。然后在执行的时候，会通过设置的targetSource的getTarget()方法获取目标bean，这里使用的是SimpleBeanTargetSource类，我们看一下该类的实现：

```java
public class SimpleBeanTargetSource extends AbstractBeanFactoryBasedTargetSource {

  @Override
  public Object getTarget() throws Exception {
    // 直接从beanFactory中获取Bean对象
    return getBeanFactory().getBean(getTargetBeanName());
  }
}
```

因此我们回到上面介绍的那几种解决方法，发现无论怎么实现，最终都是调用到了beanFactory.getBean()方法，所以@Scope注解中标注进行代理可以解决我们上面说的那种场景。

到这里，我们就介绍完了@Scope的代理的实现原理，下面我们再看一下对于scope类型的Bean是怎么创建的。在[Spring系列-Spring的Bean创建流程](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/02/15/Spring%E7%B3%BB%E5%88%97-Bean%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.html)文章中我们已经介绍了Bean的创建流程，不熟悉的童鞋可以去熟悉一下。

在AbstractBeanFactory的doGetBean()方法中有这么一段逻辑：

```java
// AbstractBeanFactory.java
protected <T> T doGetBean(
  String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
  throws BeansException {

  String beanName = transformedBeanName(name);
  Object bean;
  // ...
  try {
    RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    checkMergedBeanDefinition(mbd, beanName, args);
    // ...

    // 处理单例Bean
    if (mbd.isSingleton()) {
      sharedInstance = getSingleton(beanName, () -> {
        try {
          return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
          destroySingleton(beanName);
          throw ex;
        }
      });
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
    // 处理多例Bean
    else if (mbd.isPrototype()) {
      Object prototypeInstance = null;
      try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
      }
      finally {
        afterPrototypeCreation(beanName);
      }
      bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
    }
    else {
      // 处理其它Scope
      String scopeName = mbd.getScope();
      if (!StringUtils.hasLength(scopeName)) {
        throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
      }
      Scope scope = this.scopes.get(scopeName);
      if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
      }
      try {
        // 通过对应Scope实现类的get()方法处理Bean的获取
        Object scopedInstance = scope.get(beanName, () -> {
          beforePrototypeCreation(beanName);
          try {
            return createBean(beanName, mbd, args);
          }
          finally {
            afterPrototypeCreation(beanName);
          }
        });
        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
      }
      catch (IllegalStateException ex) {
        throw new BeanCreationException(beanName,
                                        "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                        "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                        ex);
      }
    }
  }
  catch (BeansException ex) {
    cleanupAfterBeanCreationFailure(beanName);
    throw ex;
  }
}
// ...
return (T) bean;
}
```

从上面的代理可以看出，Singleton和Prototype是特殊处理的两种Scope，否则都通过scope.get()进行处理，而我们扩展的Scope都是走scope.get()的逻辑进行处理。下一节我们就讲一下如何自定义Scope。