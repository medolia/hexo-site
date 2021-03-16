---
title: 你不好奇 ThreadPoolExecutor 构造器里的 keepAliveTime 是怎么生效的吗？
date: 2021/3/16 14:45:50
categories: 编程
tags:
  - 源码
excerpt: Java ThreadPoolExecutor 类源码节选
---

## 前言

jdk 版本： jdk11 （较于 jdk 8 无太大变动）
欲通过解读源码解决的疑问：线程池如何使构造器参数 timeout 生效。即，如何使得超过 corePoolSize 的工作线程在 timeout 规定的时间后自动回收呢？
读懂此文需要的基础：了解线程池基本的工作原理（大概了解源码的设计思想），了解阻塞队列。 

## 解读

### runWorker(Runnable)

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // 允许中断
        boolean completedAbruptly = true;
        try {
            // 只要有任务，就会一直循环而不进入销毁回收阶段
            while (task != null || (task = getTask()) != null) {
                // 任务运行中
                // ...
        } finally {
            // 销毁回收 Worker
            processWorkerExit(w, completedAbruptly);
        }
    }
```

runWorker(Runnable) 方法通过线程池内部封装任务的核心类 Worker 的 run() 调用，可以看到，每个 Worker 类首个任务通过 firstTask 引用得到，接下来的任务通过 getTask() 方法得到，只要 task 为 null（即该 Worker 已无任务分配），方法就会跳出 try 块内部的 while 循环，进入 finally 块销毁回收线程。

那么 getTask() 什么情况会返回 null 呢？往下看。

### getTask()

```java
// 只要这个方法返回 null，Worker 类就会被销毁回收
private Runnable getTask() {
        boolean timedOut = false; // 最近一次 poll() 方法是否超时？

        for (;;) {
            int c = ctl.get();
            
            // 返回 null 条件 1: 线程池处于 SHUTDOWN 状态且工作队列为空
            // 返回 null 条件 2: 线程池处于 STOP 状态
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())){
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // 是否为时限线程：允许核心线程超时（一般都不允许）或者工作线程数已经大于 corePoolSize
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            
            // 返回 null 条件 3: 工作线程数已经大于 maximumPoolSize
            // 返回 null 条件 4: 该线程为时限线程且上一次工作队列的 poll() 超时
            // 条件 3 4 判断为真且尝试减少线程数失败才会返回 null，否则会执行 continue 继续重新从条件 1 开始往下判断
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
            
            // 如果条件 1 - 4 均不成立，则该 Worker 可以继续从工作队列中拿任务做
            // 如果为时限线程，会执行阻塞队列的 poll(long, TimeUnit) 方法；
            // 如果为核心线程（timed 为 false），会执行
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

返回 null 的四个条件已经在代码注释中详细说明了。我们重点看条件 1-4 都判断不成功，worker 尝试从任务队列里拿任务（try catch 块）的逻辑。

当工作线程为时限线程（timed 为 true）时，会调用工作队列的 poll(long, TimeUnit) 方法，该方法会在倒计时结束后直接返回 null，也就意味着如果 Worker 没有在限定时间内获取到任务(r != null)，会导致 timeOut 被赋值为 true，接着进入下一次循环，然后满足条件 4 返回 null，导致 Worker 被销毁回收。

而当工作线程为核心线程（timed 为 false）时，将调用工作队列的 take() 方法，该方法会一直阻塞，直到可以得到一个非 null 值返回，如此保证每次都返回一个确定的任务，对应的 Worker 类一直呆在 try 块里的 while 循环中而不被销毁回收。

## 总结

一顿分析下来，时限线程(> corePoolSize && <= maximumPoolSize 的那部分)和核心线程（<= corePoolSize 的那部分）在线程池里完全收到两种待遇，核心线程只要线程池还在正常运行就永远不会被销毁回收（经典人上人），而时限线程每次必须在规定时间内从任务队列中拿到任务才能暂时逃离被回收的厄运，而且只要出现特殊情况（线程池关闭、工作线程数超限或规定时间内没拿到任务）时限线程就会被立刻回收（老资本家了）。

## 巨人的肩膀

[Java面试题精选【192期】面试官：线程池中多余的线程是如何回收的？](https://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247486399&idx=1&sn=01c5a16f71552306f70f923bbd1c3734&chksm=e80dbdc9df7a34df0143965e06c1eaebe6973c4a50bdd581bba4284fa1d260d81aa8a7de9036&scene=21#wechat_redirect)