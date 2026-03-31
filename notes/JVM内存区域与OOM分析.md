---
title: JVM 内存区域与 OOM — 哪些区域会炸，程序计数器凭什么没事
date: 2026-01-12 21:30:00
tags:
  - JVM
  - 内存模型
  - OOM
categories:
  - 后端开发
---

# JVM 内存区域与 OOM — 哪些区域会炸，程序计数器凭什么没事

> 面试被问到"JVM 哪些区域会发生 OOM"，当时答得不太全。回来翻了一遍 JVM 规范和源码，把每个区域的 OOM 情况都捋了一遍

## 先回顾一下 JVM 运行时数据区

JVM 运行时内存大致分这么几块：

```
JVM 运行时数据区

线程私有             线程共享
  程序计数器           堆（Heap）
  虚拟机栈             方法区（Metaspace）
  本地方法栈

还有：直接内存（Direct Memory）
```

每个区域的职责不一样，能不能 OOM、什么情况下会 OOM，也不一样。下面一个一个来说。

## 程序计数器（PC Register）— 唯一不会 OOM 的区域

先回答标题里的问题：**程序计数器不会发生 OOM**。

它是 JVM 规范中明确规定的**唯一一个**没有 `OutOfMemoryError` 的内存区域。

为什么？因为程序计数器的作用就是记录当前线程执行到了哪条字节码指令，本质上就存了一个地址值。每个线程有一个，空间极小且固定，不存在"分配不到内存"的情况。

> 如果当前线程执行的是 Java 方法，PC 记录的是字节码指令的地址；如果是 native 方法，PC 的值是 undefined。这个在 JVM 规范里写得很清楚。

所以面试答这个问题的时候，**先把程序计数器拎出来说不会 OOM，再逐个说其他区域**，思路会清晰很多。

## 堆（Heap）— 最常见的 OOM 重灾区

堆是存对象实例的地方，也是 GC 管理的主要区域。几乎所有 `new` 出来的对象都在堆上分配。

当堆空间不够用、GC 也回收不出足够的空间时，就会抛：

```
java.lang.OutOfMemoryError: Java heap space
```

写个最简单的复现代码：

```java
// VM 参数：-Xms20m -Xmx20m
public class HeapOOM {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while (true) {
            // 每次分配 1MB
            list.add(new byte[1024 * 1024]);
        }
    }
}
```

跑一下就能看到 `Java heap space` 的报错。关键是 list 一直持有引用，GC 回收不掉，堆就撑爆了。

还有一种跟堆相关的：

```
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

这个意思是 GC 花了太多时间（默认超过 98% 的时间在做 GC），但每次只回收了不到 2% 的内存。JVM 觉得这样下去没意义了，直接报错。

> 实际项目里堆 OOM 是最常见的，一般原因就两类：一是堆确实配小了，加 `-Xmx` 就行；二是有内存泄漏，对象该释放的没释放。后者才是真正要排查的，用 MAT 分析 heap dump 是常规操作。

## 虚拟机栈（JVM Stack）— StackOverflow 和 OOM 都可能

虚拟机栈是线程私有的，每个方法调用会创建一个栈帧（Stack Frame），里面放局部变量表、操作数栈、方法出口等信息。

JVM 规范里说了，这个区域有**两种异常**：

**1. StackOverflowError** — 栈深度超限

```java
// 无限递归，很快就炸
public class StackSOF {
    private int depth = 0;

    public void recursion() {
        depth++;
        recursion();
    }

