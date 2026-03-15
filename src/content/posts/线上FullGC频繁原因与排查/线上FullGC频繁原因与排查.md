---
title: 线上 Full GC 频繁怎么办？原因梳理 + 排查思路
description: "这个问题面试出现频率极高，但说实话之前我对 Full GC 的理解一直停留在\"老年代满了就触发\"这个层面。最近系统整理了一下，发现坑比想象中多得多。这篇笔记把常见原因和排查思路都串一遍。"
published: 2026-02-03
tags:
    - JVM
    - GC
    - 性能调优
    - 线上排查
category:
    - 后端开发
draft: false
---
> 这个问题面试出现频率极高，但说实话之前我对 Full GC 的理解一直停留在"老年代满了就触发"这个层面。最近系统整理了一下，发现坑比想象中多得多。这篇笔记把常见原因和排查思路都串一遍。

## 先搞清楚：什么时候会触发 Full GC

在聊"为什么频繁"之前，得先知道 Full GC 到底是什么情况下触发的。我整理了一下，大概有这么几种：

1. **老年代空间不足** —— 最常见的，老年代放不下新晋升的对象了
2. **Metaspace（元空间）不足** —— 类元数据太多，装不下了（JDK8 之前叫永久代 PermGen）
3. **显式调用 `System.gc()`** —— 代码里或某些框架主动触发的
4. **空间分配担保失败** —— Minor GC 前检查老年代剩余空间不够，直接触发 Full GC
5. **CMS GC 的 Concurrent Mode Failure** —— 并发标记还没结束，老年代就满了，退化成 Full GC
6. **G1 的 Humongous Allocation** —— 大对象分配失败

> 注意：Minor GC（Young GC）只回收年轻代，速度快、频率高，这是正常的。Full GC 是回收整个堆（年轻代 + 老年代 + 元空间），STW 时间长，频繁出现就是有问题了。

---

## 频繁 Full GC 的常见原因

### 1. 内存泄漏（最常见也最难查）

这是最典型的原因。代码里有对象被意外持有引用，GC 回收不掉，老年代越积越多，最终不断触发 Full GC。

常见的泄漏场景：

- **静态集合类** —— `static Map`、`static List` 只往里 put 不清理
- **未关闭的资源** —— 数据库连接、IO 流、HTTP 连接没 close
- **缓存没有淘汰策略** —— 自己写的 HashMap 做缓存，没有 LRU 或大小限制
- **监听器/回调没注销** —— 注册了 Listener 但从来不 remove
- **ThreadLocal 没 remove** —— 线程池场景下特别危险，线程复用导致 ThreadLocal 里的对象一直不释放

```java
// 典型的泄漏代码
public class CacheManager {
    // 只进不出的 static Map，随着时间推移会吃掉整个老年代
    private static final Map<String, Object> cache = new HashMap<>();

    public static void put(String key, Object value) {
        cache.put(key, value);
    }
    // 从来没人调 remove...
}
```

GC 日志里内存泄漏长什么样？就是那种**锯齿图的底部在不断抬高** —— 每次 Full GC 能回收一部分，但回收后的水位线一次比一次高，直到 OOM。

### 2. 堆内存设置太小

最简单的原因，但也最容易被忽视。有些服务上线的时候拍脑袋设了个 `-Xmx512m`，结果业务量上来后根本不够用。

还有一种情况：**容器环境下 JVM 没感知到容器的内存限制**。早期 JDK8 版本（8u131 之前）不认 cgroup 限制，JVM 以为自己有宿主机那么多内存，结果设的堆比容器 limit 还大，被 OOM Killer 直接杀掉。

### 3. 年轻代太小导致对象过早晋升

这个比较隐蔽。年轻代（Young Generation）太小的话，对象还没来得及被 Minor GC 回收就因为空间不足被提前晋升到老年代了。本来应该"朝生夕灭"的临时对象全跑到老年代去了，老年代很快就满了。

