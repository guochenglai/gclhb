---
title: 'Java并发编程之2——同步工具类的使用（CountDownLatch,CyclicBarrier,BlockungQueue,Semaphore）'
toc: true
mathjax: true
date: 2016-06-03 10:08:18
categories:
- Java
- Java并发编程
tags:
- Java
- 并发编程
- CountDownLatch
- CyclicBarrier
- Semaphore
description:
---
为了简化线程同步与互斥的相关操作JDK，提供了大约4中同步与互斥的工具类： 闭锁（CountDownLatch），栅栏（CyclicBarrier），阻塞队列（BlockingQueue）,信号量（semaphore）。本文将对比分析四种同步工具类的使用范例，和应用场景。

## 闭锁（CountDownLatch）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;闭锁的作用主要是允许一个或者多个线程等待某个事件的完成，当初始化一个CountDownLatch的时候需要指定同步计数器的个数（等待线程的个数），这个时候主线程的wait方法会一直阻塞，子线程每调用一次countDown方法，同步计数器就会减1,当这个值减少到0时，主线程的wait方法处就会解除阻塞。
**CountDownLatch的主要方法如下：**
- public CountDownLatch(int count);   //构造函数 传入一个同步计数器 当同步计数器大于0的时候主线程调用wait的地方会一直阻塞
- public void countDown();  //当子线程调用一次countDown方法的时候同步计数器的数值就会减1
- public void await() throws InterruptedException //当同步计数器的数值大于0的时候，主线程会一直在wait方法处等待，当同步计数器的大小为0的时候，主线程就会解除阻塞继续运行。

**CountDownLatch示例：**
```java
package com.qunar.des.current;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.CountDownLatch;

/**
 * Created by chenglaiguo on 7/21/15.
 */
public class CountDownLatchTest {
    private static Logger logger = LoggerFactory.getLogger(CountDownLatchTest.class);
    public static void main(String[] args) {
        CountDownLatch countDownLatch = new CountDownLatch(3);//构造函数传入3个同步计数器

        System.out.println("CountDownLatchTest begin ...");
        new Worker("zs",countDownLatch).start(); //启动3个线程，
        new Worker("ls",countDownLatch).start();
        new Worker("ww",countDownLatch).start();

        try {
            countDownLatch.await(); // 当同步计数器的数值大于0的时候，线程会一直在这里等待，不会执行下面的sysout，当两个子线程执行完成后同步计数器数值减到0,主线程会继续执行
        }catch (InterruptedException e){
            logger.error("CountDownLatchTest await cause InterruptedException : ",e);
        }

        System.out.println("CountDownLatchTest end ...");
    }

    static class Worker extends Thread{
        private String name;
        private CountDownLatch countDownLatch;

        public Worker(String name,CountDownLatch countDownLatch){
            this.name = name;
            this.countDownLatch = countDownLatch;
        }

        @Override
        public void run(){
            try {
                Thread.currentThread().sleep(1000);
                System.out.println(name+"complete his work...");
                countDownLatch.countDown();  // 当子线程执行完成之后，每调用一个countDown()方法，程序计数器的数值就会减1

            }catch (InterruptedException e){
                logger.error("Worker run method cause InterruptedException : ",e);
            }
        }

    }
}
```
**CountDownLatch的适用场景：**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;主线程在做一项工作之前需要一系列准备工作，只有这些准备工作都完成之后，主线程才能继续他的工作，而这些准备工作之间是相互独立的。例如  我们的搜索系统，在启动是时候主要有两部分的工作：
- 准备数据
- 建立数据之间的关联关系
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在建立数据关系之前数据的准备必须完成，而大部分的数据之间是相互独立的，所以可以分一些子任务并发运行，当所有的子任务运行完成之后，就可以建立数据关系

## 栅栏（CyclicBarrier）
**栅栏的主要作用是等待其他线程的完成，他CountDownLatch的主要区别是:** 
<font color=red>
- CountDownLatch主要是等待事件的完成，每个子线程只能countDown()一次，当子线程调用一次countDown方法,就表示当前线程的事件完成。（子线程之间不是相互等待，子线程完成之后在主线程的wait处等待）
- CyclicBarrier主要是等待其他的线程完成（所有的子线程之间会相互等待，主线程的流程不会受到子线程的影响）。他的主要方法是一个await(),这个方法每被调用一次计数器便会减1,并阻塞住当前的线程，当计数减到0的时候，阻塞会解除，在这个栅栏上面等待的所有线程都会继续运行。然后进行一次新的循环（这时如果是CountDownLatch会回到主线程运行，而不是唤醒所有在栅栏上面等待的子线程继续运行）
</font>
**CyclicBarrier的主要方法：**
- await() 这个方法每被调用一次计数器便会减1,并阻塞住当前的线程，当计数减到0的时候，阻塞会解除，在这个栅栏上面等待的所有线程都会继续运行。然后进行一次新的循环。

