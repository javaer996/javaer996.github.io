---
layout: post
read_time: true
show_date: true
title:  Spring系列-Import的实现原理
subtitle: 
date:   2022-03-04 13:34:20 +0800
description: Spring系列-Import实现原理 @Import ImportAware
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

在上篇文章[BeanDefinition解析流程-注解配置](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/03/Spring%E7%B3%BB%E5%88%97-BeanDefinition%E8%A7%A3%E6%9E%90%E8%B7%AF%E7%A8%8B-%E6%B3%A8%E8%A7%A3%E9%85%8D%E7%BD%AE.html)中我们讲解了@Import注解标注的类会被Spring当前lite类型的配置类进行解析，在ConfigurationClassParser类中有一个processImports()方法上一节我们没讲，这一节我们就详细分析一下它。

首先看一下这一个方法：

```java
// ConfigurationClassParser.java
processImports(configClass, sourceClass, getImports(sourceClass), filter, true);
```

## getImports()

在上面的processImports()方法的参数中，有一个参数调用了getImports()方法，我们来看一下这个方法：

```java
// ConfigurationClassParser.java
private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
  // 注意这里定义了两个Set集合，一个是记录导入的类，一个记录已经处理过的类
  // 之所以定义已经处理过的类，是为了避免循环Import的问题，导致陷入死循环
  Set<SourceClass> imports = new LinkedHashSet<>();
  Set<SourceClass> visited = new LinkedHashSet<>();
  // 收集@Imports注解中的value值
  collectImports(sourceClass, imports, visited);
  return imports;
}

private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
  throws IOException {
  // 因为visited是一个Set集合，所以如果该sourceClass已经存在，调用add方法就会返回false，说明之前已经处理过该sourceClass
  // 直接跳过该sourceClass的处理
  if (visited.add(sourceClass)) {
    // 循环该配置了上的所有注解
    for (SourceClass annotation : sourceClass.getAnnotations()) {
      String annName = annotation.getMetadata().getClassName();
      // 如果注解不是@Import，递归调用collectImports()方法，这里为什么要递归调用呢？
      // 因为其它注解上可能也标注了@Import注解，所以要递归处理到所有能找到的@Import注解
      // 例如Spring系列想用的@EnableXXX注解，都是在@EnableXXX注解上标注@Import注解实现的
      if (!annName.equals(Import.class.getName())) {
        collectImports(annotation, imports, visited);
      }
    }
    // 将该配置类上标注的@Import注解的value值添加到imports集合中
    imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
  }
}
```

## processImports()

该方法主要是处理配置类通过@Import注解导入的类。

```java
public @interface Import {
  Class<?>[] value();
}
```

通过@Import注解的定义我们可以看到，@Import的注解的value是class数组，也就是其可以通过在注解上配置class类型导入多个类。而且通过下面的处理逻辑我们分析出来，@Import注解导入的类分为四种情况分别处理：

- ImportSelector类型处理

  > 如果导入的类实现了ImportSelector接口，在处理该类的时候会直接回调该类的selectImports()方法

- DeferredImportSelector类型处理

  > 该接口是ImportSelector接口的子类，它有了分组group的概念，但是其与ImportSelector最大的不同在于，实现了该接口的类会比实现了ImportSelector接口的类延迟处理，等配置类都解析完才会处理该接口的导入类。

- ImportBeanDefinitionRegistrar类型处理

  > 如果导入类实现了ImportBeanDefinitionRegistrar接口，在这里会将该类添加到configClass中，在后续loadBeanDefinitions()时会调用该类的registerBeanDefinitions()方法

- 普通Bean类型处理

  > 如果导入类没有实现上述几个接口，Spring就会将其当做一个普通配置类，调用processConfigurationClass()走上一篇文章[BeanDefinition解析流程-注解配置](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/03/Spring%E7%B3%BB%E5%88%97-BeanDefinition%E8%A7%A3%E6%9E%90%E8%B7%AF%E7%A8%8B-%E6%B3%A8%E8%A7%A3%E9%85%8D%E7%BD%AE.html)配置类的解析流程

