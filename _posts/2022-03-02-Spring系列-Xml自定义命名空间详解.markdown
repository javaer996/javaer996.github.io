---
layout: post
read_time: true
show_date: true
title:  Spring系列-Xml自定义命名空间详解
subtitle: 
date:   2022-03-02 13:46:20 +0800
description: Spring系列-Xml自定义命名空间详解
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---


## 为什么需要自定义命名空间？

Spring在解析Xml配置的时候，默认只识别import、alias、bean、beans标签，其它标签都会走自定义标签解析逻辑。默认的这几种标签远远不能满足我们的使用需求，所以就需要扩展一系列的自定义命名空间。

这里的自定义命名空间并不是说一定是我们使用者定义的命名空间，Spring其实已经扩展了很多自定义命名空间供我们使用。命名空间需要有相应的命名空间处理器进行处理才能起作用。

例如我们常用的自定义命名空间处理器就有：

- ContextNamespaceHandler: 与上下文有关的命名空间处理器
- AopNamespaceHandler: 与Aop有关的命名空间处理器
- MvcNamespaceHandler: 与Mvc有关的命名空间处理器

等等。

## 自定义命名空间是怎么起作用的？

> 下面我们以context命名空间为例子进行讲解自定义命名空间到底怎么回事。

### 引入配置文件

这个配置文件与上一篇文章[BeanDefinition解析流程-XML配置](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/01/Spring%E7%B3%BB%E5%88%97-BeanDefinition%E8%A7%A3%E6%9E%90%E6%B5%81%E7%A8%8B-XML%E9%85%8D%E7%BD%AE.html)中的配置文件的区别在于:

这里增加了一行：xmlns:context="http://www.springframework.org/schema/context"，我们就是通过这一行引入了context命名空间。下面我们在xsi:schemaLocation还增加了它的xsd文件地址信息，用于让Spring知道从哪里取得context命名空间的xml配置模式，便于代码提示和校验。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd">

  <bean id="user" class="com.test.User">
    <property name="name" value="jack"/>
    <property name="age" value="18"/>
  </bean>
  <!-- 手动指定属性注入 -->
  <bean id="home" class="com.test.Home">
    <property name="user" ref="user"/>
  </bean>
  
  <!-- spring会扫描base-backage包下的所有文件，如果扫描到有@Component、@Controller、@Service 、@Repository等注解修饰的Java类，则将这些类注册为spring容器中的bean -->
  <context:component-scan base-package="com.test"/>
</beans>
```

如上所示，我们使用了context命名空间中的一个component-scan标签，这个标签与bean标签有一个区别，就是前面加了context: 前缀，这就表明这个标签是属于context命名空间的，bean标签前面没有前缀，就相当于使用的默认的命名空间，通过xmlns配置可以知道，默认的命名空间是beans。

通过这行配置，我们相当于不仅可以通过xml配置bean，还可以通过注解配置bean。那么它到底是怎么起作用的呢？我们通过跟踪Spring解析xml自定义命名空间标签的源码来讲解。

### parseCustomElement

上一篇[Spring的BeanDefinition解析流程-XML配置](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/01/Spring%E7%B3%BB%E5%88%97-BeanDefinition%E8%A7%A3%E6%9E%90%E6%B5%81%E7%A8%8B-XML%E9%85%8D%E7%BD%AE.html)中我们介绍了怎么调用到parseCustomElement()方法的，这里我们接着这个方法介绍一个自定义命名空间中的标签是怎么解析出来的。

```java
// BeanDefinitionParserDelegate.java
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
   // 首先获取到namespaceURI, 例如component-scan标签这里就是http://www.springframework.org/schema/context
   String namespaceUri = getNamespaceURI(ele);
   if (namespaceUri == null) {
      return null;
   }
   // 通过namespaceHandlerResolver获取到该命名空间所对应的处理器
   // 这里的namespaceHandlerResolver实现类默认是DefaultNamespaceHandlerResolver.java
   NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
   if (handler == null) {
      error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
      return null;
   }
   // 通过处理器处理该命名空间标签
   return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

### 获取NamespaceHandler

要想解析自定义命名空间，首先需要获取到该命名空间的处理器。

