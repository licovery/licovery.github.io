---
title: Travis-CI与自动化
date: 2019-04-17 22:49:35
tags: 
- CI(持续集成)
- 自动
categories: 编程
---

## 简介

在商业公司的项目管理里，持续集成（Continuous Integration，简称CI）是必须的一环。刚开始工作的时候，也不太懂持续集成是什么意思，慢慢接触到项目，也有了一定的理解。简单来说CI就持续编译项目源码，进行测试，并部署。每次项目组的成员提交的代码，都要走一遍编译，测试，部署的流程。而且这个过程全自动化完成。

<!-- more -->

个人感觉CI还是有很大的好处的，可以及时的暴露问题，可以把代码的测试细化到git上面的每一次commit，更容易定位问题和提升项目质量。

在某国外搜索引擎中了解到了[https://travis-ci.org](https://travis-ci.org/)，可以给github上的公开repo部署CI，教程网上一搜就一大堆，就不具体说了，说说遇到的坑和学习到的东西。

首先travis是通过一个`.travis.yml`文件来自动化构建的。`yml`似乎是一种配置文件的格式，应该跟`json`差不多，在官网上看看语法就开始用了。我的项目很简单，因为我要跑CI的repo是fork的Redis，所以已经写好了测试的脚本。我只需要把构建的流程走一下就好了。

官网的说法是，只要你把正确的.travis.yml放在repo的根目录下，并且在travis网站，用github账户登录并且激活这个仓库的构建。你以后每次push代码到仓库上，就能自动触发一次CI。

## 持续集成测试

我为某个git仓库配置了一个自动测试，每次往这个仓库push，就会触发一次构建和测试。

最后配置的.travis.xml差不多长这样：

```yml
language: c
notifications:     
  email: false
sudo : true
before_install: chmod +x runtest-ci
script : ./runtest-ci
```

意思是说项目用的C语言，每次build不需要邮件通知。使用root权限，给`runtest-ci`这个脚本执行权限，最后直接执行测试脚本。我这里面省略了部署项目的操作，因为我就是纯属想了解一下怎么自动化CI。

然后`runtest-ci`也很简单

```shell
#!/bin/sh
sudo apt-get install tcl
chmod +x runtest runtest-cluster runtest-sentinel
make test
```

tcl是redis跑test需要的一个库，网上查了一下，似乎是一个tcl语言的解析器，还不太了解。

那么build失败和成功是怎么判定的呢？其实是根据跑的shell脚本的返回值来判定。如果脚本执行返回0，就是build succ。如果返回其他值，就是build error。

刚开始用的时候，因为的测试脚本跑之前需要安装一个tcl这个库，由于我不知道默认服务器的哪个linux发行版，导致我不知道应该用apt-get还是yum或者其他的软件管理包。我是构建了一次，查看了log后才发现系统是ubuntu，可以用apt-get的。

```
$ export TRAVIS_COMPILER=gcc
$ export CC=gcc
$ export CC_FOR_BUILD=gcc
$ gcc --version
gcc (Ubuntu 4.8.4-2ubuntu1~14.04.3) 4.8.4
Copyright (C) 2013 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

而且刚开始还因为没有加sudo 导致无法执行apt-get install 命令

```
$ apt-get install tcl
E: Could not open lock file /var/lib/dpkg/lock - open (13: Permission denied)
E: Unable to lock the administration directory (/var/lib/dpkg/), are you root?
The command "apt-get install tcl" failed and exited with 100 during .
```

总结：

travis真是很方便好用的自动化CI工具，只需要配置好.travis.xml文件和关联repo就可以部署好某个repo的CI。最后附上成果图

[![Build Status](https://travis-ci.org/licovery/redis-3.0-annotated.svg?branch=unstable)](https://travis-ci.org/licovery/redis-3.0-annotated)

## 自动化部署博客

我的博客是利用`hexo`部署在github page上面的。一般来说，都是在本地完成写作后利用 `hexo g` 和 `hexo d`分别生成页面并且部署到github上。这样有几个问题：

- 每次都要做generate和deploy重复的操作，十分不爽
- 并且依赖本地的hexo，没有办法在其他机器上进行写作，除非在新机器上再次配置hexo
- 真正的数据源头，source/_posts下的md文件并没有加入版本控制，有数据丢失风险

同样利用了travis-ci的特性，每次push会触发一次任务。可以在原来github page仓库上新建一个分支`source`，把hexo根目录下的文件全部提交到仓库，并且配置`.travis.xml`文件，触发hexo自动部署到`master`分支上。

```yaml
# 指定构建环境是Node.js，当前版本是稳定版
language: node_js
# 指定需要sudo权限
sudo: required
node_js: stable

# 设置缓存文件
cache:
  directories:
    - node_modules

# 设置钩子只检测blog-source分支的push变动
branches:
  only:
    - source

#在构建之前安装hexo
before_install:
  - npm install -g hexo-cli

#安装git插件和搜索功能插件
install:
  - npm install
  - npm install hexo-deployer-git --save

# 执行清缓存，生成网页操作
script:
  - hexo clean
  - hexo generate

# 设置git提交名，邮箱；替换真实token到_config.yml文件，最后depoy部署
after_script:
  - git config user.name "neoli"
  - git config user.email "798060965@qq.com"
  # 替换同目录下的_config.yml文件中github_token字符串为travis后台刚才配置的变量，注>意此处sed命令用了双引号。单引号无效！
  - sed -i "s/github_token/${github_token}/g" ./_config.yml
  - hexo deploy
```



这里有个比较麻烦的地方，就是需要travis-ci部署网页，换言之，就是要修改仓库，需要github的仓库权限。这里可以利用github提供的`token`，生成token后要保存好，泄漏了会导致别人可以修改你的仓库。

如何把token传递给travis-ci呢？我用的是环境变量的方法，在travis-ci对于每个repo项目，都有一套环境变量，把`token`设置到环境变量中。

![](https://raw.githubusercontent.com/licovery/licovery.github.io/pic/img/travis-ci-evn-var.png)

然后在`.travis.xml`中调用（line36），写入`hexo`的`._config_yml`中，这样travis-ci就有权限修改git仓库了。

总结：

使用git分支和travis-ci实现自动化部署。在任意机器上，只需要clone source分支，进行写作，提交，就会自动发布网页，十分方便！