---
title: synchronized 和 ReentrantLock 区别是什么？代码对比着看就明白了
date: 2026-01-25 14:00:00
tags:
  - Java
  - 并发编程
  - ReentrantLock
categories:
  - 后端开发
---

# synchronized 和 ReentrantLock 区别是什么？代码对比着看就明白了

> 上一篇笔记里简单对比过这两个，但那篇重点在可重入机制。这次想把它们的差异完整展开来说，每个差异都用代码跑一遍，这样记得更牢。

## 先说相同点

别光顾着看区别，先明确它们有什么一样的：

- 都是**可重入锁**——同一线程可以重复获取同一把锁
- 都能实现**互斥**——同一时刻只有一个线程能进入临界区
- 都保证**内存可见性**——加锁/解锁会触发 happens-before 关系

理解了共同点，再看差异就清楚了——ReentrantLock 是在 synchronized 的基础上增加了更多能力。

## 区别一：用法 — 隐式 vs 显式

这是最直观的区别。

```java
// synchronized — JVM 自动管理加锁解锁
public void syncMethod() {
    synchronized (this) {
        // 临界区
    }
    // 出了代码块自动释放，不用操心
}

// ReentrantLock — 必须手动 lock/unlock
private final ReentrantLock lock = new ReentrantLock();

public void lockMethod() {
    lock.lock();
    try {
        // 临界区
    } finally {
        lock.unlock();  // 忘了这行就完蛋了
    }
}
```

synchronized 出了代码块或方法就自动释放，哪怕抛了异常也没事——JVM 在字节码层面用 `monitorenter`/`monitorexit` 保证了这一点（编译器会自动插入异常处理的 `monitorexit`）。

ReentrantLock 就得你自己来了。**必须放在 try-finally 里**，`unlock()` 写在 finally 块中。忘了 unlock 会导致锁永远不释放，其他线程全部饿死——而且这种 bug 非常难查。

> 这一点上 synchronized 明显更安全。我之前就见过生产环境因为 ReentrantLock 忘了 unlock 导致线程全卡住的事故，排查了好久。

## 区别二：可中断 — lockInterruptibly()

这是 synchronized 完全做不到的事情。

```java
// synchronized — 一旦阻塞等锁，没法打断，只能死等
synchronized (lockObj) {
    // 如果拿不到锁，线程就一直 BLOCKED，你调 interrupt() 也没用
}

// ReentrantLock — 可以被中断
private final ReentrantLock lock = new ReentrantLock();

public void interruptibleMethod() throws InterruptedException {
    lock.lockInterruptibly();  // 等锁的过程中可以被 interrupt 打断
    try {
        // 临界区
    } finally {
        lock.unlock();
    }
}
```

跑个完整例子感受一下：

```java
ReentrantLock lock = new ReentrantLock();

// 线程 A 先把锁拿了不放
Thread threadA = new Thread(() -> {
    lock.lock();
    try {
        System.out.println("A 拿到锁了，开始摸鱼...");
        Thread.sleep(100000);  // 假装干了很久
    } catch (InterruptedException e) {
    } finally {
        lock.unlock();
    }
});

// 线程 B 用 lockInterruptibly 等锁
Thread threadB = new Thread(() -> {
    try {
        System.out.println("B 开始等锁...");
        lock.lockInterruptibly();  // 可中断地等待
        try {
            System.out.println("B 拿到锁了");
        } finally {
            lock.unlock();
        }
    } catch (InterruptedException e) {
        System.out.println("B 等不下去了，被中断了，去干别的");
    }
});

threadA.start();
Thread.sleep(100);  // 确保 A 先拿到锁
threadB.start();
Thread.sleep(2000);
threadB.interrupt();  // 2 秒后中断 B
```

输出：
```
A 拿到锁了，开始摸鱼...
B 开始等锁...
B 等不下去了，被中断了，去干别的
```

如果用 synchronized，线程 B 就只能一直等着，`interrupt()` 对处于 BLOCKED 状态的线程不生效。

## 区别三：超时获取 — tryLock()

synchronized 拿不到锁就**无限期阻塞**，没有"试一下拿不到就算了"的选项。

```java
ReentrantLock lock = new ReentrantLock();

// 方式一：尝试获取，拿不到立即返回 false
if (lock.tryLock()) {
    try {
        System.out.println("拿到了");
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("没拿到，我去干别的了");
}

// 方式二：最多等 3 秒
if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try {
        System.out.println("3 秒内拿到了");
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("等了 3 秒还没拿到，不等了");
}
```

> tryLock 最大的价值在于**避免死锁**。两个线程各持有一把锁再去拿对方的锁——经典死锁场景。用 tryLock 可以实现"拿不到就退让，释放已持有的锁，过一会儿再试"，从而打破死循环。synchronized 遇到这种场景基本无解（只能靠设计上保证加锁顺序一致）。

## 区别四：公平锁

synchronized 是**非公平锁**——锁释放后哪个线程能拿到完全看运气（取决于 OS 调度），有可能某个线程一直抢不到。

ReentrantLock 可以通过构造参数选择公平或非公平：

```java
// 非公平锁（默认）— 新来的线程可以插队
ReentrantLock unfairLock = new ReentrantLock();        // fair = false
ReentrantLock unfairLock2 = new ReentrantLock(false);  // 显式指定

// 公平锁 — 严格按等待顺序获取，先来先得
ReentrantLock fairLock = new ReentrantLock(true);
```