```java
// DefaultNamespaceHandlerResolver.java
public NamespaceHandler resolve(String namespaceUri) {
  // 获取处理器映射，key为namespaceUri，value为处理器全类名或实例
  Map<String, Object> handlerMappings = getHandlerMappings();
  // 获取该命名空间对应的处理器
  Object handlerOrClassName = handlerMappings.get(namespaceUri);
  // 不存在，说明不支持该命名空间的解析
  if (handlerOrClassName == null) {
    return null;
  }
  // 如果获取到的直接是处理器实例，说明之前获取过该命名空间处理器，所以这里直接返回
  else if (handlerOrClassName instanceof NamespaceHandler) {
    return (NamespaceHandler) handlerOrClassName;
  }
  else {
    // 如果是第一次获取该命名空间处理器，一定会走到这里，此时处理器是一个全类名，是从META-INF/spring.handlers文件中获得的
    String className = (String) handlerOrClassName;
    try {
      // 加载处理器类
      Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
      if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
        throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
                                     "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
      }
      // 实例化该处理器
      NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
      // 初始化该处理器，这一步会将该处理器支持的所有标签解析器进行注册
      namespaceHandler.init();
      // 将该处理器实例放入handlerMappings，替换之前的处理器全类名
      handlerMappings.put(namespaceUri, namespaceHandler);
      return namespaceHandler;
    }
    catch (ClassNotFoundException ex) {
      throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
                                   "] for namespace [" + namespaceUri + "]", ex);
    }
    catch (LinkageError err) {
      throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
                                   className + "] for namespace [" + namespaceUri + "]", err);
    }
  }
}
```

#### getHandlerMappings()

```java
// DefaultNamespaceHandlerResolver.java
private Map<String, Object> getHandlerMappings() {
  // 第一次进来这里肯定为null，所以走下面的null处理逻辑
  Map<String, Object> handlerMappings = this.handlerMappings;
  if (handlerMappings == null) {
    synchronized (this) {
      handlerMappings = this.handlerMappings;
      // 这里是double check
      if (handlerMappings == null) {
        if (logger.isTraceEnabled()) {
          logger.trace("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
        }
        try {
          // 加载所有META-INF/spring.handlers里的配置信息
          // 默认handlerMappingsLocation为META-INF/spring.handlers
          // 这里就能获取到spring-context包中的spring.handlers信息
          Properties mappings =
            PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
          if (logger.isTraceEnabled()) {
            logger.trace("Loaded NamespaceHandler mappings: " + mappings);
          }
          handlerMappings = new ConcurrentHashMap<>(mappings.size());
          CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
          this.handlerMappings = handlerMappings;
        }
        catch (IOException ex) {
          throw new IllegalStateException(
            "Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
        }
      }
    }
  }
  return handlerMappings;
}
```

spring-context中META-INF/spring.handlers中的内容：

```properties
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler
```

注意看第一个就是指明了context命名空间的处理器为ContextNamespaceHandler。

#### init()

```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {
  @Override
  public void init() {
    registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
    registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
    registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
    // 这里我们可以看到，在context命名空间处理器中注册了component-scan标签的解析器
    registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
    registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
    registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
    registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
    registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
  }
}
```

```java
// NamespaceHandlerSupport.java
// 在NamespaceHandlerSupport中有一个Map集合，这些解析器都会存储到这个parsers集合中
// 注意这里每个命名空间处理器都有自己的解析器集合，不同处理器的解析器集合互不影响，也就是可以重名
private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();

protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
  this.parsers.put(elementName, parser);
}
```

### handler.parse()

```java
// NamespaceHandlerSupport.java
public BeanDefinition parse(Element element, ParserContext parserContext) {
  // 在这个处理器中找到该标签解析器
  BeanDefinitionParser parser = findParserForElement(element, parserContext);
  // 解析
  return (parser != null ? parser.parse(element, parserContext) : null);
}
```

#### findParserForElement()

```java
// NamespaceHandlerSupport.java
private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
  // 这里就是获取当前元素的名称，就是上面的component-scan
  String localName = parserContext.getDelegate().getLocalName(element);
  // 然后这里就从注册的解析器中拿到了component-scan的解析器ComponentScanBeanDefinitionParser
  BeanDefinitionParser parser = this.parsers.get(localName);
  if (parser == null) {
    parserContext.getReaderContext().fatal(
      "Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
  }
  return parser;
}
```

#### parser.parser()

```java
// ComponentScanBeanDefinitionParser.java
public BeanDefinition parse(Element element, ParserContext parserContext) {
  // 这里拿到配置的扫描的base-package，下面就开始通过ClassPathBeanDefinitionScanner走注解扫描解析逻辑
  String basePackage = element.getAttribute("base-package");
  basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
  String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,",; \t\n");

  // Actually scan for bean definitions and register them.
  ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
  Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
  registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

  return null;
}
```

这里需要注意的是，我们之前讲的解析逻辑是在doParser()由子类实现，但是现在这里并不是这样的。ComponentScanBeanDefinitionParser直接实现了BeanDefinitionParser接口，并没有继承AbstractBeanDefinitionParser类。

所以说，什么时候继承抽象类，什么时候直接实现接口需要具体情况具体分析，如果抽象类中的通用逻辑不适合你的功能，就不要再继承抽象类了。

至此，我们通过Spring通过Xml如何使用注解为例子讲解了Spring是如何通过自定义命名空间实现该功能的，其它的命名空间实现同理。

当然我们也可以按照这个逻辑自己编写属于自己的命名空间。