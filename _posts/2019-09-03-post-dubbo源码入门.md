---
layout: post
comments: true
title: "Post Dubbo源码入门"
date: 2019-09-03 08:50:54
image: '/assets/img/'
description:
main-class:
color:
tags:
- Dubbo
categories:
twitter_text:
introduction:
---

## Dubbo源码入门

#### 1、简介

在熟悉Dubbo源码之前，我们先认识一下java的SPI（Service Provider Interface）机制，这是一种服务发现机制。其本质是将接口实现类的权限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。



#### 2、SPI示例

step1

创建一个maven应用，