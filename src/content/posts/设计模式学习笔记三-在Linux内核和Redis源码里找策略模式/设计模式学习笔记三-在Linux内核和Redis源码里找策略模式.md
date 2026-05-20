---
title: "设计模式学习笔记（三）：在 Linux 内核和 Redis 源码里找策略模式"
description: "鸭子例子把策略模式的思路讲明白了，但工业级代码里它长什么样？这一篇我钻进 Linux 内核的 TCP 拥塞控制和 Redis 的内存淘汰策略源码，去看同一个模式在两种语言、两种工程语境下的不同落地。一个是开放式插件、一个是封闭式枚举，对照着读才反应过来：模式是不变的，工程是给它加上骨肉的……"
published: 2026-05-20
tags:
    - 设计模式
    - 策略模式
    - 源码阅读
    - Linux 内核
    - Redis
category:
    - 设计模式
draft: false
---

> 接着上一篇继续。鸭子例子用《Head First 设计模式》把策略模式自然地推导了出来，但教学例子终归是教学例子。这一篇我钻进了两个优秀的开源项目——Linux 内核 和 Redis——的源码里，去看工业级的策略模式到底长什么样。看完之后最大的感受是：**模式是不变的，工程是给它加上骨肉的。**

## 为什么要看源码里的策略模式

鸭子例子很好，但它有一个"教学例子综合症"：**结构太干净了**。

真实工程里，你会遇到 C 这种没有 `interface` 关键字的语言、会遇到性能压榨到极致以至于愿意放弃一点抽象的场景、会遇到一个策略需要在运行时被注册 / 卸载 / 切换的复杂需求。这些情况下，策略模式还成立吗？长什么样？

我挑了两个例子：

1. **Linux 内核 TCP 拥塞控制**——纯 C 用函数指针表实现的策略模式，计网领域绕不开的经典
2. **Redis 内存淘汰策略**——同样是 C，但表达方式完全不同，更偏向"策略作为配置项"

放在一起看，会发现一件很有意思的事：**策略模式的精髓和语言无关、和语法无关，只和"把变化点抽出来"这件事有关。**

## 案例一：Linux 内核 TCP 拥塞控制

### 背景：为什么 TCP 必须支持多种拥塞控制算法

TCP 在发送数据时，需要根据网络拥塞情况调节发送速率。这个"怎么调"就是拥塞控制算法。

历史上算法非常多，各自适用场景不同：

- **Reno / NewReno**：经典的"加性增、乘性减"
- **Cubic**：Linux 长期的默认算法，在高带宽长延迟链路上比 Reno 好得多
- **BBR**：Google 提出，基于带宽和延迟建模，不依赖丢包判定拥塞，长肥管道里非常猛
- **Vegas**：基于延迟的算法
- **DCTCP**：数据中心专用

没有一个算法能通吃所有场景。**所以 Linux 必须允许"换算法"，而且最好能动态换。**

这就是策略模式天然的应用场景：**算法可以随网络环境、随业务场景切换，但 TCP 主流程不应该因为多了一个算法就要改一遍**。

### 策略接口本体：`struct tcp_congestion_ops`

整个设计的核心，是 `include/net/tcp.h` 里的一个结构体：

```c
struct tcp_congestion_ops {
    /* (a) 和 (b) 二选一 —— 算法必须提供其中之一：
     * (a) "经典"响应：沿用 TCP 主流程的默认拥塞控制套路，只负责
     *     "每个 ACK 怎么推进 cwnd"。Reno / Cubic 走这条。
     */
    void (*cong_avoid)(struct sock *sk, u32 ack, u32 acked);
    /* (b) "自定义"响应：完全接管拥塞控制，主流程在 ACK 处理完后
     *     把决策权交给算法。BBR 走这条。
     */
    void (*cong_control)(struct sock *sk, u32 ack, int flag,
                         const struct rate_sample *rs);

    u32  (*ssthresh)(struct sock *sk);              /* 必填 */
    u32  (*undo_cwnd)(struct sock *sk);             /* 必填 */

    void (*set_state)(struct sock *sk, u8 new_state);                 /* 可选 */
    void (*cwnd_event)(struct sock *sk, enum tcp_ca_event ev);        /* 可选 */
    void (*cwnd_event_tx_start)(struct sock *sk);                     /* 可选 */
    void (*in_ack_event)(struct sock *sk, u32 flags);                 /* 可选 */
    void (*pkts_acked)(struct sock *sk, const struct ack_sample *);   /* 可选 */
    void (*init)(struct sock *sk);                                    /* 可选 */
    /* ... 还有 sndbuf_expand / min_tso_segs / get_info / release ... */

    char              name[TCP_CA_NAME_MAX];   /* "reno" / "cubic" / "bbr" ... */
    struct module    *owner;
    struct list_head  list;                    /* 全局注册链表的节点 */
    u32               key;                     /* name 的 jhash，加速查找 */
    u32               flags;                   /* 能力声明，如 NEEDS_ECN */
};
```

