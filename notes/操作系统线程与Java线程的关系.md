---
title: 操作系统线程和 Java 线程到底是什么关系？
date: 2026-01-17 18:30:00
tags:
  - 操作系统
  - Java
  - 线程模型
categories:
  - 后端开发
---

# 操作系统线程和 Java 线程到底是什么关系？

> 之前一直说"Java 线程就是操作系统线程"，但具体怎么对应的、底层怎么实现的，其实没太深究过。这次从线程模型的演变、HotSpot 的实现机制，一直到 Java 21 的虚拟线程，把这条线完整梳理了一遍。

## 先聊操作系统层面的线程

操作系统里的线程分两种：

**内核线程（Kernel-Level Thread）**：由操作系统内核直接管理和调度的线程。创建、销毁、切换都需要通过系统调用陷入内核态，开销比较大，但能被操作系统感知到，可以分配到不同的 CPU 核心上真正并行执行。

**用户线程（User-Level Thread）**：在用户空间由应用程序自己管理的线程，内核完全不知道它的存在。优点是创建和切换极快（不需要进内核），缺点是内核把整个进程当一个调度单元，一个用户线程阻塞了，整个进程都会被阻塞。

而用户线程和内核线程之间的对应关系，就产生了三种经典的线程模型：

| 模型 | 映射关系 | 特点 |
|------|---------|------|
| 1:1 | 一个用户线程对应一个内核线程 | 能真正并行，但创建开销大 |
| M:1 | 多个用户线程映射到一个内核线程 | 轻量快速，但无法利用多核 |
| M:N | 多个用户线程映射到多个内核线程 | 兼顾两者，但实现复杂 |

> 搞清楚这三种模型之后，Java 线程的演变脉络就很清晰了。

## Java 线程的三个时代

### 第一阶段：绿色线程（Green Threads）— M:1 模型

Java 1.1 的时候，线程是 JVM 自己实现的，叫做**绿色线程**（Green Threads）。这个名字来自 Sun 公司的 "Green Team" 项目组。

这个阶段的架构大概是这样的：

```
Java 线程 1
Java 线程 2     JVM 调度   ← 全部运行在用户空间
Java 线程 3
Java 线程 4
  ↓
唯一的一个内核线程             ← OS 只看到一个线程
```

所有 Java 线程跑在同一个内核线程上，由 JVM 自己做调度切换。

**问题很明显**：操作系统觉得这就是个单线程程序，只会分配一个 CPU 核心给它。哪怕你有 8 核 CPU，Java 程序也只能用一个核。而且一个线程做了阻塞 I/O，整个进程都卡住。

> 所以绿色线程在 Java 1.2/1.3 的时候就被抛弃了。不过理解这段历史对理解后面的虚拟线程很有帮助。

### 第二阶段：原生线程（Native Threads）— 1:1 模型

从 Java 1.2 开始（在 Solaris 上更早），Java 线程切换到了**原生线程**实现。这也是目前绝大多数 JVM 使用的模型。

**核心结论：现代 Java 中，每个 `java.lang.Thread` 都对应一个操作系统的内核线程。**

在 Linux 上就是 pthread，在 Windows 上就是 Windows Thread。

```
java.lang.Thread
    ↓
JavaThread（HotSpot C++ 对象）
    ↓
OSThread（HotSpot C++ 对象）
    ↓
pthread_create() / CreateThread()   ← 系统调用
    ↓
操作系统内核线程
```

### HotSpot 内部具体怎么做的？

当你在 Java 里 `new Thread().start()` 的时候，HotSpot 内部大致发生了这些事：

```java
// Java 层面
Thread t = new Thread(() -> {
    System.out.println("Hello from thread");
});
t.start(); // 这里会调用 native 方法
```

`start()` 方法最终会调到 JVM 的 native 代码，流程是：

1. **创建 JavaThread 对象**（C++ 层面）：准备线程本地存储（TLS）、分配缓冲区、同步对象、程序计数器
2. **创建 OSThread 对象**：封装操作系统线程相关的信息
3. **调用 `pthread_create()`**（Linux）或 `CreateThread()`（Windows）：创建真正的内核线程
4. **内核线程启动后回调 `run()` 方法**：最终执行你在 Java 里写的 Runnable

用 `jstack` 做线程 dump 的时候可以看到每个 Java 线程的 `nid`（native thread id），这就是操作系统层面的线程 ID：

```
"main" #1 prio=5 os_prio=0 tid=0x00007f4a8c00a800 nid=0x1a03 runnable
                                                     ↑
                                            这就是 OS 线程 ID
```

在 Linux 上可以直接用 `top -H -p <pid>` 看到这些线程，跟 jstack 里的 nid 能对上。

> 这就是为什么说 Java 线程本质上就是操作系统线程——不是"模拟"的，是真的一一对应的内核线程。线程的调度、优先级管理、上下文切换，全都是操作系统在做，JVM 不自己调度。

### 1:1 模型的代价

