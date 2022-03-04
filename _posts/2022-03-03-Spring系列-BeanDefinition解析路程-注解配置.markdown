---
layout: post
read_time: true
show_date: true
title:  Spring系列-BeanDefinition解析流程-注解配置
date:   2022-03-03 14:28:20 +0800
description: Spring系列-BeanDefinition解析流程-注解配置
img: posts/common/spring.png
tags: [spring]
author: tengjiang
toc: yes
---

|| 该文章主要介绍了Spring是如何加载解析我们使用注解方式配置的BeanDefinition的，讲解了ConfigurationClassPostProcessor后置处理器的解析逻辑。||

前面我们在[BeanDefinition解析流程-XML配置](https://www.tengjiang.site/Spring%E7%B3%BB%E5%88%97-BeanDefinition%E8%A7%A3%E6%9E%90%E6%B5%81%E7%A8%8B-XML%E9%85%8D%E7%BD%AE.html)文章中讲解了原始Xml配置方法是怎么加载BeanDefinition的，这次我们来看看通过注解方式是怎么加载BeanDefinition的。

> 这里需要说明的是，BeanDefinition的解析位置不是固定的，写法不同，解析开始的位置不同。可能由于Spring加载方式太灵活了，导致大家在看源码的时候感觉很混乱，摸不清这个BeanDefinition到底会在什么地方才会解析。

我们在使用注解配置的时候，最常用的有两种方式：

- 直接在类上加@Component注解

  > @Controller、@Service、@@Repository等注解也是依据于@Component注解，在这些注解类上都添加有@Component注解

  ```java
  @Component
  public class User {
  }
  ```

- 在@Configuration标注的类中使用@Bean标识

  ```java
  @ComponentScan("com.demo")
  @Configuration
  public class HomeConfig {
    @Bean
    public Home home() {
      return new Home();
    }
  }
  ```

在讲解之前，我们首先需要让Spring知道去哪里扫描，才能找到添加了@Component或@Configuration注解的类。下面我们通过AnnotationConfigApplicationContext为例进行说明。

AnnotationConfigApplicationContext可以通过多种方式告诉Spring去哪里找到这些类。例如：

```java
// 第一种方式，直接在参数中指定要扫描的包
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext("com.demo");

// 第二种方式，在参数中指定一个类，在这个类上使用@ComponentScan("com.demo")注解标注要扫描的包
// 这种方式，Spring默认会将指定的这个类作为一个Bean进行处理
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(HomeConfig.class);
```

通过第一种方式扫描有一个问题，就是这种方式只能扫描到标注了@Component注解的类，@Configuration注解无效。

所以建议大家使用第二种方式，第二种方式比第一种方式更强大。

### ConfigurationClassPostProcessor

通过注解的方式依赖于一个非常重要的类ConfigurationClassPostProcessor，这个类是专门处理配置类的类。这里需要说明的是，这个类会将什么样的类作为配置类呢？

ConfigurationClassPostProcessor将配置类分为了两种：full和lite。

- full: 被@Configuration标识的类
- lite: 被@Component、@ComponentScan、@Import、@ImportResource标识，或者类中的方法被@Bean标识的类

那么这个类是什么时候生效的呢？

在之前的文章[Spring的Bean创建流程](Spring%E7%B3%BB%E5%88%97-Bean%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.html)我们讲了在refresh()方法中的invokeBeanFactoryPostProcessors()方法中，会注册BeanDefinition，而ConfigurationClassPostProcessor就是在这一步中生效的，因为它不仅实现了BeanFactoryPostProcessor还实现了BeanDefinitionRegistryPostProcessor。

在这个类中有两个入口方法:

- BeanDefinitionRegistryPostProcessor接口的postProcessBeanDefinitionRegistry()方法
- BeanFactoryPostProcessor接口的postProcessBeanFactory()方法

通过之前的文章我们知道了Spring在执行invokeBeanFactoryPostProcessors()方法时会先调用postProcessBeanDefinitionRegistry()方法，然后再调用postProcessBeanFactory()方法，那么接下来我们就从这两个方法入手去看看到底做了什么。

### postProcessBeanDefinitionRegistry()

```java
// ConfigurationClassPostProcessor.java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
  // 这里的判断主要是为了防止重复执行
  int registryId = System.identityHashCode(registry);
  // 防止同一个方法多次调用
  if (this.registriesPostProcessed.contains(registryId)) {
    throw new IllegalStateException(
      "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
  }
  // 防止先调用了postProcessBeanFactory()又调用了postProcessBeanDefinitionRegistry()
  // 正常来说是要先调用postProcessBeanDefinitionRegistry()再调用postProcessBeanFactory()，如果先调用了
  // postProcessBeanFactory()其实postProcessBeanDefinitionRegistry()中调用的逻辑已经被执行过了
  if (this.factoriesPostProcessed.contains(registryId)) {
    throw new IllegalStateException(
      "postProcessBeanFactory already called on this post-processor against " + registry);
  }
  this.registriesPostProcessed.add(registryId);
  // 重要！！！ 处理配置类
  processConfigBeanDefinitions(registry);
}
```

#### processConfigBeanDefinitions()

```java
// ConfigurationClassPostProcessor.java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
  List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
  /**
   * 这里获取已经注册的所有BeanDefinition的名称，这里最重要的是我们能拿到上面我们创建  	 
   * AnnotationConfigApplicationContext时传入的HomeConfig的名称，我们说过了，我们传入的这个类，Spring会将它当做一个
   * Bean处理，当然就会为它创建BeanDefinition，所以我们这里可以拿到它。它被我们标注了@ComponentScan("com.demo")注解，
   * 所以在接下来的解析中就会扫描com.demo包下的文件，进而就可以找到我们所有定义的Bean。
   */
  String[] candidateNames = registry.getBeanDefinitionNames();
  for (String beanName : candidateNames) {
    // 通过beanName拿到BeanDefinition，因为BeanDefinition中封装了该Bean的所有信息
    BeanDefinition beanDef = registry.getBeanDefinition(beanName);
    if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
      if (logger.isDebugEnabled()) {
        logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
      }
    }
    // 检查该Bean是否是一个配置类，前面我们已经说了会将什么样的类作为配置类，分为full和litel两种
    else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
      // 如果是一个配置类，将这个类通过BeanDefinitionHolder包装后放入配置类集合，便于下面统一处理
      configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
    }
  }

  // 如果没有发现配置类，直接返回不再处理
  if (configCandidates.isEmpty()) {
    return;
  }

  // 对配置类进行排序，使用@Order标注，数字小的排在前面
  configCandidates.sort((bd1, bd2) -> {
    int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
    int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
    return Integer.compare(i1, i2);
  });

  // 如果应用程序上下文提供了bean名称生成器，则将该生成器作为Spring配置类的名称生成器
  // 一般不设置
  SingletonBeanRegistry sbr = null;
  if (registry instanceof SingletonBeanRegistry) {
    sbr = (SingletonBeanRegistry) registry;
    if (!this.localBeanNameGeneratorSet) {
      BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
        AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
      if (generator != null) {
        this.componentScanBeanNameGenerator = generator;
        this.importBeanNameGenerator = generator;
      }
    }
  }

  if (this.environment == null) {
    this.environment = new StandardEnvironment();
  }

  // 通过ConfigurationClassParser解析配置类
  ConfigurationClassParser parser = new ConfigurationClassParser(
    this.metadataReaderFactory, this.problemReporter, this.environment,
    this.resourceLoader, this.componentScanBeanNameGenerator, registry);

  Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
  Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
  do {
    // 开始解析
    parser.parse(candidates);
    parser.validate();
    
    // 拿到所有解析出来的配置类信息
    Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
    configClasses.removeAll(alreadyParsed);

    // Read the model and create bean definitions based on its content
    if (this.reader == null) {
      this.reader = new ConfigurationClassBeanDefinitionReader(
        registry, this.sourceExtractor, this.resourceLoader, this.environment,
        this.importBeanNameGenerator, parser.getImportRegistry());
    }
    // 通过ConfigurationClassBeanDefinitionReader加载所有配置类的BeanDefinition
    // 注意上面的parse方法只是解析，并没有将其加载为BeanDefinition，BeanDefinition是在这里依据签名解析出来的信息
    // 进行加载的
    this.reader.loadBeanDefinitions(configClasses);
    alreadyParsed.addAll(configClasses);
    // 清除已经处理完的配置类
    candidates.clear();
    // 如果现在BeanDefinition的数量大于之前的BeanDefinition的数量，说明通过配置类又引入了新的BeanDefinition
    // 接下来要继续处理新引入的BeanDefintion，因为新引入的BeanDefinition可能也是一个配置类，它可能又会引入新的
    // BeanDefinition，直到不再引入新BaenDefinition为止
    if (registry.getBeanDefinitionCount() > candidateNames.length) {
      String[] newCandidateNames = registry.getBeanDefinitionNames();
      Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
      Set<String> alreadyParsedClasses = new HashSet<>();
      // 将目前已经解析过的类添加到alreadyParsedClasses，避免后续重复处理
      for (ConfigurationClass configurationClass : alreadyParsed) {
        alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
      }
      for (String candidateName : newCandidateNames) {
        if (!oldCandidateNames.contains(candidateName)) {
          // 走到这里说明该candidateName是一个新引入的，然后获取它的BeanDefinition，判断其是否还是一个配置类
          // 并且还没有被处理过，如果是，将其添加candidates集合中，以它为基础再次处理
          BeanDefinition bd = registry.getBeanDefinition(candidateName);
          if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
              !alreadyParsedClasses.contains(bd.getBeanClassName())) {
            candidates.add(new BeanDefinitionHolder(bd, candidateName));
          }
        }
      }
      candidateNames = newCandidateNames;
    }
  }
  // 直到candidates为空才停止处理，如果不为空，说明经过处理后一直有新的BeanDefinition加入进来
  while (!candidates.isEmpty());

  // 将ImportRegistry注册为单例bean，以支持实现ImportAware @Configuration类
  // 为什么注册这个单例Bean？
  // 下一篇文章我们会讲到@Import原理，其中导入类实现ImportSelector接口的方式都可以在导入类方法中获取到importMetadata，
  // 但是没有实现该接口的导入类怎么获取到importMetadata呢？这就需要ImportAware接口了，如果导入类实现了ImportAware接口，
  // 在后续实例化Bean的时候，在ImportAwareBeanPostProcessor后置处理器中就会从该单例Bean中取对应的importMetadata设置
  // 到实例中，这样导入类就获取到了importMetadata信息。
  if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
    sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
  }

  if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
    // Clear cache in externally provided MetadataReaderFactory; this is a no-op
    // for a shared cache since it'll be cleared by the ApplicationContext.
    ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
  }
}
```

#### checkConfigurationClassCandidate()

```java
// ConfigurationClassUtils.java
public static boolean checkConfigurationClassCandidate(
  BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

  String className = beanDef.getBeanClassName();
  if (className == null || beanDef.getFactoryMethodName() != null) {
    return false;
  }

  // ...省略无关代码

  Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
  // 如果该类有@Configuration注解，并且允许代理BeanMethods，该类就是一个full类型的配置类
  if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
    beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
  }
  // 如果该类没有@Configuration，则判断其是否一个lite配置类
  else if (config != null || isConfigurationCandidate(metadata)) {
    beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
  }
  else {
    return false;
  }
  return true;
}
```

```java
// ConfigurationClassUtils.java
private static final Set<String> candidateIndicators = new HashSet<>(8);
static {
  candidateIndicators.add(Component.class.getName());
  candidateIndicators.add(ComponentScan.class.getName());
  candidateIndicators.add(Import.class.getName());
  candidateIndicators.add(ImportResource.class.getName());
}

public static boolean isConfigurationCandidate(AnnotationMetadata metadata) {
  // 不能是一个接口
  if (metadata.isInterface()) {
    return false;
  }

  // 是否标注有如上所列的注解
  for (String indicator : candidateIndicators) {
    if (metadata.isAnnotated(indicator)) {
      return true;
    }
  }
  // 如果都没有，看该类中是否存在使用@Bean标注的方法
  try {
    return metadata.hasAnnotatedMethods(Bean.class.getName());
  }
  catch (Throwable ex) {
    if (logger.isDebugEnabled()) {
      logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + ex);
    }
    return false;
  }
}
```

#### parser.parse()

```java
// ConfigurationClassParser.java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
  for (BeanDefinitionHolder holder : configCandidates) {
    BeanDefinition bd = holder.getBeanDefinition();
    try {
      // 由于我们这里是解析注解标注的Bean,所以会走这个方法
      if (bd instanceof AnnotatedBeanDefinition) {
        parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
      }
      else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
        parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
      }
      else {
        parse(bd.getBeanClassName(), holder.getBeanName());
      }
    }
    catch (BeanDefinitionStoreException ex) {
      throw ex;
    }
    catch (Throwable ex) {
      throw new BeanDefinitionStoreException(
        "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
    }
  }
  // 执行需要推迟执行的导入类
  this.deferredImportSelectorHandler.process();
}
```

```java
// ConfigurationClassParser.java
protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
  // 这里会将其封装为一个ConfigurationClass，这个类非常重要，因为在这个类里面保存了通过该配置类以各种方式导入的信息，
  // 该类会保存到ConfigurationClassParser的configurationClasses集合中，解析完成后的后续处理，都是依据这个集合中
  // 的信息进行处理
  processConfigurationClass(new ConfigurationClass(metadata, beanName), DEFAULT_EXCLUSION_FILTER);
}
```

```java
// ConfigurationClassParser.java
protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
  if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
    return;
  }
  
  // 如果同一个configClassName在多处配置了import，将它们合并到一块处理
  ConfigurationClass existingClass = this.configurationClasses.get(configClass);
  if (existingClass != null) {
    if (configClass.isImported()) {
      if (existingClass.isImported()) {
        existingClass.mergeImportedBy(configClass);
      }
      return;
    }
    else {
      // 如果该Bean被显式的定义了，我们则将显式定义的Bean作为最新的，忽略import的
      // 这说明显式定义的Bean优先级比导入的更高
      this.configurationClasses.remove(configClass);
      this.knownSuperclasses.values().removeIf(configClass::equals);
    }
  }
  
  // 递归处理配置类及其超类层次结构
  SourceClass sourceClass = asSourceClass(configClass, filter);
  do {
    // 处理配置类，如果配置类有父类，会将父类作为SourceClass返回，然后继续处理其父类，否则返回null
    sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
  }
  while (sourceClass != null);

  this.configurationClasses.put(configClass, configClass);
}
```

```java
// ConfigurationClassParser.java
protected final SourceClass doProcessConfigurationClass(
  ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
  throws IOException {

  if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
    // 递归处理内嵌类，如果内嵌类中也是一个配置类，则会再次递归调用processConfigurationClass()
    processMemberClasses(configClass, sourceClass, filter);
  }

  // 处理@PropertySource注解，该注解用于引入properties文件
  for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
    sourceClass.getMetadata(), PropertySources.class,
    org.springframework.context.annotation.PropertySource.class)) {
    if (this.environment instanceof ConfigurableEnvironment) {
      // 解析properties文件中的内容，并将其添加到environment.getPropertySources()中
      processPropertySource(propertySource);
    }
    else {
      logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                  "]. Reason: Environment must implement ConfigurableEnvironment");
    }
  }

  // 处理@ComponentScan注解
  Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
    sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
  if (!componentScans.isEmpty() &&
      !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
    // 由此可以看到，可以在一个类上标注多个@ComponentScan注解，会循环处理所有注解
    for (AnnotationAttributes componentScan : componentScans) {
      // 扫描出该注解指定的包下的所有的BeanDefinition，内部使用ClassPathBeanDefinitionScanner进行处理
      // ClassPathBeanDefinitionScanner的内容等待后续讲解
      Set<BeanDefinitionHolder> scannedBeanDefinitions =
        this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
      for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
        BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
        if (bdCand == null) {
          bdCand = holder.getBeanDefinition();
        }
        // 如果扫描出来的Bean还是配置类，则递归解析新扫描出来的配置，可以看到，这里又调回了parse()方法
        if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
          parse(bdCand.getBeanClassName(), holder.getBeanName());
        }
      }
    }
  }

  // 处理@Import注解，通过该注解可以直接导入Bean
  // 这里比较复杂，下一篇详细讲解
  processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

  // 处理@ImportResource注解，该注解可以导入spring的xml配置文件，然后解析该配置文件中定义的Bean
  // 这里将@ImportResource引入的文件路径添加到configClass的importedResource中等待后续处理
  AnnotationAttributes importResource =
    AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
  if (importResource != null) {
    String[] resources = importResource.getStringArray("locations");
    Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
    for (String resource : resources) {
      String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
      configClass.addImportedResource(resolvedResource, readerClass);
    }
  }

  // 解析该类中使用@Bean标注的方法，将BeanMethod添加到configClass中，等待后续处理
  Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
  for (MethodMetadata methodMetadata : beanMethods) {
    configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
  }

  // 递归处理该类实现的接口，判断接口中有没有默认方法标注了@Bean，如果有，将BeanMethod添加到configClass中，等待后续处理
  processInterfaces(configClass, sourceClass);

  // 如果该配置类有父类，将父类作为SourceClass返回，处理其父类
  if (sourceClass.getMetadata().hasSuperClass()) {
    String superclass = sourceClass.getMetadata().getSuperClassName();
    if (superclass != null && !superclass.startsWith("java") &&
        !this.knownSuperclasses.containsKey(superclass)) {
      this.knownSuperclasses.put(superclass, configClass);
      return sourceClass.getSuperClass();
    }
  }
  // 如果没有父类直接返回null
  return null;
}
```

### loadBeanDefinitions()

上面步骤完成了解析，在这一步会将上面解析出来的信息加载成BeanDefinition，通过ConfigurationClassBeanDefinitionReader.loadBeanDefinitions()。

```java
// ConfigurationClassBeanDefinitionReader.java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
  TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
  // 处理前面解析出来的每一个ConfigurationClass
  for (ConfigurationClass configClass : configurationModel) {
    loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
  }
}
```

#### loadBeanDefinitionsForConfigurationClass()

```java
// ConfigurationClassBeanDefinitionReader.java
private void loadBeanDefinitionsForConfigurationClass(
  ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

  if (trackedConditionEvaluator.shouldSkip(configClass)) {
    String beanName = configClass.getBeanName();
    if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
      this.registry.removeBeanDefinition(beanName);
    }
    this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
    return;
  }
  // 注册通过@Import导入的BeanDefinition，类型为AnnotatedGenericBeanDefinition
  if (configClass.isImported()) {
    registerBeanDefinitionForImportedConfigurationClass(configClass);
  }
  // 注册通过@Bean标注的BeanDefinition，类型为ConfigurationClassBeanDefinition
  for (BeanMethod beanMethod : configClass.getBeanMethods()) {
    loadBeanDefinitionsForBeanMethod(beanMethod);
  }
  // 注册通过@ImportResource导入的spring.xml配置文件定义的BeanDefinition，这一步回到了上一篇xml配置那一套流程了
  loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
  // 注册通过实现ImportBeanDefinitionRegistrar接口导入的BeanDefinition
  loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

我们以registerBeanDefinitionForImportedConfigurationClass()为例看一下是怎么注册BeanDefinition的。

```java
// ConfigurationClassBeanDefinitionReader.java
private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
  // 拿到注解元数据
  AnnotationMetadata metadata = configClass.getMetadata();
  // 创建一个注解Bean定义：AnnotatedGenericBeanDefinition
  AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);
  // 解析scope元数据
  ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
  // 设置scope
  configBeanDef.setScope(scopeMetadata.getScopeName());
  // 生成beanName
  String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
  // 处理BeanDefinition通用注解
  AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);
  
  BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
  // 看是否需要对该Bean创建Scope代理，之后将scope的时候会详细介绍
  definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
  // 注册BeanDefinition
  this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
  configClass.setBeanName(configBeanName);

  if (logger.isTraceEnabled()) {
    logger.trace("Registered bean definition for imported class '" + configBeanName + "'");
  }
}
```

```java
// AnnotationConfigUtils.java
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
  // 处理@Lazy注解
  AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
  if (lazy != null) {
    abd.setLazyInit(lazy.getBoolean("value"));
  }
  else if (abd.getMetadata() != metadata) {
    lazy = attributesFor(abd.getMetadata(), Lazy.class);
    if (lazy != null) {
      abd.setLazyInit(lazy.getBoolean("value"));
    }
  }
  // 处理@Primary注解
  if (metadata.isAnnotated(Primary.class.getName())) {
    abd.setPrimary(true);
  }
  // 处理@DependsOn注解
  AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
  if (dependsOn != null) {
    abd.setDependsOn(dependsOn.getStringArray("value"));
  }
  // 处理@Role注解
  AnnotationAttributes role = attributesFor(metadata, Role.class);
  if (role != null) {
    abd.setRole(role.getNumber("value").intValue());
  }
  // 处理@Description注解  
  AnnotationAttributes description = attributesFor(metadata, Description.class);
  if (description != null) {
    abd.setDescription(description.getString("value"));
  }
}
```

### postProcessBeanFactory()

在上面的postProcessBeanDefinitionRegistry()已经找到并注册了所有的BeanDefinition，在这个方法中会对注册的BeanDefinition进行增强，增强的目录是为了避免重复创建Bean。

```java
// ConfigurationClassPostProcessor.java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  int factoryId = System.identityHashCode(beanFactory);
  if (this.factoriesPostProcessed.contains(factoryId)) {
    throw new IllegalStateException(
      "postProcessBeanFactory already called on this post-processor against " + beanFactory);
  }
  this.factoriesPostProcessed.add(factoryId);
  if (!this.registriesPostProcessed.contains(factoryId)) {
    // 因为从上面的postProcessBeanDefinitionRegistry()已经处理过了，所以这里不再处理了
    processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
  }
	// 增强配置类，这里只对full类型的配置类进行增强
  enhanceConfigurationClasses(beanFactory);
  beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}
