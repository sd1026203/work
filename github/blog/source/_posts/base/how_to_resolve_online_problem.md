---
title: 一些线上问题的排查方法
tags: java基础
categories: "java基础"
date: 2017-05-05
filename: base/how_to_resolve_online_problem.md
---
# 工具简介
## jstack
### 用法
		jstack -l pid > jstack.log  线上机器需要切换至有权限用户
		
### 线程信息中几种状态含义
1. RUNNABLE 状态是线程正在正常运行中, 当然可能会有某种耗时计算/IO等待的操作/CPU时间片切换等, 这个状态下发生的等待一般是其他系统资源, 而不是锁.
2. BLOCKED  这个状态下, 是在多个线程有同步操作的场景, 比如正在等待另一个线程的synchronized 块的执行释放, 或者可重入的 synchronized块里别人调用wait() 方法, 也就是这里是线程在等待进入临界区
3. WAITING（线 程池中从BlockingQueue.take()方法处常见）  这个状态下是指线程拥有了某个锁之后, 调用了他的wait方法, 等待其他线程/锁拥有者调用 notify / notifyAll 一遍该线程可以继续下一步操作, 这里要区分 BLOCKED 和 WATING 的区别, 一个是在临界点外面等待进入, 一个是在临界点里面wait等待别人notify, 线程调用了join方法 join了另外的线程的时候, 也会进入WAITING状态, 等待被他join的线程执行结束
4. TIMED_WAITING  这个状态就是有限的(时间限制)的WAITING, 一般出现在调用wait(long), join(long)等情况下, 另外一个线程sleep后, 也会进入TIMED_WAITING状态
5. TERMINATED 这个状态下表示 该线程的run方法已经执行完毕了, 基本上就等于死亡了(当时如果线程被持久持有, 可能不会被回收)

### waiting状态及括号内容说明

1. sleeping: sleep状态, 调用了sleep()方法可进入此状态;
2. parking: 线程挂起, 可能是在等待其他线程唤醒;
3. on object monitor ： object.wait()方法可使线程进入这个状态

### 线程基本状态信息	

1. waiting for monitor entry, 线程在等待进入临界区;一般此时线程状态都是Blocked
2. waiting on condition, 等待另一个条件来唤醒自己; 此时线程状态一般为WAITING(parking)或者TIMED_WAITING(parking或者sleeping).
3. in Object.wait(), 获取monitor之后, 又调用了wait()方法.此时线程状态一般为WAITING(on object monitor)或者TIMED_WAITING(on object monitor);

### 补充说明
使用时可以多dump几份，如果都指向同一个问题，才可以确定。查看线程所处的状态，如果某一批线程最新所在栈帧都是同一个方法的栈帧，或者都再等待同一个锁，则需要关注改方法的性能，以及过度使用锁同步的可能

## jmap
### 用法
		jmap -dump:live,format=b,file=someFile pid
		将someFile 拷贝至本机（推荐）
		jhat someFile 直至加在完毕
		访问localhost:7000查看信息

### 补充说明
jmap具有停止服务, 强制执行gc等功能, 对线上服务慎用.

## netstat
### 用法		
		netstat -anp | grep pid | sort -k5或
		netstat -ap | grep pid |sort -k5
		区别：-n打印的是ip，不加的-n打印机器名
		sort -k5是按照所连接的其他机器ip排序，从而达到将相同ip聚合到一起便于观察的目的

### 打印信息含义
具体请参考 [链接](http://11376164.blog.51cto.com/11366164/1795032)

1. ESTABLISHED：(Connection established.)代表一个打开的连接
2. TIME-WAIT：(In 2 MSL (twice the maximum segment length) quiet wait after close. )等待足够的时间以确保远程TCP接收到连接中断请求的确认 — 常见于客户端
3. CLOSE-WAIT：等待从本地用户发来的连接中断请求，常见于服务端

  