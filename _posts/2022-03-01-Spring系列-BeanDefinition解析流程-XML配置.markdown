---
layout: post
read_time: true
show_date: true
title:  Spring系列-BeanDefinition解析流程-XML配置
subtitle: 
date:   2022-03-01 16:13:20 +0800
description: Spring系列-Spring的BeanDefinition解析流程-XML配置
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---


Spring的xml配置是很早之前的使用方式了，现在基本都使用注解方式。但是了解xml配置方式是怎么执行的对我们理解Spring的原理有很大的帮助。

下面我们将以ClassPathXmlApplicationContext为例，探索一个Spring是怎么将我们的配置解析成BeanDefinition的。

## 代码示例

### 入口

```java
public static void main(String[] args) throws IOException {
  ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
  Home bean = applicationContext.getBean(Home.class);
  User user = bean.getUser();
  System.out.println(user.getName());
  System.out.println(user.getAge());
}
```

### User类

```java
@Data
public class User {
  private String name;
  private String age;
}
```

### Home类

```java
@Data
public class Home {
    private User user;
}
```

### spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">

  <bean id="user" class="com.test.User">
    <property name="name" value="jack"/>
    <property name="age" value="18"/>
  </bean>
  <!-- 手动指定属性注入 -->
  <bean id="home" class="com.test.Home">
    <property name="user" ref="user"/>
  </bean>
  <!-- 自动属性注入, 取值：no、byName、byType、constructor 
  <bean id="home" class="com.test.Home" autowire="byName"></bean>
	-->
</beans>
```

## 通过xml加载BeanDefinition源码跟踪

```java
// ClassPathXmlApplicationContext.java
public ClassPathXmlApplicationContext(
  String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
  throws BeansException {
  super(parent);
  setConfigLocations(configLocations);
  if (refresh) {
    // Spring入口，refresh()方法
    refresh();
  }
}
```

```java
// AbstractApplicationContext.java
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    prepareRefresh();
    // 获取BeanFacotory，该方法中会执行加载BeanDefinition逻辑
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
    // ...省略无关代码
  }
}

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
  // 刷新BeanFactory，ClassPathXmlApplicationContext中该方法是在AbstractRefreshableApplicationContext实现的
  refreshBeanFactory();
  return getBeanFactory();
}
```

```java
// AbstractRefreshableApplicationContext.java
protected final void refreshBeanFactory() throws BeansException {
  if (hasBeanFactory()) {
    destroyBeans();
    closeBeanFactory();
  }
  try {
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    // 在此处开始加载BeanDefinition，而该方法是在AbstractXmlApplicationContext实现的
    loadBeanDefinitions(beanFactory);
    this.beanFactory = beanFactory;
  }
  catch (IOException ex) {
    throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
  }
}
```

```java
// AbstractXmlApplicationContext.java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
  // 创建XmlBeanDefinitionReader
  XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
  beanDefinitionReader.setEnvironment(this.getEnvironment());
  // 设置resourceLoader，不是ResourcePatternResolver类型的
  beanDefinitionReader.setResourceLoader(this);
  beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
  initBeanDefinitionReader(beanDefinitionReader);
  // 加载BeanDefinition
  loadBeanDefinitions(beanDefinitionReader);
}

protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
  // 最终调用的AbstractBeanDefinitionReader中的loadBeanDefinitions()方法
  Resource[] configResources = getConfigResources();
  if (configResources != null) {
    reader.loadBeanDefinitions(configResources);
  }
  String[] configLocations = getConfigLocations();
  if (configLocations != null) {
    reader.loadBeanDefinitions(configLocations);
  }
}
```

```java
// AbstractBeanDefinitionReader.java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
  ResourceLoader resourceLoader = getResourceLoader();
  if (resourceLoader == null) {
    throw new BeanDefinitionStoreException(
      "Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
  }
  // 这里是通过location加载所有的Resource，Resource是Spring抽象出来的，用于描述所有能加载的资源
  // 因为这里是加载spring.xml,resourceLoader不是ResourcePatternResolver类型的，所以不会走这里。
  if (resourceLoader instanceof ResourcePatternResolver) {
    // Resource pattern matching available.
    try {
      Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
      int count = loadBeanDefinitions(resources);
      if (actualResources != null) {
        Collections.addAll(actualResources, resources);
      }
      if (logger.isTraceEnabled()) {
        logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
      }
      return count;
    }
    catch (IOException ex) {
      throw new BeanDefinitionStoreException(
        "Could not resolve bean definition resource pattern [" + location + "]", ex);
    }
  }
  else {
    // 代码会走到这里
    Resource resource = resourceLoader.getResource(location);
    // 通过spring.xml的Resource对象去加载我们定义的BeanDefinition
    // 该方法是由XmlBeanDefinitionReader实现的
    int count = loadBeanDefinitions(resource);
    if (actualResources != null) {
      actualResources.add(resource);
    }
    if (logger.isTraceEnabled()) {
      logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
    }
    return count;
  }
}
```

```java
// XmlBeanDefinitionReader.java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
  return loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
  // ...
  try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
    InputSource inputSource = new InputSource(inputStream);
    if (encodedResource.getEncoding() != null) {
      inputSource.setEncoding(encodedResource.getEncoding());
    }
    // spring中以do开头的方法都是真正开始干活的
    return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
  }
  // ...
}

protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
  throws BeanDefinitionStoreException {
  try {
    // 这里是解析spring.xml文件，将该文件解析成Document对象，便于下面操作
    Document doc = doLoadDocument(inputSource, resource);
    // 解析Document对象，注册BeanDefinition
    int count = registerBeanDefinitions(doc, resource);
    if (logger.isDebugEnabled()) {
      logger.debug("Loaded " + count + " bean definitions from " + resource);
    }
    return count;
  }
  catch (BeanDefinitionStoreException ex) {
    throw ex;
  }
  // ...
}

public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
  // 创建BeanDefinitionDocumentReader对象，用于从Document对象中解析出BeanDefinition
  // 默认实现类是DefaultBeanDefinitionDocumentReader.java
  BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
  int countBefore = getRegistry().getBeanDefinitionCount();
  // 解析Document，注册BeanDefinition
  documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
  return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

```java
// DefaultBeanDefinitionDocumentReader.java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
  this.readerContext = readerContext;
  doRegisterBeanDefinitions(doc.getDocumentElement());
}

protected void doRegisterBeanDefinitions(Element root) {
  // 任何嵌套的 <beans> 元素都会在这个方法中引起递归。为了正确传播和保留 <beans> default-* 属性， 
  // 跟踪当前（父）委托，该委托可能为空。使用对父级的引用创建新的（子级）委托以用于回退目的，然后最终将 
  // this.delegate 重置为其原始（父级）引用。这种行为模拟了一堆委托，而实际上不需要一个委托。
  BeanDefinitionParserDelegate parent = this.delegate;
  // 依据于父委托创建一个新委托
  this.delegate = createDelegate(getReaderContext(), root, parent);
  
  if (this.delegate.isDefaultNamespace(root)) {
    String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
    if (StringUtils.hasText(profileSpec)) {
      String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
        profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
      // We cannot use Profiles.of(...) since profile expressions are not supported
      // in XML config. See SPR-12458 for details.
      if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
        if (logger.isDebugEnabled()) {
          logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                       "] not matching: " + getReaderContext().getResource());
        }
        return;
      }
    }
  }
  preProcessXml(root);
  // 解析Document定义的BeanDefinition
  parseBeanDefinitions(root, this.delegate);
  postProcessXml(root);
  this.delegate = parent;
}

protected BeanDefinitionParserDelegate createDelegate(
  XmlReaderContext readerContext, Element root, @Nullable BeanDefinitionParserDelegate parentDelegate) {
  BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
  // 创建新委托的时候，会初始化一个默认属性，如果parentDelegate不为空，则以parentDelegate中的默认属性为准
  delegate.initDefaults(root, parentDelegate);
  return delegate;
}

protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
  // 这里解析BeanDefinition的时候分为两种情况：
  // 第一种是使用的默认命名空间：http://www.springframework.org/schema/beans
  // 第二种是非默认命名空间，例如：http://www.springframework.org/schema/context
  if (delegate.isDefaultNamespace(root)) {
    NodeList nl = root.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
      Node node = nl.item(i);
      if (node instanceof Element) {
        Element ele = (Element) node;
        if (delegate.isDefaultNamespace(ele)) {
          // 解析默认命名空间
          parseDefaultElement(ele, delegate);
        }
        else {
          // 解析自定义命名空间
          delegate.parseCustomElement(ele);
        }
      }
    }
  }
  else {
    delegate.parseCustomElement(root);
  }
}
```