```
正常情况：
  短命对象在 Eden → S0/S1 之间来回倒，几次 Minor GC 就被回收了

年轻代太小时：
  短命对象在 Eden 放不下 → 直接晋升到老年代 → 老年代迅速填满 → Full GC
```

可以通过 `-XX:NewRatio` 或 `-Xmn` 调整年轻代大小。一般建议年轻代占堆的 1/3 到 1/2。

### 4. 大对象直接进老年代

默认配置下，超过一定大小的对象会直接分配到老年代（跳过年轻代）。如果你的代码频繁创建大数组、大字符串，老年代会被快速填满。

```java
// 每次请求都创建一个大数组，直接进老年代
byte[] buffer = new byte[4 * 1024 * 1024]; // 4MB
```

G1 里这叫 Humongous Object（巨型对象），超过 Region 大小一半的对象就算。这类对象的分配和回收都比较特殊，容易引发 Full GC。

### 5. System.gc() 被显式调用

有些框架或第三方库里会调 `System.gc()`，比如 NIO 的 DirectByteBuffer 在回收堆外内存时。GC 日志里会标记为 `System.gc()` 触发的，很好辨认。

> 可以用 `-XX:+DisableExplicitGC` 禁掉，但要小心 —— 如果你用了堆外内存（DirectByteBuffer），禁掉后可能导致堆外内存泄漏。这时候建议换成 `-XX:+ExplicitGCInvokesConcurrent`，让 System.gc() 触发的是并发 GC 而不是 Full GC。

### 6. Metaspace 不够

JDK8 以后类元数据存在 Metaspace（本地内存），默认没有上限。但如果你用了大量动态代理（CGLib、反射）、热部署、OSGi 这类会动态生成类的技术，Metaspace 会持续增长。

我的 API 网关项目里就用了 CGLib 动态代理来生成 Session 代理类，如果代理类没有被正确复用，理论上也会有 Metaspace 膨胀的风险（虽然实际规模不大）。

### 7. CMS 的 Concurrent Mode Failure

如果用的 CMS 收集器，并发标记阶段老年代又满了，就会触发 Concurrent Mode Failure，退化成单线程的 Serial Old GC 做 Full GC，STW 时间非常长。

本质原因是**并发回收的速度赶不上对象分配的速度**。

---

## 排查思路：我会按什么顺序来

碰到线上 Full GC 频繁，我的排查步骤是这样的：

### 第一步：看 GC 日志，确认频率和类型

这是最基本的。先确认 Full GC 的频率、耗时、每次回收了多少内存。

```bash
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/var/log/gc.log

-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags
```

拿到日志后看几个关键指标：

| 指标 | 正常 | 异常 |
|------|------|------|
| Full GC 频率 | 几小时一次或更低 | 几分钟甚至几秒一次 |
| Full GC 后老年代占用 | 回收后降到 30% 以下 | 回收后仍然在 70%+ |
| GC 吞吐量 | > 98% | < 95% |

> 重点看 Full GC 后老年代有没有降下来。**如果每次 Full GC 后老年代占用都在涨，基本可以确定是内存泄漏**。如果能降下来但马上又满了，可能是流量太大或者堆太小。

