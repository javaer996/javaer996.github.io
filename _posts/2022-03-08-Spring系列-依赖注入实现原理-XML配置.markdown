---
layout: post
read_time: true
show_date: true
title:  Spring系列-依赖注入实现原理-XML配置
date:   2022-03-08 15:16:20 +0800
description: Spring系列-依赖注入实现原理-XML配置 自动注入
img: posts/common/spring.png
tags: [spring]
author: tengjiang
toc: yes
---

|| 该文章主要介绍了Spring通过XML配置的Bean属性是如何进行依赖注入的。||

在之前使用Spring的时候，我们都是使用Xml的方式进行配置，虽然现在大部分项目都改为了使用注解方式，但是我们这里还是讲解一下之前的Xml的方式Spring是怎样进行依赖注入的。下一节，我们将讲解注解方式Spring是如何进行注入的。

首先还是使用前面定义的两个类，Home和User。

```java
@Data
public class Home {
  private User user;
}

@Data
public class User {
  private String name;
  private String age;
}
```

在使用Xml配置的时候，我们通常会进行如下配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">
    <!-- 定义一个类型为User，name为user的Bean，并注入name和age属性 -->
    <bean id="user" class="com.demo.User">
        <property name="name" value="jack"/>
        <property name="age" value="18"/>
    </bean>
  
    <!-- home中注入user有两种方式，Spring不推荐使用第二种方式，虽然第二种方式更简洁 -->
    <!-- 如果两种方式同时指定，以第一种方式为准 -->
    <!-- 第一种，通过property指定user字段注入user Bean -->
    <bean id="home" class="com.demo.Home">
        <property name="user" ref="user"/>
    </bean>
  
    <!-- 第二种，通过autowire="byName"指定自动注入，此时Spring会自动根据名称注入user Bean -->
    <bean id="home" class="com.demo.Home" autowire="byName">
    </bean>