从 Java 视角看，这就是一个 interface。只是 C 没有 `interface` 关键字，所以用**函数指针组成的结构体**来表达"我承诺会提供这些行为"——也就是 C 版本的虚函数表。

但比起 Java 接口，生产级的接口设计有几个细节让我看完才意识到：

- **必填 / 可选区分**。注释里 `required` 表示算法必须实现，`optional` 表示可以留 NULL，调用方调用前会做 NULL 检查。这相当于 Java 接口里"抽象方法 vs default 方法"的二分，只不过 C 版本要靠调用方手写 `if (ops->xxx) ops->xxx(...)`。
- **二选一钩子**。`cong_avoid` 和 `cong_control` 是二选一关系——前者走 TCP 经典套路，后者完全接管。**同一个接口允许不同抽象层级的算法挂进来**，这是接口设计上的余量，讲鸭子例子时根本想象不到。**好的策略接口不强求所有算法长一样，只规定最小的必要契约，其它都是可选的。**
- **元数据字段**。`name`、`flags`、`list`、`key` 不是函数指针，是策略对自己的"自我介绍"。比如 `flags & TCP_CONG_NEEDS_ECN` 让主流程不调用钩子就能问"这个算法需不需要 ECN"——**接口除了承诺行为，还能携带能力声明**。

### 一个算法 = 一份 ops 实现

来看 Reno 是怎么"实现"这个接口的（`net/ipv4/tcp_cong.c`）：

```c
struct tcp_congestion_ops tcp_reno = {
    .flags     = TCP_CONG_NON_RESTRICTED,
    .name      = "reno",
    .owner     = THIS_MODULE,
    .ssthresh  = tcp_reno_ssthresh,
    .cong_avoid = tcp_reno_cong_avoid,
    .undo_cwnd = tcp_reno_undo_cwnd,
};
```

只填了必填的三个钩子（`ssthresh` / `cong_avoid` / `undo_cwnd`），其他全留 NULL——这就是策略模式里"接口只规定最小契约，实现按需填空"的工程版本。

Cubic（`net/ipv4/tcp_cubic.c`）多填了几个可选钩子：

```c
static struct tcp_congestion_ops cubictcp __read_mostly = {
    .init                 = cubictcp_init,
    .ssthresh             = cubictcp_recalc_ssthresh,
    .cong_avoid           = cubictcp_cong_avoid,
    .set_state            = cubictcp_state,
    .undo_cwnd            = tcp_reno_undo_cwnd,     /* 直接复用 Reno 的实现 */
    .cwnd_event_tx_start  = cubictcp_cwnd_event_tx_start,
    .pkts_acked           = cubictcp_acked,
    .owner                = THIS_MODULE,
    .name                 = "cubic",
};
```

注意一个细节：`.undo_cwnd = tcp_reno_undo_cwnd` 直接把 Reno 的实现拿来用。**策略实现之间可以拼装组合**——这是函数指针表比 Java 类继承更灵活的一点，Cubic 不需要"继承 Reno"才能复用它的某个钩子，挑哪个用哪个，完全字段级别。

BBR（`net/ipv4/tcp_bbr.c`）就长得不一样了：

