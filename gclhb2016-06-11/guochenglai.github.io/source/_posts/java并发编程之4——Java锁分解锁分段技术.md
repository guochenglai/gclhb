---
title: java并发编程之4——Java锁分解锁分段技术
toc: true
mathjax: true
date: 2016-06-04 20:02:04
categories:
- Java
- Java并发编程
tags:
- 并发编程
- Java
- 锁分析
description:
---
并发编程的所有问题，最后都转换成了，“有状态bean”的状态的同步与互斥修改问题。而最后提出的解决“有状态bean”的同步与互斥修改问题的方案是为所有修改这个状态的方法都加上锁，这样也就可以保证他们在修改bean的状态的时候是顺序进行的。但是这样整个过程的瓶颈也就是被加锁的这段代码。由此就产生了很多对程序加锁的优化思想，从整体上来看，可以分为两个部分：对单个锁的算法的优化。和对锁粒度的细分。
## 单个锁的优化
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;自旋锁：非自旋锁在未获取锁的情况会被阻塞，之后再唤醒尝试获得锁。而JDK的阻塞和唤醒是基于操作系统实现的，会有系统资源的开销。自旋锁就是线程不停地循环尝试获得锁，而不会将自己阻塞,这样不会浪费系统的资源开销，但是会浪费CPU的资源。所有现在的JDK都的是先自旋等待，如果自旋等待一段时间之后还没有获取到锁，就会将当前线程阻塞。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;锁消除：当JVM分析代码发现某个方法只被单个线程安全访问，而且这个方法是同步方法，那么JVM就会去掉这个方法的锁。
***单个锁优化的瓶颈***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对单个锁优化的效果就像提高单个CPU的处理能力一样，最终会由于各个方面的限制而达到一个平衡点，到达这个点之后优化单个锁的对高并发下面锁的优化效果越来越低。所以将一个锁进行粒度细分带来的效果会很明显，如果一个锁保护的代码块被拆分成两个锁来保护，那么程序的效率就大约能够提高到2倍，这个比单个锁的优化带来的效果要明显很多。常见的 锁粒度细分技术有：锁分解和锁分段      
## 锁粒度细分
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;锁的粒度细分主要有：锁分解和锁分段两种方式。他们的核心都是降低锁竞争发生的可能性。
### 锁分解
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;锁分解的实现方式主要有两种：
#### 缩小锁的范围
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;锁分解的核心是将无关的代码块，如果在一个方法中有一部分的代码与锁无关，一部分的代码与锁有关，那么可以缩小这个锁的返回，这样锁操作的代码块就会减少，锁竞争的可能性也会减少
**传统写法：**
```java
 public synchronized void synchronizedOnMethod(){ //粗粒度直接在方法上加synchronized,这样会提高锁冲突的概率
            prefix();
            try {
                TimeUnit.SECONDS.sleep(1);
            }catch (InterruptedException e){

            }
            post();
        }

        private void post(){
            try {
                TimeUnit.SECONDS.sleep(1);
            }catch (InterruptedException e){

            }
        }

        private void prefix(){
            try {
                TimeUnit.SECONDS.sleep(1);
            }catch (InterruptedException e){

            }
        }
    }

```
**修正写法**
```
//假设prefix和post方法是线程安全的（与锁无关的代码）
static class SynchronizedClazz{
        public void mineSynOnMethod(){
            prefix();
            synchronized (this){ //synchronized代码块只保护有竞争的代码
                try {
                    TimeUnit.SECONDS.sleep(1);
                }catch (InterruptedException e){

                }
            }
            post();

        }

```
#### 减少锁的粒度
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果一个锁需要保护多个相互独立的变量，那么可以将一个锁分解为多个锁，并且每个锁保护一个变量，比较对比下面的两段代码，
<font color=red>假设 allUsers和allComputers是两个相互独立的变量</font>

**传统写法**
```
static class DecomposeClazz{
        private final Set<String> allUsers = new HashSet<String>();
        private final Set<String> allComputers = new HashSet<String>();
        
        public synchronized void addUser(String user){ //公用一把锁
            allUsers.add(user);
        }
        
        public synchronized void addComputer(String computer){
            allComputers.add(computer);
        }
    }
```
**修正写法**
```java
static class DecompossClazz2{
        private final Set<String> allUsers = new HashSet<String>();
        private final Set<String> allComputers = new HashSet<String>();

        public void addUser(String user){ //分解为两把锁
            synchronized (allUsers){
                allUsers.add(user);
            }
        }

        public void addComputer(String computer){
            synchronized (allComputers){
                allComputers.add(computer);
            }
        }
    }
```
### 锁分段
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如上的方法把一个锁分解为2个锁时候，采用两个线程时候，大约能够使程序的效率提升一倍，但是当竞争激烈的时候，单一个锁上面的竞争还是很激烈，我们还可以将锁分解技术进一步扩展到一组独立的对象例如ConcurrentHashMap的锁分段技术
```java
package com.qunar.des.lock;

import java.util.HashMap;
import java.util.Map;

/**
 * Created by chenglaiguo on 8/14/15.
 */
public class MyConcurrentHashMap<K,V> {
    private final int LOCK_COUNT = 16;
    private final Map<K,V> map;
    private final Object[] locks ;

    public MyConcurrentHashMap() {
        this.map = new HashMap<K,V>();
        locks = new Object[LOCK_COUNT];
        for (int i=0;i<LOCK_COUNT;i++){
            locks[i] = new Object();
        }
    }
    
    private int keyHashCode(K k){
        return Math.abs(k.hashCode() % LOCK_COUNT);
    }
    
    public V get(K k){
        int keyHashCode = keyHashCode(k);
        synchronized (locks[keyHashCode % LOCK_COUNT]){
            return map.get(k);
        }
    }
    
}
```