GC 日志可以用 [GCeasy](https://gceasy.io) 在线分析，直接上传日志文件就行，图表很直观。

### 第二步：看监控，关联业务指标

Full GC 频繁是从什么时候开始的？跟什么事件相关？

- 是**发版之后**开始的？→ 大概率是新代码引入了内存泄漏
- 是**流量高峰**才出现？→ 可能是堆太小或年轻代太小
- 是**缓慢增长**的？→ 典型的内存泄漏，对象慢慢堆积
- 是**突然出现**的？→ 可能是某个大查询一次性加载了大量数据

### 第三步：jstat 实时观察各代内存变化

```bash
jstat -gcutil <pid> 1000 30
```

输出大概长这样：

```
  S0     S1     E      O      M     CCS    YGC   YGCT   FGC   FGCT    GCT
  0.00  62.34  45.12  89.71  94.56  91.23   124  1.234   15   8.456  9.690
  0.00  62.34  67.89  91.03  94.56  91.23   124  1.234   16   9.012 10.246
```

重点看 O（老年代占比）和 FGC（Full GC 次数）：
- O 持续在 80%+ 且 FGC 在涨 → 老年代有问题
- M（Metaspace）持续涨 → 可能是类加载泄漏
- E（Eden）频繁跳 100% → 年轻代可能太小

### 第四步：dump 堆内存，找到罪魁祸首

这是定位问题的关键一步。

```bash
jmap -dump:format=b,file=heapdump.hprof <pid>

-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/heapdump.hprof
```

（注意：dump 的时候 JVM 会暂停，大概每 GB 堆耗时 2 秒左右。线上 4GB 的堆就要停 8 秒，要评估影响）

拿到 dump 文件后用 **Eclipse MAT** 打开分析：

1. 先看 **Leak Suspects Report** —— MAT 自动帮你找可疑的泄漏点
2. 看 **Dominator Tree** —— 按 Retained Heap 排序，最大的那几个对象就是嫌疑人
3. 对嫌疑对象 **右键 → Path to GC Roots → exclude weak/soft references** —— 看是谁持有了它的强引用，导致 GC 回收不掉

大部分情况下，到这一步就能定位到是哪个类、哪个集合在泄漏了。

### 第五步：jstack 看线程（辅助手段）

如果怀疑是某个请求导致的，可以用 jstack 抓线程快照：

```bash
jstack <pid> > thread_dump.txt
```

看有没有大量线程卡在同一个地方（比如都在等数据库查询返回一个巨大的结果集）。

---

## 常见场景速查表

| 现象 | 可能原因 | 排查方向 |
|------|---------|---------|
| Full GC 后老年代占用不降 | 内存泄漏 | Heap Dump → MAT 分析 |
| Full GC 后能降但马上又满 | 堆太小 / 对象创建太快 | 加大堆 / 优化代码减少对象创建 |
| GC 日志显示 `System.gc()` | 显式调用 | grep 代码或加 `-XX:+DisableExplicitGC` |
| Metaspace 持续增长 | 动态类加载泄漏 | 检查反射 / CGLib / 热部署相关代码 |
| Young GC 很频繁且对象快速晋升 | 年轻代太小 | 调大 `-Xmn` 或 `-XX:NewRatio` |
| CMS `Concurrent Mode Failure` | 并发回收速度不够 | 降低 CMSInitiatingOccupancyFraction |
| 容器里 OOM Killed | JVM 没感知容器内存限制 | 加 `-XX:+UseContainerSupport`（JDK8u191+） |

---

## 最后说几点实战建议

**1. 生产环境永远要开 GC 日志。** 性能开销几乎可以忽略，但出问题的时候就是救命的。

**2. 一定要配 `-XX:+HeapDumpOnOutOfMemoryError`。** OOM 的时候不 dump 你就只能干瞪眼了。

**3. 别上来就调参数。** 很多人一看 Full GC 频繁就开始调 `-Xmx`、`-XX:NewRatio`... 调参数只能缓解症状，如果是内存泄漏，堆调再大也只是延缓了 OOM 的时间。**先定位根因，再决定是改代码还是调参数。**

**4. 容器环境要特别注意。** JDK8u191 之前的版本需要手动设置 `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap`。之后的版本默认支持了，但还是建议显式设置堆大小，别全靠自动检测。

---

## 小结

Full GC 频繁本质上就两类原因：**要么内存不够用（堆太小），要么内存被占着不释放（内存泄漏）**。排查顺序就是：GC 日志确认现象 → 监控关联时间线 → jstat 实时观察 → Heap Dump 定位根因。掌握这套流程，不管面试还是实际排查都够用了。