既然是真正的内核线程，那每个 Java 线程的开销就不小：

- **内存开销**：每个线程默认分配约 1MB 的栈空间（通过 `-Xss` 可调），再加上内核的数据结构，一个线程大概要占 1~2MB
- **创建开销**：需要系统调用进入内核态，大约 100~300 微秒
- **切换开销**：上下文切换需要保存/恢复寄存器、刷新 TLB 等
- **数量限制**：受操作系统限制，通常几千到几万个线程就是极限了

```java
// 试一下创建大量线程，很快就会爆
public class ThreadLimit {
    public static void main(String[] args) {
        int count = 0;
        try {
            while (true) {
                new Thread(() -> {
                    try { Thread.sleep(Long.MAX_VALUE); } catch (Exception e) {}
                }).start();
                count++;
            }
        } catch (OutOfMemoryError e) {
            System.out.println("最多创建了 " + count + " 个线程");
            // 通常在几千到一两万之间
        }
    }
}
```

这就是为什么传统 Java Web 服务器（比如 Tomcat）要用线程池——线程太贵了，不能每个请求都创建一个。

### 第三阶段：虚拟线程（Virtual Threads）— M:N 模型

Java 21 引入了**虚拟线程**（Virtual Threads，Project Loom），这是一次很大的变化。

你可以把它理解为"有了现代技术加持的绿色线程"——又回到了 JVM 自己管理线程，但这次解决了绿色线程的痛点。

架构是这样的：

```
虚拟线程（可以有几百万个）                    ← JVM 调度
  VT1  VT2  VT3  VT4  VT5 ... VTn
   ↓         ↓         ↓    （挂载/卸载）
载体线程 / Carrier Threads（ForkJoinPool）    ← 少量平台线程
  CT1       CT2       CT3       CT4
   ↓         ↓         ↓         ↓
操作系统内核线程                               ← 1:1 对应载体线程
  KT1       KT2       KT3       KT4
```

核心思路就是：**虚拟线程不直接绑定内核线程**。JVM 维护一个小的平台线程池（载体线程），虚拟线程被"挂载"到载体线程上执行，遇到阻塞操作时自动"卸载"，让载体线程去执行其他虚拟线程。

```java
// Java 21 创建虚拟线程，就这么简单
Thread vt = Thread.ofVirtual().start(() -> {
    System.out.println("我是虚拟线程");
});

// 或者用 Executors
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // 轻松创建 10 万个虚拟线程，毫无压力
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return "done";
        });
    }
}
```

虚拟线程的关键特性：
- 每个虚拟线程只占约 200 字节（对比平台线程的 1~2MB）
- 可以轻松创建百万级虚拟线程
- 遇到 I/O 阻塞会自动挂起，不占用载体线程
- 但 CPU 密集型任务没有优势（因为最终还是要占着载体线程跑）

> 有个需要注意的坑：`synchronized` 代码块会导致虚拟线程被"钉住"（pinning）在载体线程上，阻塞时无法卸载。Java 24 修复了这个问题。如果用的是 Java 21，建议用 `ReentrantLock` 替代 `synchronized`。

## 把三个时代放在一起看

| 时代 | 版本 | 模型 | Java 线程对应 | 调度者 |
|------|------|------|-------------|--------|
| 绿色线程 | 1.1 | M:1 | 多个 → 1 个内核线程 | JVM |
| 原生线程 | 1.2~20 | 1:1 | 1 个 → 1 个内核线程 | 操作系统 |
| 虚拟线程 | 21+ | M:N | 多个 → N 个载体线程 → N 个内核线程 | JVM + 操作系统 |

> 从 M:1 → 1:1 → M:N，Java 的线程模型走了一个螺旋上升的过程。绿色线程的理念没有错，但当时技术不成熟；虚拟线程在 20 多年后用更成熟的方式实现了同样的目标。

## 面试答题思路

如果被问到"操作系统线程和 Java 线程的关系"，我觉得可以这样组织：

1. **先说当前主流**：现代 JVM（HotSpot）采用 1:1 线程模型，每个 `java.lang.Thread` 对应一个操作系统内核线程（Linux 上是 pthread，Windows 上是 Win32 Thread）
2. **说一下实现细节**：`Thread.start()` 底层通过 JNI 调用 `pthread_create()` 创建内核线程，线程的调度由操作系统完成
3. **提历史演变**：早期用过绿色线程（M:1），无法利用多核被淘汰
4. **聊最新发展**：Java 21 引入虚拟线程（M:N），JVM 自己调度，一个虚拟线程不再独占一个内核线程
5. **加分项**：说一下 1:1 模型的代价（内存、创建开销、数量限制），自然引出虚拟线程要解决的问题

## 小结

理清这条线之后，很多之前零散的知识点就串起来了——为什么创建线程有开销、为什么要用线程池、为什么虚拟线程是 Java 并发的重大升级。核心就一句话：**目前主流 JVM 里 Java 线程就是操作系统线程（1:1），但虚拟线程正在改变这个局面。**
