---
title: 可重入锁
date: 2017-04-24
tags: [锁]
categories: "锁"
file: reentrantlock.md
---
本文中所说的可重入锁是广义上的, 不单指Java中的ReentrantLock.
可重入锁，也叫做递归锁，指的是同一线程 外层函数获得锁之后 ，内层递归函数仍然有获取该锁的代码，但不受影响。
在JAVA环境下 ReentrantLock 和synchronized 都是 可重入锁.
```java
public class Test implements Runnable{
	public synchronized void get(){
		System.out.println(Thread.currentThread().getId());
		set();
	}

	public synchronized void set(){
		System.out.println(Thread.currentThread().getId());
	}

	@Override
	public void run() {
		get();
	}
	public static void main(String[] args) {
		Test ss=new Test();
		new Thread(ss).start();
		new Thread(ss).start();
		new Thread(ss).start();
	}
}
```
```java
public class TestReentrantLock implements Runnable {
    ReentrantLock lock = new ReentrantLock();

    public void get() {
        lock.lock();
        System.out.println(Thread.currentThread().getId());
        set();
        lock.unlock();
    }

    public void set() {
        lock.lock();
        System.out.println(Thread.currentThread().getId());
        lock.unlock();
    }

    @Override
    public void run() {
        get();
    }

    public static void main(String[] args) {
        TestReentrantLock ss = new TestReentrantLock();
        new Thread(ss).start();
        new Thread(ss).start();
        new Thread(ss).start();
    }
}
```
两个例子最后的结果都是正确的，即 同一个线程id被连续输出两次。结果如下：

Threadid: 8
Threadid: 8
Threadid: 10
Threadid: 10
Threadid: 9
Threadid: 9

可重入锁最大的作用是避免死锁.

```java
public class SpinLock {
	private AtomicReference<Thread> owner =new AtomicReference<>();
	public void lock(){
		Thread current = Thread.currentThread();
		while(!owner.compareAndSet(null, current)){
		}
	}
	public void unlock (){
		Thread current = Thread.currentThread();
		owner.compareAndSet(current, null);
	}
}
```
对于自旋锁来说，
1、若有同一线程两调用lock() ，会导致第二次调用lock位置进行自旋，产生了死锁
说明这个锁并不是可重入的。（在lock函数内，应验证线程是否为已经获得锁的线程）
2、若1问题已经解决，当unlock（）第一次调用时，就已经将锁释放了。实际上不应释放锁。
（采用计数次进行统计）
修改之后，如下：
```java
public class SpinLock1 {
	private AtomicReference<Thread> owner =new AtomicReference<>();
	private int count =0;
	public void lock(){
		Thread current = Thread.currentThread();
		if(current==owner.get()) {
			count++;
			return ;
		}

		while(!owner.compareAndSet(null, current)){

		}
	}
	public void unlock (){
		Thread current = Thread.currentThread();
		if(current==owner.get()){
			if(count!=0){
				count--;
			}else{
				owner.compareAndSet(current, null);
			}

		}

	}
}
```
该自旋锁即为可重入锁。 