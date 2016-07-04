---
title: java并发编程之3——Java锁的分析
toc: true
mathjax: true
date: 2016-06-03 12:13:42
categories:
- Java
- Java并发编程
tags:
- Java
- 并发编程
description: 在java虚拟机上面每个对象和类在逻辑上面都是和一个监视器相关联的，对于对象来说，监视器保护的是对象的实例变量，对于类来说，监视器保护的是类的类变量，为了实现监视器的排他能力，JVM为每个对象分配一个锁（称为内置锁），任何时候只有一个线程可以获得这个锁，当前线程访问实例对象不需要重新获取锁（锁的重入，在后面会介绍），   当只有一个线程获取到该对象的锁之后，在他释放这个锁之前其他线程只能等待。
---
 在分析Java的锁之前首先解释一下JVM对的内存分配模型

## JVM内存模型
**JVM的内存公分为5个部分，主要包括：**
- 方法区
- 堆
- 栈
- 程序计数器
- 本地方法栈，
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其中方法区和堆是由多线程共享的，其他的区域数据都是由线程独享的，所有在这两个部分保存的数据是所有的线程都可以访问的，我们的锁机制主要也就是保护以下两个方面的内容：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1  保存在堆中的类实例对象（包括实例对象的变量，方法）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2  保存在方法区的类变量 

## JVM监视器
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在java虚拟机上面每个对象和类在逻辑上面都是和一个监视器相关联的，对于对象来说，监视器保护的是对象的实例变量，对于类来说，监视器保护的是类的类变量，为了实现监视器的排他能力，JVM为每个对象分配一个锁（称为内置锁），任何时候只有一个线程可以获得这个锁，当前线程访问实例对象不需要重新获取锁（锁的重入，在后面会介绍），   当只有一个线程获取到该对象的锁之后，在他释放这个锁之前其他线程只能等待。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;开发人员不需要加锁，对象锁是java虚拟机自己使用的，以前我一直认为在代码块或者在方法上面加上synchronized关键字就是调用当前对象的内置锁，其实真相是： 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当在一个代码块或者方法上加synchronized关键字，就是标示要监视这段代码 （告诉JVM我要监视这段代码，）每当进入被监视的代码块的时候JVM就会自动使用当前对象的内置锁，锁上当前的对象。

## 对象锁
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;java中用synchronized监视的代码块获取的锁都是对象的锁，例如：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1  在普通的方法上面添加的锁，就是调用这个方法对应的对象的锁（当这个对象在多个线程之间共享的时候）就会出现线程的同步与互斥
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2  在静态方法上面添加锁获取到的是这个类对象的锁

