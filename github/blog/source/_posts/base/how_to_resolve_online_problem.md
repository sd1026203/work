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
```bash
jstack -l pid > jstack.log  线上机器需要切换至有权限用户
```
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