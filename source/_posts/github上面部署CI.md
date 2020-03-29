---
title: 为某个github的项目部署CI
date: 2019-04-17 22:49:35
tags: CI(持续集成)
categories: 编程
---

在商业公司的项目管理里，持续集成（Continuous Integration，简称CI）是必须的一环。刚开始工作的时候，也不太懂持续集成是什么意思，慢慢接触到项目，也有了一定的理解。简单来说CI就持续编译项目源码，进行测试，并部署。每次项目组的成员提交的代码，都要走一遍编译，测试，部署的流程。而且这个过程全自动化完成。

<!-- more -->



个人感觉CI还是有很大的好处的，可以及时的暴露问题，可以把代码的测试细化到git上面的每一次commit，更容易定位问题和提升项目质量。



在某国外搜索引擎中了解到了[https://travis-ci.org](https://travis-ci.org/)，可以给github上的公开repo部署CI，教程网上一搜就一大堆，就不具体说了，说说遇到的坑和学习到的东西。



首先travis是通过一个`.travis.yml`文件来自动化构建的。`yml`似乎是一种配置文件的格式，应该跟`json`差不多，在官网上看看语法就开始用了。我的项目很简单，因为我要跑CI的repo是fork的Redis，所以已经写好了测试的脚本。我只需要把构建的流程走一下就好了。

官网的说法是，只要你把正确的.travis.yml放在repo的根目录下，并且在travis网站，用github账户登录并且激活这个仓库的构建。你以后每次push代码到仓库上，就能自动触发一次build



最后配置的.travis.xml差不多长这样：

```yml
language: c
notifications:     
  email: false
sudo : true
before_install: chmod +x runtest-ci
script : ./runtest-ci
```

意思是说项目用的C语言，每次build不需要邮件通知。使用root权限，给`runtest-ci`这个脚本执行权限，最后直接执行测试脚本。我这里面省略了部署项目的操作，因为我就是纯属想了解一下怎么自动化CI



然后`runtest-ci`也很简单

```shell
#!/bin/sh
sudo apt-get install tcl
chmod +x runtest runtest-cluster runtest-sentinel
make test
```

tcl是redis跑test需要的一个库，网上查了一下，似乎是一个tcl语言的解析器，还不太了解



让我比较疑惑的一个问题是，build失败和成功是怎么判定的呢？我的想法是应该是根据跑的shell脚本的返回来判定。如果脚本执行返回0，就是build succ。如果返回其他值，就是build error



还有刚开始用的时候，因为的测试脚本跑之前需要安装一个tcl这个库，由于我不知道默认服务器的哪个linux发行版，导致我不知道应该用apt-get还是yum或者其他的软件管理包。我是构建了一次，查看了log后才发现系统是ubuntu，可以用apt-get的。

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



总结一下：

travis真是很方便好用的自动化CI工具，只需要配置好.travis.xml文件和关联repo就可以部署好某个repo的CI。最后附上成果图

[![Build Status](https://travis-ci.org/licovery/redis-3.0-annotated.svg?branch=unstable)](https://travis-ci.org/licovery/redis-3.0-annotated)