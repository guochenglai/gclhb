---
title: java并发编程之1——synchronized和volatile
toc: true
mathjax: true
date: 2016-06-02 20:03:31
categories: 
- Java
- Java并发编程
tags: 
- 并发编程
- synchronized
- Java
description: 总的来说synchronized主要是解决线程互斥性问题，volatitle主要是解决线程可见性的问题。synchronized主要用来保证临界的代码在同一时刻只有一个线程在访问获取修改。那么synchronized保护的变量一定是线程安全的。volatile能够保证在valatile变量之前的代码在valatile修饰的代码之前执行，由volatile修饰的代码之后的代码在voliate修饰的代码之后运行。
---
总的来说synchronized主要是解决线程互斥性问题，volatitle主要是解决线程可见性的问题。

## synchronized
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;synchronized主要用来保证临界的代码在同一时刻只有一个线程在访问获取修改。那么synchronized保护的变量一定是线程安全的。

### **synchronized的线程安全性**           
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;synchronized 拿到的锁是java对象的内置锁，同一个对象的内置锁在一个时间点只有一个线程可以获取（注意：这里是线程，不是调用，涉及到可重入锁，将在后面介绍）所以由synchronized加锁的代码块一定是线程安全的。
### **synchronized的变量可见性**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;synchronized 加锁的变量在统一时刻只有一个线程可以看到，所以有synchronized保护的变量在多个线程之间一定是可见的。
### **对象锁**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;synchronized加锁的代码块拿到的锁是当前对象的内置锁，注意是对象的锁。
```
public class LockTestClass {

        private Object object = new Object();

	//无锁
	public void noSynMethod(long threadID, ObjThread thread) {
		System.out.println("nosyn: class obj is " + thread + ", threadId is"
				+ threadID);
	}

	//当前对象锁
	public synchronized void synOnMethod() {
		System.out.println("synOnMethod begins" + ", time = "
				+ System.currentTimeMillis() + "ms");

	}

	//锁代码块，用的还是当前对象的锁
	public void synInMethod() {
		synchronized (this) {
			System.out.println("synInMethod begins" + ", time = "
					+ System.currentTimeMillis() + "ms");

		}

	}

	//使用object对象的锁，所以与当前调用对象的锁不冲突
	public void synMethodWithObj() {
		synchronized (object) {
			System.out.println("synMethodWithObj begins" + ", time = "
					+ System.currentTimeMillis() + "ms");

		}
	}

	//调用的是类对象的锁
	public static synchronized void increament() {
		System.out.println("class synchronized  time = "
				+ System.currentTimeMillis() + "ms");

	}

}
```

 
## volatile
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;volatile主要是告诉JVM当前在寄存器的变量是不稳定的，需要当前线程直接从内存中存取。主要的作用是保证一个变量在各个线程之间是直接可见的。

### **变量的可见性**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;valatile 是JDK中一个轻量级的保证变量可见性的标识，当一个变量前面添加了synchronized那么CPU读取这个变量的时候就不会在寄存器读取，直接从内存中读取，所以每个线程读取到的变量的值，都是当前变量的最新的值。
### **非线程安全性**
volatile 保护的变量能够保证一个变量在多个线程之间可见，但是并不能保证这个变量在同一时刻只有一个线程来修改他，所以会出现经典的i++的问题。当多个线程修改同一个变量的时候同样会出现线程安全性的问题。

### **顺序执行**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;volatile能够保证在valatile变量之前的代码在valatile修饰的代码之前执行，由volatile修饰的代码之后的代码在voliate修饰的代码之后运行。

例如：
```
    int a = 1;
    int b = 2;
    int c = 3;
    如果cpu分析得到 abc三个变量在后面的代码中并没有什么关联关系，CPU的执行顺序并不一定按照 abc的顺序执行


    int a = 1;
    volatile int b = 2;
    int c = 3;
    由于volatile修饰变量b，那么a一定在b之前被cpu执行，c一定在b之后被cpu执行

```