```c
static struct tcp_congestion_ops tcp_bbr_cong_ops __read_mostly = {
    .flags                = TCP_CONG_NON_RESTRICTED,
    .name                 = "bbr",
    .owner                = THIS_MODULE,
    .init                 = bbr_init,
    .cong_control         = bbr_main,        /* ← 这里！不是 cong_avoid */
    .sndbuf_expand        = bbr_sndbuf_expand,
    .undo_cwnd            = bbr_undo_cwnd,
    .cwnd_event_tx_start  = bbr_cwnd_event_tx_start,
    .ssthresh             = bbr_ssthresh,
    .min_tso_segs         = bbr_min_tso_segs,
    .get_info             = bbr_get_info,
    .set_state            = bbr_set_state,
};
```

完美对应上一篇里讲的策略模式结构，只是多了一些工程细节：

| 鸭子例子                | TCP 拥塞控制                |
| ----------------------- | --------------------------- |
| `FlyBehavior` 接口      | `tcp_congestion_ops` 结构体 |
| `FlyWithWings` 实现类   | `tcp_reno` 实例             |
| `FlyNoWay` 实现类       | `cubictcp` 实例             |
| （鸭子例子里没有）      | `tcp_bbr_cong_ops` 实例（用 `cong_control` 接管整个流程） |
| `Duck` 持有 FlyBehavior | sock 持有 `icsk_ca_ops`     |
| `performFly()` 委托调用 | 内核流程通过 ops 指针调用   |

### 策略的"私有状态"挂在哪里？

读鸭子例子的时候，我没注意到一个问题：`FlyBehavior` 的实现类是没有状态的，`FlyWithWings` 就一个 `fly()` 方法，你 new 一个所有鸭子共用就行。

但 Cubic / BBR 是有状态的算法——Cubic 要记 `bic_K`、`last_max_cwnd`、`epoch_start` 这些；BBR 要存 `min_rtt`、`bw` 估计、状态机当前阶段……**这些状态是 per-connection 的**，显然不能放在共享的 ops 实例上（整个系统只有一份 `cubictcp` 实例，千万连接共用）。

那挂哪儿？读 `include/net/inet_connection_sock.h` 我看到了 Linux 的答案：

```c
struct inet_connection_sock {
    /* ... 一堆 TCP socket 字段 ... */

    const struct tcp_congestion_ops *icsk_ca_ops;   /* ← 策略指针 */

    /* ... */

    /* ★ 算法私有数据区：104 字节的 inline 缓冲区 */
    u64 icsk_ca_priv[104 / sizeof(u64)];
#define ICSK_CA_PRIV_SIZE sizeof_field(struct inet_connection_sock, icsk_ca_priv)
};

/* 算法访问自己私有数据的入口 */
static inline void *inet_csk_ca(const struct sock *sk)
{
    return (void *)inet_csk(sk)->icsk_ca_priv;
}
```

socket 上预留了 104 字节的内联缓冲区，谁是当前算法谁就把这块内存当成自己的私有结构体来用。Cubic 在 init 钩子里：

```c
struct bictcp *ca = inet_csk_ca(sk);   /* 把通用内存 cast 成自己的结构体 */
ca->cnt = 0;
ca->last_max_cwnd = 0;
/* ... */
```

模块注册时还有编译期校验，免得 struct 撑爆这块区域：

```c
BUILD_BUG_ON(sizeof(struct bictcp) > ICSK_CA_PRIV_SIZE);
```

这个设计我觉得特别巧：

- ops 是**全局共享**的（整个系统只有一份 `cubictcp` 实例，所有连接共用）
- per-connection 的**私有状态**挂在 socket 上，而不是 ops 上
- 用一块预留内存 + cast，不用动态分配，缓存友好

教学例子里之所以遇不到这问题，是因为鸭子的 fly 行为天然无状态。**一旦策略需要带 per-instance 状态，工程上就要解决"状态挂哪儿"——挂在客户端上，而不是策略上**。这是教科书没强调，但读源码才能体会到的一环。

### 委托调用是什么样

TCP 主流程里大量这种代码（`tcp_input.c`，简化）：

```c
/* 拥塞避免阶段，通过 ops 指针调用，根本不 care 是哪个算法 */
icsk->icsk_ca_ops->cong_avoid(sk, ack, acked);
```