公平锁内部多了一步判断——获取锁之前先调 `hasQueuedPredecessors()` 看看队列里有没有排在前面的线程，有的话就乖乖排队。

```java
// AQS 中公平锁的获取逻辑（简化版）
protected final boolean tryAcquire(int acquires) {
    if (state == 0) {
        // 公平锁的关键：先检查有没有排在前面的线程
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(currentThread);
            return true;
        }
    }
    // ... 重入逻辑 ...
}

// 非公平锁的获取逻辑（简化版）
protected final boolean tryAcquire(int acquires) {
    if (state == 0) {
        // 非公平锁：不管队列，直接 CAS 抢
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(currentThread);
            return true;
        }
    }
    // ...
}
```

> 公平锁虽然看起来更"正义"，但吞吐量比非公平锁低不少——因为线程切换有开销。所以默认都是非公平的。只有在对顺序有严格要求的场景（比如银行排队）才用公平锁。

## 区别五：Condition 多条件等待

这个差异在实际编码中非常有用。

synchronized 只能用 `wait()` / `notify()` / `notifyAll()`，它们是绑定在**同一个对象**的 monitor 上的——也就是说只有一个等待队列。你 `notifyAll()` 的时候，所有等待的线程都会被唤醒，不管它们是因为什么条件在等。

ReentrantLock 可以创建**多个 Condition**，每个 Condition 有自己的等待队列，可以精准唤醒。

经典的生产者-消费者问题，对比一下：

**synchronized 版本：**

```java
class Buffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 10;

    public synchronized void put(int value) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();  // 满了就等，但等在同一个队列里
        }
        queue.offer(value);
        notifyAll();  // 只能唤醒所有人，生产者消费者一起醒
    }

    public synchronized int get() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();  // 空了就等，也在同一个队列里
        }
        int value = queue.poll();
        notifyAll();  // 还是唤醒所有人
        return value;
    }
}
```

问题在于 `notifyAll()` 是"大喇叭"——不管你是生产者还是消费者，全都醒过来抢锁。醒过来发现条件不满足又得 wait 回去，白白浪费 CPU。

**ReentrantLock + Condition 版本：**

```java
class Buffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 10;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();   // 单独的"没满"条件
    private final Condition notEmpty = lock.newCondition();  // 单独的"没空"条件

    public void put(int value) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // 满了？在"没满"条件上等
            }
            queue.offer(value);
            notEmpty.signal();   // 放了东西，通知"没空"条件上等的消费者
        } finally {
            lock.unlock();
        }
    }

    public int get() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // 空了？在"没空"条件上等
            }
            int value = queue.poll();
            notFull.signal();    // 取了东西，通知"没满"条件上等的生产者
            return value;
        } finally {
            lock.unlock();
        }
    }
}
```

> 生产者只唤醒消费者，消费者只唤醒生产者——精准通知，不浪费。这在高并发场景下性能差异还是挺明显的。

## 区别六：锁的作用域

```java
// synchronized — 必须在同一个代码块或方法内获取和释放
public void method() {
    synchronized (lock) {
        // 进来就锁，出去就放
    }
    // 不可能在方法 A 里 lock，在方法 B 里 unlock
}

// ReentrantLock — 可以跨方法
class CrossMethodLock {
    private final ReentrantLock lock = new ReentrantLock();

    public void begin() {
        lock.lock();  // 这里获取
        // 做一些事情...
    }

    public void end() {
        // 做一些事情...
        lock.unlock();  // 这里释放
    }
}
```

这个能力在某些框架级别的场景里挺有用的，比如在一个方法里获取锁、经过一系列回调后在另一个方法里释放。当然这也让代码更容易出错，所以一般不推荐。

## 区别七：底层实现

| | synchronized | ReentrantLock |
|---|---|---|
| 实现层面 | JVM 内置，`monitorenter`/`monitorexit` 字节码 | Java API 层面，基于 AQS |
| 锁优化 | JVM 自动做偏向锁/轻量级锁/自旋优化 | 没有这些优化，但有 CAS + CLH 队列 |
| 锁状态 | 对象头 Mark Word | AQS 的 `volatile int state` |
| 等待队列 | ObjectMonitor 的 EntryList/WaitSet | AQS 的 CLH 双向链表 |

synchronized 经过这么多年的 JVM 优化（偏向锁、自适应自旋、锁消除、锁粗化等），在低竞争场景下性能已经非常好了，跟 ReentrantLock 差距极小。

> 实际项目里，性能基本不构成选择依据。选哪个主要看你需不需要 ReentrantLock 的那些额外特性。

## 怎么选？一张图说清楚

```
需要锁吗？

  简单的互斥同步 → synchronized（优先选这个）
  需要尝试获取锁 / 超时等待？ → ReentrantLock.tryLock()
  需要等锁时可被中断？ → ReentrantLock.lockInterruptibly()
  需要公平排队？ → new ReentrantLock(true)
  需要多个等待条件（精准唤醒）？ → ReentrantLock + Condition
  需要跨方法持有锁？ → ReentrantLock
```

> 我的原则是：**能用 synchronized 就别上 ReentrantLock**。synchronized 更简洁、更安全、JVM 还帮你做优化。只有当 synchronized 满足不了需求（上面列的那些场景）的时候，才换 ReentrantLock。

## 小结

synchronized 是 JVM 的"亲儿子"，简单安全有优化；ReentrantLock 是 JUC 的"瑞士军刀"，功能强大但用起来要小心。核心差异就这几条：可中断、可超时、可公平、多条件。记住这几个关键词，面试也好实战也好就够用了。