**CyclicBarrier示例：**
```java
package com.qunar.des.current;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.CyclicBarrier;

/**
 * Created by chenglaiguo on 7/24/15.
 */
public class CyclicBarrierTest {
    private static Logger logger = LoggerFactory.getLogger(CyclicBarrierTest.class);
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3);

        System.out.println("cyclicBarrierTest begin...");
        new Runner(cyclicBarrier,"zs").start();  // 启动三个子线程
        new Runner(cyclicBarrier,"ls").start();
        new Runner(cyclicBarrier,"ww").start();

        System.out.println("cyclicBarrierTest end ....");  //主线程不会阻塞，所以启动子线程之后，这里会马上执行
    }

    static class Runner extends Thread{
        private CyclicBarrier cyclicBarrier ;
        private String name;

        public Runner(CyclicBarrier cyclicBarrier,String name){
            this.cyclicBarrier = cyclicBarrier;
            this.name = name;
        }

        @Override
        public void run(){
            try {
                Thread.currentThread().sleep(1000);
                System.out.println(name+": is ready....");
                cyclicBarrier.await(); // 三个子线程会在这里进行第一步的等待 （与主线程无关）当三个子线程都执行完成之后，会同时进行下一步的工作

                Thread.currentThread().sleep(1000);
                System.out.println(name+": is running...");
                cyclicBarrier.await(); // 三个子线程会在这里进行第二步的等待（与主线程无关）当三个子线程都执行完成之后，会同时进入下一步的操作

                Thread.currentThread().sleep(1000);
                System.out.println(name+": is complete..."); //执行完成之后，子线程退出

            }catch (Exception e){
                logger.error("Runner run method cause exception : ",e);

            }

        }
    }
}
```
## 阻塞队列（BlockingQueue）
**BlockingQueue是Java提供的一种阻塞队列实现方式，他主要有四种实现：**
- ArrayBlockingQueue  有界的阻塞队列，在实例化的时候需要指明有界队列的大小，队列采用先进先出的策略
- LinkedBlockQueue  有两种构造方法默认的是不传递任何参数，这种方式是一个无界的阻塞队列，另一种方式和ArrayBlockingQueue相同
- SynchronousBlockQueue 比较特殊（所有的读写操作都是读取交替运行才可以）
- PriorityBolckQueue 有优先级的阻塞队列
**阻塞队列的主要方法（以ArrayBlockingQueue为列子说明）：**
- piblic  ArrayBlockQueue(int count) 构造方法制定阻塞队列的大小
- piblic void put() 方法在队列满的时候会阻塞直到有队列成员被消费，
- public T take() 方法在队列空的时候会阻塞，直到有队列成员被放进来

**BlockingQueue示例：**
```java
package com.qunar.des.current;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Random;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

/**
 * Created by chenglaiguo on 7/24/15.
 */
public class BlockingQueueTest {

    private static Logger logger = LoggerFactory.getLogger(BlockingQueueTest.class);

    public static void main(String[] args) {
        BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<String>(5);

        for (int i=0;i<1000;i++){
            new Consumer(blockingQueue).start();
            new Producer(blockingQueue).start();
        }

    }

    static class Consumer extends Thread{
        private BlockingQueue<String> blockingQueue;

        public Consumer(BlockingQueue<String> blockingQueue){
            this.blockingQueue = blockingQueue;
        }

        @Override
        public void run(){
            try {
                Thread.currentThread().sleep(new Random().nextInt(1000));
                String product = blockingQueue.take();  //如果队列为空 当前线程会在这里阻塞
                System.out.println(Thread.currentThread().getName()+" : consumer : "+product);
            }catch (InterruptedException e){
                logger.error("Consumer run method cause InterruptedException",e);
            }
        }
    }


    static class Producer extends Thread{
        private BlockingQueue<String> blockingQueue;
        public Producer(BlockingQueue<String> blockingQueue){
            this.blockingQueue = blockingQueue;
        }

        @Override
        public void run(){
            try {
                Thread.currentThread().sleep(new Random().nextInt(1000));
                String productName = "product"+new Random().nextInt(1000000);
                System.out.println(Thread.currentThread().getName()+"  : produce :"+productName);
                blockingQueue.put(productName); //如果队列已经满了当前线程会在这里阻塞
            }catch (InterruptedException e){
                logger.error("Producer run method cause InterruptedException",e);
            }
        }
    }

}
```
## 信号量（Semaphore）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;信号量通过构造函数设定一个数量的许可（permit）,然后通过acquire获取许可，通过release释放许可，当acquire获取到许可之后当前的线程就可以执行，否则当前线程阻塞。Semaphore通常用于限制可以访问资源的数量，例如数据库连接池，线程池。单个信号量的semaphore可以用来实现互斥锁
**Semaphore的主要方法如下：**
- Semaphore(int permits, boolean fair) //创建具有给定的许可数和给定的公平设置的Semaphore。  如果以公平方式执行，则线程将会按到达的顺序（FIFO）执行，如果是非公平，则可以后请求的有可能排在队列的头部
- acquire() 获取一个许可，如果没有就等待,当前线程阻塞直到获取到许可之后才会继续执行
- release() 释放一个许可。