```

#### enhanceConfigurationClasses()

对full类型的配置类进行增强。

```java
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
  Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
  for (String beanName : beanFactory.getBeanDefinitionNames()) {
    BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
    Object configClassAttr = beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE);
    // ...忽略无关代码
    // 可以看到这里判断了，只有是full类型的配置类才会放到configBeanDefs集合中
    if (ConfigurationClassUtils.CONFIGURATION_CLASS_FULL.equals(configClassAttr)) {
      // ...
      configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
    }
  }
  // 如果没有full类型的配置了，直接返回了
  if (configBeanDefs.isEmpty()) {
    return;
  }
  
  ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
  for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
    AbstractBeanDefinition beanDef = entry.getValue();
    // 设置该Bean需要代理
    beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
    Class<?> configClass = beanDef.getBeanClass();
    // 通过该配置类，生成增强子类，将增强后的子类设置到BeanDefinition的baenClass，这也就意味了等待实例化该Bean的时候，
    // 实例化的是增强后的子类的实例
    // 这里创建增强类是用的CGLIB的方式生成的，其中添加了一个非常重要的拦截器：BeanMethodInterceptor
    Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
    if (configClass != enhancedClass) {
      if (logger.isTraceEnabled()) {
        logger.trace(String.format("Replacing bean definition '%s' existing class '%s' with " +
                                   "enhanced class '%s'", entry.getKey(), configClass.getName(), enhancedClass.getName()));
      }
      beanDef.setBeanClass(enhancedClass);
    }
  }
}
```

#### BeanMethodInterceptor.intercept()

这个拦截器的作用主要是为了拦截我们的full配置类中的方法调用，防止单例Bean的重复创建。参考下面这个例子：

```java
@Configuration
public class HomeConfig {
  @Bean
  public Home home() {
    // 这里user()方法的调用的目地其实是拿到下面创建的那个单例Bean
    // 如果不对这种情况进行处理，调用user()默认会直接new出一个新的User对象来，这样就不是拿到的Spring中的单例Bean实例了
    // 为了避免这种写法导致的程序异常，所以Spring通过增加HomeConfig类，对这种情况进行了特殊处理
    return new Home(user());
  }
  
