---
layout: post
title: "Java线程"
date: 2017-02-26 12:31:00 +0800
catalog: true
image: /images/head.png
tags:
    - Java
    - Java线程
category: tech-blog
---

## **Java创建多线程的方法**
1. 继承Thread类
2. 实现Runnable接口
3. 实现Callable接口  

## **Java中多线程两种实现方式的区别？/Thread类与Runnable接口实现多线程的区别？**
* Thread类是Runnable接口的子类，使用Runnable接口实现多线程可以避免单继承局限。
* Runnable接口实现的多线程比继承Thread类实现的多线程更加清楚的描述数据共享的概念。

## **Java多线程的代码实现**
```Java
    class MyThread extends Thread{
        @Override
        public void run(){
        }
    }
    class MyThread implements Runnable{
        @Override
        public void run(){
        }
    }
    class MyThread implements Callable<String>{
        @Override
        public String call() throws Exception{
            return null;
        }
    }
    //实现Callable接口的线程应该这样启动
    MyThread mt = new MyThread();
    FutureTask<String> task = new Future<String>(mt);
    new Thread(task).start;
    /**
    查看Java源码，FutureTask<V>实现了Runnable接口，因此可以直接传入Thread参数。线程执行完成后可以通过FutureTask的父接口Future中的get()方法获得返回的数据。
    */
```
## **为什么启动线程需要使用Thread类里的start方法而不是直接调用run()？**
Thread类里的start方法中调用了native方法start0()。在native方法中需要给线程分配内存资源。如果直接调用run方法，则跟一般的对象没有什么区别，不会有系统资源的分配，也就没有启动新的线程。

  >每一个JVM进程启动时至少启动几个线程？
   1. Main线程：程序的主要执行，以及启动子线程
   2. gc线程：负责垃圾回收  

默认情况下，如果设置了多个线程对象，那么所有的线程对象将一起进入到run()方法。所谓的一起进入实际上是因为先后顺序实在是太短了，但实际上有区别。所以有可能造成数据的错误。
   
