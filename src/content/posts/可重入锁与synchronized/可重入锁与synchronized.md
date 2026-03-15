---
title: 什么是可重入锁？synchronized 是可重入的吗？
description: "刚学多线程的时候一直在用 synchronized，但从来没想过一个问题：如果一个 synchronized 方法调了另一个 synchronized 方法，同一个线程要拿两次锁，会不会把自己锁死？答案是不会——因为 synchron..."
published: 2026-01-22
tags:
    - Java
    - 并发编程
    - 可重入锁
category:
    - 后端开发
draft: false
---
> 刚学多线程的时候一直在用 synchronized，但从来没想过一个问题：如果一个 synchronized 方法调了另一个 synchronized 方法，同一个线程要拿两次锁，会不会把自己锁死？答案是不会——因为 synchronized 是**可重入锁**。但"可重入"到底是怎么实现的？底层靠什么机制？这次把这个问题完整捋一遍。

## 什么是可重入锁

可重入锁（Reentrant Lock），就是**同一个线程可以多次获取同一把锁，不会把自己死锁**。

先看一个最直观的场景：

```java
class Reentrant {
    public synchronized void a() {
        b();  // 在持有锁的情况下，又去调另一个需要同一把锁的方法
        System.out.println("here I am, in a()");
    }
    public synchronized void b() {
        System.out.println("here I am, in b()");
    }
}
```

调用 `a()` 的时候，当前线程获取了 this 对象的锁。然后 `a()` 内部调了 `b()`，而 `b()` 也是 synchronized 的，也需要 this 的锁。

如果锁**不可重入**，线程就会卡在 `b()` 的入口——因为锁已经被自己持有了，它等着自己释放锁才能进去，但它又必须进去才能继续执行然后释放锁...经典死锁，自己锁死自己。

但实际运行结果是正常的：
```
here I am, in b()
here I am, in a()
```

> 所以可重入的含义就是：**线程拿着锁的时候，再次请求同一把锁，能直接获取到，不会阻塞**。这在面向对象编程里非常重要，因为方法互相调用太常见了。

## 如果没有可重入会怎样？

假设一个不可重入的锁（伪代码）：

```java
class NonReentrantLock {
    private boolean isLocked = false;

    public synchronized void lock() throws InterruptedException {
        while (isLocked) {
            wait();  // 已经被锁了就等
        }
        isLocked = true;
    }

    public synchronized void unlock() {
        isLocked = false;
        notify();
    }
}
```

这个锁只记录了"有没有被锁"，不关心是谁锁的。如果同一个线程拿着锁再 `lock()` 一次，`isLocked` 已经是 true，直接 `wait()` 了——死锁。

可重入锁需要额外记录两个东西：**谁持有锁** + **重入了几次**。

```java
class SimpleReentrantLock {
    private boolean isLocked = false;
    private Thread lockedBy = null;   // 记录持有者
    private int holdCount = 0;         // 记录重入次数

    public synchronized void lock() throws InterruptedException {
        Thread current = Thread.currentThread();
        while (isLocked && lockedBy != current) {
            wait();  // 别的线程持有，才需要等
        }
        isLocked = true;
        lockedBy = current;
        holdCount++;  // 每次获取，计数 +1
    }

    public synchronized void unlock() {
        if (Thread.currentThread() == lockedBy) {
            holdCount--;  // 每次释放，计数 -1
            if (holdCount == 0) {
                isLocked = false;
                lockedBy = null;
                notify();  // 计数到 0 才真正释放
            }
        }
    }
}
```

> 关键逻辑就是：获取锁时判断"是不是同一个线程"，如果是就直接放行并把计数器加 1；释放锁时计数器减 1，减到 0 才真正释放。

## synchronized 的可重入机制 — 从字节码到 Monitor

回到正题：**synchronized 是可重入锁**。它的可重入机制是 JVM 在 Monitor（监视器）层面实现的。

### 字节码层面

synchronized 代码块在编译后会生成 `monitorenter` 和 `monitorexit` 两条字节码指令：

```java
public void method() {
    synchronized (this) {
        // 业务代码
    }
}
```

编译后的字节码（简化版）：

```
 0: aload_0               // 把 this 压栈
 1: monitorenter          // 获取 this 的 monitor 锁
 2: // ... 业务代码 ...
 5: aload_0
 6: monitorexit           // 正常退出时释放锁
 7: goto 15
 8: astore_1              // 异常处理开始
 9: aload_0
10: monitorexit           // 异常退出时也要释放锁！
11: aload_1
12: athrow                // 重新抛出异常
15: return
```

注意到了吗——**`monitorexit` 出现了两次**！一次是正常退出时释放，一次是在异常处理里释放。编译器自动加了一个类似 `finally` 的逻辑，保证锁一定会被释放，不会因为异常导致死锁。

（这也是 synchronized 比手动 lock/unlock 更安全的原因之一——不用担心忘了释放锁）

对于同步方法，不会有显式的 monitorenter/monitorexit，而是在方法的访问标志里加了 `ACC_SYNCHRONIZED`：

