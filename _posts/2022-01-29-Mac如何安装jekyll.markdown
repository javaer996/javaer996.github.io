---
layout: post
read_time: true
show_date: true
title:  Mac如何安装jekyll?
date:   2022-01-29 10:19:20 +0800
description: 记录Mac安装jekyll遇到的问题以及解决方式.
img: posts/20210312/nnet_optimization.jpg
tags: [mac, jekyll, ruby, gem, rvm, brew]
author: tengjiang
# github: amaynez/TicTacToe/blob/7bf83b3d5c10adccbeb11bf244fe0af8d9d7b036/entities/Neural_Network.py#L199
# mathjax: yes # leave empty or erase to prevent the mathjax javascript from loading
toc: yes # leave empty or erase for no TOC
---

> 1. mac自带ruby软件，但是安装的时候不能使用自带的ruby软件，因为自带的权限为system，权限不够
>
> 2. 先安装rvm，使用rvm可以管理多个不同版本的ruby软件  brew install rvm
>
> 3. 使用rvm安装ruby  rvm install --head
>
> 4. 将ruby版本切换到我们新安装的ruby版本上     rvm use 3.0.0
>
> 5. 切换gem源(通过gem安装jekyll)：gem sources -r https://rubygems.org/ -a https://gems.ruby-china.com/
>
> 6. 通过gem安装jekyll: sudo gem install jekyll  或 sudo gem install github-pages
>
> 7. 安装是可能会先让升级brew，但是升级会报如下错误：
>
>    ![image-20220129082320182](https://s2.loli.net/2022/01/29/18NSntUuBFIkKa9.png)
>
> 此时只要先删除/usr/local/Homebrew/Library/Taps/homebrew/homebrew-core再重新执行brew upgrade即可。
>
> (如果升级太慢，可以切换brew国内源)
>
> 8. 安装的时候可能还会报openssl/ssl.h找不到：
>
>    ![image-20220129083255393](https://s2.loli.net/2022/01/29/rmkspMbDtNwB2vl.png)
>
> 此时先看openssl是否已安装，通过brew install openssl安装openssl，如果已安装不会重复安装，如果安装好之后依旧找不到，可以通过 brew info openssl查看openssl的安装路径。然后将include包下的内容复制到xcode的/Applications/[Xcode](https://so.csdn.net/so/search?q=Xcode&spm=1001.2101.3001.7020).app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/文件夹下即可。
>
> 9. 将jekyll安装目录设置到PATH路径。
>
> 10. 如上，应该即可正确安装jekyll。
>
> 11. 安装好jekyll后，可能启动你还会出现如下问题：
>
>     ![image-20220129085428933](https://s2.loli.net/2022/01/29/NDlYbZOct21E5z9.png)
>
> 执行 sudo gem install kramdown-parser-gfm 安装kramdown-parser-gfm。
>
> 执行sudo gem install bundler:2.2.3安装bundler。
>
> 修改项目目录下的gemfile，添加 gem "kramdown-parser-gfm"
>
> gemfile中会设置source，可以通过bundle config mirror.https://rubygems.org https://gems.ruby-china.com配置国内源
>
> 如果仍报错，缺少什么安装什么，注意安装的时候要使用自己安装的ruby版本，切换版本用rvm use 3.0.0，然后执行bundle install。
>
> 最后执行jekyll server启动。