    public static void main(String[] args) {
        StackSOF sof = new StackSOF();
        try {
            sof.recursion();
        } catch (StackOverflowError e) {
            System.out.println("栈深度：" + sof.depth);
            // 默认配置下大概能到几千到一万多层
        }
    }
}
```

**2. OutOfMemoryError** — 栈扩展时申请不到内存

这个在实际中不太常见，但理论上如果 JVM 允许动态扩展栈（HotSpot 实际上不允许动态扩展，栈大小在线程创建时就固定了），扩展时申请不到内存就会 OOM。

不过有一种间接的 OOM 跟栈有关：**创建太多线程**，每个线程都要分配栈空间，最终耗尽内存：

```java
// 警告：这个代码可能让系统卡死，谨慎运行！
// VM 参数：-Xss2m（给每个线程栈分大点更容易复现）
public class ThreadOOM {
    public static void main(String[] args) {
        while (true) {
            new Thread(() -> {
                try { Thread.sleep(Long.MAX_VALUE); } catch (Exception e) {}
            }).start();
        }
    }
}
// 报错：java.lang.OutOfMemoryError: unable to create new native thread
```

（这个代码**千万别在生产环境跑**...我在本机试的时候系统直接卡了好一会儿）

## 方法区 / 元空间（Metaspace）— 类加载太多就炸

方法区存的是类的元数据：类名、字段、方法信息、常量池等。

- JDK 7 及之前叫 **永久代（PermGen）**，大小固定，容易 OOM
- JDK 8 开始改为 **元空间（Metaspace）**，使用本地内存，默认不设上限（但可以通过 `-XX:MaxMetaspaceSize` 限制）

PermGen 时代的报错：
```
java.lang.OutOfMemoryError: PermGen space
```

Metaspace 时代的报错：
```
java.lang.OutOfMemoryError: Metaspace
```

什么时候会触发？加载的类太多了。用 CGLib 之类的字节码框架动态生成大量类就能复现：

```java
// VM 参数：-XX:MaxMetaspaceSize=10m
public class MetaspaceOOM {
    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(MetaspaceOOM.class);
            enhancer.setUseCache(false); // 关闭缓存，每次生成新类
            enhancer.setCallback((MethodInterceptor) (obj, method, arg, proxy)
                -> proxy.invokeSuper(obj, arg));
            enhancer.create();
        }
    }
}
```

> 实际项目里 Metaspace OOM 常见于：频繁热部署（比如 Tomcat 反复 redeploy）、大量使用动态代理、Groovy 等脚本语言动态编译类。JDK 17 引入了 Elastic Metaspace（JEP 387），会更积极地把不用的元空间还给操作系统，情况好了不少。

## 本地方法栈（Native Method Stack）

和虚拟机栈类似，只不过是给 native 方法用的。同样会抛 `StackOverflowError` 和 `OutOfMemoryError`。

在 HotSpot 里，本地方法栈和虚拟机栈其实是合在一起的，所以一般不单独讨论。

## 直接内存（Direct Memory）— 容易被忽略

直接内存不属于 JVM 运行时数据区，但也会导致 OOM。

NIO 引入的 `ByteBuffer.allocateDirect()` 可以分配堆外内存，这块内存不归 GC 管（只有 DirectByteBuffer 对象被回收时才会释放对应的直接内存）。

```java
// VM 参数：-XX:MaxDirectMemorySize=10m
public class DirectMemoryOOM {
    public static void main(String[] args) throws Exception {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);

        while (true) {
            // 通过 Unsafe 直接分配本地内存
            unsafe.allocateMemory(1024 * 1024);
        }
    }
}
```

报错信息：
```
java.lang.OutOfMemoryError: Direct buffer memory
```

> 直接内存的问题比较隐蔽，因为 heap dump 里看不到它。如果发现 heap dump 很小但进程内存占用很高，就要怀疑是不是直接内存泄漏了。用 `-XX:NativeMemoryTracking=detail` 可以追踪。

## 汇总一下

| 内存区域 | 会 OOM 吗 | 错误信息 | 常见原因 |
|---------|----------|---------|---------|
| 程序计数器 | **不会** | — | 唯一安全的区域 |
| 堆 | 会 | `Java heap space` / `GC overhead limit exceeded` | 内存泄漏、堆配置太小 |
| 虚拟机栈 | 会 | `StackOverflowError`（栈深度）/ OOM（线程太多） | 递归过深、创建过多线程 |
| 本地方法栈 | 会 | 同上 | 同上 |
| 方法区/元空间 | 会 | `PermGen space` / `Metaspace` | 加载类过多、频繁热部署 |
| 直接内存 | 会 | `Direct buffer memory` | NIO 堆外内存泄漏 |

## 面试怎么答

如果被问到"哪些区域会 OOM"，我觉得可以这样组织：

1. **先说结论**：除了程序计数器，其他所有区域都可能 OOM
2. **说原因**：程序计数器只存一个指令地址，空间固定且极小，JVM 规范明确规定它不会抛 OOM
3. **挑重点展开**：堆（最常见）→ 元空间（类加载相关）→ 栈（递归/线程数）→ 直接内存（NIO 相关）
4. **加分项**：提一下排查手段，比如 `-XX:+HeapDumpOnOutOfMemoryError` 自动 dump、MAT 分析、NMT 追踪本地内存

## 小结

把每个区域的 OOM 场景都跑了一遍代码之后理解深了不少。核心就一句话：**程序计数器是唯一不会 OOM 的区域**，其他的，只要条件够极端，都有可能。实际开发中堆 OOM 占大头，其次是 Metaspace，直接内存的问题虽然少但一旦出了会比较难查。
