---
layout: post
title: "Java线程的同步与死锁"
date: 2017-02-26 20:27:17 +0800
catalog: true
image: /images/head.png
tags:
    - Java
    - Java线程
category: tech-blog
---

## **同步问题的引出**
多个线程访问同一个资源时需要考虑到的问题。
## **同步操作**
### **Synchronized关键字**

[Sychronized关键字](http://www.codeceo.com/article/java-synchronized.html)

Synchronized关键字有两种使用方式：
1. 同步代码块
2. 同步方法
>Java中有四种代码块
1. 普通代码块
2. 构造块
3. 静态块
4. 同步块

* 同步操作与异步操作相比，异步操作的执行速度要高于同步操作，但是同步操作时数据的安全性较高，属于安全的线程操作。

## **死锁**
>请解释多个线程访问同一资源时需要考虑到哪些情况？有可能带来哪些问题？
多个线程访问同一资源时一定要处理好同步，可以使用同步代码块或同步方法解决。但是过多的使用同步，有可能造成死锁。



## **生产者与消费者**
[生产者与消费者问题](http://c.biancheng.net/cpp/html/2600.html)
#### **问题的引出**
生产者和消费者是指两个不同的线程对象，操作同一资源的情况，具体操作流程如下：
* 生产者负责生产数据，消费者负责消费数据。
* 生产者每生产完一组数据，消费者就要取走一组数据。

```Java
package com.xiaofei;

class Info {
    private String title;
    private String content;

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }
}

class Productor implements Runnable {

    private Info info;

    public Productor(Info info) {
        this.info = info;
    }

    @Override
    public void run() {
        for (int i = 0; i < 20; i++) {
            if (i % 2 == 0) {
                info.setTitle("A数据的title" + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                info.setContent("A数据的content" + i);
            } else {
                info.setTitle("B数据的title" + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                info.setContent("B数据的content" + i);
            }
        }
    }
}

class Customer implements Runnable {

    private Info info;

    public Customer(Info info) {
        this.info = info;
    }

    @Override
    public void run() {
        for (int i = 0; i < 20; i++) {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(info.getTitle() + "-" + info.getContent());
        }
    }
}

public class Main {

    public static void main(String[] args) {
        // write your code here
        Info info = new Info();
        new Thread(new Productor(info)).start();
        new Thread(new Customer(info)).start();
    }
}
```
上面代码中出现了两个问题，同步问题和互斥问题。能看到的现象可能如下：
*B数据的title1-A数据的content0*
*A数据的title2-B数据的content1*
*A数据的title2-B数据的content1*
*B数据的title3-A数据的content2*
*A数据的title4-B数据的content3*
*A数据的title6-B数据的content5*
*A数据的title6-B数据的content5*
*B数据的title7-A数据的content6*
*A数据的title8-B数据的content7*
*...*
设置的信息错位，重复取出数据。原因是两个线程同时访问同一个对象的时候，没有处理同步与互斥的问题。在生产者线程去生产数据的时候，两次中间有微小的时间差，当消费者线程消费数据时可能还没等到A完全生产完数据就去消费，造成数据错位或者重复取出数据。
```Java
package com.xiaofei;

class Info {
    private String title;
    private String content;
    private int flag = 0; //0表示可以生产，1表示可以消费
    public synchronized void set(String title,String content){ //生产者线程使用set方法模拟生产
        if(flag == 1){//此时需要等消费者消费完成后再生产
            try {
                super.wait(); //生产线程等待，释放该对象的锁
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        this.title = title;
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        this.content = content;
        flag = 1; //生产完毕，可以进行消费
        super.notify(); //唤醒消费线程
    }

    public synchronized void get(){//消费者线程使用get方法模拟消费
        if(flag == 0){ //还没有进行生产，需要等待生产者生产
            try {
                super.wait();//消费线程等待，释放该对象的锁
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(title + "-" + content);
        flag = 0; //消费完成，生产者可以进行生产
        super.notify(); //唤醒生产者线程
    }
}

class Productor implements Runnable {

    private Info info;

    public Productor(Info info) {
        this.info = info;
    }

    @Override
    public void run() {
        for (int i = 0; i < 20; i++) {
            if (i % 2 == 0) {
                info.set("A数据的title" + i,"A数据的content" + i);
            } else {
                info.set("B数据的title" + i,"B数据的content" + i);
            }
        }
    }
}

class Customer implements Runnable {

    private Info info;

    public Customer(Info info) {
        this.info = info;
    }

    @Override
    public void run() {
        for (int i = 0; i < 20; i++) {
            info.get();
        }
    }
}

public class Main {

    public static void main(String[] args) {
        // write your code here
        Info info = new Info();
        new Thread(new Productor(info)).start();
        new Thread(new Customer(info)).start();
    }
}
```
**wait()，notify(),notifyAll()方法必须在持有对象的资源锁的时候调用，否则会抛出IllegalMonitorStateException异常。**
>查看wait()方法的源代码可以看到里面有这段注释：

```Java
/*   This method should only be called by a thread that is the owner
     * of this object's monitor. See the {@code notify} method for a
     * description of the ways in which a thread can become the owner of
     * a monitor.
     *
     * @throws  IllegalMonitorStateException  if the current thread is not
     *               the owner of the object's monitor.
    * @throws  InterruptedException if any thread interrupted the
     *             current thread before or while the current thread
     *             was waiting for a notification.  The <i>interrupted
     *             status</i> of the current thread is cleared when
     *             this exception is thrown.
*/
```

*(Monitor:监视器)*
>sleep()方法与wait()方法的区别
sleep()方法是Thread类里面的方法，作用是使线程休眠一段时间，休眠时间结束自动唤醒线程，可以自己设置休眠的时间。wait()方法是Object类里面的方法，必须要手动调用notify()方法去唤醒使用wait()的休眠的线程。

在 Java 中，可以使用 synchronized 关键字来标记一个方法或者代码块，当某个线程调用该对象的synchronized方法或者访问synchronized代码块时，这个线程便获得了**该对象的锁**，其他线程暂时无法访问这个方法，只有等待这个方法执行完毕或者代码块执行完毕，这个线程才会释放该对象的锁，其他线程才能执行这个方法或者代码块。

    synchronized(lock){
        //...
    }
当在某个线程中执行这段代码块，该线程会获取对象lock的锁，从而使得其他线程无法同时访问该代码块。其中，lock 可以是 this，代表获取当前对象的锁，也可以是类中的一个属性，代表获取该属性的锁。特别地， 实例同步方法 与 synchronized(this)同步块 是互斥的，因为它们锁的是同一个对象。但与 synchronized(非this)同步块 是异步的，因为它们锁的是不同对象。

* 当一个线程正在访问一个对象的 synchronized 方法，那么其他线程不能访问该对象的其他 synchronized 方法。这个原因很简单，因为一个对象只有一把锁，当一个线程获取了该对象的锁之后，其他线程无法获取该对象的锁，所以无法访问该对象的其他synchronized方法。

* 当一个线程正在访问一个对象的 synchronized 方法，那么其他线程能访问该对象的非 synchronized 方法。这个原因很简单，访问非 synchronized 方法不需要获得该对象的锁，假如一个方法没用 synchronized 关键字修饰，说明它不会使用到临界资源，那么其他线程是可以访问这个方法的，

* 如果一个线程 A 需要访问对象 object1 的 synchronized 方法 fun1，另外一个线程 B 需要访问对象 object2 的 synchronized 方法 fun1，即使 object1 和 object2 是同一类型），也不会产生线程安全问题，因为他们访问的是不同的对象，所以不存在互斥问题。
### **可重入性**
一般地，当某个线程请求一个由其他线程持有的锁时，发出请求的线程就会阻塞。然而，由于 Java 的内置锁是可重入的，因此如果某个线程试图获得一个已经由它自己持有的锁时，那么这个请求就会成功。**可重入锁最大的作用是避免死锁**。

***
>占有锁的线程释放锁一般会是以下三种情况之一：
1. 占有锁的线程执行完了该代码块，然后释放对锁的占有；
2. 占有锁线程执行发生异常，此时JVM会让线程自动释放锁；
3. 占有锁线程进入 WAITING 状态从而释放锁，例如在该线程中调用wait()方法等。

[lock框架](http://www.codeceo.com/article/java-lock-framework.html)