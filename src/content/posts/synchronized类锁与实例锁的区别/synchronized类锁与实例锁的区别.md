---
title: synchronized 锁 People.class 和锁 people 实例，到底差在哪？
description: "这个问题刚看到的时候觉得\"不就是锁的对象不一样嘛\"，但仔细想想，两者的行为差异其实挺大的，涉及到锁的粒度、作用范围、还有一些容易踩的坑。用代码跑了几个场景之后才真正理解透。"
published: 2026-01-20
tags:
    - Java
    - 并发编程
    - synchronized
category:
    - 后端开发
draft: false
---
> 这个问题刚看到的时候觉得"不就是锁的对象不一样嘛"，但仔细想想，两者的行为差异其实挺大的，涉及到锁的粒度、作用范围、还有一些容易踩的坑。用代码跑了几个场景之后才真正理解透。

## 先明确一个前提：synchronized 到底锁的是什么

synchronized 的本质是给一个**对象**加锁，这个对象叫做**监视器对象（Monitor Object）**。

不管你怎么写——同步方法、同步代码块——最终都是在某个对象上加锁。关键就在于：**锁的是哪个对象？**

```java
// 写法一：锁实例对象
synchronized (people) {
    // ...
}

// 写法二：锁 Class 对象
synchronized (People.class) {
    // ...
}
```

`people` 是一个 People 类的实例，堆上的一个普通对象。

`People.class` 是 People 这个类对应的 **Class 对象**，整个 JVM 里只有一个（同一个类加载器下）。

> 这就是根本区别：一个锁的是"某一个人"，一个锁的是"人类这个概念"。

## 用代码看区别

### 场景一：锁实例对象

```java
public class People {
    public void doSomething() {
        synchronized (this) {  // 锁的是当前实例
            System.out.println(Thread.currentThread().getName() + " 开始");
            try { Thread.sleep(2000); } catch (Exception e) {}
            System.out.println(Thread.currentThread().getName() + " 结束");
        }
    }
}

// 测试
People p1 = new People();
People p2 = new People();

new Thread(() -> p1.doSomething(), "线程A").start();
new Thread(() -> p1.doSomething(), "线程B").start();  // 同一个实例
new Thread(() -> p2.doSomething(), "线程C").start();  // 不同实例
```

运行结果：
```
线程A 开始
线程C 开始      ← C 和 A 同时跑！因为锁的是不同对象
线程A 结束
线程B 开始      ← B 要等 A 释放 p1 的锁
线程C 结束
线程B 结束
```

**线程 A 和线程 B** 竞争的是同一个对象 `p1` 的锁，所以互斥。
**线程 C** 锁的是 `p2`，跟 `p1` 的锁完全无关，所以可以同时跑。

（这里我刚开始搞混了一个点——以为是"同一个类的对象就会互斥"，其实不是，**锁的粒度是对象级别的**，不同实例的锁互不影响）

### 场景二：锁 Class 对象

```java
public class People {
    public void doSomething() {
        synchronized (People.class) {  // 锁的是 Class 对象
            System.out.println(Thread.currentThread().getName() + " 开始");
            try { Thread.sleep(2000); } catch (Exception e) {}
            System.out.println(Thread.currentThread().getName() + " 结束");
        }
    }
}

// 测试
People p1 = new People();
People p2 = new People();

new Thread(() -> p1.doSomething(), "线程A").start();
new Thread(() -> p1.doSomething(), "线程B").start();
new Thread(() -> p2.doSomething(), "线程C").start();
```

运行结果：
```
线程A 开始
线程A 结束
线程C 开始      ← 即使是不同实例，也得排队！
线程C 结束
线程B 开始
线程B 结束
```

三个线程全部串行执行。因为 `People.class` 在 JVM 里只有一个，不管你用哪个实例去调，锁的都是同一个 Class 对象。

> 一句话总结：**实例锁是"一把钥匙开一扇门"，类锁是"一把钥匙锁整栋楼"**。

## 跟 synchronized 方法的等价关系

理解了上面的区别，再看同步方法就很清晰了：

```java
// 普通同步方法 ←→ synchronized(this)
public synchronized void method1() { ... }
// 等价于
public void method1() {
    synchronized (this) { ... }
}

// 静态同步方法 ←→ synchronized(People.class)
public static synchronized void method2() { ... }
// 等价于
public static void method2() {
    synchronized (People.class) { ... }
}
```

所以当你看到 `static synchronized` 方法时，它锁的就是类的 Class 对象，跟 `synchronized(People.class)` 效果一样。

### 一个重要推论：实例锁和类锁互不干扰

```java
public class People {
    // 锁 this
    public synchronized void instanceMethod() {
        System.out.println("实例方法：" + Thread.currentThread().getName());
        try { Thread.sleep(2000); } catch (Exception e) {}
    }

    // 锁 People.class
    public static synchronized void classMethod() {
        System.out.println("静态方法：" + Thread.currentThread().getName());
        try { Thread.sleep(2000); } catch (Exception e) {}
    }
}

People p = new People();
new Thread(() -> p.instanceMethod(), "线程A").start();
new Thread(() -> People.classMethod(), "线程B").start();
```

