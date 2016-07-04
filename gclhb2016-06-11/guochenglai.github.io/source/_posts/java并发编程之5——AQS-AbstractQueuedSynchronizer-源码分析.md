---
title: java并发编程之5——AQS-AbstractQueuedSynchronizer-源码分析
toc: true
mathjax: true
date: 2016-06-06 10:10:34
categories:
- Java
- Java并发编程
- 源码分析
tags:
- Java
- 并发编程 
- AQS
- 源码分析
description:  AbstractQueudSynchronizer（AQS）是道格李java并发编程的基础，内部主要包括Node和ConditionObject两个内部类，基于Node节点构建了一个FIFO队列，用来存储等待锁的线程的队列。基于ConditionObject节点也构造了一个FIFO队列，用于存储因为某种原因已经获取到锁而又主动释放锁的线程的队列。
---
# 什么是AQS
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AbstractQueudSynchronizer（AQS）是道格李java并发编程的基础，内部主要包括Node和ConditionObject两个内部类，基于Node节点构建了一个FIFO队列，用来存储等待锁的线程的队列。基于ConditionObject节点也构造了一个FIFO队列，用于存储因为某种原因已经获取到锁而又主动释放锁的线程的队列。在concurrent包下面的大部分的工具类都是以他为基础，包括CountDownLatch,Lock,ReadWriteLock,Semaphare,条件队列....等等。
# 如何利用AQS编写自己的同步工具类
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所有基于AQS实现的同步工具类的实现方法都可以划归为以下三步：
- 子类定义一个名称为Sync的静态内部类，该类继承子AQS
- Sync类实现tryAcquire/tryRelease(独占式) tryAcquireShared/tryReleaseShare(共享式) 来管理状态，所有对状态的管理都是通过AQS的getState/setState/compareAndSetState/三个方法来管理的
- 构造函数实例化Sync类在实例化Sync类的同时，指定同步类的同步状态的值
```
               public  int getState() 返回同步器的状态

               public void setState(int arg) 设置同步状态

               public  boolean compareAndSetState(int expect,int target) 原子的方式更新状态，如果现在的状态是expect就更新的target
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AQS可以被定义为共享模式和排他模式，当他被定义为排他模式的时候，他会阻止其他线程获取同步器的状态，当他被定义为共享模式的时候其他线程就可以获取同步器的同步状态。

# AQS原理概述
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AQS是一个运用模板方法设计模式的典型示例。AQS是Java并发框架的基础抽象类，他提供了：如何让线程入队，如何让线程出队，线程如何从等待队列转移到条件队列，以及线程如何从条件队列转移到等待队列等，模板方法。而每一个AQS的子类，所要做的事情是：如何决定线程的出队入队。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如：Lock是基于AQS实现的同步工具类。引用场景是当前线程如果获取到了锁就执行锁保护的代码，当前线程没有获取到锁，就将当前线程加入到等待队列，直到有线程执行完锁的代码之后主动释放锁，当前线程才能继续尝试获取到锁。
他们的实现方式如下：
  - Lock类有一个同步状态用来标识当前线程是否可以获取到锁。
  - 如果当前线程没有获取到锁，就调用AQS的方法，让当前线程进入到等待队列。
  - 其他线程执行完成之后，调用AQS的方法告知等待锁的线程队列，可以获取锁了。
  - AQS决定让哪一个线程去获取锁，并将其移除等待队列。
<font color=red>***所以一句话：AQS实现线程的入队，子类决定线程的入队。***</font>
# AQS源码分析
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在查看AQS的源代码的时候请时刻记住：AQS的主要功能就是围绕等待线程的出队入队。
<font color=red>**注意本文对源代码按照功能进行了一定的重排序，试图提取出代码的逻辑主线，和整体流程。所以代码的顺序和源代码的顺序有一定的前后顺序调整**</font>
## Node节点的定义
```java
   static final class Node {
        //标识当前节点是共享模式
        static final Node SHARED = new Node();
        //标识当前节点是排他模式
        static final Node EXCLUSIVE = null;
        
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
        
        /*
        * 标识线程的等待状态 取值有以上四种  
        * 1 表示等待线程节点已取消 
        * -1 表示当前线程节点需要被激活
        * -2 表示当前线程节点，等待在条件队列上而不是等待队列上
        * -3 表示当前线程节点的后续节点的acquireShare方法能够被无条件执行
        */
        volatile int waitStatus;
        
        //等待队列的前驱节点
        volatile Node prev;
        //等待队列的后继节点
        volatile Node next;
        //当前节点的线程
        volatile Thread thread;
        //条件队列的等待节点
        Node nextWaiter;
        //判断当前节点是否是共享节点
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        //获取当前节点的前驱节点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        // 默认构造函数，用来建立初始化节点
        Node() { 
        }

        // 等待队列使用的节点
        Node(Thread thread, Node mode{ 
            this.nextWaiter = mode;
            this.thread = thread;
        }

        //条件队列使用的节点
        Node(Thread thread, int waitStatus) {
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```
## AQS内部变量定义
```java
    //等待队列的头结点，当调用setHead方法的时候才会初始化，属于懒加载
    private transient volatile Node head;
    //等待队列的尾节点
    private transient volatile Node tail;
    //AQS队列的状态
    private volatile int state;

```
## CAS操作
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;本方法调用的是JDK的内部unsafe类，是基于硬件保证原子的更新。
<font color=red>如果这个值是“expert”就更新为“update”</font>是无锁并发的基础。
```java
 protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```
## 独占模式构建等待队列的实现
**构建等待队列有很多的变种，有的加入了中断，有的加入了时间判断，但是根本的原理是一样的。这个例子是以无中断，无时间判断来讲解的。后面查看源代码的时候，会继续提到其他的方法。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;acquire是AQS提供的非共享模式获取锁的入口，首先执行tryAcquire方法（<font color=red>由具体的子类实现，不同的子类有不同的实现方式，这个在后续分析Lock源代码的时候会结合起来。</font>），如果失败，表示该线程获取锁失败，就调用addWaiter方法，将当前线程加入到等待队列中，然后返回当前线程的node节点。将node节点传递给acquireQueued方法，如果node节点的前驱节点是头结点，就再次尝试获取到锁，如果获取锁成功（成功返回的是false不会执行selfInterrupt方法），就讲该节点设置为头结点，如果获取失败，就将当前节点的线程挂起。
```java

public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;将传入的节点加入到等待队列。
```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode); //将节点构造为等待节点（前面的构造方法已经讲过）
        Node pred = tail;
        if (pred != null) { //如果为节点不为空，表示已经有节点了，就将该节点添加到等待队列
            //下面的方法是先将node节点的前驱节点指向尾节点，然后CAS将尾节点设置为node节点（设置尾节点的时候采用的是位移运算，所以看不到直接的设置地方）
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        
        //如果尾节点为null，就构造初始队列将节点添加到队列的尾部
        enq(node);
        return node;
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;等待队列为空的情况下，无锁地构造初始等待队列。
```java
private Node enq(final Node node) {
        for (;;) { //for循环，直到队列构造成功
            Node t = tail;
            if (t == null) { //第一次循环：队列为空，就构造一个节点设置为头结点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {//第二次循环执行到这里 首先将node节点的前驱节点指向尾节点，然后CAS设置node节点为尾节点
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        
```

<font color=red>(在挂起当前线程之前进行最后一次挣扎)！！！！！</font>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在节点添加到了等待队列中的时候再次尝试获取到锁 这里的arg一般是0或者1。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;本方法的主要作用是看CAS是否能够成功，成功表示当前线程获取到了锁返回false（注意 成功返回false和我们平时理解的正好相反）,如果失败就将当前线程挂起，在AQS提供的公用的acquire方法中调用了他
```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取node节点的前驱
                final Node p = node.predecessor();
                /**
                *如果node节点的前驱节点是头结点，
                *并且CAS更新同步状态成功（表示获取到锁了）
                *就返回false，这里的tryAcquire是由AQS的不同子类实现的。
                *后面我会写专门的文章来解析这部分
                */
                if (p == head && tryAcquire(arg)) { //这里是独占方式所以tryAcquire返回的是boolean，只能是成功或者失败，对比后面分析的tryAcquireShared
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //如果上一个操作没有成功，就判断当前node节点的线程是否应该被挂起，
                //如果是，就尝试挂起这个节点对应的线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;设置头结点
```java
private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
```
## 独占模式激活并移除等待队列的节点
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;激活并移除等待节点的过程，和加入等待节点的过程正好相反。首先调用子类的tryRelease方法，如果失败，就返回，如果tryRelease方法释放锁成功。就拿到队列的头结点。然后激活头结点的后继节点，激活的过程是，首先找到头结点的第一个后继有效节点，将其从队列中移除，然后激活这个节点对应的线程。
```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;激活node节点的后继节点
```java
private void unparkSuccessor(Node node) {
        //获取当前节点的状态
        /**
        * 1 表示等待线程节点已取消 
        * -1 表示当前线程节点需要被激活
        * -2 表示当前线程节点，等待在条件队列上而不是等待队列上
        * -3 表示当前线程节点的后续节点的acquireShare方法能够被无条件执行
        */
        int ws = node.waitStatus;
        
        //如果节点的状态不是已取消,就讲节点的状态设置为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        //获取node节点的后继节点
        Node s = node.next;
        //如果后继节点不合法，就一直循环，直到找到一个合法的后继节点
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //激活s节点，其实就是唤醒这个node节点对应的线程
            LockSupport.unpark(s.thread);
    }
```

## 共享模式构建等待队列的实现
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;共享模式构建等待队列的实现的流程和独占模式构建等待队列的实现是一样的，唯一的不一样的地方是“tryAcquireShared”这个由子类实现的方法。他的过程是：首先尝试获取共享锁（注意这里返回的是整数，这是实现共享模式的关键。）如果失败（小于0），就构建一个共享节点添加到等待队列。并将当前线程挂起。
```java
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
```java
private void doAcquireShared(int arg) {
        
        在addWaiter中以Node.SHARE构建一个node节点，添加到node队列的结尾并返回构建的节点
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                获取node节点的前驱节点
                final Node p = node.predecessor();
                
                如果node节点的前驱节点是头结点，注意这里和独占式的区别，独占式在这里CAS设置状态
                if (p == head) {
                    获取节点的状态
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        将node节点设置为头结点 ，如果r大于0，原来的头结点的状态小于0，就获取node节点的后继节点，如果后继节点为null或者后继节点是共享节点，就激活node节点的后继节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

## 共享模式激活并移除等待队列的节点
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;共享模式释放节点的流程和独占模式释放节点的流程基本一致。首先尝试更新释放状态tryReleaseShared方法，由具体的子类实现，如果成功就激活头节点的后继节点。
```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
    
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;激活头结点的后继有效节点。
```java
private void doReleaseShared() {
        for (;;) {
            Node h = head; //获取头结点
            if (h != null && h != tail) {
                int ws = h.waitStatus; //获取头结点的状态
                if (ws == Node.SIGNAL) {//如果头节点线程节点需要被激活，就尝试更新头结点的状态为0，如果更新状态失败，就继续循环，如果更新状态成功，就激活头结点的有效后继节点。
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)){ //更新状态失败，就继续循环
                        continue;
                 }
                    //更新状态成功就激活头结点的有效后继节点
                    unparkSuccessor(h);
                }else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) //如果头结点的初始状态为0，就CAS将状态更新为-3，如果成功，就判断头结点是否被修改，
                    continue; //CAS 失败就一直循环
            }
            if (h == head) //如果头结点指针没有变化，就一直循环，否则，退出循环
                break;
        }
    }
```
```java
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; //记录下原来的头结点 
        setHead(node);//将node节点设置为头结点
       
        if (propagate > 0 || h == null || h.waitStatus < 0) {
            Node s = node.next; 
            if (s == null || s.isShared()) //如果是共享节点就激活头结点的后继节点
                doReleaseShared();
        }
    }

```
## 节点的取消
将节点的前驱有效节点，和后继有效节点连接起来，取消当前节点
```java
private void cancelAcquire(Node node) {
        if (node == null)
            return;
        node.thread = null;
        //获取node节点的有效的前驱节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // 找到node节点的后继节点
        Node predNext = pred.next;

        //将node节点的状态设置为取消
        node.waitStatus = Node.CANCELLED;

        // 如果node节点是尾节点，就将node节点的前驱节点设置为尾节点，并将node前驱节点的后继节点设置为null
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            //如果node节点不是尾节点  
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {//如果node前驱节点不是头结点，就将前驱节点的状态设置为SINGAL。
                Node next = node.next;//找到node节点的后继节点
                if (next != null && next.waitStatus <= 0)//前后链接取消node节点
                    compareAndSetNext(pred, predNext, next);
            } else { //如果node前驱节点是头节点，激活node节点的后继有效节点
                unparkSuccessor(node);
            }

            node.next = node; 
        }
    }
```
根据node节点和node节点的前驱节点的状态（只有前驱节点的状态为SIGNAL时，后继节点才应该被挂起）判断node节点对应的线程是否应该挂起
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //获取前驱节点的状态
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL) //如果前驱节点状态为SIGNAL，说明当前节点应该被挂起
            return true;
        if (ws > 0) {//如果前驱节点被取消就一直往前找，直到找到有效的前驱节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node; //将node节点的前驱无效节点删除
        } else { //如果前驱节点的状态非以上两种，就设置前驱节点的状态，并且，返回false
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

```
## 其他代码分析。
上面分析了AQS的整体执行流程图。下面这些方法其实和上面的方法功能都一样，只不过是加了一些中断判断，时间判断等。这里就不一一列出了。
## ConditionObject节点的定义
```java
 public class ConditionObject implements Condition, java.io.Serializable {
        private transient Node firstWaiter;
        private transient Node lastWaiter;
        public ConditionObject() { }
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
        public final void awaitUninterruptibly() {
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean interrupted = false;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if (Thread.interrupted())
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted)
                selfInterrupt();
        }
        private static final int REINTERRUPT =  1;
        private static final int THROW_IE    = -1;
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            long lastTime = System.nanoTime();
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;

                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;
                lastTime = now;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return nanosTimeout - (System.nanoTime() - lastTime);
        }

        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            if (deadline == null)
                throw new NullPointerException();
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() > abstime) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
            if (unit == null)
                throw new NullPointerException();
            long nanosTimeout = unit.toNanos(time);
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            long lastTime = System.nanoTime();
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;
                lastTime = now;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

       
        final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
            return sync == AbstractQueuedSynchronizer.this;
        }

       
        protected final boolean hasWaiters() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    return true;
            }
            return false;
        }

       
        protected final int getWaitQueueLength() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int n = 0;
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    ++n;
            }
            return n;
        }

        
        protected final Collection<Thread> getWaitingThreads() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            ArrayList<Thread> list = new ArrayList<Thread>();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION) {
                    Thread t = w.thread;
                    if (t != null)
                        list.add(t);
                }
            }
            return list;
        }
}
```
## 构建条件队列的实现
## 移除条件队列的实现

