---
layout: post
read_time: true
show_date: true
title:  SpringBoot集成Swagger2?
date:   2022-01-29 17:35:20 +0800
description: SpringBoot集成Swagger2步骤.
img: posts/common/Swagger.png
tags: [springboot,swagger2]
author: tengjiang
toc: yes
---

|| 该文章主要介绍SpringBoot如何集成Swagger2。 ||

<!-- more -->

## 一、添加MAVEN依赖

```xml
<!-- 版本号 -->
<swagger2.version>3.0.0</swagger2.version>
<knife4j-spring-boot-starter.version>3.0</knife4j-spring-boot-starter.version>


<!-- swagger2 begin-->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>${swagger2.version}</version>
</dependency>
<!-- 使用了bootstrap版本的swagger ui -->
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-spring-boot-starter</artifactId>
    <version>${knife4j-spring-boot-starter.version}</version>
</dependency>
<!-- swagger2 end-->
```

## 二、添加Swagger2Config配置文件

```java
@Configuration
@EnableSwagger2
public class Swagger2Config {

    @Value("${version}")
    private String version;
	
	// 如果需要分组可以配置多个，如果不需要分组只创建一个默认Bean就可以了，groupName前面的0，1是为了排序
    @Bean
    public Docket createDefaultRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .enable(true)
				// 指定分组
                .groupName("0-全部接口")  
				// 描述信息
                .apiInfo(apiInfo("SERVER 服务", "提供前端所需所有接口"))
                .select()
				// 需要扫描的包路径
                .apis(RequestHandlerSelectors.basePackage("com.XXX.server.controller"))
                .paths(PathSelectors.any())
                .build();
    }



    public ApiInfo apiInfo(String title, String description) {
        return new ApiInfoBuilder()
                .title(title)
                .description(description)
                .contact(new Contact("IT中心", null, null))
                .version(version)
                .build();
    }
}
```

## 三、添加ServerConfig配置类 (如果有，在之前的配置文件中新增即可)

```java
@Configuration
public class ServerConfiguration implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {  
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
        registry.addResourceHandler("swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");
    }

}
```

## 四、在application.yml文件中添加版本配置

```yaml
version: @project.version@
```

## 五、如果项目中有请求拦截器，需要忽略拦截

```yaml
- /swagger-resources/**
- /v2/api-docs
```

## 六、如果项目中有返回值拦截器，返回值忽略拦截

```yaml
- springfox.documentation.swagger.*
- springfox.documentation.swagger2.*
```

##  七、在接口上添加注解(不添加的话也能扫描到，只是没有中文解释)

1. 在类上添加 **@Api(tags = "描述类的作用")**
2. 在方法上添加 **@ApiOperation("描述方法的作用")**
3. 参数描述有多种方式
   1. 描述单个参数可以在参数上用 **@ApiParam(value="参数描述", requried = true)**
   2. 描述单个参数可以在方法上用 **@ApiImplicitParam(name="code" , value="参数描述", requried = true)**
   3. 统一描述多个参数可以在方法上用 
      **@ApiImplicitParams({**
            **@ApiImplicitParam(name="code" , value="编号", requried = true),**
            **@ApiImplicitParam(name="name" , value="姓名", requried = true)**
      **})**
   4. 如果参数是实体类，在类上添加 **@ApiModel** ，然后在每个字段上添加 **@ApiModelProperty(value = "字段描述", example = "示例值")**
4. 定义错误码
   1. 描述单个错误码，在方法上用
      **@ApiResponse(responseCode = "360102", description = "菜单编号不存在")**
   2. 描述多个错误码，在方法上用
      **@ApiResponses({**
         **@ApiResponse(responseCode = "360102", description = "菜单编号不存在"),**
         **@ApiResponse(responseCode = "360102", description = "菜单编号不存在")**
      **})**

## 八、访问文档( [http://ip:port/doc.html](http://ipport/))