### 获取同一个对象的锁同时只有一个线程可以执行 
```java
package com.qunar.des.current;

/**
 * Created by chenglaiguo on 7/27/15.
 */
public class SynchronizedTest {
    public static void main(String[] args) {
        MySynchronized mySynchronized = new MySynchronized();

        for (int i=0;i<10;i++){
            new SynchronizedThread(mySynchronized).start();
        }

    }

    static class SynchronizedThread extends Thread{

        private MySynchronized mySynchronized ;

        public SynchronizedThread(MySynchronized mySynchronized){
            this.mySynchronized = mySynchronized;
        }

        @Override
        public void run(){
            mySynchronized.synchronizedInMethod();
            mySynchronized.synchronizedOnMethod();
        }
    }

    static class MySynchronized{
        public synchronized void synchronizedOnMethod(){ //同步方法 获取的锁是当前对象的锁，也就是SynchronizedThread中创建的对象mySynchronized的锁
            try {
                System.out.println(Thread.currentThread().getName()+" acquire lock");
                Thread.currentThread().sleep(100);
                System.out.println(Thread.currentThread().getName()+"   synchronizedOnMethod");
            }catch (InterruptedException e){

            }finally {
                System.out.println(Thread.currentThread().getName()+" release lock");
            }
        }
        public synchronized void synchronizedInMethod(){
            synchronized (this){  //同步代码块 这里用this的锁就是调用这个方法对象的锁 也就是SynchronizedThread中创建的对象mySynchronized的锁
                try {
                    System.out.println(Thread.currentThread().getName()+" acquire lock");
                    Thread.currentThread().sleep(100);
                    System.out.println(Thread.currentThread().getName()+"   synchronizedInMethod");
                }catch (InterruptedException e){

                }finally {
                    System.out.println(Thread.currentThread().getName()+" release lock");
                }
            }
        }
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;程序的运行结果如下：  
```java
Thread-0 acquire lock
Thread-0   synchronizedInMethod
Thread-0 release lock
Thread-9 acquire lock
Thread-9   synchronizedInMethod
Thread-9 release lock
Thread-9 acquire lock
Thread-9   synchronizedOnMethod
Thread-9 release lock
Thread-8 acquire lock
Thread-8   synchronizedInMethod
Thread-8 release lock
Thread-8 acquire lock
Thread-8   synchronizedOnMethod
Thread-8 release lock
Thread-7 acquire lock
Thread-7   synchronizedInMethod
Thread-7 release lock
Thread-7 acquire lock
Thread-7   synchronizedOnMethod
Thread-7 release lock
Thread-6 acquire lock
Thread-6   synchronizedInMethod
Thread-6 release lock
Thread-6 acquire lock
Thread-6   synchronizedOnMethod
Thread-6 release lock
Thread-5 acquire lock
Thread-5   synchronizedInMethod
Thread-5 release lock
Thread-5 acquire lock
Thread-5   synchronizedOnMethod
Thread-5 release lock
Thread-4 acquire lock
Thread-4   synchronizedInMethod
Thread-4 release lock
Thread-4 acquire lock
Thread-4   synchronizedOnMethod
Thread-4 release lock
Thread-3 acquire lock
.....
```

可以看到下面的输出都是两个线程交替输出，原因 ： 两个方法拿到的锁都是同一个对象的锁，同一时刻只有一个线程可以拿到这个锁可以执行。
### 获取不同对象的锁互不影响
```java
package com.qunar.des.current;

/**
 * Created by chenglaiguo on 7/27/15.
 */
public class SynchronizedTest {
    public static void main(String[] args) {
        MySynchronized mySynchronized = new MySynchronized();

        for (int i=0;i<10;i++){
            new SynchronizedThread(mySynchronized).start();
        }

    }

    static class SynchronizedThread extends Thread{

        private MySynchronized mySynchronized ;

        public SynchronizedThread(MySynchronized mySynchronized){
            this.mySynchronized = mySynchronized;
        }

        @Override
        public void run(){
            mySynchronized.synchronizedOnMethod();
            mySynchronized.synchronizedWithInnerLock();
        }
    }

    static class MySynchronized{
        private Object innerLock = new Object();

        public synchronized void synchronizedOnMethod(){ //synchronized方法调用的是当前对象的锁这里是SynchronizedTest中实例化的mySynchronized的锁
            try {
                System.out.println(Thread.currentThread().getName()+" acquire lock");
                Thread.currentThread().sleep(100);
                System.out.println(Thread.currentThread().getName()+"   synchronizedOnMethod");
            }catch (InterruptedException e){

            }finally {
                System.out.println(Thread.currentThread().getName()+" release lock");
            }
        }