```java
//  ConfigurationClassParser.java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
                            Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
                            boolean checkForCircularImports) {

  if (importCandidates.isEmpty()) {
    return;
  }

  if (checkForCircularImports && isChainedImportOnStack(configClass)) {
    this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
  }
  else {
    this.importStack.push(configClass);
    try {
      for (SourceClass candidate : importCandidates) {
        // 导入的类实现了ImportSelector接口
        if (candidate.isAssignable(ImportSelector.class)) {
          // 加载并实例化该类
          Class<?> candidateClass = candidate.loadClass();
          ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                                                                         this.environment, this.resourceLoader, this.registry);
          Predicate<String> selectorFilter = selector.getExclusionFilter();
          if (selectorFilter != null) {
            exclusionFilter = exclusionFilter.or(selectorFilter);
          }
          // 如果实现的是DeferredImportSelector接口，通过deferredImportSelectorHandler进行处理，这里只是将其
          // 放入了一个延迟执行的集合，等待后续延迟处理
          if (selector instanceof DeferredImportSelector) {
            this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
          }
          else {
            // 实现ImportSelector接口直接调用selectImports()方法
            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
            // 递归调用processImports方法，处理导入类上的@Import注解
            processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
          }
        }
        // 实现了ImportBeanDefinitionRegistrar接口，直接将该类添加到configClass的importBeanDefinitionRegistrar
        // 字段中，等到后续处理
        else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
          Class<?> candidateClass = candidate.loadClass();
          ImportBeanDefinitionRegistrar registrar =
            ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                                                 this.environment, this.resourceLoader, this.registry);
          configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
        }
        else {
          // 如果是普通配置Bean，将以被导入类名为key，导入类注解原信息为value注册到importStack
          // 这里的importStack也就是上一节我们讲的为了适配ImportAware接口注册那个单例Bean
          // 具体位置为：Spring系列-BeanDefinition解析流程-注解配置的processConfigBeanDefinitions()下的121行代码
          this.importStack.registerImport(
            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
          // 然后以该类作为配置类，递归调用processConfigurationClass()走处理配置类逻辑
          processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
        }
      }
    }
    catch (BeanDefinitionStoreException ex) {
      throw ex;
    }
    catch (Throwable ex) {
      throw new BeanDefinitionStoreException(
        "Failed to process import candidates for configuration class [" +
        configClass.getMetadata().getClassName() + "]", ex);
    }
    finally {
      this.importStack.pop();
    }
  }
}
```

其中我们需要主要介绍下实现了DeferredImportSelector接口的导入类的处理方式，其它的要么我们已经介绍过，要么很简单就不解释了。

## DeferredImportSelectorHandler

实现了DeferredImportSelector接口的导入类，会交给DeferredImportSelectorHandler类去处理。

```java
private class DeferredImportSelectorHandler {

  @Nullable
  private List<DeferredImportSelectorHolder> deferredImportSelectors = new ArrayList<>();

  public void handle(ConfigurationClass configClass, DeferredImportSelector importSelector) {
    DeferredImportSelectorHolder holder = new DeferredImportSelectorHolder(configClass, importSelector);
    // 我们看到上面deferredImportSelectors已经赋予一个空集合了，为什么这里还要判断空呢？
    // 这就需要看下面的process()方法，在处理的时候，将deferredImportSelectors又置为null了。
    // 而process()方法是在配置类都处理完成后，最后调用的处理deferredImportSelector类型，在处理的过程中，会将
    // deferredImportSelectors置为空，如果此时又有需要处理的deferredImportSelector,那么立即分组处理
    if (this.deferredImportSelectors == null) {
      // 可以看到，在这种情况下只注册了一个holder，所以此时分组的意义不是太大了
      DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
      handler.register(holder);
      handler.processGroupImports();
    }
    else {
      // 如上所示，正常来说第一次进来的时候，deferredImportSelectors不为null，是一个实例化号的集合，这里仅仅只是做了
      // 一个简单的add将该deferredImportSelector包装后的DeferredImportSelectorHolder添加到集合中，然后等待延迟处理
      this.deferredImportSelectors.add(holder);
    }
  }
  
  // 延迟处理执行逻辑，当所有的配置类都处理完成后，就会调用process方法，处理延迟import方法
  // 这一步是在所有候选配置类执行完parse()方法后调用的
  // 在ConfigurationClassParser.parse(Set<BeanDefinitionHolder> configCandidates)这个重载方法中
  public void process() {
    List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
    // 这里将deferredImportSelectors置为null了，所以说，process执行期间，就不能在往该处理集合中添加延迟处理类了
    // 该次处理完成后，会将deferredImportSelectors集合重置
    this.deferredImportSelectors = null;
    try {
      if (deferredImports != null) {
        // 创建延迟处理分组处理器，里面封装了分组逻辑以及分组处理逻辑
        DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
        // 对deferredImports进行排序，延迟导入类可以通过实现Order接口或使用@Order注解排序
        deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
        // 将延迟导入类分组注册
        deferredImports.forEach(handler::register);
        // 分组处理延迟导入类
        handler.processGroupImports();
      }
    }
    finally {
      // 执行完后，重置deferredImportSelectors集合
      this.deferredImportSelectors = new ArrayList<>();
    }
  }
}
```

