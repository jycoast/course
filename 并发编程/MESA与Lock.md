# synchronized 与 Lock

前面我们提到过在并发编程领域，有两大核心问题：一个是**互斥**，即同一时刻只允许一个线程访问共享资源；另一个是**同步**，即线程之间如何通信、协作。这两大问题，管程都是能够解决的。**Java SDK 并发包通过 Lock 和 Condition 两个接口来实现管程，其中 Lock 用于解决互斥问题，Condition 用于解决同步问题**。

Java 语言本身提供的 synchronized 也是管程的一种实现，既然 Java 从语言层面已经实现了管程了，那为什么还要在 SDK 里提供另外一种实现呢？

## 再造管程的理由

对于死锁问题，可以采用**破坏不可抢占条件**方案，即占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。

但synchronized 没有办法解决。原因是 synchronized 申请资源的时候，如果申请不到，线程直接进入阻塞状态了，而线程进入阻塞状态，啥都干不了，也释放不了线程已经占有的资源。

如果我们重新设计一把互斥锁去解决这个问题，那该怎么设计呢？

**能够响应中断**。synchronized 的问题是，持有锁 A 后，如果尝试获取锁 B 失败，那么线程就进入阻塞状态，一旦发生死锁，就没有任何机会来唤醒阻塞的线程。但如果阻塞状态的线程能够响应中断信号，也就是说当我们给阻塞的线程发送中断信号的时候，能够唤醒它，那它就有机会释放曾经持有的锁 A。这样就破坏了不可抢占条件了。

**支持超时**。如果线程在一段时间之内没有获取到锁，不是进入阻塞状态，而是返回一个错误，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。

**非阻塞地获取锁**。如果尝试获取锁失败，并不进入阻塞状态，而是直接返回，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件。

这三种方案可以全面弥补 synchronized 的问题。

```java
// 支持中断的API
void lockInterruptibly() throws InterruptedException;

// 支持超时的API
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

// 支持非阻塞获取锁的API
boolean tryLock();

```

# MESA与Lock

![image-20250424154532935](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250424154532935.png)

# AQS 源码探究

## CLH队列

![image.png](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/171d2d42c77b2765~tplv-t2oaga2asx-image.image)

## 加锁过程

如果同时有**三个线程**并发抢占锁，此时**线程一**抢占锁成功，**线程二**和**线程三**抢占锁失败，具体执行流程如下：

![image.png](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/171d2d42c5d2eb2c~tplv-t2oaga2asx-image.image)

那么此时AQS内部的数据是：

![image.png](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/171d2d42c8e5e268~tplv-t2oaga2asx-image.image)

## 解锁过程

![image.png](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/171d2d4332df6fd5~tplv-t2oaga2asx-image.image)

执行完后等待队列数据如下：

![image.png](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/171d2d434393c58f~tplv-t2oaga2asx-image.image)

# 思考题

1. 死锁是什么，使用synchronized 写一个会发生死锁的例子
2. Lock是如何保证可见性的？