        public void synchronizedWithInnerLock(){
            synchronized (innerLock) { //调用的是innerlock的锁
                try {
                    System.out.println(Thread.currentThread().getName()+" acquire lock");
                    Thread.currentThread().sleep(100);
                    System.out.println(Thread.currentThread().getName() + "   synchronizedWithInnerLock");
                } catch (InterruptedException e) {

                }finally {
                    System.out.println(Thread.currentThread().getName()+" release lock");
                }
            }
        }

    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;程序的输入如下： ```java
Thread-0 acquire lock
Thread-0   synchronizedOnMethod
Thread-0 release lock
Thread-0 acquire lock
Thread-9 acquire lock
Thread-0   synchronizedWithInnerLock
Thread-9   synchronizedOnMethod
Thread-0 release lock
Thread-9 release lock
Thread-9 acquire lock
Thread-8 acquire lock
Thread-9   synchronizedWithInnerLock
Thread-8   synchronizedOnMethod
Thread-9 release lock
Thread-8 release lock
Thread-8 acquire lock
Thread-7 acquire lock
Thread-8   synchronizedWithInnerLock
Thread-8 release lock
Thread-7   synchronizedOnMethod
Thread-7 release lock
Thread-7 acquire lock
Thread-6 acquire lock
Thread-7   synchronizedWithInnerLock
Thread-6   synchronizedOnMethod
Thread-6 release lock
Thread-7 release lock
Thread-5 acquire lock
Thread-6 acquire lock
Thread-5   synchronizedOnMethod
Thread-6   synchronizedWithInnerLock
Thread-5 release lock
Thread-6 release lock
Thread-5 acquire lock
Thread-4 acquire lock
Thread-5   synchronizedWithInnerLock
Thread-5 release lock
Thread-4   synchronizedOnMethod
Thread-4 release lock
Thread-4 acquire lock
Thread-3 acquire lock
Thread-4   synchronizedWithInnerLock
Thread-4 release lock
Thread-3   synchronizedOnMethod
Thread-3 release lock
Thread-3 acquire lock
Thread-2 acquire lock
...
```
两个方法可以同时获取到锁，可以同时执行。原因：这两个方法上面使用的锁不是相同的锁，所以他们之间的获取和释放没有任何关系
### 对象锁和类对象锁
```java 
package com.qunar.des.current;

/**
 * Created by chenglaiguo on 7/27/15.
 */
public class SynchronizedTest {
    public static void main(String[] args) {
        MySynchronized mySynchronized = new MySynchronized();

        for (int i=0;i<10;i++){
            new SynchronizedThread(mySynchronized).start();
        }

    }

    static class SynchronizedThread extends Thread{

        private MySynchronized mySynchronized ;

        public SynchronizedThread(MySynchronized mySynchronized){
            this.mySynchronized = mySynchronized;
        }

        @Override
        public void run(){
            mySynchronized.synchronizedOnMethod();
            MySynchronized.synchronizedOnClass();
        }
    }

    static class MySynchronized{

        public synchronized void synchronizedOnMethod(){ //获取的锁是当前对象的锁
            try {
                System.out.println(Thread.currentThread().getName()+" acquire lock");
                Thread.currentThread().sleep(100);
                System.out.println(Thread.currentThread().getName()+"   synchronizedOnMethod");
            }catch (InterruptedException e){

            }finally {
                System.out.println(Thread.currentThread().getName()+" release lock");
            }
        }

        public static synchronized void synchronizedOnClass(){ //获取的是类对象的锁
            try {
                System.out.println(Thread.currentThread().getName()+" acquire lock");
                Thread.currentThread().sleep(100);
                System.out.println(Thread.currentThread().getName()+"   synchronizedOnClass");
            }catch (InterruptedException e){

            }finally {
                System.out.println(Thread.currentThread().getName()+" release lock");
            }
        }


    }
}
```
程序的输入如下：在多线程访问的情况下两个方法可以同时进入，当前对象的锁和类对象的锁不是同一把锁，所以可以同时访问。
```java
Thread-0 acquire lock
Thread-0   synchronizedOnMethod
Thread-0 release lock
Thread-0 acquire lock
Thread-9 acquire lock
Thread-0   synchronizedOnClass
Thread-9   synchronizedOnMethod
Thread-0 release lock
Thread-9 release lock
Thread-9 acquire lock
Thread-8 acquire lock
Thread-9   synchronizedOnClass
Thread-9 release lock
Thread-8   synchronizedOnMethod
Thread-8 release lock
Thread-8 acquire lock
Thread-7 acquire lock
Thread-8   synchronizedOnClass
Thread-8 release lock
Thread-7   synchronizedOnMethod
Thread-7 release lock
Thread-7 acquire lock
Thread-6 acquire lock
Thread-7   synchronizedOnClass
Thread-6   synchronizedOnMethod
Thread-7 release lock
Thread-6 release lock
Thread-6 acquire lock
Thread-5 acquire lock
Thread-6   synchronizedOnClass
Thread-5   synchronizedOnMethod
Thread-5 release lock
Thread-6 release lock
Thread-5 acquire lock
Thread-4 acquire lock
Thread-5   synchronizedOnClass
Thread-5 release lock
Thread-4   synchronizedOnMethod
Thread-4 release lock
Thread-4 acquire lock
Thread-3 acquire lock
Thread-4   synchronizedOnClass
Thread-3   synchronizedOnMethod
Thread-4 release lock
Thread-3 release lock
Thread-3 acquire lock
Thread-2 acquire lock
Thread-3   synchronizedOnClass
Thread-2   synchronizedOnMethod
Thread-2 release lock
Thread-3 release lock
Thread-2 acquire lock
Thread-1 acquire lock
Thread-2   synchronizedOnClass
Thread-2 release lock
Thread-1   synchronizedOnMethod
Thread-1 release lock
Thread-1 acquire lock
Thread-1   synchronizedOnClass
Thread-1 release lock
```
	
### 类对象锁的互斥性

```java
package com.qunar.des.current;

/**
 * Created by chenglaiguo on 7/27/15.
 */
public class SynchronizedTest {
    public static void main(String[] args) {
        MySynchronized mySynchronized = new MySynchronized();

        for (int i=0;i<10;i++){
            new SynchronizedThread(mySynchronized).start();
        }

    }

    static class SynchronizedThread extends Thread{

        private MySynchronized mySynchronized ;

        public SynchronizedThread(MySynchronized mySynchronized){
            this.mySynchronized = mySynchronized;
        }

        @Override
        public void run(){
            mySynchronized.synchronizedOnStaticMethod();
            MySynchronized.synchronizedOnClass();
        }
    }

    static class MySynchronized{


        public void synchronizedOnStaticMethod(){
           synchronizedOnMethod(); //调用static对象的锁，就是类对象的锁
        }


        private static synchronized void synchronizedOnMethod(){
            try {
                System.out.println(Thread.currentThread().getName()+" acquire lock");
                Thread.currentThread().sleep(100);
                System.out.println(Thread.currentThread().getName()+"   synchronizedOnMethod");
            }catch (InterruptedException e){

            }finally {
                System.out.println(Thread.currentThread().getName()+" release lock");
            }
        }

        public static synchronized void synchronizedOnClass(){ //获取当前类对象的锁
            try {
                System.out.println(Thread.currentThread().getName()+" acquire lock");
                Thread.currentThread().sleep(100);
                System.out.println(Thread.currentThread().getName()+"   synchronizedOnClass");
            }catch (InterruptedException e){

            }finally {
                System.out.println(Thread.currentThread().getName()+" release lock");
            }
        }


    }
}
```
程序的输出如下： 两个方法不会同时执行，都是用的同一个类对象的锁
```java
Thread-0 acquire lock
Thread-0   synchronizedOnMethod
Thread-0 release lock
Thread-9 acquire lock
Thread-9   synchronizedOnMethod
Thread-9 release lock
Thread-9 acquire lock
Thread-9   synchronizedOnClass
Thread-9 release lock
Thread-8 acquire lock
Thread-8   synchronizedOnMethod
Thread-8 release lock
Thread-8 acquire lock
Thread-8   synchronizedOnClass
Thread-8 release lock
Thread-7 acquire lock
Thread-7   synchronizedOnMethod
Thread-7 release lock
Thread-7 acquire lock
Thread-7   synchronizedOnClass
Thread-7 release lock
Thread-6 acquire lock
Thread-6   synchronizedOnMethod
Thread-6 release lock
Thread-6 acquire lock
Thread-6   synchronizedOnClass
Thread-6 release lock
Thread-5 acquire lock
Thread-5   synchronizedOnMethod
Thread-5 release lock
Thread-5 acquire lock
Thread-5   synchronizedOnClass
Thread-5 release lock
Thread-4 acquire lock
Thread-4   synchronizedOnMethod
Thread-4 release lock
Thread-4 acquire lock
Thread-4   synchronizedOnClass
Thread-4 release lock
Thread-3 acquire lock
Thread-3   synchronizedOnMethod
Thread-3 release lock
Thread-3 acquire lock
Thread-3   synchronizedOnClass
Thread-3 release lock
Thread-2 acquire lock
Thread-2   synchronizedOnMethod
Thread-2 release lock
Thread-2 acquire lock
Thread-2   synchronizedOnClass
Thread-2 release lock
Thread-1 acquire lock
Thread-1   synchronizedOnMethod
Thread-1 release lock
Thread-1 acquire lock
Thread-1   synchronizedOnClass
Thread-1 release lock
Thread-0 acquire lock
Thread-0   synchronizedOnClass
Thread-0 release lock
```