## DeferredImportSelectorGroupingHandler

```java
private class DeferredImportSelectorGroupingHandler {
  
  // 用于保存deferredImportSelector分组结果，每一个value都是一个组
  private final Map<Object, DeferredImportSelectorGrouping> groupings = new LinkedHashMap<>();
  // 保存配置类的注解元数据对象和配置类的映射关系
  // 这里的配置类指的是导入DeferredImportSelector实现类的配置类
  private final Map<AnnotationMetadata, ConfigurationClass> configurationClasses = new HashMap<>();

  // 将deferredImportSelector分组注册
  public void register(DeferredImportSelectorHolder deferredImport) {
    // 获取该deferredImportSelector的组
    // getImportGroup()是DeferredImportSelector的一个接口，我们可以通过该接口返回组实现类类型
    Class<? extends Group> group = deferredImport.getImportSelector().getImportGroup();
    // computeIfAbsent如果key不存在则添加，如果存在则返回对应的value
    // 如果组实现类不为null，则用组实现类作为分组key，如果实现类为null，则用当前DeferredImportSelectorHolder对象作为key
    // 如果组实现类都为null，则相当于不分组
    // 如果组实现类已存在，则返回该组对应的DeferredImportSelectorGrouping对象，该对象中保存了改组所有的
    // deferredImportSelector, 如果不存在，则创建一个默认组，组的默认实现类为DefaultDeferredImportSelectorGroup
    // 通过DefaultDeferredImportSelectorGroup可以获取到该组下所有deferredImportSelector导入的类
    DeferredImportSelectorGrouping grouping = this.groupings.computeIfAbsent(
      (group != null ? group : deferredImport),
      key -> new DeferredImportSelectorGrouping(createGroup(group)));
    // 将该deferredImportSelector加入改组，上一行代码相当于创建一个组
    grouping.add(deferredImport);
    // 将配置类注解元数据对象和配置类放入集合中暂存，方便下面获取
    this.configurationClasses.put(deferredImport.getConfigurationClass().getMetadata(),
                                  deferredImport.getConfigurationClass());
  }
  
  // 处理分组导入
  public void processGroupImports() {
    // 循环每一个组
    for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
      Predicate<String> exclusionFilter = grouping.getCandidateFilter();
      // 获取该组导入的所有类并循环处理
      grouping.getImports().forEach(entry -> {
        // 这里用了上一步存放起来的配置类的数据，只需要需要configurationClass是因为下面processImports()方法需要
        ConfigurationClass configurationClass = this.configurationClasses.get(entry.getMetadata());
        try {
          // 对导入的类再次处理@Import，处理级联导入的问题（即导入的类又导入了其它的类）
          processImports(configurationClass, asSourceClass(configurationClass, exclusionFilter),
                         Collections.singleton(asSourceClass(entry.getImportClassName(), exclusionFilter)),
                         exclusionFilter, false);
        }
        catch (BeanDefinitionStoreException ex) {
          throw ex;
        }
        catch (Throwable ex) {
          throw new BeanDefinitionStoreException(
            "Failed to process import candidates for configuration class [" +
            configurationClass.getMetadata().getClassName() + "]", ex);
        }
      });
    }
  }
  // 创建分组
  private Group createGroup(@Nullable Class<? extends Group> type) {
    // 如果分组类为null，默认使用DefaultDeferredImportSelectorGroup.class
    Class<? extends Group> effectiveType = (type != null ? type : DefaultDeferredImportSelectorGroup.class);
    // 实例化组对象，可以看到这里传入到环境变量对象等一些参数
    return ParserStrategyUtils.instantiateClass(effectiveType, Group.class,
                                                ConfigurationClassParser.this.environment,
                                                ConfigurationClassParser.this.resourceLoader,
                                                ConfigurationClassParser.this.registry);
  }
}
```

