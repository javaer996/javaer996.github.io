---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring的BeanDefinition详解
date:   2022-02-22 04:28:20 +0800
description: Spring的BeanDefinition详解，分析每一个字段的作用
img: posts/common/spring.png
tags: [spring]
author: tengjiang
toc: yes
---

|| 该文章主要介绍了Spring的BeanDefinition的结构，以及每个字段的作用，并分析了每个实现类都在什么情况下使用。||

## BeanDefinition是什么

BeanDefinition是Spring用来保存关于Bean定义信息的。在Spring中，我们可以定义Bean的各种属性，生命周期，初始化等，而且不仅可以使用xml配置，还可以使用注解配置，这些配置最终都会保存在BeanDefinition对象中，Spring后续关于Bean的流程都是依据BeanDefinition展开的。

## BeanDefinition整体架构

![beanDefinition](https://s2.loli.net/2022/02/16/h98sDqHS1zvifZQ.png)

如上图所示，BeanDefinition接口继承自两个接口：BeanMetadataElement和AttributeAccessor。

而且BeanDefinition的实现类众多，包括：

- RootBeanDefinition
- ChildBeanDefinition
- ClassDerivedBeanDefinition
- GenericBeanDefinition
- AnnotatedGenericBeanDefinition
- ScannedGenericBeanDefinition
- ConfigurationClassBeanDefinition

下面我们就从基本接口定义到实现类逐个讲解。

## BeanMetadataElement

该接口的定义很简单，就一个方法，用于获取Bean的源对象。

```java
public interface BeanMetadataElement {
  @Nullable
  Object getSource();
}
```

## AttributeAccessor

该接口主要定义了对对象元数据访问接口。

```java
public interface AttributeAccessor {
  void setAttribute(String name, @Nullable Object value);

  @Nullable
  Object getAttribute(String name);

  @Nullable
  Object removeAttribute(String name);

  boolean hasAttribute(String name);

  String[] attributeNames();
}
```

## BeanDefinition

该接口是BeanDefinition的顶级接口，定义了一些要作为一个BeanDefinition需要实现的最基本的方法。

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
  
  // spring中bean的scope默认是singleton，即单例的
  String SCOPE_SINGLETON = "singleton";
  
  // spring中bean的scope如果是prototype，代表该bean是多例的
  String SCOPE_PROTOTYPE = "prototype";

  // 代表该bean是用户定义的
  int ROLE_APPLICATION = 0;

  // 代表该bean是某些配置的支撑部分
  int ROLE_SUPPORT = 1;

  // 代表该bean是spring内部定义的
  int ROLE_INFRASTRUCTURE = 2;

  // 设置该bean的父类名
  void setParentName(@Nullable String parentName);

  @Nullable
  String getParentName();

  // 设置该bean的全类名
  void setBeanClassName(@Nullable String beanClassName);

  @Nullable
  String getBeanClassName();

  // 设置该bean的作用域，singleton，prototyp
  void setScope(@Nullable String scope);

  @Nullable
  String getScope();

  // 设置该bean是否懒加载，如果设置了懒加载，该bean在启动的时候不会创建，在第一次使用的时候才会创建
  void setLazyInit(boolean lazyInit);

  boolean isLazyInit();

  // 设置该bean的依赖bean，可以控制bean创建的先后顺序
  void setDependsOn(@Nullable String... dependsOn);

  @Nullable
  String[] getDependsOn();
  
  // 设置该bean是否自动装配
  void setAutowireCandidate(boolean autowireCandidate);

  boolean isAutowireCandidate();

  // 设置该bean是否是首选，如果设置为首选，在spring根据条件查找到多个符合条件的bean时，会返回首选bean
  void setPrimary(boolean primary);

  boolean isPrimary();

  // 设置FactoryBean名称，FactoryBean也是一个bean，它可以用来控制bean的生成
  void setFactoryBeanName(@Nullable String factoryBeanName);

  @Nullable
  String getFactoryBeanName();

  // 设置FactoryMethod的名称
  // 如果该bean是一个工厂bean，通过设置的FacotoryMethodName方法可以获取该工厂bean生成的bean
  void setFactoryMethodName(@Nullable String factoryMethodName);

  @Nullable
  String getFactoryMethodName();
  
  // 获取该bean构造方法参数
  ConstructorArgumentValues getConstructorArgumentValues();

  // 判断该bean构造方法是否有参数
  default boolean hasConstructorArgumentValues() {
    return !getConstructorArgumentValues().isEmpty();
  }

  // 获取该bean属性集合
  MutablePropertyValues getPropertyValues();

  // 判断该bean是否有属性
  default boolean hasPropertyValues() {
    return !getPropertyValues().isEmpty();
  }
  
  // 判断该bean是否是单例的
  boolean isSingleton();

  // 判断该bean是否是多例的
  boolean isPrototype();

  // 判断该bean是否是抽象的，抽象的不能实例化
  boolean isAbstract();

  // 获取该bean的角色，ROLE_APPLICATION，ROLE_SUPPORT，ROLE_INFRASTRUCTURE
  int getRole();

  // 获取该bean的描述信息
  @Nullable
  String getDescription();

  // 获取该bean的资源描述信息
  @Nullable
  String getResourceDescription();

  // 获取该bean的原始bean定义，如果当前beanDefinition是一个代理对象，那么可以通过该方法获取原始的beanDefinition
  @Nullable
  BeanDefinition getOriginatingBeanDefinition();
}
```

## AttributeAccessorSupport

该类是AttributeAccessor的抽象实现类，定义了一个Map用于保存属性信息，作为AttributeAccessor接口的一个默认接口实现。

```java
public abstract class AttributeAccessorSupport implements AttributeAccessor, Serializable {