或者带 NULL 检查的可选钩子：

```c
if (icsk->icsk_ca_ops->pkts_acked)
    icsk->icsk_ca_ops->pkts_acked(sk, &sample);
```

但最能说明问题的不是这些调用本身，而是**整个 `tcp_input.c` 文件里 grep 不到任何具体算法名**——reno / cubic / bbr / vegas 一个都没有。

这就是策略模式真正想消灭的东西。我以前读教科书时一直以为"消灭 if-else"是终点，直到读到这里才反应过来：**真正的终点是"客户端代码里完全看不见具体策略"**。if-else 只是表象，真正消灭的是"客户端对具体策略的耦合"。

唯一一处看起来像 dispatch 的代码长这样（`tcp_input.c::tcp_cong_control`）：

```c
if (icsk->icsk_ca_ops->cong_control) {
    icsk->icsk_ca_ops->cong_control(sk, ack, flag, rs);
    return;
}
/* 否则走默认套路：cwnd_reduction / cong_avoid */
```

第一眼像 `if (是 BBR) 走这条`，但仔细看条件——它判断的是"算法**有没有实现** cong_control 钩子"，不是"算法叫不叫 bbr"。**主流程依然不知道具体算法是谁**，它只问接口字段。

### 注册机制：对应"开闭原则"

新加一个算法怎么接入？在自己的 `module_init` 里调一次 `tcp_register_congestion_control()` 就行：

```c
static int __init bbr_register(void)
{
    BUILD_BUG_ON(sizeof(struct bbr) > ICSK_CA_PRIV_SIZE);
    return tcp_register_congestion_control(&tcp_bbr_cong_ops);
}
module_init(bbr_register);
```

整个 TCP 核心代码**不需要任何修改**。这就是开闭原则在内核里的样子：

- 对扩展开放：加新算法只是新增一个 `.c` 文件 + 一个 ops 实例 + 一次注册调用，可以编译成 `.ko` 单独加载
- 对修改关闭：`tcp_input.c` / `tcp_output.c` / `tcp_cong.c` 这些核心代码不会因为新增算法而改动

继续读 `tcp_cong.c` 里 `tcp_register_congestion_control` 的实现，我看到几个值得记一笔的工程细节：

```c
int tcp_register_congestion_control(struct tcp_congestion_ops *ca)
{
    int ret = tcp_validate_congestion_control(ca);   /* 1. 校验必填钩子 */
    if (ret) return ret;

    ca->key = jhash(ca->name, ...);                   /* 2. name → key 建索引 */

    spin_lock(&tcp_cong_list_lock);
    if (tcp_ca_find_key(ca->key)) {
        ret = -EEXIST;                                /* 3. 重名检测 */
    } else {
        list_add_tail_rcu(&ca->list, &tcp_cong_list); /* 4. 挂全局链表，RCU 保护 */
    }
    spin_unlock(&tcp_cong_list_lock);
    return ret;
}
```

注册中心做了几件事：

1. **校验必填钩子** —— `ssthresh / undo_cwnd / (cong_avoid|cong_control)` 必须有，否则 `-EINVAL` 注册失败。这是接口契约的运行时保障。
2. **建索引** —— 既能按 name 查，又能按 jhash 出来的 key 查，加速查找。
3. **RCU 保护读路径** —— 用算法时是个超高频读、超低频写的场景，RCU 是合适的并发原语。
4. **模块自动加载** —— `tcp_ca_find_autoload` 里找不到算法时会 `request_module("tcp_%s", name)`，触发内核按需加载 `.ko`，实现"算法即插件"。

教学例子里的策略模式只有"接口 + 实现"两层，生产级策略模式还要解决**注册、查找、并发、热插拔**这一整套基础设施。读到这里我才意识到，"策略模式"这四个字背后能扩展出多少东西。

### 两种切换粒度

更妙的是，Linux 提供了两种切换粒度：

```bash
# 系统级：改全局默认
sysctl -w net.ipv4.tcp_congestion_control=bbr

# 连接级：在单个 socket 上设
setsockopt(fd, IPPROTO_TCP, TCP_CONGESTION, "bbr", 3);
```

