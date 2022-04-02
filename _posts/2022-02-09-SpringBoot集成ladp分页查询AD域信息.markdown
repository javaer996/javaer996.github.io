---
layout: post
read_time: true
show_date: true
title:  SpringBoot集成ldap分页查询AD域用户信息
subtitle: 人生的道路上，你没有耐心去等待成功，那么你只有用一生去面对失败。
date:   2022-02-09 10:11:20 +0800
description: SpringBoot集成ldap分页查询AD域用户信息
categories: [教程]
tags: [springboot,ldap]
author: tengjiang
toc: yes
---

## 什么是LDAP

LDAP 的全称是 Lightweight Directory Access Protocol，**「轻量目录访问协议」**。

LDAP **「是一个协议」**，约定了 Client 与 Server 之间的信息交互格式、使用的端口号、认证方式等内容。而 **「LDAP 协议的实现」**，有着众多版本，例如微软的 Active Directory 是 LDAP 在 Windows 上的实现，AD 实现了 LDAP 所需的树形数据库、具体如何解析请求数据并到数据库查询然后返回结果等功能。再例如 OpenLDAP 是可以运行在 Linux 上的 LDAP 协议的开源实现。而我们平常说的 LDAP Server，一般指的是安装并配置了 Active Directory、OpenLDAP 这些程序的服务器。

## LDAP结构

| 属性                     | 解释                                                         |
| :----------------------- | :----------------------------------------------------------- |
| dn（Distinguished Name） | 一条记录的位置 ，如上图，我们要想描述baby这个节点，描述如下cn=baby,ou=marketing,ou=pepple,dc=mydomain,dc=org |
| dc(domain compoent)      | 一条记录所属区域 域名部分                                    |
| ou (Organization Unit)   | 一条记录所属组织                                             |
| cn/uid（Common Name）    | 一条记录的名字/ID                                            |
| Entry                    | 条目记录数                                                   |

## LDAP查询语法

```javascript
search语法: attribute operator value
search filter options:( "&" or "|" (filter1) (filter2) (filter3) ...) ("!" (filter))
```

**详细解释**

> **=(等于)**

查找”Name”属性为”John”的所有对象:

```javascript
(Name=John)
```

这条语句会返回”name”为”john”的所有对象，以便强调LDAP语句的开始和结束

> **&(逻辑与)**

如果具有多个条件，并且希望所有条件都能满足，则使用该语法。

```javascript
(&(Name=John)(live=Dallas))
```

以上语句查询居住在Dallas，并且名为John的所有人员

> **!(逻辑非)**

此操作符用来排除具有特定属性的对象:

```javascript
(!Name=John)
```

查找所有”name”不为”John”的人员

> **通配符 \***

可以用通配符表示值可以等于任何内容

```javascript
(title=*)
```

查找具有职务头衔的所有人员

```javascript
(Name=Jo*)
```

查找所有”Name”以”Jo”开头的人员

最后，举一个较复杂的例子:

```javascript
(&(Name=John)(|(live=Dallas)(live=Austin)))
```

查找所有居住在Dallas或Austin，并且名为John的人员

## SpringBoot集成LDAP并进行分页查询

### 添加Maven依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-ldap</artifactId>
  <version>2.0.4.RELEASE</version>
</dependency>
```

### 分页查询

```java
public static void main(String[] args) {
  
  String ldapUserName = "xxxx";
  String ldapPassword = "xxxx";

  LdapContext ctx = null;
  Hashtable<String, String> HashEnv = new Hashtable<String, String>();
  HashEnv.put(Context.PROVIDER_URL, ldapUrl);
  // LDAP访问安全级别(none,simple,strong)
  HashEnv.put(Context.SECURITY_AUTHENTICATION, "simple");   
  //AD的用户名
  HashEnv.put(Context.SECURITY_PRINCIPAL, ldapUsername);
  //AD的密码    
  HashEnv.put(Context.SECURITY_CREDENTIALS, ldapPassword);  
  HashEnv.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.ldap.LdapCtxFactory");
  //连接超时设置为3秒
  HashEnv.put("com.sun.jndi.ldap.connect.timeout", "3000"); 
  int pageSize = 15;
  int currentPage = 0;
  int total = 0;
  try {
    // 初始化连接
    ctx = new InitialLdapContext(HashEnv, null);
    // 设置分页大小
    ctx.setRequestControls(new Control[]{new PagedResultsControl(pageSize, Control.CRITICAL)});
    // 查询过滤器
    String filter = "userprincipalname=*@*";
    SearchControls searchControls = new SearchControls();
    searchControls.setCountLimit(pageSize);
    // 设置查询该树和子树的数据
    searchControls.setSearchScope(SearchControls.SUBTREE_SCOPE);
    // 设置返回值
    String[] s = {"usncreated", "userprincipalname"};
    searchControls.setReturningAttributes(s);
    byte[] cookie = null;
    do {
      // 查询
      NamingEnumeration results = ctx.search("ou=软件开发,DC=TEST", filter, searchControls);
      while (results != null && results.hasMoreElements()) {
        SearchResult entry = (SearchResult) results.next();
        total++;
        if (currentPage == pageNo) {
          // *****需要返回的页面的数据在这里操作*****

        }
      }
      Control[] controls = ctx.getResponseControls();
      if (controls != null) {
        for (int i = 0; i < controls.length; i++) {
          if (controls[i] instanceof PagedResultsResponseControl) {
            PagedResultsResponseControl prrc = (PagedResultsResponseControl) controls[i];
            cookie = prrc.getCookie();
          }
        }
      }
      // 将cookie信息设置进去，下次查询的时候会在本次查询的基础上向后查询，如果没有cookie了，就说明没有数据了
      ctx.setRequestControls(new Control[]{new PagedResultsControl(pageSize, cookie, Control.CRITICAL)});
      currentPage++;
    } while (cookie != null);

  } catch (Exception e) {
    log.error("LDAP获取信息失败", e);
  } finally {
    if (null != ctx) {
      try {
        ctx.close();
      } catch (Exception e) {
        e.printStackTrace();
      }
    }
  }
}
```