  /** Map with String keys and Object values */
  private final Map<String, Object> attributes = new LinkedHashMap<>();

  @Override
  public void setAttribute(String name, @Nullable Object value) {
    Assert.notNull(name, "Name must not be null");
    if (value != null) {
      this.attributes.put(name, value);
    }
    else {
      removeAttribute(name);
    }
  }

  @Override
  @Nullable
  public Object getAttribute(String name) {
    Assert.notNull(name, "Name must not be null");
    return this.attributes.get(name);
  }

  @Override
  @Nullable
  public Object removeAttribute(String name) {
    Assert.notNull(name, "Name must not be null");
    return this.attributes.remove(name);
  }

  @Override
  public boolean hasAttribute(String name) {
    Assert.notNull(name, "Name must not be null");
    return this.attributes.containsKey(name);
  }

  @Override
  public String[] attributeNames() {
    return StringUtils.toStringArray(this.attributes.keySet());
  }

  ... // 忽略其它代码
}
```

## BeanMetadataAttributeAccessor

该类继承了AttributeAccessorSupport并实现了BeanMetadataElement接口，其中重写了AttributeAccessorSupport中的部分接口。

```java
public class BeanMetadataAttributeAccessor extends AttributeAccessorSupport implements BeanMetadataElement {

  @Nullable
  private Object source;

  public void setSource(@Nullable Object source) {
    this.source = source;
  }

  @Override
  @Nullable
  public Object getSource() {
    return this.source;
  }

  public void addMetadataAttribute(BeanMetadataAttribute attribute) {
    super.setAttribute(attribute.getName(), attribute);
  }

  @Nullable
  public BeanMetadataAttribute getMetadataAttribute(String name) {
    return (BeanMetadataAttribute) super.getAttribute(name);
  }

  @Override
  public void setAttribute(String name, @Nullable Object value) {
    super.setAttribute(name, new BeanMetadataAttribute(name, value));
  }

  @Override
  @Nullable
  public Object getAttribute(String name) {
    BeanMetadataAttribute attribute = (BeanMetadataAttribute) super.getAttribute(name);
    return (attribute != null ? attribute.getValue() : null);
  }