后者背后的代码路径就是 `tcp_set_congestion_control()` —— 按 name 查注册表 → 找到 ops → 替换 `icsk->icsk_ca_ops` → 清零 `icsk_ca_priv` → 调新算法的 init 钩子重填。

这就是策略模式那句"**算法的变化独立于使用算法的客户**"的工程版本——客户（socket）可以在任意时刻决定用哪个算法，核心代码完全不感知。

### 这一段我自己最大的感受

我以前读《计算机网络》的时候，看到"TCP 拥塞控制"知道大概意思，但很难想象它在工程里是怎么"长"出来的。

读了这块源码之后，我才突然反应过来：**那些算法之所以能"列"成一个表给我背，本质上就是因为内核用策略模式把它们封装成了可替换的单元。**

不是这些算法天生就长成可枚举的样子，而是**好的架构让它们可以被枚举、被替换、被增量加入**。

而且这一次读细了之后，我对策略模式本身也多了几个理解：

- **接口可以渐进演化**——`cong_avoid` 不够用了就加 `cong_control`，老算法不动，新算法填新钩子
- **策略实现之间可以互相组合**——cubic 直接复用 reno 的 `undo_cwnd`，字段级别拼装
- **per-instance 状态要挂在客户端上，不是策略上**——`icsk_ca_priv` 那 104 字节是关键
- **接口除了行为契约，还可以承载能力声明**——`flags & TCP_CONG_NEEDS_ECN` 这种查询
- **生产级策略模式还需要一整套"注册中心"基础设施**——校验 / 索引 / 并发 / 自动加载

这些都是教学例子讲不到的层级。**模式是不变的，工程是给它加上骨肉的。**

这就是上一篇说的"提前在代码中加入弹性"在大型系统里的真实样子。

## 案例二：Redis 内存淘汰策略

### 背景

Redis 是内存数据库，内存用满了得有人腾地方。怎么腾？通过 `redis.conf` 里的 `maxmemory-policy` 配置：

- `noeviction`：满了就报错，谁也不删
- `allkeys-lru` / `volatile-lru`：LRU
- `allkeys-lfu` / `volatile-lfu`：LFU
- `allkeys-random` / `volatile-random`：随机
- `volatile-ttl`：挑快过期的删
- `allkeys-lrm` / `volatile-lrm`：以最近一次修改时间为依据

`volatile-*` 只在带 TTL 的 key 里挑，`allkeys-*` 在所有 key 里挑——两个维度叉乘加上 `noeviction` 一共 10 种。

### 策略的"枚举身份"

`server.h` 里定义了 `MAXMEMORY_FLAG_*` 和 `MAXMEMORY_VOLATILE_LRU` / `ALLKEYS_LRU` / … / `NO_EVICTION` 这一组宏。每个宏对应一个策略对象，是 int 编码而不是结构体。高 8 位是策略 id，低 8 位是属性 flag。

这一点比较值得学习——不仅记录了策略对象本身，还把状态一同记录进去；想获取某个状态的值时，可以快速通过位运算获取，不需要 switch-case 一个个值匹配。采用 `int + 位运算` 来伪装成对象，在 C 里非常常见。

### 客户端持有"策略指针"

客户端（整个 Redis 实例）持有的就是 `server.maxmemory_policy` 这个枚举值。新增策略需要在 `evict.c` 的 `performEvictions()` 主循环里再加一个 if 分支——也就是说，Redis 的策略模式是弱化版的：**固定 10 种，不开放注册，不支持运行时扩展。**

策略名 → 枚举值的注册表在 `config.c`：

```c
configEnum maxmemory_policy_enum[] = {
    {"volatile-lru",   MAXMEMORY_VOLATILE_LRU},
    {"volatile-lfu",   MAXMEMORY_VOLATILE_LFU},
    {"volatile-random",MAXMEMORY_VOLATILE_RANDOM},
    {"volatile-ttl",   MAXMEMORY_VOLATILE_TTL},
    {"volatile-lrm",   MAXMEMORY_VOLATILE_LRM},
    {"allkeys-lru",    MAXMEMORY_ALLKEYS_LRU},
    {"allkeys-lfu",    MAXMEMORY_ALLKEYS_LFU},
    {"allkeys-random", MAXMEMORY_ALLKEYS_RANDOM},
    {"allkeys-lrm",    MAXMEMORY_ALLKEYS_LRM},
    {"noeviction",     MAXMEMORY_NO_EVICTION},
    {NULL, 0}
};
```