## DeferredImportSelectorGrouping

```java
private static class DeferredImportSelectorGrouping {
  // 组对象
  private final DeferredImportSelector.Group group;
  // 该组对应的所有deferredImportSelector
  private final List<DeferredImportSelectorHolder> deferredImports = new ArrayList<>();

  DeferredImportSelectorGrouping(Group group) {
    this.group = group;
  }

  public void add(DeferredImportSelectorHolder deferredImport) {
    this.deferredImports.add(deferredImport);
  }

  public Iterable<Group.Entry> getImports() {
    // 循环处理所有的deferredImportSelector
    for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
      // 这一步会将该deferredImportSelector导入的类全部放到imports集合中
      // imports集合是DefaultDeferredImportSelectorGroup默认分组实现类中的一个字段
      this.group.process(deferredImport.getConfigurationClass().getMetadata(),
                         deferredImport.getImportSelector());
    }
    // 返回该组中的imports集合
    return this.group.selectImports();
  }
  // 这是过滤器
  public Predicate<String> getCandidateFilter() {
    Predicate<String> mergedFilter = DEFAULT_EXCLUSION_FILTER;
    for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
      Predicate<String> selectorFilter = deferredImport.getImportSelector().getExclusionFilter();
      if (selectorFilter != null) {
        mergedFilter = mergedFilter.or(selectorFilter);
      }
    }
    return mergedFilter;
  }
}
```

## DefaultDeferredImportSelectorGroup

```java
private static class DefaultDeferredImportSelectorGroup implements Group {
  // 存放该组所有deferredImportSelector导入的类信息
  private final List<Entry> imports = new ArrayList<>();

  @Override
  public void process(AnnotationMetadata metadata, DeferredImportSelector selector) {
    // 处理该组所有deferredImportSelector，将导入的类信息封装为Entry添加到imports集合
    for (String importClassName : selector.selectImports(metadata)) {
      this.imports.add(new Entry(metadata, importClassName));
    }
  }

  // 拿到上面process()处理后导入的所有类信息
  @Override
  public Iterable<Entry> selectImports() {
    return this.imports;
  }
}
```

## ParserStrategyUtils

```java
// ParserStrategyUtils.java
static <T> T instantiateClass(Class<?> clazz, Class<T> assignableTo, Environment environment,
                              ResourceLoader resourceLoader, BeanDefinitionRegistry registry) {

  Assert.notNull(clazz, "Class must not be null");
  Assert.isAssignable(assignableTo, clazz);
  if (clazz.isInterface()) {
    throw new BeanInstantiationException(clazz, "Specified class is an interface");
  }
  ClassLoader classLoader = (registry instanceof ConfigurableBeanFactory ?
                             ((ConfigurableBeanFactory) registry).getBeanClassLoader() : resourceLoader.getClassLoader());
  // 实例化
  T instance = (T) createInstance(clazz, environment, resourceLoader, registry, classLoader);
  // 如果实现了aware接口，执行适配器接口
  // 为什么要在这里执行适配器接口，在bean的初始化流程中不是会执行实现了适配器接口的方法吗？
  // 因为这里还是在获取BeanDefinititon的阶段中，还没走到初始化单例Bean的流程，所以这里的实例化对实现了aware接口
  // 的类进行了适配
  ParserStrategyUtils.invokeAwareMethods(instance, environment, resourceLoader, registry, classLoader);
  return instance;
}
```