```java
// 普通同步方法：JVM 看到 ACC_SYNCHRONIZED 标志
// 自动在方法进入时 monitorenter this
// 方法退出时 monitorexit this
public synchronized void doWork() { ... }

// 静态同步方法：
// monitorenter 的是 Class 对象
public static synchronized void doStaticWork() { ... }
```

### Monitor 的计数器机制

JVM 规范是这么描述 `monitorenter` 的：

> 如果当前线程已经是该 monitor 的持有者，**进入计数器加 1**；如果 monitor 没有被持有，当前线程成为持有者，计数器设为 1；如果被其他线程持有，当前线程阻塞等待。

`monitorexit` 的逻辑：

> 计数器减 1。**当计数器为 0 时，释放 monitor**，其他等待的线程才有机会获取。

用一个流程来理解：

```
线程 A 调用 synchronized a() 方法：
  → monitorenter → 计数器 = 1（线程 A 持有）

  a() 内部调用 synchronized b() 方法：
    → monitorenter → 发现持有者就是自己 → 计数器 = 2

    b() 执行完毕：
    → monitorexit → 计数器 = 1（还没释放！）

  a() 继续执行，执行完毕：
  → monitorexit → 计数器 = 0 → 锁真正释放！
```

> 这就是可重入的底层实现——**每个 monitor 维护一个持有线程和一个计数器**，进一次加一、出一次减一，减到零才释放。

### 对象头里的体现

上一篇笔记说过，锁信息存在对象头的 Mark Word 里。在重量级锁状态下，Mark Word 指向一个 Monitor 对象，这个 Monitor 对象里就有：

```
ObjectMonitor（C++ 结构，简化版）
  _owner       // 持有锁的线程
  _count       // 重入计数器
  _WaitSet     // wait() 的线程队列
  _EntryList   // 阻塞等待锁的线程队列
  _recursions  // 重入次数
```

HotSpot 源码中 `ObjectMonitor` 有个 `_recursions` 字段，专门记录重入次数。每次同一个线程获取锁，`_recursions++`；每次释放，`_recursions--`。

## ReentrantLock — 另一种可重入锁

Java 里还有一个显式的可重入锁：`java.util.concurrent.locks.ReentrantLock`。名字里就带着 "Reentrant"。

```java
private final ReentrantLock lock = new ReentrantLock();

public void a() {
    lock.lock();
    try {
        System.out.println("a() - hold count: " + lock.getHoldCount());
        b();  // 重入
    } finally {
        lock.unlock();
    }
}

public void b() {
    lock.lock();
    try {
        System.out.println("b() - hold count: " + lock.getHoldCount());
    } finally {
        lock.unlock();  // 每次 lock 必须对应一次 unlock
    }
}
```

输出：
```
a() - hold count: 1
b() - hold count: 2    ← 重入后计数变成 2
```

ReentrantLock 内部基于 AQS（AbstractQueuedSynchronizer），用一个 `state` 字段记录重入次数，原理跟 Monitor 的计数器是一样的。

### synchronized vs ReentrantLock 对比

两者都是可重入的，但功能上有差异：

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 可重入 | 是 | 是 |
| 锁释放 | 自动（出代码块/方法就释放） | 手动（必须 `unlock()`） |
| 可中断等待 | 不支持 | `lockInterruptibly()` |
| 超时获取 | 不支持 | `tryLock(timeout)` |
| 公平锁 | 不支持（非公平） | 构造参数可选公平/非公平 |
| 多条件等待 | 只有 `wait/notify` | 可创建多个 `Condition` |
| 异常安全 | JVM 保证释放 | 必须 `try-finally` 手动释放 |
| 跨方法持有 | 不行（块级作用域） | 可以（`lock()` 和 `unlock()` 在不同方法） |

> 一句话：synchronized 胜在简单安全，ReentrantLock 胜在灵活强大。能用 synchronized 搞定的就别上 ReentrantLock，除非你需要它的高级特性。

## 可重入在实际场景中的意义

可重入不只是个理论概念，实际代码里到处都依赖它：

**场景一：递归调用**
```java
public synchronized int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);  // 递归重入同一把锁
}
```

**场景二：同步方法互相调用**
```java
public synchronized void save(User user) {
    validate(user);    // 调另一个同步方法
    doPersist(user);
}

public synchronized void validate(User user) {
    // 校验逻辑
}
```

**场景三：父子类方法调用**
```java
class Parent {
    public synchronized void doWork() {
        System.out.println("parent");
    }
}

class Child extends Parent {
    public synchronized void doWork() {
        super.doWork();  // 子类拿着锁，调父类的同步方法，也需要重入
        System.out.println("child");
    }
}
```

如果锁不可重入，上面这些代码全都会死锁。

## 小结

可重入锁的核心就是：**同一线程可以反复获取同一把锁，通过计数器跟踪重入次数，减到零才真正释放**。synchronized 天然就是可重入的，JVM 在 Monitor 层面通过 `_recursions` 计数器实现；ReentrantLock 通过 AQS 的 `state` 字段实现，原理一样。理解了这个机制，很多之前觉得"理所当然"的代码就知道为什么能正常跑了。