  // 这里会创建一个单例Bean
  @Bean
  public User user(){
    return new User("Jack", 18);
  }
}
```

拦截器实现：

```java
public Object intercept(Object enhancedConfigInstance, Method beanMethod, Object[] beanMethodArgs,
                        MethodProxy cglibMethodProxy) throws Throwable {

  ConfigurableBeanFactory beanFactory = getBeanFactory(enhancedConfigInstance);
  String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod);

  // 如果该beanMethod上有@Scope注解，需要对scope进行处理
  if (BeanAnnotationHelper.isScopedProxy(beanMethod)) {
    // 如果有scope注解，这里拿到的scopedBeanName是：scopedTarget.beanName
    String scopedBeanName = ScopedProxyCreator.getTargetBeanName(beanName);
    if (beanFactory.isCurrentlyInCreation(scopedBeanName)) {
      beanName = scopedBeanName;
    }
  }

  // 首先，检查请求的bean是否是FactoryBean。如果是这样，创建一个子类代理来拦截对getObject()的调用并返回任何缓存的bean
  // 实例。这确保在@Bean方法中调用FactoryBean的语义与在 XML 中引用 FactoryBean 的语义相同。
  if (factoryContainsBean(beanFactory, BeanFactory.FACTORY_BEAN_PREFIX + beanName) &&
      factoryContainsBean(beanFactory, beanName)) {
    Object factoryBean = beanFactory.getBean(BeanFactory.FACTORY_BEAN_PREFIX + beanName);
    if (factoryBean instanceof ScopedProxyFactoryBean) {
      // Scoped proxy factory beans are a special case and should not be further proxied
    }
    else {
      // It is a candidate FactoryBean - go ahead with enhancement
      return enhanceFactoryBean(factoryBean, beanMethod.getReturnType(), beanFactory, beanName);
    }
  }
  // 判断当前调用的方法是否为@Bean方法本身，如果是，直接调用方法本身逻辑
  // 例如：Spring来调用我们的@Bean方法,则返回true，如果在别的方法中调用@Bean方法，则返回false
  // 注意不要在该方法中调用自己，这样只会导致死循环！！！
  if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
    return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
  }
  // 解析Bean引用，其实就是去beanFactory中那Bean实例，而不是调用方法去执行方法逻辑
  return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);
}
```
增强FactoryBean类型
```java
private Object enhanceFactoryBean(final Object factoryBean, Class<?> exposedType,
                                  final ConfigurableBeanFactory beanFactory, final String beanName) {
  // ...省略无关代码
  return createCglibProxyForFactoryBean(factoryBean, beanFactory, beanName);
}