这张表是**编译期静态数组**，不是运行时注册。想加一种新策略，必须改这个 .c 文件并重新编译 Redis。这是一个【闭集】，对"开闭原则"是有让步的——对修改并不封闭。**这是 Redis 牺牲扩展性换来简单和性能的取舍。**

### 顺便安利下我的 skill

题外话：上面这几段源码是我借助自己写的一个 skill 拉下来的。如果直接告诉 AI 你想了解的内容，可能会出现幻觉，给出的源码可能有较大偏差；如果直接去下载源码呢，一方面 Linux、Redis 这些源码本身较大、文件太多，自己找太麻烦，需要批量下载；另一方面，太多杂乱的文件结构和海量代码，注意力有限，只想关注当前真正关心、想学习的代码部分。

我这个 skill 的输入是开源库地址 + 感兴趣的部分（比如"Redis 中内存淘汰中关于策略模式的应用"），然后它会只下载相关文件（避免下载 Linux 这种动辄好几 G 的流量，也挺贵），删减完全无关此次学习的代码部分，同时给剩余代码加上详细的文件大纲注释、把英文注释替换为中文，并写出一份阅读指南，这样我们就可以顺着阅读指南去看源码了。

skill 地址：<https://github.com/xiaozhongyaonvli/skill-lab/tree/main/repo-theme-extract>

### 淘汰候选池

Redis 会维护一个淘汰候选池，多次 `performEvictions()` 调用之间共享一个淘汰候选池——每次采样的好候选都会被加入这个池子里。池里的条目按照空闲时间升序排列，如果采用的是 LFU，则是频率倒数。

### 题外话：近似 LRU 而不是严格 LRU

在学习过程中也了解了一些关于 Redis 淘汰算法的具体实现：

Redis 采用的是近似 LRU，而不是严格 LRU。严格 LRU 需要维护一条双向链表，每次访问都要把节点移到头部，这在大量 key + 多线程访问下开销不可接受。

近似算法：每次想淘汰时，从 DB 里随机抽样 N 个 key，将这 N 个候选与淘汰池里现有的候选比较，把空闲时间更长的塞进池子；真正要淘汰时，从池子里取空闲最久的干掉。

同时注意内存计量中，AOF 缓冲区不计入已用内存（如果淘汰向它们推 DEL，反过来又让它们变大，就会陷入"淘汰越多缓冲越大、需要再淘汰"的死循环）；同样的，槽迁移时类似副本，也接收 DEL，不计入。

Redis 淘汰中还有一个异步淘汰定时器：一旦被 `performEvictions` 触发，就不停回调，直到内存恢复或者再也没东西可淘汰，避免单次命令处理里占用过长时间。

### performEvictions() 的处理流程

整个核心部分在于 `performEvictions()` 函数，根据策略分发。大致处理流程如下：

```
1. 检查是否超 maxmemory，没超直接返回 EVICT_OK
2. 如果策略是 NO_EVICTION，直接返回 EVICT_FAIL
3. 进入大循环 while (mem_freed < mem_tofree):
   * 下面就是策略分发 *
   if (policy 是 LRU/LFU/LRM/VOLATILE_TTL):
       走采样 + 淘汰池路径，挑出 bestkey
   else if (policy 是 ALLKEYS_RANDOM / VOLATILE_RANDOM):
       走纯随机路径，挑出 bestkey
4. 将 bestkey 删掉，累计 mem_freed，可能转入异步定时器
```

再读具体的函数代码，发现：跨所有 DB 采样，避免局部偏向；如果当前 DB key 太少，提早跳出。

### 题外话：关于多 DB

这里补充些 DB 相关的知识——实际我在操作 Redis 中，比如 RESP 或者 Another Redis 中，是能看到有多个 DB 的，但我只用 db 0，所以也不太关注，这里看到了，去详细了解了下。