  @Override
  @Nullable
  public Object removeAttribute(String name) {
    BeanMetadataAttribute attribute = (BeanMetadataAttribute) super.removeAttribute(name);
    return (attribute != null ? attribute.getValue() : null);
  }
}
```

## AbstractBeanDefinition

该类是bean定义的抽象类，实现了一些通用逻辑，定义了一些字段用于标识bean的各种情况。

```java
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
    implements BeanDefinition, Cloneable {

  public static final String SCOPE_DEFAULT = "";

  // 不自动注入
  public static final int AUTOWIRE_NO = AutowireCapableBeanFactory.AUTOWIRE_NO;

  // 依据名称注入
  public static final int AUTOWIRE_BY_NAME = AutowireCapableBeanFactory.AUTOWIRE_BY_NAME;

  // 依据类型注入
  public static final int AUTOWIRE_BY_TYPE = AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE;

  // 依据构造方法注入
  public static final int AUTOWIRE_CONSTRUCTOR = AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR;

  // 已过期，自动推断注入方式
  @Deprecated
  public static final int AUTOWIRE_AUTODETECT = AutowireCapableBeanFactory.AUTOWIRE_AUTODETECT;

  // 不需要检查依赖
  public static final int DEPENDENCY_CHECK_NONE = 0;

  // 检查对象引用的依赖
  public static final int DEPENDENCY_CHECK_OBJECTS = 1;

  // 检查简单属性的依赖
  public static final int DEPENDENCY_CHECK_SIMPLE = 2;

  // 对象属性和简单属性都检查依赖
  public static final int DEPENDENCY_CHECK_ALL = 3;

  public static final String INFER_METHOD = "(inferred)";

  // 用于保存bean的class对象
  @Nullable
  private volatile Object beanClass;

  // 默认scope为"", 但是下面方法判断中，如果scope是""代表是单例的
  @Nullable
  private String scope = SCOPE_DEFAULT;
  
  // 默认当前bean不是抽象的
  private boolean abstractFlag = false;

  // 默认当前bean不是懒加载的
  private boolean lazyInit = false;

  // 默认注入默认为不自动注入
  private int autowireMode = AUTOWIRE_NO;

  // 默认不进行依赖检查（Spring3.0之后弃用这个属性）
  private int dependencyCheck = DEPENDENCY_CHECK_NONE;

  // 用于保存当前的bean创建前，必须先创建哪些bean
  @Nullable
  private String[] dependsOn;

  // 如果设置为false, 容器在自动装配对象的时候，不会将该bean作为其他bean依赖的bean，但是该bean依赖的bean
  // 还是能够自动装配进来的
  private boolean autowireCandidate = true;

  // 当发生自动装配的时候，假如某个bean会发现多个,那么标注了primary为true的首先被注入
  private boolean primary = false;

  private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();

  @Nullable
  private Supplier<?> instanceSupplier;

  // 默认允许访问非public的构造方法
  private boolean nonPublicAccessAllowed = true;

  // 默认构造函数使用宽松模式构造的方式
  // 在大部分情况下都是使用宽松模式，即使多个构造函数的参数数量相同、类型存在父子类、接口实现类关系也能正常创建bean
  private boolean lenientConstructorResolution = true;

  // 假如是通过@Configuration的@Bean的方式扫描进来的组件，那么该属性用于保存我们的是由哪个配置类
  @Nullable
  private String factoryBeanName;

  // 用于保存我们的标注了@Bean的方法名称或者xml配置中配置的factory-method
  @Nullable
  private String factoryMethodName;

  // 保存了构造参数的值
  @Nullable
  private ConstructorArgumentValues constructorArgumentValues;

  // 保存了普通属性的集合
  @Nullable
  private MutablePropertyValues propertyValues;

  // 保存了重写属性的集合，look-method 和replace-method
  @Nullable
  private MethodOverrides methodOverrides;

  // 保存了配置的初始化方法名，比如xml中配置的init-method
  @Nullable
  private String initMethodName;

  // 保存了配置的销毁方法名，比如xml中配置的destory-method 
  @Nullable
  private String destroyMethodName;
  
  // 是否执行 init-method 方法
  private boolean enforceInitMethod = true;

  // 是否执行 destroy-method 方法
  private boolean enforceDestroyMethod = true;

  // 是否是用户自定义的bean，创建AOP时为true
  private boolean synthetic = false;
  
  // 默认是用户定义的
  private int role = BeanDefinition.ROLE_APPLICATION;

  protected AbstractBeanDefinition() {
    this(null, null);
  }

  ...// 省略部分代码
  
  @Override
  public boolean isSingleton() {
    return SCOPE_SINGLETON.equals(this.scope) || SCOPE_DEFAULT.equals(this.scope);
  }

  @Override
  public boolean isPrototype() {
    return SCOPE_PROTOTYPE.equals(this.scope);
  }

  ...// 省略部分代码
}
```

## AnnotatedBeanDefinition

该接口继承自BeanDefinition，添加了获取注解元数据信息的方法。

```java
public interface AnnotatedBeanDefinition extends BeanDefinition {
    AnnotationMetadata getMetadata();

    @Nullable
    MethodMetadata getFactoryMethodMetadata();
}
```

## RootBeanDefinition

RootBeanDefinition是合并后的 BeanDefinition 对象。在Spring初始化Bean的之前，会将具有层次性关系的BeanDefinition进行合并，生成一个 RootBeanDefinition，用于后续实例化和初始化。

```java
public class RootBeanDefinition extends AbstractBeanDefinition {