</beans>
```

在之前的文章[Spring系列-Spring的Bean创建流程](https://www.tengjiang.site/Spring%E7%B3%BB%E5%88%97-Bean%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.html)我们已经从整体上讲解了Bean的创建流程，在Bean创建的过程中，依赖注入是发生在doGetBean()中的populateBean()方法中，下面我们再次看下populateBean()方法。

## populateBean()

```java
// AbstractAutowireCapableBeanFactory.java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
  // ...忽略注入无关代码
 
  PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
  int resolvedAutowireMode = mbd.getResolvedAutowireMode();
  // 这里是处理自动注入的代码，也就是如果我们在xml中配置了autowire="byName"或者autowire="byType"，便会走下面的
  // 自动注入代码，如果不配置，默认是autowire="NO", 也就是不自动注入
  // 需要注意的是，这里并没有真正进行注入，只是把需要注入的信息都准备好了，真正的注入是在最后的applyPropertyValues()
  if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
    MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
    // 通过名称自动注入
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
      autowireByName(beanName, mbd, bw, newPvs);
    }
    // 通过类型自动注入
    if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
      autowireByType(beanName, mbd, bw, newPvs);
    }
    pvs = newPvs;
  }

  // 如果有InstantiationAwareBeanPostProcessor后置处理器，通过后置处理器处理属性，我们可以定义这种后置处理器修改获取
  // 添加注入属性，注解属性注入就是在后置处理器中处理的，这篇我们先不考虑这里
  boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
  boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
  PropertyDescriptor[] filteredPds = null;
  if (hasInstAwareBpps) {
    if (pvs == null) {
      pvs = mbd.getPropertyValues();
    }
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
        PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
        if (pvsToUse == null) {
          if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
          }
          pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
          if (pvsToUse == null) {
            return;
          }
        }
        pvs = pvsToUse;
      }
    }
  }
  if (needsDepCheck) {
    if (filteredPds == null) {
      filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
    }
    checkDependencies(beanName, mbd, filteredPds, pvs);
  }

  if (pvs != null) {
    // 应用属性值，也就是注入属性
    applyPropertyValues(beanName, mbd, bw, pvs);
  }
}
```
## autowireByName()

这个方法比较简单，因为bean名称不能重复，所以该方法实际上就是通过属性名同beanFactory中获取Bean对象，然后保存到pvs中，以便接下来的属性赋值。

```java
// AbstractAutowireCapableBeanFactory.java
protected void autowireByName(
  String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
  // 获取不满足的非简单属性，这是什么意思？
  // 就是获取符合条件注入条件但是在pvs不存在注入属性值的非简单属性(也就是bean)，下面我们会详解
  String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
  for (String propertyName : propertyNames) {
    // 如果该属性是一个Bean，直接通过getBean获取该属性要注入的bean实例，然后添加到pvs集合中
    if (containsBean(propertyName)) {
      Object bean = getBean(propertyName);
      pvs.add(propertyName, bean);
      // 注册Bean之间的相互依赖关系
      registerDependentBean(propertyName, beanName);
      if (logger.isTraceEnabled()) {
        logger.trace("Added autowiring by name from bean name '" + beanName +
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
```

### unsatisfiedNonSimpleProperties()

获取符合条件注入条件但是在pvs不存在注入属性值的非简单属性。

```java
// AbstractAutowireCapableBeanFactory.java
protected String[] unsatisfiedNonSimpleProperties(AbstractBeanDefinition mbd, BeanWrapper bw) {
  Set<String> result = new TreeSet<>();
  // 已经存在的属性值信息
  PropertyValues pvs = mbd.getPropertyValues();
  // 该bean的所有属性描述信息对象
  PropertyDescriptor[] pds = bw.getPropertyDescriptors();
  // 这一步就是找该bean所有属性描述对象中符合自动注入条件，但是在pvs中不存在属性，如果pvs中已经存在就不用在寻找了
  for (PropertyDescriptor pd : pds) {
    // 存在属性的set方法
    if (pd.getWriteMethod() != null 
        // 不属于要排除的属性依赖
        && !isExcludedFromDependencyCheck(pd) 
        // pvs中不存在该属性
        && !pvs.contains(pd.getName()) 
        // 该属性不是一个简单属性
        && !BeanUtils.isSimpleProperty(pd.getPropertyType())) {
      result.add(pd.getName());
    }
  }
  return StringUtils.toStringArray(result);
}
```
判断该依赖属性是否需要排除？如果需要排除返回true, 不需要排除返回false
```java
// AbstractAutowireCapableBeanFactory.java
// 什么样的属性属于要排除的属性呢？
protected boolean isExcludedFromDependencyCheck(PropertyDescriptor pd) {
         // 1. 没有set方法的属性要排除
         // 2. 是CGLIB生成的方法，需要排除
         // 3. 是代理对象而且被代理对象中不存在该属性set方法需要被排除
  return (AutowireUtils.isExcludedFromDependencyCheck(pd) ||
          // 4. 如果该属性类型为忽略依赖类型的集合中，则排除。可以通过getBeanFactory().ignoreDependencyInterface()
          // 设置需要排除的类型
          this.ignoredDependencyTypes.contains(pd.getPropertyType()) ||
          // 5. 如果该属性的set方法为忽略依赖接口中定义的方法，则排除
          // 在创建该类以及执行prepareBeanFactory()的时候，已经将一系列的Aware接口设置为了忽略接口
          // 引为这些接口定义的方法有一个共同的特点，就是以set开头，都是setXXX()的格式
          AutowireUtils.isSetterDefinedInInterface(pd, this.ignoredDependencyInterfaces));
}
```
判断什么样的方法需要被排除。
```java
// AutowireUtils.java
public static boolean isExcludedFromDependencyCheck(PropertyDescriptor pd) {
  Method wm = pd.getWriteMethod();
  // set方法为空不排除
  // 因为前面已经判断了WriteMethod != null, 所以这里肯定不为空，暂时不知道这里是适配哪里的逻辑
  if (wm == null) {
    return false;
  }
  // 如果该方法的声明类不是CGLIB代理的对象，则不需要排除
  if (!wm.getDeclaringClass().getName().contains("$$")) {
    return false;
  }
  
  // 如果是CGLIB代理类，则拿到superClass，即被代理类，看被代理上是否定义了该属性的set方法，如果定义了，也不排除
  Class<?> superclass = wm.getDeclaringClass().getSuperclass();
  return !ClassUtils.hasMethod(superclass, wm);
}
```
该属性的set方法是否为忽略依赖接口中定义的方法。
```java
// AutowireUtils.java
public static boolean isSetterDefinedInInterface(PropertyDescriptor pd, Set<Class<?>> interfaces) {
  Method setter = pd.getWriteMethod();
  if (setter != null) {
    Class<?> targetClass = setter.getDeclaringClass();
    for (Class<?> ifc : interfaces) {
      // 判断该属性的声明类是否是该接口类型，并且该set方法是否在接口中进行了声明
      if (ifc.isAssignableFrom(targetClass) && ClassUtils.hasMethod(ifc, setter)) {
        return true;
      }
    }
  }
  return false;
}
```

判断是否为简单属性。

```java
// BeanUtils.java
public static boolean isSimpleProperty(Class<?> type) {
  // 如果是数组，获取数组类型进行判断
  return isSimpleValueType(type) || (type.isArray() && isSimpleValueType(type.getComponentType()));
}

public static boolean isSimpleValueType(Class<?> type) {
  return (Void.class != type && void.class != type &&
          // 判断是否是基本类型或包装类型
          (ClassUtils.isPrimitiveOrWrapper(type) ||
           Enum.class.isAssignableFrom(type) ||
           CharSequence.class.isAssignableFrom(type) ||
           Number.class.isAssignableFrom(type) ||
           Date.class.isAssignableFrom(type) ||
           Temporal.class.isAssignableFrom(type) ||
           URI.class == type ||
           URL.class == type ||
           Locale.class == type ||
           Class.class == type));
}
```

### registerDependentBean()

```java
// AbstractAutowireCapableBeanFactory.java
public void registerDependentBean(String beanName, String dependentBeanName) {
  String canonicalName = canonicalName(beanName);
  // 保存该bean与依赖bean的依赖关系
  synchronized (this.dependentBeanMap) {
    Set<String> dependentBeans =
      this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
    if (!dependentBeans.add(dependentBeanName)) {
      return;
    }
  }
  // 保存依赖bean与该bean的依赖关系
  synchronized (this.dependenciesForBeanMap) {
    Set<String> dependenciesForBean =
      this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
    dependenciesForBean.add(canonicalName);
  }
}
```

## autowireByType()

该方法比autowireByName()要复杂，因为根据类型注入的话，一个类型还可能会有很多Bean实例对象，此时要挑选出最合适的一个。

```java
// AbstractAutowireCapableBeanFactory.java
protected void autowireByType(
  String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
  // 获取类型转换器，用于类型转化
  TypeConverter converter = getCustomTypeConverter();
  if (converter == null) {
    // 如果类型转换器为null，将beanWrapper设置为converter，因为BeanWrapper实现了类型转换器接口，也是一个类型转换器
    converter = bw;
  }

  Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
  // 同autowireByName()一样，先找到需要处理的自动注入的属性名
  String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
  for (String propertyName : propertyNames) {
    try {
      PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
			// 不要尝试按类型自动装配 Object 类型：永远不会有意义，即使它在技术上是一个不满意的、不简单的属性。
      if (Object.class != pd.getPropertyType()) {
        // 获取该属性set方法的参数对象
        MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
        // 在优先后处理器的情况下，不允许使用 Eager init 进行类型匹配
        boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
        DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
        // 解析依赖，这里的逻辑比较复杂，而且包含了其它的解析逻辑，所以这块在下一篇文章注解配置里讲解
        Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
        if (autowiredArgument != null) {
          // 添加到pvs
          pvs.add(propertyName, autowiredArgument);
        }
        for (String autowiredBeanName : autowiredBeanNames) {
          // 注册依赖关系
          registerDependentBean(autowiredBeanName, beanName);
          if (logger.isTraceEnabled()) {
            logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
                         propertyName + "' to bean named '" + autowiredBeanName + "'");
          }
        }
        autowiredBeanNames.clear();
      }
    }
    catch (BeansException ex) {
      throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
    }
  }
}
```

## filterPropertyDescriptorsForDependencyCheck()

该方法其实过滤掉不需要处理的属性后，将过滤后的属性返回并缓存起来。

```java
// AbstractAutowireCapableBeanFactory.java
protected PropertyDescriptor[] filterPropertyDescriptorsForDependencyCheck(BeanWrapper bw, boolean cache) {
  PropertyDescriptor[] filtered = this.filteredPropertyDescriptorsCache.get(bw.getWrappedClass());
  if (filtered == null) {
    filtered = filterPropertyDescriptorsForDependencyCheck(bw);
    if (cache) {
      PropertyDescriptor[] existing =
        this.filteredPropertyDescriptorsCache.putIfAbsent(bw.getWrappedClass(), filtered);
      if (existing != null) {
        filtered = existing;
      }
    }
  }
  return filtered;
}
// 看一看到该方法中又用到了上面我们说的isExcludedFromDependencyCheck()方法
protected PropertyDescriptor[] filterPropertyDescriptorsForDependencyCheck(BeanWrapper bw) {
  List<PropertyDescriptor> pds = new ArrayList<>(Arrays.asList(bw.getPropertyDescriptors()));
  pds.removeIf(this::isExcludedFromDependencyCheck);
  return pds.toArray(new PropertyDescriptor[0]);
}
```

## checkDependencies()

检查依赖关系，其实就是判断需要注入依赖的属性是否都有了可以注入的值。

```java
// AbstractAutowireCapableBeanFactory.java
protected void checkDependencies(
  String beanName, AbstractBeanDefinition mbd, PropertyDescriptor[] pds, @Nullable PropertyValues pvs)
  throws UnsatisfiedDependencyException {

  int dependencyCheck = mbd.getDependencyCheck();
  for (PropertyDescriptor pd : pds) {
    // 如果属性set方法存在，当时pvs中到现在还没有该属性的赋值信息，则进行校验
    if (pd.getWriteMethod() != null && (pvs == null || !pvs.contains(pd.getName()))) {
      boolean isSimple = BeanUtils.isSimpleProperty(pd.getPropertyType());
                            // 是否是校验所有，则直接报错
      boolean unsatisfied = (dependencyCheck == AbstractBeanDefinition.DEPENDENCY_CHECK_ALL) ||
        // 如果该属性是简单属性，并且需要校验简单属性，则报错
        (isSimple && dependencyCheck == AbstractBeanDefinition.DEPENDENCY_CHECK_SIMPLE) ||
        // 如果该属性不是简单属性，并且需要校验非简单属性，则报错
        (!isSimple && dependencyCheck == AbstractBeanDefinition.DEPENDENCY_CHECK_OBJECTS);
      if (unsatisfied) {
        throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, pd.getName(),
                                                 "Set this property value or disable dependency checking for this bean.");
      }
    }
  }
}
```

## applyPropertyValues()

真正的为属性赋值。

```java
// AbstractAutowireCapableBeanFactory.java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs){
  // 如果没有可以注入的属性，则返回
  if (pvs.isEmpty()) {
    return;
  }

  if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
    ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
  }

  MutablePropertyValues mpvs = null;
  List<PropertyValue> original;

  // MutablePropertyValues 是 PropertyValues 的默认实现类，它允许简单的属性操作，并提供构造函数来支持从Map
  // 进行深度复制和构造
  if (pvs instanceof MutablePropertyValues) {
    mpvs = (MutablePropertyValues) pvs;
    if (mpvs.isConverted()) {
      // 如果需要注入的属性都已经进行了转换，则可以直接注入，说明在前面的步骤已经处理过了
      try {
        bw.setPropertyValues(mpvs);
        return;
      }
      catch (BeansException ex) {
        throw new BeanCreationException(
          mbd.getResourceDescription(), beanName, "Error setting property values", ex);
      }
    }
    original = mpvs.getPropertyValueList();
  }
  else {
    original = Arrays.asList(pvs.getPropertyValues());
  }
  // 如果没有进行过转换，则获取类型转换器，进行转换
  TypeConverter converter = getCustomTypeConverter();
  if (converter == null) {
    converter = bw;
  }
  // 在bean工厂实现中使用的Helper类，解析bean定义对象中包含的值,转换为应用于目标bean实例的实际值。
  BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

  // 创建深拷贝，解决值的任何引用
  List<PropertyValue> deepCopy = new ArrayList<>(original.size());
  boolean resolveNecessary = false;
  for (PropertyValue pv : original) {
    if (pv.isConverted()) {
      // 如果该属性已经转换过了，则直接添加到深拷贝集合，说明之前处理了
      deepCopy.add(pv);
    }
    else {
      String propertyName = pv.getName();
      Object originalValue = pv.getValue();
      // 这里是自动注入属性的标记对象，如果标记为这个对象，最后是通过AutowireCapableBeanFactory#resolveDependency()
      // 解析出实际的注入对象
      if (originalValue == AutowiredPropertyMarker.INSTANCE) {
        Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
        if (writeMethod == null) {
          throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
        }
        originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
      }
      // 解析属性注入值，如果像我们开头所说的第一种配置方式，这里获取的originalValue为RuntimeBeanReference，
      // 这代表运行时解析，这一步就会将RuntimeBeanReference解析为真正的bean对象
      Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
      Object convertedValue = resolvedValue;
      // 如果有set方法，并且不是内嵌属性就可以进行转换
      boolean convertible = bw.isWritableProperty(propertyName) &&
        !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
      if (convertible) {
        // 对值进行转换，对于我们bean对象注入来说没什么太大作用，因为转换之后还是该Bean，但是对其它类型不一致的就有必要了
        // 例如将字符串转换为Integer等，类型转换在后面会专门写文章讲解
        convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
      }
      // 如果解析后的值和原始值一样，并且该值可转换，则设置转换后的值
      // 这里是在某些情况下才会走，因为Spring很复杂，而且方法通用，所以代码需要兼容各种情况
      if (resolvedValue == originalValue) {
        if (convertible) {
          pv.setConvertedValue(convertedValue);
        }
        deepCopy.add(pv);
      }
      // 这里也是处理某种情况
      else if (convertible && originalValue instanceof TypedStringValue &&
               !((TypedStringValue) originalValue).isDynamic() &&
               !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
        pv.setConvertedValue(convertedValue);
        deepCopy.add(pv);
      }
      else {
        // 这里将值设置为需要解析，并添加到深拷贝集合
        // 像我们的第一种配置，就会走到这里
        resolveNecessary = true;
        deepCopy.add(new PropertyValue(pv, convertedValue));
      }
    }
  }
  // 如果mpvs不为空，而且不需要解析，说明所有需要注入的属性都已经解析了，则设置mpvs为已转换，这里设置的目的是这里处理了
  // 后面的步骤就不用再次处理了，否则后面还需要处理转换
  if (mpvs != null && !resolveNecessary) {
    mpvs.setConverted();
  }

  try {
    // 对属性进行赋值
    bw.setPropertyValues(new MutablePropertyValues(deepCopy));
  }
  catch (BeansException ex) {
    throw new BeanCreationException(
      mbd.getResourceDescription(), beanName, "Error setting property values", ex);
  }
}
```

### resolveValueIfNecessary()

该方法就是根据值的不同类型进行不同的处理，实际上就是解析为真正的注入对象。因为在解析BeanDefinition的时候，不同的情况我们需要在pvs中设置不同的占位，这里就是根据签名不同的占位执行不同的解析。

```java
// BeanDefinitionValueResolver.java
public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
  // 像我们上面的第一种配置ref="user",就是在BeanDefinition保存为RuntimeBeanReference，这里则会真正去拿到真正的Bean
  if (value instanceof RuntimeBeanReference) {
    RuntimeBeanReference ref = (RuntimeBeanReference) value;
    return resolveReference(argName, ref);
  }
  else if (value instanceof RuntimeBeanNameReference) {
    String refName = ((RuntimeBeanNameReference) value).getBeanName();
    refName = String.valueOf(doEvaluate(refName));
    if (!this.beanFactory.containsBean(refName)) {
      throw new BeanDefinitionStoreException(
        "Invalid bean name '" + refName + "' in bean reference for " + argName);
    }
    return refName;
  }
  else if (value instanceof BeanDefinitionHolder) {
    // Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
    BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
    return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
  }
  else if (value instanceof BeanDefinition) {
    // Resolve plain BeanDefinition, without contained name: use dummy name.
    BeanDefinition bd = (BeanDefinition) value;
    String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
      ObjectUtils.getIdentityHexString(bd);
    return resolveInnerBean(argName, innerBeanName, bd);
  }
  else if (value instanceof DependencyDescriptor) {
    Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
    Object result = this.beanFactory.resolveDependency(
      (DependencyDescriptor) value, this.beanName, autowiredBeanNames, this.typeConverter);
    for (String autowiredBeanName : autowiredBeanNames) {
      if (this.beanFactory.containsBean(autowiredBeanName)) {
        this.beanFactory.registerDependentBean(autowiredBeanName, this.beanName);
      }
    }
    return result;
  }
  else if (value instanceof ManagedArray) {
    // May need to resolve contained runtime references.
    ManagedArray array = (ManagedArray) value;
    Class<?> elementType = array.resolvedElementType;
    if (elementType == null) {
      String elementTypeName = array.getElementTypeName();
      if (StringUtils.hasText(elementTypeName)) {
        try {
          elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
          array.resolvedElementType = elementType;
        }
        catch (Throwable ex) {
          // Improve the message by showing the context.
          throw new BeanCreationException(
            this.beanDefinition.getResourceDescription(), this.beanName,
            "Error resolving array type for " + argName, ex);
        }
      }
      else {
        elementType = Object.class;
      }
    }
    return resolveManagedArray(argName, (List<?>) value, elementType);
  }
  else if (value instanceof ManagedList) {
    // May need to resolve contained runtime references.
    return resolveManagedList(argName, (List<?>) value);
  }
  else if (value instanceof ManagedSet) {
    // May need to resolve contained runtime references.
    return resolveManagedSet(argName, (Set<?>) value);
  }
  else if (value instanceof ManagedMap) {
    // May need to resolve contained runtime references.
    return resolveManagedMap(argName, (Map<?, ?>) value);
  }
  else if (value instanceof ManagedProperties) {
    Properties original = (Properties) value;
    Properties copy = new Properties();
    original.forEach((propKey, propValue) -> {
      if (propKey instanceof TypedStringValue) {
        propKey = evaluate((TypedStringValue) propKey);
      }
      if (propValue instanceof TypedStringValue) {
        propValue = evaluate((TypedStringValue) propValue);
      }
      if (propKey == null || propValue == null) {
        throw new BeanCreationException(
          this.beanDefinition.getResourceDescription(), this.beanName,
          "Error converting Properties key/value pair for " + argName + ": resolved to null");
      }
      copy.put(propKey, propValue);
    });
    return copy;
  }
  else if (value instanceof TypedStringValue) {
    // Convert value to target type here.
    TypedStringValue typedStringValue = (TypedStringValue) value;
    Object valueObject = evaluate(typedStringValue);
    try {
      Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
      if (resolvedTargetType != null) {
        return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
      }
      else {
        return valueObject;
      }
    }
    catch (Throwable ex) {
      // Improve the message by showing the context.
      throw new BeanCreationException(
        this.beanDefinition.getResourceDescription(), this.beanName,
        "Error converting typed String value for " + argName, ex);
    }
  }
  else if (value instanceof NullBean) {
    return null;
  }
  else {
    return evaluate(value);
  }
}
```

### convertForProperty()

这里就是将输入的注入值进行类型转换，转换为我们定义的注入类型。

```java
private Object convertForProperty(
  @Nullable Object value, String propertyName, BeanWrapper bw, TypeConverter converter) {
  // 转换最终都会调用到this.typeConverterDelegate.convertIfNecessary(),这个我们后面再说
  if (converter instanceof BeanWrapperImpl) {
    return ((BeanWrapperImpl) converter).convertForProperty(value, propertyName);
  }
  else {
    PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
    MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
    return converter.convertIfNecessary(value, pd.getPropertyType(), methodParam);
  }
}
```

### setPropertyValues()

```java
// AbstractNestablePropertyAccessor.java
public void setPropertyValue(String propertyName, @Nullable Object value) throws BeansException {
  AbstractNestablePropertyAccessor nestedPa;
  try {
    // 获取内嵌属性访问器
    // 什么是内嵌属性？例如：home.user.age, names[0], users["Jack"].age等等
    nestedPa = getPropertyAccessorForPropertyPath(propertyName);
  }
  catch (NotReadablePropertyException ex) {
    throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName,
                                           "Nested property in path '" + propertyName + "' does not exist", ex);
  }
  /**
   protected static class PropertyTokenHolder {
    // 对应bean中的属性名称，如嵌套属性user['Jack'].age 在bean中的属性名称为user
    public String actualName;
    // 将原始的嵌套属性处理成标准的token，如user["Jack"].age处理成user[Jack].age
    public String canonicalName;
    // 这个数组存放的是嵌套属性[]中的内容，如user["Jack"].age处理成 ["Jack", "age"]
    public String[] keys;
  */
  // 获取属性token，其实就是将上面的内嵌属性转换为好解析的格式进行保存
  PropertyTokenHolder tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
  // 赋值
  nestedPa.setPropertyValue(tokens, new PropertyValue(propertyName, value));
}
```

```java
// AbstractNestablePropertyAccessor.java
protected AbstractNestablePropertyAccessor getPropertyAccessorForPropertyPath(String propertyPath) {
  // 这里是获取内嵌属性第一个分隔符的位置，其实就是找第一个.或者[的位置
  int pos = PropertyAccessorUtils.getFirstNestedPropertySeparatorIndex(propertyPath);
  // 如果位置>-1，则说明是内嵌属性，递归处理内嵌属性，获取每一个内嵌属性访问器
  // 每一个内嵌属性都需要有各自的属性访问器进行处理
  if (pos > -1) {
    String nestedProperty = propertyPath.substring(0, pos);
    String nestedPath = propertyPath.substring(pos + 1);
    // 获取属性访问器
    AbstractNestablePropertyAccessor nestedPa = getNestedPropertyAccessor(nestedProperty);
    // 将内嵌属性去除上一层后进行递归处理，例如：home.user.age去除home后再处理user.age
    return nestedPa.getPropertyAccessorForPropertyPath(nestedPath);
  }
  else {
    return this;
  }
}
```

```java
// AbstractNestablePropertyAccessor.java
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
  if (tokens.keys != null) {
    // 处理内嵌属性
    processKeyedProperty(tokens, pv);
  }
  else {
    // 处理普通属性
    processLocalProperty(tokens, pv);
  }
}
```

```java
// AbstractNestablePropertyAccessor.java
private void processLocalProperty(PropertyTokenHolder tokens, PropertyValue pv) {
  // 获取属性处理器进行属性注入
  PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);

  Object oldValue = null;
  try {
    Object originalValue = pv.getValue();
    Object valueToApply = originalValue;
    if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
      if (pv.isConverted()) {
        // 如果属性值已经转换过，直接获取转换过的属性值
        valueToApply = pv.getConvertedValue();
      }
      else {
        if (isExtractOldValueForEditor() && ph.isReadable()) {
          try {
            // 如果没有转换过，获取原始属性值
            oldValue = ph.getValue();
          }
          catch (Exception ex) {
            if (ex instanceof PrivilegedActionException) {
              ex = ((PrivilegedActionException) ex).getException();
            }
            if (logger.isDebugEnabled()) {
              logger.debug("Could not read previous value of property '" +
                           this.nestedPath + tokens.canonicalName + "'", ex);
            }
          }
        }
        // 进行转换
        valueToApply = convertForProperty(
          tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());
      }
      pv.getOriginalPropertyValue().conversionNecessary = (valueToApply != originalValue);
    }
    // 注入
    ph.setValue(valueToApply);
  }
  catch (TypeMismatchException ex) {
    throw ex;
  }
```

## setValue

```java
// BeanPropertyHandler.java
public void setValue(@Nullable Object value) throws Exception {
  Method writeMethod = (this.pd instanceof GenericTypeAwarePropertyDescriptor ?
                        ((GenericTypeAwarePropertyDescriptor) this.pd).getWriteMethodForActualAccess() :
                        this.pd.getWriteMethod());
  if (System.getSecurityManager() != null) {
    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
      ReflectionUtils.makeAccessible(writeMethod);
      return null;
    });
    try {
      AccessController.doPrivileged((PrivilegedExceptionAction<Object>)
                                    () -> writeMethod.invoke(getWrappedInstance(), value), acc);
    }
    catch (PrivilegedActionException ex) {
      throw ex.getException();
    }
  }
  else {
    // 通过反射，调用set方法进行赋值
    ReflectionUtils.makeAccessible(writeMethod);
    writeMethod.invoke(getWrappedInstance(), value);
  }
}
```

至此，我们就说完了Xml配置的属性依赖注入的逻辑，但是里面还有很多的细节我们还没有说到，具体细节后面会慢慢讲到。

