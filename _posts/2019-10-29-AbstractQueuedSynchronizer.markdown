---
layout: post
title:  "AbstractQueuedSynchronizer 原理分析 - 独占/共享模式"
date:   2019-10-29 22:34:50 +0800
categories: java
---
# 简介
AbstractQueuedSynchronizer （抽象队列同步器，以下简称 AQS）出现在 JDK 1.5 中，由大师 Doug Lea 所创作。AQS 是很多同步器的基础框架，比如 ReentrantLock、CountDownLatch 和 Semaphore 等都是基于 AQS 实现的。除此之外，我们还可以基于 AQS，定制出我们所需要的同步器。

AQS 的使用方式通常都是通过内部类继承 AQS 实现同步功能，通过继承 AQS，可以简化同步器的实现。如前面所说，AQS 是很多同步器实现的基础框架。弄懂 AQS 对理解 Java 并发包里的组件大有裨益，这也是我学习 AQS 并写出这篇文章的缘由。另外，需要说明的是，AQS 本身并不是很好理解，细节很多。在看的过程中药有一定的耐心，做好看多遍的准备。好了，其他的就不多说了，开始进入正题吧。

# 原理概述
在 AQS 内部，通过维护一个FIFO 队列来管理多线程的排队工作。在公平竞争的情况下，无法获取同步状态的线程将会被封装成一个节点，置于队列尾部。入队的线程将会通过自旋的方式获取同步状态，若在有限次的尝试后，仍未获取成功，线程则会被阻塞住。大致示意图如下：



当头结点释放同步状态后，且后继节点对应的线程被阻塞，此时头结点

线程将会去唤醒后继节点线程。后继节点线程恢复运行并获取同步状态后，会将旧的头结点从队列中移除，并将自己设为头结点。大致示意图如下：

# 重要方法介绍
本节将介绍三组重要的方法，通过使用这三组方法即可实现一个同步组件。

第一组方法是用于访问/设置同步状态的，如下：
| 方法        | 说明           | 
| ------------- |:-------------:|
| int getState()     | 获取同步状态 |
| void setState()     | 设置同步状态      |
| boolean compareAndSetState(int expect, int update) | 通过 CAS 设置同步状态      |

# 源码分析
## 节点结构
在并发的情况下，AQS 会将未获取同步状态的线程将会封装成节点，并将其放入同步队列尾部。同步队列中的节点除了要保存线程，还要保存等待状态。不管是独占式还是共享式，在获取状态失败时都会用到节点类。所以这里我们要先看一下节点类的实现，为后面的源码分析进行简单铺垫。源码如下：

```java
static final class Node {

    /** 共享类型节点，标记节点在共享模式下等待 */
    static final Node SHARED = new Node();
    
    /** 独占类型节点，标记节点在独占模式下等待 */
    static final Node EXCLUSIVE = null;

    /** 等待状态 - 取消 */
    static final int CANCELLED =  1;
    
    /** 
     * 等待状态 - 通知。某个节点是处于该状态，当该节点释放同步状态后，
     * 会通知后继节点线程，使之可以恢复运行 
     */
    static final int SIGNAL    = -1;
    
    /** 等待状态 - 条件等待。表明节点等待在 Condition 上 */
    static final int CONDITION = -2;
    
    /**
     * 等待状态 - 传播。表示无条件向后传播唤醒动作，详细分析请看第五章
     */
    static final int PROPAGATE = -3;

    /**
     * 等待状态，取值如下：
     *   SIGNAL,
     *   CANCELLED,
     *   CONDITION,
     *   PROPAGATE,
     *   0
     * 
     * 初始情况下，waitStatus = 0
     */
    volatile int waitStatus;

    /**
     * 前驱节点
     */
    volatile Node prev;

    /**
     * 后继节点
     */
    volatile Node next;

    /**
     * 对应的线程
     */
    volatile Thread thread;

    /**
     * 下一个等待节点，用在 ConditionObject 中
     */
    Node nextWaiter;

    /**
     * 判断节点是否是共享节点
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * 获取前驱节点
     */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    /** addWaiter 方法会调用该构造方法 */
    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }

    /** Condition 中会用到此构造方法 */
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```
