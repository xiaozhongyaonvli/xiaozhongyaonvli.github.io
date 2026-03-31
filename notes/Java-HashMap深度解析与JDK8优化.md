---
title: HashMap 深度解析 — 从底层结构到 JDK 1.8 的那些优化
date: 2026-01-10 10:00:00
tags:
  - Java
  - HashMap
  - 数据结构
categories:
  - 后端开发
---

# HashMap 深度解析 — 从底层结构到 JDK 1.8 的那些优化

> 最近重新翻了一遍 HashMap 的源码，发现之前很多地方都是一知半解。这次整理一下，主要聚焦底层数据结构和 JDK 1.8 做了哪些改进，贴着源码来说。

## 先说基本结构

HashMap 底层本质上就是一个 **Node 数组**（`Node<K,V>[] table`），每个数组位置叫做一个"桶"（bucket）。每个 Node 里面存了 hash 值、key、value，还有一个 next 指针指向下一个节点——也就是说每个桶其实是一个链表。

来看 Node 的源码定义：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    // ...
}
```

很简洁对吧，就是一个标准的单链表节点，附带了 hash 和 key-value。

## 几个关键常量

源码里有一批常量，刚开始看容易忽略，但其实很重要：

```java
// 默认初始容量 16，必须是 2 的幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 16

// 最大容量 2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子 0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 链表转红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;

// 红黑树退化为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;

// 树化的最小表容量，容量不到 64 时优先扩容而不是树化
static final int MIN_TREEIFY_CAPACITY = 64;
```

这里有个容易忽略的点：**树化不是只看链表长度到 8 就转**，还得数组容量到 64 以上才行。如果容量不够，HashMap 会选择先 `resize()` 扩容，而不是直接转红黑树。这个设计其实挺合理的——如果表很小，冲突多是正常的，扩容才是正道。

## hash() 方法 — 扰动函数

HashMap 算桶位置不是直接拿 key 的 hashCode，而是做了一次"扰动"：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这行代码做的事情是：**把高 16 位和低 16 位异或**。为什么要这么干？

因为计算桶下标用的是 `(n - 1) & hash`，当 n 比较小的时候（比如默认 16），实际只有低 4 位在起作用，高位完全被忽略了。这样的话如果两个 key 的 hashCode 只在高位不同，就会冲突。

> 把高位"混"到低位里去，能让哈希分布更均匀。JDK 1.7 是做了 4 次移位 + 4 次异或，1.8 简化成了一次，够用了而且更快。

## put() 方法 — 插入流程

`put()` 实际调用的是 `putVal()`，这是 HashMap 最核心的方法之一。我把源码简化注释一下：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;

    // 1. 表为空或长度为 0，先 resize 初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // 2. 算出桶下标，如果该位置为空直接放进去
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;

        // 3. 桶里第一个节点就匹配了（hash 相同且 key equals）
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;

        // 4. 如果是红黑树节点，走树的插入逻辑
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

        // 5. 链表遍历
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 到链表尾部了，插入新节点（尾插法！）
                    p.next = newNode(hash, key, value, null);
                    // 插入后如果链表长度 >= 8，尝试树化
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 链表中找到了相同的 key
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }

        // 6. 如果找到了已存在的 key，替换 value
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }

    ++modCount;
    // 7. 超过阈值就扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

整个流程梳理下来就是：

1. 表没初始化？先 `resize()` 创建
2. 桶是空的？直接放
3. 桶不空？看第一个节点是不是要找的 key
4. 是红黑树？走 `putTreeVal`
5. 是链表？**尾插法**遍历到尾部插入，顺便检查是否需要树化
6. 找到重复 key 就覆盖 value
7. 插入完看看 size 是否超过阈值，超了就扩容

（这里有个关键点：**JDK 1.7 用的是头插法，1.8 改成了尾插法**，后面会说为什么）

## resize() — 扩容机制（1.8 的优化很巧妙）

扩容是 HashMap 性能的关键。当元素数量超过 `容量 × 负载因子` 时就会触发。新容量是旧容量的 2 倍。

JDK 1.7 的做法比较暴力：遍历所有节点，对每个节点重新计算 `hash & (newCap - 1)` 得到新下标。

**1.8 的优化就很聪明了**——因为新容量是旧容量的 2 倍，所以 `(newCap - 1)` 比 `(oldCap - 1)` 就是在高位多了一个 1。对于每个节点，只需要看它的 hash 在这个新增的高位上是 0 还是 1：

- 如果是 **0**：位置不变，还在原来的桶
- 如果是 **1**：新位置 = 原位置 + 旧容量

```java
// resize() 中重新分配节点的核心逻辑（简化版）
Node<K,V> loHead = null, loTail = null; // 低位链表（留在原位）
Node<K,V> hiHead = null, hiTail = null; // 高位链表（移动到原位+oldCap）
Node<K,V> next;
do {
    next = e.next;
    // 关键：用 e.hash & oldCap 判断高位是 0 还是 1
    if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    } else {
        if (hiTail == null)
            hiHead = e;
        else
            hiTail.next = e;
        hiTail = e;
    }
} while ((e = next) != null);