private Object createInterfaceProxyForFactoryBean(final Object factoryBean, Class<?> interfaceType,
                                                  final ConfigurableBeanFactory beanFactory, final String beanName) {
  return Proxy.newProxyInstance(
    factoryBean.getClass().getClassLoader(), new Class<?>[] {interfaceType},
    (proxy, method, args) -> {
      // 重点看这，对FactoryBean的getObject()进行处理，直接从beanFactory中获取bean实例
      if (method.getName().equals("getObject") && args == null) {
        return beanFactory.getBean(beanName);
      }
      return ReflectionUtils.invokeMethod(method, factoryBean, args);
    });
}
```

增强普通bean

```java
private Object resolveBeanReference(Method beanMethod, Object[] beanMethodArgs,
                                    ConfigurableBeanFactory beanFactory, String beanName) {

  boolean alreadyInCreation = beanFactory.isCurrentlyInCreation(beanName);
  try {
    if (alreadyInCreation) {
      beanFactory.setCurrentlyInCreation(beanName, false);
    }
    boolean useArgs = !ObjectUtils.isEmpty(beanMethodArgs);
    if (useArgs && beanFactory.isSingleton(beanName)) {
      for (Object arg : beanMethodArgs) {
        if (arg == null) {
          useArgs = false;
          break;
        }
      }
    }
    // 重点看这里，也是通过beanFactory直接获取bean实例
    Object beanInstance = (useArgs ? beanFactory.getBean(beanName, beanMethodArgs) :
                           beanFactory.getBean(beanName));
    if (!ClassUtils.isAssignableValue(beanMethod.getReturnType(), beanInstance)) {
      if (beanInstance.equals(null)) {
        beanInstance = null;
      }
      else {
        try {
          BeanDefinition beanDefinition = beanFactory.getMergedBeanDefinition(beanName);
        }
        catch (NoSuchBeanDefinitionException ex) {
        }
        throw new IllegalStateException(msg);
      }
    }
    Method currentlyInvoked = SimpleInstantiationStrategy.getCurrentlyInvokedFactoryMethod();
    if (currentlyInvoked != null) {
      String outerBeanName = BeanAnnotationHelper.determineBeanNameFor(currentlyInvoked);
      beanFactory.registerDependentBean(beanName, outerBeanName);
    }
    return beanInstance;
  }
  finally {
    if (alreadyInCreation) {
      beanFactory.setCurrentlyInCreation(beanName, true);
    }
  }
}
```

回顾ConfigurationClassPostProcessor对配置类进行解析的过程，我们发现Spring在解析配置类的时候，到处充斥着循环和递归，正是这些循环和递归使得Spring可以将各种嵌套情况都处理到。

至此，我们就说完了Spring加载使用注解方式配置的BeanDefinition的流程。