结果：**两个线程同时执行**。因为一个锁的是实例 `p`，另一个锁的是 `People.class`，根本不是同一把锁。

> 这个点很容易被忽略。如果你在一个类里同时有实例同步方法和静态同步方法，它们之间是**不会互斥**的。

## 锁传进来的实例参数时要注意什么

题目说的是"锁一个传过来的 people 实例"，这种写法在实际代码里很常见：

```java
public void transfer(People from, People to, int amount) {
    synchronized (from) {
        synchronized (to) {
            // 转账逻辑
        }
    }
}
```

这种写法有几个坑：

### 坑一：传进来的引用可能变

```java
private People target;

public void doWork() {
    synchronized (target) {  // 如果 target 被重新赋值了呢？
        // ...
    }
}
```

如果 `target` 不是 final 的，在某些时机被其他线程改成了另一个对象，那后来的线程锁的就是新对象了——**两个线程锁在了不同对象上，根本没有互斥效果**。

> 所以最佳实践是：**synchronized 锁的对象引用应该是 final 的**。

```java
private final Object lock = new Object();  // 推荐
synchronized (lock) { ... }
```

### 坑二：锁 null 会空指针

```java
People p = null;
synchronized (p) { ... }  // 直接 NullPointerException！
```

synchronized 方法不会有这个问题（因为你能调方法说明对象不为 null），但 synchronized 代码块传入 null 就会炸。

### 坑三：嵌套锁不同实例可能死锁

```java
// 线程 1：synchronized(A) → synchronized(B)
// 线程 2：synchronized(B) → synchronized(A)
// 经典死锁场景
```

如果你锁传进来的两个实例，一定要保证**加锁顺序一致**，不然就是教科书级别的死锁。一个常见的解决方案是按对象的 hashCode 或 id 排序后再加锁。

## 底层机制：对象头里的锁信息

不管是锁实例还是锁 Class 对象，底层机制是一样的。每个 Java 对象（包括 Class 对象）在内存里都有一个**对象头（Object Header）**，里面有个叫 **Mark Word** 的区域，存的就是锁状态。

``` 
Object Header
  Mark Word              Class Pointer
  (锁状态/hashCode/GC 等)  (指向类元数据)
```

Mark Word 的内容会根据锁状态变化：

| 锁状态 | Mark Word 存储内容 | 标志位 |
|-------|-------------------|--------|
| 无锁 | hashCode、GC 分代年龄 | `01` |
| 偏向锁 | 持有锁的线程 ID | `01`（偏向位=1） |
| 轻量级锁 | 指向栈上 Lock Record 的指针 | `00` |
| 重量级锁 | 指向 Monitor 对象的指针 | `10` |

锁升级的过程：**无锁 → 偏向锁 → 轻量级锁 → 重量级锁**（只能升级不能降级）。

- **偏向锁**：只有一个线程用的时候，把线程 ID 写进 Mark Word，后续进入不需要 CAS。JDK 15 之后默认关闭了
- **轻量级锁**：有竞争了，用 CAS 自旋尝试获取
- **重量级锁**：竞争激烈，自旋拿不到就调操作系统的 Mutex，线程阻塞挂起

> 不管你 `synchronized(people)` 还是 `synchronized(People.class)`，走的都是这套机制。区别只是 Mark Word 属于哪个对象——实例的 Mark Word 还是 Class 对象的 Mark Word。`People.class` 本身也是个 Java 对象（`java.lang.Class` 的实例），所以它也有自己的对象头和 Mark Word。

## 怎么选？用实例锁还是类锁

| 场景 | 选择 | 原因 |
|------|------|------|
| 保护实例变量 | 锁实例（`this` 或实例引用） | 不同实例的变量互不影响，没必要全局串行 |
| 保护静态变量 | 锁 Class 对象 | 静态变量全局只有一份，必须全局互斥 |
| 需要跨实例互斥 | 锁 Class 对象或共享的锁对象 | 保证所有实例都排队 |
| 细粒度控制 | 锁独立的 `final Object lock` | 避免暴露 this 或 Class 对象被外部锁住 |

说个实际的例子——单例模式的双重检查锁：

```java
public class Singleton {
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {  // 这里必须锁 Class
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

这里锁 Class 对象是因为 `instance` 是静态变量，所有线程都可能访问，必须用类级别的锁来保护。如果锁某个实例，那不同线程用不同实例调，根本锁不住。

## 小结

回到最初的问题——`synchronized(People.class)` 和 `synchronized(people)` 的区别，核心就是**锁的粒度不同**。实例锁只管"自己这个对象"，不同实例之间不互斥；类锁管"整个类的所有实例"，全局只有一把。底层机制都是一样的对象头 Mark Word 那套东西，只是 Mark Word 属于不同的对象而已。选哪个取决于你要保护的数据是实例级别的还是类级别的。
