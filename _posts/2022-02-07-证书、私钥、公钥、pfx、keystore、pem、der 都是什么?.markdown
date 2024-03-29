---
layout: post
read_time: true
show_date: true
title:  证书、私钥、公钥、pfx、keystore、pem、der 都是什么?
subtitle: 
date:   2022-02-07 09:06:20 +0800
description: 证书、私钥、公钥、pfx、keystore、pem、der
categories: [教程]
tags: [数字证书]
author: tengjiang
toc: yes
---

刚开始接触证书，很容易就会被各种名词整的迷迷糊糊，通过搜索引擎一搜索，我们会发现很多介绍文章，如果没接触过这一块的话，一上来很多的名字就会把人绕晕了。什么csr，crt，cer，keystore等等。

我们知道，现在的网站为了数据的安全，往往都会使用证书进行签名或者加密数据。可以证书的各种后缀让人无从下手，不知道该用什么，以及怎么使用。

### 什么是CA？

CA就相当于一个认证机构，只要经过这个机构签名的证书我们就可以当做是可信任的。我们的浏览器中，已经被写入了默认的CA根证书。

### 什么是证书？

证书就是将我们的公钥和相关信息写入一个文件，CA用它们的私钥对我们的公钥和相关信息进行签名后，将签名信息也写入这个文件后生成的一个文件。

#### 证书格式(是一种标准)
> x509            这种证书只有公钥，不包含私钥；

> pcks#7       这种主要是用于签名或者加密；

> pcks#12     这种含有私钥，同时也含有公钥，但是有口令保护；

#### 编码方式
> .pem 后缀的证书都是base64编码；

> .der   后缀的证书都是二进制格式；

#### 证书
> .csr              后缀的文件是用于向ca申请签名的请求文件；

> .crt    .cer     后缀的文件都是证书文件（编码方式不一定，有可能是.pem,也有可能是.der）；

#### 私钥
> .key   后缀的文件是私钥文件；

#### 包含证书和私钥
> .keystore  .jks   .truststore 后缀的文件，是java搞的，java可以用这个格式。这个文件中包含证书和私钥，但是获取私钥需要密码才可以, 这是一个证书库，里面可以保存多个证书和密钥，通过别名可以获取到。（不同的后缀是为了区分不同的用途以及保存的信息的差异）；

> .pfx 主要用于windows平台，浏览器可以使用，也是包含证书和私钥，获取私钥需要密码才可以；

### 知识点

1. 使用公钥操作数据属于加密；

2. 使用私钥操作数据属于签名；

3. 公钥和私钥可以互相加解密；

4. 不同格式的证书之间可以互相转换；

5. 公钥可以对外公开，但是私钥千万不要泄露，要妥善保存。

#### 注意

在我们备份证书信息的时候，最好使用.jks或者.pfx文件进行保存，这样备份的证书文件可以被完整的导出。

我们在使用证书的时候，要根据不同平台，不同应用的要求，转换成不同的格式进行使用。