---
layout: post
title: "String,StringBuffer与StringBuild对比"
date: 2017-06-16 09:53:30 +0800
catalog: true
image: /images/head.png
tags:
    - Java
    - Java基础
category: tech-blog
---

# **String**
String 是字符串**常量**。
是不可变对象，每次改变都等于生成了一个新的String对象，然后指向新的对象。所以经常改变内容的字符串最好不要用String。

# **StringBuffer**
StringBuffer是字符串**变量**。
并且是线程安全的。
每次改变都是对StringBuffer对象本身操作。
但是在某些特别的情况下，String的字符串拼接其实是被JVM解释成了StringBuffer对象的拼接，所以这些时候String对象的速度并不会比StringBuffer慢。而下面这种情况，String的效率要远远高于StringBuffer：
```
String s = "This is only a" + " simple" + "test";
StringBuffer sb = new StringBuffer("This is only a").append(" simple").append(" test");
```
这其实是编译器优化的结果， "This is only a" + " simple" + "test"这个在JVM眼里，就是“This is only a simple test”。

> 在大部分的情况下，StringBuffer > String 效率

StringBuffer主要操作是append和insert方法，可以重载这些方法，以接受任意类型的数据。每个方法都能有效的将给定的数据转换成字符串，然后将该字符串的字符追加或者插入到字符串缓冲区中。append方法始终将这些字符添加到缓冲区的末端，而insert方法则在指定的点添加字符。

例如，如果s引用一个当前内容是"start"的字符串缓冲区对象，则此方法调用s.append("le")会使字符串缓冲区包含"startle"，而s.insert(4,"le")将更改字符串缓冲区使之包含"starlet"。

# **StringBuilder**
字符串**变量**。
非线程安全。
可变字符序列是5.0新增的。
提供与StringBuffer兼容的API。但不保证同步。
用在字符串缓冲区被单个线程使用的时候。

> 在大多数情况下， StringBuilder > StringBuffer 效率