if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;       // 原位置
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;  // 原位置 + oldCap
}
```

> 这段代码我第一次看的时候觉得有点绕，后来画了个图就明白了。举个例子：假设 oldCap = 16（二进制 10000），hash 值如果在第 5 位是 0，那 `hash & 16` 就等于 0，留原位；如果是 1，就移到 原位 + 16。**不需要重新计算完整的桶下标**，只需要一次位运算，这个优化确实精妙。

而且注意到没有，这里也是**尾插法**——用 loTail/hiTail 维护尾指针。这保证了链表节点的相对顺序不变。

## 为什么 1.8 改成了尾插法？

JDK 1.7 的头插法有个致命问题：**多线程并发扩容时可能产生环形链表**，导致死循环。

原因是这样的：头插法会在 resize 时反转链表的顺序。假设线程 A 和线程 B 同时在扩容，A 扩容到一半被挂起，B 完成了扩容（链表已反转）。A 恢复后继续按照旧的引用关系操作，这时候就可能形成环。

JDK 1.8 改用尾插法之后，resize 过程中节点的相对顺序保持不变，就不会出现这个问题。

（不过要注意，**HashMap 本身仍然不是线程安全的**，并发场景该用 ConcurrentHashMap）

## 链表转红黑树 — treeifyBin

当某个桶的链表长度达到 8，会触发 `treeifyBin()`：

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 如果表容量不到 64，优先扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        // 先把 Node 链表转成 TreeNode 双向链表
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);  // 真正的树化操作
    }
}
```

流程分两步：
1. 先把普通的 Node 链表转成 TreeNode 双向链表
2. 再调用 `treeify()` 构建红黑树

TreeNode 的定义也值得看一眼：

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // 用于退化为链表时的反向指针
    boolean red;
    // ...
}
```

它继承了 `LinkedHashMap.Entry`（也就间接继承了 `HashMap.Node`），所以 TreeNode 同时维护着树结构和链表结构，方便在需要退化的时候快速转回链表。

> 这里有个设计上的取舍：TreeNode 比普通 Node 占用更多内存，所以只在冲突确实严重的时候才会树化。这也是为什么树化阈值选 8 而不是更小的数——在正常的哈希分布下，链表长度超过 8 的概率极低（泊松分布计算出来大概是千万分之六）。

## 为什么选红黑树而不是 AVL 树？

这个问题面试也经常问。简单来说：

- **AVL 树**更严格平衡，查找快一点，但插入/删除时旋转操作更多
- **红黑树**平衡性稍宽松，但插入/删除效率更高

HashMap 里面不只是查找，插入和删除也很频繁，红黑树在这个场景下是更好的折中。

## JDK 1.7 vs 1.8 对比总结

| 特性 | JDK 1.7 | JDK 1.8 |
|------|---------|---------|
| 数据结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| 插入方式 | 头插法 | 尾插法 |
| hash 扰动 | 4次移位 + 4次异或 | 1次移位 + 1次异或 |
| 扩容重定位 | 重新计算所有 hash | 高位判断，`hash & oldCap` |
| 并发扩容问题 | 可能产生环形链表 | 保持顺序，无环形链表风险 |
| 初始化时机 | 构造时分配 | 懒加载，首次 put 才分配 |

## 小结

HashMap 的源码看起来不多，但设计上的细节真的很多。1.8 的几个优化方向都很明确：扰动函数精简、扩容时的位运算优化、链表转红黑树降低最坏情况复杂度、尾插法避免并发环形链表。每个改动都不是随便来的，背后都有很扎实的考量。建议有时间的话直接打开 JDK 源码对着看，比光读文章理解深很多。
