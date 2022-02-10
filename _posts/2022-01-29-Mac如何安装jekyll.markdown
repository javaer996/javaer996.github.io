---
layout: post
read_time: true
show_date: true
title:  Mac如何安装jekyll?
date:   2022-01-29 10:19:20 +0800
description: 记录Mac安装jekyll遇到的问题以及解决方式.
img: posts/common/jekyll.png
tags: [mac,jekyll]
author: tengjiang
# github: amaynez/TicTacToe/blob/7bf83b3d5c10adccbeb11bf244fe0af8d9d7b036/entities/Neural_Network.py#L199
# mathjax: yes # leave empty or erase to prevent the mathjax javascript from loading
toc: yes # leave empty or erase for no TOC
---

|| 该文章主要介绍Mac如何安装jekyll，以及在安装过程中遇到了哪些问题。||

<!-- more -->

## 一、安装Ruby

> 其实mac自带ruby，但是安装的时候不能使用自带的ruby软件，因为自带的权限为system，导致安装的时候权限不够安装失败，
>
> 所以我们需要自己安装一个ruby软件。

###  使用rvm安装ruby

> RVM 是一个命令行工具，可以提供一个便捷的多版本 Ruby 环境的管理和切换。

1. 安装rvm

   ```shell
   curl -L get.rvm.io | bash -s stable
   source ~/.bashrc
   source ~/.bash_profile
   ```

2. 使用rvm安装ruby

   ```shell
   rvm install ruby-3.0.0   // 安装最新的ruby软件（我安装的是3.0.0）
   ```

3. 切换ruby为我们刚安装的版本

   ```shell
   rvm use 3.0.0
   ```

## 二、安装jekyll

### 使用gem安装jekyll

> RubyGems软件允许您轻松下载、安装和使用ruby在您的系统软件包。 这个软件包被称为“Gem”和包含一个Ruby包应用程序或库。
>
> 因为国内访问gem源比较慢，设置安装失败，所以可以切换国内源提升速度。

1. 切换gem源

   ```shell
   gem sources -r https://rubygems.org/ -a https://gems.ruby-china.com/
   ```

2. 安装jekyll

   ```shell
   sudo gem install jekyll      		// 第一种方式
   sudo gem install github-pages   // 第二种方式(该方式会按照github pages的版本安装jekyll)
   ```

3. 设置PATH路径

   在.bash_profile文件中添加如下内容，路径替换成自己的真实路径

   ```shell
   export jekyll='/Users/xxx/.rvm/rubies/ruby-3.0.0'
   export PATH=$PATH:$jekyll/bin
   ```

## 三、下载jekyll主题

通过[jekyll主题网站](http://jekyllthemes.org/)下载喜欢的主题

## 四、启动jekyll服务

解析主题压缩包，进入主题文件夹，执行以下指令：

```shell
jekyll server
```

## 五、（重点）安装过程中的异常梳理

### 1. 安装jekyll的时候可能会让升级homebrew，但是升级的时候报如下错误：

![image-20220129082320182](https://s2.loli.net/2022/01/29/18NSntUuBFIkKa9.png)

>  **解决方案：**
>
>  先删除/usr/local/Homebrew/Library/Taps/homebrew/homebrew-core再重新执行brew upgrade即可。(如果升级太慢，可以切换brew国内源)

##### 2. 安装jkeyll的时候可能还会报openssl/ssl.h找不到错误 ![image-20220129083255393](https://s2.loli.net/2022/01/29/rmkspMbDtNwB2vl.png)

> **解决方案：**
>
> 1. 此时先看openssl是否已安装，通过brew install openssl安装openssl，如果已安装不会重复安装；
>
> 2. 如果安装好之后依旧找不到，可以通过 brew info openssl查看openssl的安装路径。然后将include包下的内容复制到xcode
>
>    的/Applications/[Xcode](https://so.csdn.net/so/search?q=Xcode&spm=1001.2101.3001.7020).app/Contents/Developer/Platforms/MacOSX.platform
>    
>    /Developer/SDKs/MacOSX.sdk/usr/include/文件夹下即可。

### 3. 使用rvm use xxx报错RVM is not a function
> **解决方案：**
> 在使用rvm use xxx命令之前先执行如下命令：
> ```shell
> source ~/.rvm/scripts/rvm
> ```

### 3. 启动jekyll服务的时候可能会报如下错误
![image-20220129085428933](https://s2.loli.net/2022/01/29/NDlYbZOct21E5z9.png)
>**解决方案：**
>
>1.  执行 sudo gem install kramdown-parser-gfm 安装kramdown-parser-gfm；
>
>2. 执行sudo gem install bundler:2.2.3安装bundler；
>
>3. 修改项目目录下的gemfile，添加 gem "kramdown-parser-gfm"；
>
>4. gemfile中会设置source，可以通过bundle config mirror.https://rubygems.org https://gems.ruby-china.com配置国内源。
>5. 执行bundle install 。

**最后：如果仍报错，缺少什么安装什么，注意安装的时候要使用自己安装的ruby版本，切换版本用rvm use 3.0.0。**