  // BeanDefinitionHolder存储有Bean的名称、别名、BeanDefinition
  @Nullable
  private BeanDefinitionHolder decoratedDefinition;
  
  // 是java反射包的接口，通过它可以查看Bean的注解信息
  @Nullable
  private AnnotatedElement qualifiedElement;

  // 标识这个BeanDefinition是否是陈旧的，如果是true的话，在创建实例的时候会重新合并BeanDefinition
  // 在markBeanAsCreated()会将该值变为true
  // 至于为什么需要重新合并，是因为在bean工厂后置处理器中可能会对BeanDefinition进行修改,所以后续可能需要重新合并
  volatile boolean stale;
  
  // 是否允许缓存
  boolean allowCaching = true;

  // 工厂方法是否唯一
  boolean isFactoryMethodUnique = false;

  // 封装了java.lang.reflect.Type,提供了泛型相关的操作
  @Nullable
  volatile ResolvableType targetType;

  // 缓存class，表明RootBeanDefinition存储哪个类的信息
  @Nullable
  volatile Class<?> resolvedTargetType;

  // 是否是FactoryBean
  @Nullable
  volatile Boolean isFactoryBean;

  // 工厂方法的返回类型
  @Nullable
  volatile ResolvableType factoryMethodReturnType;

  // 工厂方法
  @Nullable
  volatile Method factoryMethodToIntrospect;

  // 下面四个变量的锁
  final Object constructorArgumentLock = new Object();

  // 缓存已经解析的构造函数或是工厂方法，Executable是Method、Constructor类型的父类
  @Nullable
  Executable resolvedConstructorOrFactoryMethod;

  // 构造函数参数是否解析完毕
  boolean constructorArgumentsResolved = false;

  // 已解析出来的构造函数参数
  @Nullable
  Object[] resolvedConstructorArguments;

  // 待解析的构造函数参数
  @Nullable
  Object[] preparedConstructorArguments;

  // 下面两个变量的锁
  final Object postProcessingLock = new Object();

  // 是否已通过MergedBeanDefinitionPostProcessor后置处理器处理
  boolean postProcessed = false;

  // 在生成代理的时候用来判断是否已经生成代理
  @Nullable
  volatile Boolean beforeInstantiationResolved;

  // 实际缓存的类型是Constructor、Field、Method类型，它们都是Member的子类型
  // 在checkConfigMembers()的时候会将需要注入的信息添加进来，保证需要注入的同一个字段或方法等只会注入一次
  // 例如:如果在同一个字段上既有@Resource注解，又有@Autowired注解，由于@Resource注解先解析，所以会忽略@Autowired注解
  @Nullable
  private Set<Member> externallyManagedConfigMembers;

  // 缓存的是通过注解已经执行过的初始化方法，防止同一个初始化方法执行多次
  // 例如：一个类实现了InitializingBean接口，在afterPropertiesSet()方法上又添加了@PostConstruct注解，由于注解方式会
  // 先执行，所以在后续执行InitializingBean接口初始化时，就不再执行这个已经执行过的方法了
  @Nullable
  private Set<String> externallyManagedInitMethods;

  // 同理，这里是销毁方法，例如：DisposableBean的destroy()方法上又添加了@PreDestroy注解
  @Nullable
  private Set<String> externallyManagedDestroyMethods;

  public RootBeanDefinition() {
    super();
  }
  // ...省略部分代码
}
```

## ChildBeanDefinition

该类可以让子 BeanDefinition 定义拥有从父 BeanDefinition 那里继承配置的能力（已经被GenericBeanDefinition所替代了）。

## GenericBeanDefinition

通过xml配置的bean会解析为GenericBeanDefinition，其可以动态设置父 Bean，替代了原来的ChildBeanDefinition。比起ChildBeanDefinition更为灵活，ChildBeanDefinition在实例化的时候必须要指定一个parentName，而GenericBeanDefinition不需要。

```java
public class GenericBeanDefinition extends AbstractBeanDefinition {

  // 该bean父bean名称
  @Nullable
  private String parentName;

  public GenericBeanDefinition() {
    super();
  }

  public GenericBeanDefinition(BeanDefinition original) {
    super(original);
  }

  @Override
  public void setParentName(@Nullable String parentName) {
    this.parentName = parentName;
  }