### 解析默认命名空间

> 默认命名空间：http://www.springframework.org/schema/beans

```java
// DefaultBeanDefinitionDocumentReader.java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
  // 解析import标签，该方法会递归到前面的AbstractBeanDefinitionReader中的loadBeanDefinitions()方法
  // 加载刚刚通过import标签导入的BeanDefinition
  if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
    importBeanDefinitionResource(ele);
  }
  // 解析alias标签，注册别名
  else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
    processAliasRegistration(ele);
  }
  // 解析bean标签
  else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
    processBeanDefinition(ele, delegate);
  }
  // 解析内嵌beans标签
  else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
    // 递归处理
    doRegisterBeanDefinitions(ele);
  }
}

protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  // 解析bean标签中的属性，创建BeanDefinition
  BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
  if (bdHolder != null) {
    bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
    try {
      // 注册BeanDefinition
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
```java
// BeanDefinitionParserDelegate.java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
  String id = ele.getAttribute(ID_ATTRIBUTE);
  String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
  // ...
  // 继续调用
  AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
  if (beanDefinition != null) {
    // ...
    String[] aliasesArray = StringUtils.toStringArray(aliases);
    return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
  }

  return null;
}

// 这里实际要解析标签，并创建BeanDefinition实例对象了
public AbstractBeanDefinition parseBeanDefinitionElement(
  Element ele, String beanName, @Nullable BeanDefinition containingBean) {
  // ...
  try {
    // 创建BeanDefinition对象，实际类型是GenericBeanDefinition
    // 由BeanDefinitionReaderUtils进行创建
    AbstractBeanDefinition bd = createBeanDefinition(className, parent);
    // 下面是解析各种属性
    parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
    bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

    parseMetaElements(ele, bd);
    parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
    parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

    parseConstructorArgElements(ele, bd);
    parsePropertyElements(ele, bd);
    parseQualifierElements(ele, bd);

    bd.setResource(this.readerContext.getResource());
    bd.setSource(extractSource(ele));

    return bd;
  }
  catch (ClassNotFoundException ex) {
    error("Bean class [" + className + "] not found", ele, ex);
  }
  // ...
  return null;
}

protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
  throws ClassNotFoundException {
  // 交给BeanDefinitionReaderUtils进行创建
  return BeanDefinitionReaderUtils.createBeanDefinition(
    parentName, className, this.readerContext.getBeanClassLoader());
}
```

```java
// BeanDefinitionReaderUtils.java
public static AbstractBeanDefinition createBeanDefinition(
  @Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {
  // 实际创建GenericBeanDefinition类型的BeanDefinition
  GenericBeanDefinition bd = new GenericBeanDefinition();
  bd.setParentName(parentName);
  if (className != null) {
    if (classLoader != null) {
      bd.setBeanClass(ClassUtils.forName(className, classLoader));
    }
    else {
      bd.setBeanClassName(className);
    }
  }
  return bd;
}
```

### 解析自定义命名空间

```java
// BeanDefinitionParserDelegate.java
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
  // 获取命名空间
  String namespaceUri = getNamespaceURI(ele);
  if (namespaceUri == null) {
    return null;
  }
  // 获取自定义命名空间处理器
  NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
  if (handler == null) {
    error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
    return null;
  }
  // 通过自定义命名空间处理器解析
  return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

> 自定义命名空间处理器一般都会实现NamespaceHandlerSupport。

```java
// NamespaceHandlerSupport.java
public BeanDefinition parse(Element element, ParserContext parserContext) {
  // 找到该命名空间处理器下注册的该元素处理器
  BeanDefinitionParser parser = findParserForElement(element, parserContext);
  // 通过找到的BeanDefinition解析器解析自定义元素到BeanDefinition
  return (parser != null ? parser.parse(element, parserContext) : null);
}

private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
  // 通过元素的localName，找到该命名空间处理器下注册的该localName的BeanDefinition解析器
  String localName = parserContext.getDelegate().getLocalName(element);
  BeanDefinitionParser parser = this.parsers.get(localName);
  if (parser == null) {
    parserContext.getReaderContext().fatal(
      "Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
  }
  return parser;
}
```

```java
// AbstractBeanDefinitionParser.java
public final BeanDefinition parse(Element element, ParserContext parserContext) {
  AbstractBeanDefinition definition = parseInternal(element, parserContext);
  if (definition != null && !parserContext.isNested()) {
    try {
      String id = resolveId(element, definition, parserContext);
      if (!StringUtils.hasText(id)) {
        parserContext.getReaderContext().error(
          "Id is required for element '" + parserContext.getDelegate().getLocalName(element)
          + "' when used as a top-level tag", element);
      }
      String[] aliases = null;
      if (shouldParseNameAsAliases()) {
        String name = element.getAttribute(NAME_ATTRIBUTE);
        if (StringUtils.hasLength(name)) {
          aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
        }
      }
      BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
      registerBeanDefinition(holder, parserContext.getRegistry());
      if (shouldFireEvents()) {
        BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
        postProcessComponentDefinition(componentDefinition);
        parserContext.registerComponent(componentDefinition);
      }
    }
    catch (BeanDefinitionStoreException ex) {
      String msg = ex.getMessage();
      parserContext.getReaderContext().error((msg != null ? msg : ex.toString()), element);
      return null;
    }
  }
  return definition;
}

protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
  // 创建BeanDefinition，通过BeanDefinitionBuilder包装了一层， 便于下面的赋值
  BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
  String parentName = getParentName(element);
  if (parentName != null) {
    builder.getRawBeanDefinition().setParentName(parentName);
  }
  Class<?> beanClass = getBeanClass(element);
  if (beanClass != null) {
    builder.getRawBeanDefinition().setBeanClass(beanClass);
  }
  else {
    String beanClassName = getBeanClassName(element);
    if (beanClassName != null) {
      builder.getRawBeanDefinition().setBeanClassName(beanClassName);
    }
  }
  builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
  BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
  if (containingBd != null) {
    // Inner bean definition must receive same scope as containing bean.
    builder.setScope(containingBd.getScope());
  }
  if (parserContext.isDefaultLazyInit()) {
    // Default-lazy-init applies to custom bean definitions as well.
    builder.setLazyInit(true);
  }
  // 执行自定义命名空间元素的解析
  doParse(element, parserContext, builder);
  return builder.getBeanDefinition();
}

// 这里的解析是一个protected方法，真正的解析是留给子类实现的，因为不同的命名空间解析逻辑都不同
protected void doParse(Element element, BeanDefinitionBuilder builder) {
}
```

以上，就是通过Xml方式配置的BeanDefinition的主要解析逻辑了，但是自定义命名空间这里想必大家还有疑惑，自定义命名空间怎么定义的？又是怎么注册的？下一篇，将带大家揭晓。