Redis 支持多个 DB（逻辑数据库），默认是 16 个。多个 DB 不是多个 Redis 实例，而是同一个 Redis 进程里的逻辑隔离，本质上是多个 key 空间（namespace）。

那为什么大家可能都不太关注这个呢？因为实际生产环境中几乎废弃多 DB：

1. **Redis Cluster 不支持多 DB**——Cluster 要分片、哈希槽、节点路由，多 DB 会让路由复杂化，所以官方直接禁掉。
2. **本质底层仍是同一个实例**——内存共享（DB1 爆了 DB0 也受影响）、CPU 共享、网络共享、持久化共享（RDB/AOF 仍然是整个实例）、单线程共享（一个 DB 卡顿全实例受影响）。
3. **运维困难**——无法单独备份，无法单独迁移、限流、扩容。

现在主流采用：1）key 前缀（兼容 Cluster，方便迁移、统计、删除）；2）不同业务用不同 Redis 实例。

### if/else 分发出现具体策略名，合理吗？

代码中直接出现了具体策略的名字，在《Head First 设计模式》里是不推荐的——客户端代码不应该知道具体策略。但 Redis 采用 if/else 分发的弱化策略模式，我认为有两个原因：

第一，策略数量很少，只有 10 个，且是封闭集合，所以 if/else 也没有多大的困扰，即使不使用指针——因为策略模式的核心实际还是把可变的部分移出，所以究竟是指针还是 if/else 并没有太大关系。

第二，前面也大概介绍了一些淘汰策略的具体实现，可以发现它们的执行路径并不太相同（随机、采样、池子），所以无法强行给出一个统一接口，除非这个接口包含所有，就会很臃肿和松散。

### 在能复用的地方还是尽量复用

不过 Redis 中比如在池子这类的实现中，同一个变量在不同策略下含义不同，但为了后续插入池子代码的复用，LFU 使用 `255 - 频率`，TTL 则用 `MAX - expire`，就是为了将策略统一到"值越大越该淘汰"的语义。

`idle` 这个变量在三种策略下含义完全不同，但后面插入池子、比较大小的代码完全复用。策略的差异只体现在"算 idle 这个变量"这一行——这是即使在"弱化策略模式"里也仍然在尽力共享代码的体现。

## 我从这一段源码阅读里拿到的最大收获

**1. 设计模式不是"长什么样"，而是"为什么这样"**

上一篇看完鸭子例子之后，我心里其实建立了一个"标准长相"：接口 + 一组实现类 + 客户端持有引用并委托。但读完这两段源码我才意识到，这只是策略模式的**一种表现形式**，而不是它的本体。

它的本体是那条设计原则：**找出应用中可能需要变化的部分，把它们独立出来。**

满足这一点的，无论用接口、用函数指针表、还是用一组集中的 if-else，都可以叫策略模式。

**2. 看源码要分清"模式"和"骨架"**

写源码博客经常掉进的陷阱是：贴一堆代码，然后说"看，这就是 XX 模式"。但其实模式的精髓往往不在那段代码本身，而在它**没有出现在哪里**。

比如 Linux TCP 拥塞控制的策略模式，精髓不在 `tcp_reno = { ... }` 这种实例定义里——那只是骨架。精髓在 `tcp_input.c` 这种核心文件里**完全没有出现任何具体算法名**。"算法不污染核心流程"，这种"没看见"的东西，才是模式的力量。

**3. AI 时代的源码阅读价值反而上升了**

vibe coding 时代，写代码这件事被 AI 大幅承担了。但**读懂别人的代码、判断架构好坏**这件事，反而变得更重要——因为只有这样你才知道该指挥 AI 往哪里走。

阅读优秀开源项目的源码，本质上是给自己积累"架构品味"。看多了你会自然知道：哦，这种地方应该把变化抽出来；哦，这种场景下 if-else 反而比接口更合适。这种判断力，是无法通过让 AI 生成代码积累的。

---

单看书例子还是太单薄了，缺少一些具体的应用。无论是 TCP 里关于接口最小化的那种克制，还是 Redis 里把状态一起塞进 int、用位运算快速取出的那种省，都是让我感到眼前一亮、教学例子里看不到的点。