  @Override
  @Nullable
  public String getParentName() {
    return this.parentName;
  }

  @Override
  public AbstractBeanDefinition cloneBeanDefinition() {
    return new GenericBeanDefinition(this);
  }
  // ...省略部分代码
}
```

## AnnotatedGenericBeanDefinition

通过 @Import导入，@Configuration 注解标记配置类会被解析为AnnotatedGenericBeanDefinition，其父类是GenericBeanDefinition。

```java
public class AnnotatedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {
  
  // 注解元数据
  private final AnnotationMetadata metadata;

  // 工厂方法元数据
  @Nullable
  private MethodMetadata factoryMethodMetadata;

  public AnnotatedGenericBeanDefinition(Class<?> beanClass) {
    setBeanClass(beanClass);
    this.metadata = AnnotationMetadata.introspect(beanClass);
  }

  public AnnotatedGenericBeanDefinition(AnnotationMetadata metadata, MethodMetadata factoryMethodMetadata) {
    this(metadata);
    Assert.notNull(factoryMethodMetadata, "MethodMetadata must not be null");
    setFactoryMethodName(factoryMethodMetadata.getMethodName());
    this.factoryMethodMetadata = factoryMethodMetadata;
  }
  // ...省略部分代码
}
```

## ScannedGenericBeanDefinition

通过 @Service、@Controller、@Repository 以及 @Component 等注解标记的 Bean会被解析为ScannedGenericBeanDefinition，其父类是GenericBeanDefinition，并实现了AnnotatedBeanDefinition接口。

```java
public class ScannedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {

  // 注解源数据
  private final AnnotationMetadata metadata;

  public ScannedGenericBeanDefinition(MetadataReader metadataReader) {
    Assert.notNull(metadataReader, "MetadataReader must not be null");
    this.metadata = metadataReader.getAnnotationMetadata();
    setBeanClassName(this.metadata.getClassName());
    setResource(metadataReader.getResource());
  }
  // ...省略部分代码
}
```

## ConfigurationClassBeanDefinition

通过 @Bean 注解标记的 Bean 会被解析为 ConfigurationClassBeanDefinition，其父类是RootBeanDefinition，并实现了AnnotatedBeanDefinition接口。

```java
private static class ConfigurationClassBeanDefinition extends RootBeanDefinition implements AnnotatedBeanDefinition {
  
  // configClass元数据
  private final AnnotationMetadata annotationMetadata;
  
  // @Bean方法元数据
  private final MethodMetadata factoryMethodMetadata;
  
  // 标识该Bean是从哪个类创建出来的，也就是@Bean是从哪个类中定义的
  private final String derivedBeanName;
  
  public ConfigurationClassBeanDefinition(
    ConfigurationClass configClass, MethodMetadata beanMethodMetadata, String derivedBeanName) {

    this.annotationMetadata = configClass.getMetadata();
    this.factoryMethodMetadata = beanMethodMetadata;
    this.derivedBeanName = derivedBeanName;
    setResource(configClass.getResource());
    setLenientConstructorResolution(false);
  }
  
  // ...省略部分代码
}
```

## ClassDerivedBeanDefinition

通过GenericApplicationContext.registerBean()方法注册的bean会被解析为ClassDerivedBeanDefinition，其父类是RootBeanDefinition。registerBean()方法可以在Spring启动之后通过class类向Spring容器中注册Bean（spring 5.0提供）。

```java
private static class ClassDerivedBeanDefinition extends RootBeanDefinition {

  public ClassDerivedBeanDefinition(Class<?> beanClass) {
    super(beanClass);
  }

  public ClassDerivedBeanDefinition(ClassDerivedBeanDefinition original) {
    super(original);
  }
  
  // 重写了获取class构造器的方法
  @Override
  @Nullable
  public Constructor<?>[] getPreferredConstructors() {
    Class<?> clazz = getBeanClass();
    Constructor<?> primaryCtor = BeanUtils.findPrimaryConstructor(clazz);
    if (primaryCtor != null) {
      return new Constructor<?>[] {primaryCtor};
    }
    Constructor<?>[] publicCtors = clazz.getConstructors();
    if (publicCtors.length > 0) {
      return publicCtors;
    }
    return null;
  }

  @Override
  public RootBeanDefinition cloneBeanDefinition() {
    return new ClassDerivedBeanDefinition(this);
  }
}
```

至此，BeanDefinition的知识点就讲完了。