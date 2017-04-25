---
title: 自旋锁
date: 2017-04-23
tags: [锁]
categories: "锁"
file: spinlock.md
---
# 自旋锁
自旋锁是采用让当前线程不停的在循环体内执行实现的， 当循环的条件被其它线程改变时才能进入临界区。代码如下： 
```java
 	public class SpinLock { 	 
	  private AtomicReference<Thread> sign =newAtomicReference<>(); 	 
 	    public void lock(){
 	      Thread current = Thread.currentThread();
 	      while(!sign .compareAndSet(null, current)){
 	      }
 	  }
 	 
 	  public void unlock (){
 	    Thread current = Thread.currentThread();
 	    sign .compareAndSet(current, null);
 	  }
 	}
```
注：该例子为非公平锁，获得锁的先后顺序，不会按照进入lock的先后顺序进行。
lock函数使用了CAS原子操作， 将owner设置为当前线程， 并且预测原来的值为空。unlock函数将owner设置为null，并且预测值为当前线程。
当有第二个线程调用lock操作时由于owner值不为空，导致循环一直被执行，直至第一个线程调用unlock函数将owner设置为null，第二个线程才能进入临界区。
由于自旋锁只是将当前线程不停地执行循环体，不进行线程状态的改变，所以响应速度更快。但当线程数不停增加时，性能下降明显，因为每个线程都需要执行，占用CPU时间。
如果线程竞争不激烈，并且保持锁的时间段。适合使用自旋锁。
# 自旋锁的其它种类
锁作为并发共享数据，保证一致性的工具，在JAVA平台有多种实现(如 synchronized 和 ReentrantLock等等 ) 。这些已经写好提供的锁为我们开发提供了便利，但是锁的具体性质以及类型却很少被提及。本系列文章将分析JAVA下常见的锁名称以及特性，为大家答疑解惑。在自旋锁中 另有三种常见的锁形式:TicketLock ，CLHlock 和MCSlock。
## TicketLock
```java
 	package com.alipay.titan.dcc.dal.entity; 	 
 	import java.util.concurrent.atomic.AtomicInteger; 	 
 	public class TicketLock {
 	    private AtomicInteger serviceNum = new AtomicInteger();
 	    private AtomicInteger ticketNum  = new AtomicInteger();
 	    private static final ThreadLocal<Integer> LOCAL      = new ThreadLocal<Integer>();
 	    public void lock() {
 	        int myticket = ticketNum.getAndIncrement();
 	        LOCAL.set(myticket);
 	        while (myticket != serviceNum.get()) {
 	        }
 	    }
 	 
 	    public void unlock() {
 	        int myticket = LOCAL.get();
 	        serviceNum.compareAndSet(myticket, myticket + 1);
 	    }
 	}
```
每次都要查询一个serviceNum 服务号，影响性能（必须要到主内存读取，并阻止其他cpu修改）。