```java
// ParserStrategyUtils.java
private static void invokeAwareMethods(Object parserStrategyBean, Environment environment,
                                       ResourceLoader resourceLoader, BeanDefinitionRegistry registry, @Nullable ClassLoader classLoader) {

  if (parserStrategyBean instanceof Aware) {
    if (parserStrategyBean instanceof BeanClassLoaderAware && classLoader != null) {
      ((BeanClassLoaderAware) parserStrategyBean).setBeanClassLoader(classLoader);
    }
    if (parserStrategyBean instanceof BeanFactoryAware && registry instanceof BeanFactory) {
      ((BeanFactoryAware) parserStrategyBean).setBeanFactory((BeanFactory) registry);
    }
    if (parserStrategyBean instanceof EnvironmentAware) {
      ((EnvironmentAware) parserStrategyBean).setEnvironment(environment);
    }
    if (parserStrategyBean instanceof ResourceLoaderAware) {
      ((ResourceLoaderAware) parserStrategyBean).setResourceLoader(resourceLoader);
    }
  }
}
```

至此，Import导入的逻辑就梳理完了。

## ImportAware

接下来我们扩展一下有关Import的功能，ImportAware接口的实现原理。

```java
public interface ImportAware extends Aware {

  // 设置导入类的注解元数据，也就是能获取到导入了实现该接口的类的注解元数据 
  void setImportMetadata(AnnotationMetadata importMetadata);
}
```

举个例子：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

  boolean proxyTargetClass() default false;

  boolean exposeProxy() default false;
}
```

EnableAspectJAutoProxy注解上使用@Import导入了AspectJAutoProxyRegistrar类，当我们在某一个配置类上标注了@EnableAspectJAutoProxy注解后，Spring会将AspectJAutoProxyRegistrar导入，我们怎么才能在AspectJAutoProxyRegistrar类中获取到@EnableAspectJAutoProxy注解中的proxyTargetClass这些信息呢？

实际上如果AspectJAutoProxyRegistrar实现了ImportSelector接口，那么在调用selectImport()接口时，就会将@EnableAspectJAutoProxy注解元数据信息当做参数传入。

```java
public interface ImportSelector {

  // 这里传入的importingClassMetadata实际上就是@EnableAspectJAutoProxy的注解元信息
  String[] selectImports(AnnotationMetadata importingClassMetadata);

  @Nullable
  default Predicate<String> getExclusionFilter() {
    return null;
  }

}
```

但是如果AspectJAutoProxyRegistrar类没有实现ImportSelector接口呢？

这时候就需要ImportAware接口出场了，ImportAware就是为了解决这个问题而存在的。

前面处理Import的时候我们说过了在processImports()方法中有这么一行代码：

```          java
// 将被导入类和导入类注解元信息保存到importStack中
this.importStack.registerImport(currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
```

上一篇文章[BeanDefinition解析流程-注解配置](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/03/Spring%E7%B3%BB%E5%88%97-BeanDefinition%E8%A7%A3%E6%9E%90%E8%B7%AF%E7%A8%8B-%E6%B3%A8%E8%A7%A3%E9%85%8D%E7%BD%AE.html)中我们也强调了一行代码：

```java
// 这就是将上面的importStack对象注册成一个单例Bean
sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
```

如果导入类实现了ImportAware接口，那么在初始化该类的时候，会被**ImportAwareBeanPostProcessor**后置处理器处理：

```java
private static class ImportAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter {

  private final BeanFactory beanFactory;

  public ImportAwareBeanPostProcessor(BeanFactory beanFactory) {
    this.beanFactory = beanFactory;
  }

  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) {
    // 处理实现了ImportAware接口的类
    if (bean instanceof ImportAware) {
      // 这里获取到的就是我们上面注册的importStack单例Bean
      ImportRegistry ir = this.beanFactory.getBean(IMPORT_REGISTRY_BEAN_NAME, ImportRegistry.class);
      // 通过该类名获取到对应导入类的注解元信息
      AnnotationMetadata importingClass = ir.getImportingClassFor(ClassUtils.getUserClass(bean).getName());
      if (importingClass != null) {
        // 调用setImportMetadata()方法传入
        ((ImportAware) bean).setImportMetadata(importingClass);
      }
    }
    return bean;
  }
}
```

综上，就是ImportAware的作用已经实现原理。