**Semaphore示例：**
```java
package com.qunar.des.current;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.Semaphore;

/**
 * Created by chenglaiguo on 7/24/15.
 */
public class SemaphoreTest {

    private static final Logger logger = LoggerFactory.getLogger(SemaphoreTest.class);

    public static void main(String[] args) {
        DataPool dataPool = new DataPool();
        for (int i=0;i<100;i++){  //100个线程获取10个链接，可以演示他们的同步与阻塞
             new ConnectionConsumer(dataPool).start();
        }
    }

    static class ConnectionConsumer extends Thread{
        DataPool dataPool ;
        public ConnectionConsumer(DataPool dataPool){
            this.dataPool = dataPool;
        }

        @Override
        public void run(){
            try {
                Object connection = dataPool.acquireDataConnection(); //获取链接
                System.out.println(Thread.currentThread().getName()+"  acquire "+connection.toString());

                Thread.currentThread().sleep(1000); //消费链接

                dataPool.releaseDataConnection(); //释放链接
                System.out.println(Thread.currentThread().getName()+"  release "+connection.toString());
            }catch (InterruptedException e){
                logger.error("ConnectionConsumer run cause InterruptedException : ",e);
            }

        }
    }

    static class DataPool{   //在这里模拟一个容量为10的数据库连接池
        private final int PERMIT_COUNT = 10;
        private final Semaphore dataPoolSize = new Semaphore(PERMIT_COUNT,true);  //使用公平的信号量，这个信号量中有10个许可
        private Object[] allDataConnection = new Object[PERMIT_COUNT];
        private boolean[] isUsed = new boolean[PERMIT_COUNT];

        DataPool() {  //初始化10个数据库链接池
            allDataConnection[0]="connection 1";
            allDataConnection[1]="connection 2";
            allDataConnection[2]="connection 3";
            allDataConnection[3]="connection 4";
            allDataConnection[4]="connection 5";
            allDataConnection[5]="connection 6";
            allDataConnection[6]="connection 7";
            allDataConnection[7]="connection 8";
            allDataConnection[8]="connection 9";
            allDataConnection[9]="connection 10";
        }

        public Object acquireDataConnection(){ //获取一个数据库链接
            try {
                dataPoolSize.acquire();  //首先获取一个信号量的许可，如果当前线程获取到了许可则当前线程可以继续执行，如果当前线程没有获取到许可，则当前线程会阻塞等待
            }catch (InterruptedException e){
                logger.error("DataPool getDataConnection cause InterruptedException : ",e);
            }
            return acquireAvailableDataConnection(); // 如果当前线程获取到一个信号量的许可之后，就可以获取到一个链接
        }

        public void releaseDataConnection(){ // 释放一个许可，注意：这里是在连接池中任意找到一个链接释放，在实际开发中需要释放的线程应该和释放信号量的那个线程占有的链接是同一个 （这里是简化做法，可能释放其他线程占有的链接）
            dataPoolSize.release();
            for (int i=0;i<PERMIT_COUNT;i++){
                if (isUsed[i]){ // 找到一个被占有的链接释放
                    isUsed[i]=false;
                    return;
                }
            }
        }

        private synchronized Object acquireAvailableDataConnection(){ // 当一个线程获取到信号量之后，就可以找到一个没有使用的链接返回
            for (int i=0;i<PERMIT_COUNT;i++){
                if (!isUsed[i]){ //返回一个未被使用的链接
                    isUsed[i]=true;
                    return allDataConnection[i];
                }
            }
            return null;
        }
    }
}

```
