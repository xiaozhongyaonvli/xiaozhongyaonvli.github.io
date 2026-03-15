---
title: 面试场景题：100GB 日志文件，统计接单最多的 Top10 车主
description: "这道题在面试里属于\"海量数据处理\"分类，考的不是写 SQL 或调 API，而是你在**内存有限**的约束下怎么思考问题。核心矛盾就一个：100GB 的文件，内存可能只有 4~8GB，一次性加载不了，怎么办？"
published: 2026-02-18
tags:
    - 海量数据
    - 算法
    - 面试
    - TopK
category:
    - 后端开发
draft: false
---
> 这道题在面试里属于"海量数据处理"分类，考的不是写 SQL 或调 API，而是你在**内存有限**的约束下怎么思考问题。核心矛盾就一个：100GB 的文件，内存可能只有 4~8GB，一次性加载不了，怎么办？

## 先把题目拆清楚

原始数据长这样（一行一条日志）：

```
driver_id,order_id,status
D10001,OD20001,ACCEPTED
D10001,OD20001,ARRIVED      ← 同一个车主、同一个订单、不同状态
D10001,OD20001,COMPLETED    ← 还是同一单，要去重
D10001,OD20002,ACCEPTED     ← 同一个车主、不同订单
D10002,OD20003,ACCEPTED
...
```

要做的事：
1. **去重**：相同 `driver_id + order_id` 只算一个订单（不管有多少种 status）
2. **计数**：统计每个 driver_id 的去重后订单数
3. **排序**：取接单量最多的 Top 10

难点在于：100GB 文件，内存放不下。

---

## 思路一：哈希分治（面试首选答案）

这是最经典的解法，也是面试官最想听到的。核心思想：**大文件放不下，那就切成小文件，每个小文件能放进内存。**

### 第一步：哈希分片 —— 把大文件拆成小文件

```
100GB 大文件
    ↓ 逐行读取，对 driver_id 取哈希
    ↓ hash(driver_id) % N → 写入对应的小文件
    ↓
file_0  file_1  file_2  ...  file_N
```

假设内存 4GB，我们可以设 N=100，平均每个小文件约 1GB，HashMap 能放得下。

**关键点：按 `driver_id` 取哈希，不是随机切分。** 这样保证了同一个车主的所有记录一定落在同一个小文件里，后面每个小文件可以独立完成去重和计数，不需要跨文件合并。

```java
// 伪代码：哈希分片
int N = 100;
BufferedWriter[] writers = new BufferedWriter[N];
// ... 初始化 N 个文件输出流

try (BufferedReader reader = new BufferedReader(new FileReader("huge_log.csv"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        String driverId = line.split(",")[0];
        int bucket = (driverId.hashCode() & 0x7FFFFFFF) % N;
        writers[bucket].write(line);
        writers[bucket].newLine();
    }
}
```

（注意 `& 0x7FFFFFFF` 是为了把 hashCode 的负数变成正数，不然取模会得到负的下标）

### 第二步：逐个小文件处理 —— 去重 + 计数

对每个小文件，把它整个加载到内存里处理：

```java
// 处理一个小文件
// key = driver_id, value = 该车主的去重订单集合
Map<String, Set<String>> driverOrders = new HashMap<>();

try (BufferedReader reader = new BufferedReader(new FileReader("file_" + i))) {
    String line;
    while ((line = reader.readLine()) != null) {
        String[] parts = line.split(",");
        String driverId = parts[0];
        String orderId = parts[1];
        // status 直接忽略，不参与去重
        driverOrders.computeIfAbsent(driverId, k -> new HashSet<>())
                     .add(orderId);  // HashSet 自动去重
    }
}

// 转成 driver_id → 订单数
Map<String, Integer> driverCount = new HashMap<>();
for (Map.Entry<String, Set<String>> entry : driverOrders.entrySet()) {
    driverCount.put(entry.getKey(), entry.getValue().size());
}
```

> 这里用 `HashSet<String>` 存 orderId 来去重。同一个 driver_id + order_id 不管出现多少次（不同 status），Set 里只会存一份。

### 第三步：小顶堆取全局 Top 10

每个小文件处理完会得到一批 `(driver_id, count)` 键值对。我们维护一个**大小为 10 的小顶堆**，把所有小文件的结果都过一遍：

```java
// 小顶堆，堆顶是最小值
PriorityQueue<int[]> minHeap = new PriorityQueue<>(10,
    (a, b) -> a[1] - b[1]  // 按 count 升序
);

// 遍历每个小文件的 driverCount 结果
for (Map.Entry<String, Integer> entry : driverCount.entrySet()) {
    if (minHeap.size() < 10) {
        minHeap.offer(new int[]{entry.getKey(), entry.getValue()});
    } else if (entry.getValue() > minHeap.peek()[1]) {
        // 比堆顶大，踢掉堆顶，放进来
        minHeap.poll();
        minHeap.offer(new int[]{entry.getKey(), entry.getValue()});
    }
}
// 最终堆里就是全局 Top 10
```

> 为什么用小顶堆而不是大顶堆？因为我们要的是"最大的 10 个"。小顶堆的堆顶是当前 Top10 里最小的那个，新来一个元素只要比堆顶大就有资格进来。堆始终保持大小为 10，每次操作 O(log10) ≈ O(1)，比全量排序高效得多。

### 完整流程图

```
100GB 原始文件
  ↓
  ① 逐行读，按 driver_id 哈希分成 100 个小文件
  ↓
  file_0  file_1  file_2  ...  file_99
  ↓
  ② 逐个加载到内存
     HashSet 对 (driver_id, order_id) 去重
     统计每个 driver_id 的订单数
  ↓
  D10001→35  D10002→12  D10003→28  ...
  (每个小文件产出一批 driver→count)
  ↓
  ③ 所有结果过一遍小顶堆（大小=10）
  ↓
  Top 10 车主（按接单数降序）
```

### 复杂度分析

| 阶段 | 时间 | 空间 |
|------|------|------|
| 哈希分片 | O(n) 全量读一遍 | O(1) 只需要少量缓冲区 |
| 单文件去重计数 | O(m) 每个文件内线性扫描 | O(m) HashMap + HashSet |
| 小顶堆 Top10 | O(n' log 10) ≈ O(n') | O(10) |
| **总计** | **O(n)** 两遍扫描 | **O(100GB / 100) ≈ 1GB** |

---

## 思路二：两阶段去重（优化内存）

上面思路一有个潜在问题：如果某个车主特别活跃，他的订单特别多，那 `HashSet<String>` 存他所有的 orderId 可能很占内存。

优化思路：**先去重，再计数。分成两步走。**

### 第一步：去重阶段

还是哈希分片，但这次按 `driver_id + order_id` 的组合取哈希：

```java
// 按 driver_id + order_id 组合哈希分片
String key = driverId + "_" + orderId;
int bucket = (key.hashCode() & 0x7FFFFFFF) % N;
```

每个小文件内部，用 HashSet 对 `driver_id + order_id` 去重，去重后写入中间文件，每行只保留 `driver_id`：

```
去重前（file_0）:           去重后（dedup_0）:
D10001,OD20001,ACCEPTED     D10001
D10001,OD20001,COMPLETED    D10001    ← 这条被去掉了（重复的 OD20001）
D10001,OD20002,ACCEPTED     D10001
D10002,OD20003,ACCEPTED     D10002
```

等等... 这样写出来的中间文件里同一个 driver_id 会出现多次（每个去重后的订单一次），后面计数的时候只要数每个 driver_id 出现了几次就行。

### 第二步：计数阶段

对去重后的中间文件，再按 `driver_id` 哈希分片一次，然后用 HashMap<String, Integer> 计数。这次不用存 orderId 了，HashMap 的 value 就是一个 int，内存压力小很多。

> 这个两阶段的方案适合订单量特别大、去重集合放不进内存的极端情况。代价是多了一轮磁盘 IO。

---

## 思路三：外部排序

另一个经典思路：**先排序，再计数。**

### 第一步：外部排序

对 100GB 文件按 `driver_id, order_id` 做外部排序（External Sort）：

1. 把文件切成能装进内存的块（比如每块 1GB）
2. 每块在内存里排序后写回磁盘
3. 多路归并 —— 同时打开所有排序好的块文件，用最小堆做 K 路归并，输出全局有序的文件

排序后数据长这样：

```
D10001,OD20001,ACCEPTED
D10001,OD20001,ARRIVED      ← 相邻且相同的 driver+order
D10001,OD20001,COMPLETED    ← 连续出现
D10001,OD20002,ACCEPTED
D10001,OD20002,COMPLETED
D10002,OD20003,ACCEPTED
D10002,OD20004,ACCEPTED
...
```

### 第二步：顺序扫描 + 去重计数

排好序之后，**相同 driver_id + order_id 的记录一定是连续的**，直接顺序扫描就能去重 + 计数，不需要 HashMap，O(1) 内存就够了：

```java
String prevDriver = null, prevOrder = null;
int currentCount = 0;
PriorityQueue<Map.Entry<String, Integer>> minHeap = ...;

while ((line = reader.readLine()) != null) {
    String[] parts = line.split(",");
    String driverId = parts[0], orderId = parts[1];

    if (driverId.equals(prevDriver)) {
        // 同一个车主
        if (!orderId.equals(prevOrder)) {
            // 新订单（去重：只有 orderId 变了才算新的一单）
            currentCount++;
            prevOrder = orderId;
        }
        // 否则是同一个订单的不同状态，跳过
    } else {
        // 换了一个车主，把上一个的结果入堆
        if (prevDriver != null) {
            updateHeap(minHeap, prevDriver, currentCount);
        }
        prevDriver = driverId;
        prevOrder = orderId;
        currentCount = 1;
    }
}
// 别忘了最后一个车主
updateHeap(minHeap, prevDriver, currentCount);
```

这个方案的好处是**计数阶段几乎不占内存**（只需要存上一行的 driver_id 和 order_id）。代价是外部排序本身比较耗时，需要多轮磁盘读写。

---

## 思路四：MapReduce（分布式方案）

如果面试官追问"如果数据量更大，单机处理不了怎么办"，就聊 MapReduce：

```
100GB → Mapper
  输入: driver_id,order_id,status
  输出: <driver_id_order_id, 1>  ← key 是组合键，自动去重
          ↓ Shuffle & Sort
        Reducer
  输入: <driver_id_order_id, [1,1,1]>
  去重: 每个 key 只计 1 次
  输出: <driver_id, 1>
          ↓ 第二轮 MR
        Reducer
  按 driver_id 聚合求和
  每个 Reducer 维护本地 Top10 小顶堆
  cleanup() 输出本地 Top10
          ↓ 单 Reducer 汇总
        全局 Top 10
```

或者更简洁的一轮方案：

```java
// Mapper: 输出 <driverId, orderId>
map(line) {
    emit(driverId, orderId);
}

// Reducer: 收到某个 driverId 的所有 orderId，用 Set 去重后计数
reduce(driverId, Iterator<orderId> values) {
    Set<String> uniqueOrders = new HashSet<>();
    for (orderId : values) {
        uniqueOrders.add(orderId);
    }
    emit(driverId, uniqueOrders.size());
}
```

然后再跑一轮 MR 或者在 Reducer 的 `cleanup()` 里用小顶堆取 Top10。

---

## 思路五：Linux 命令行（快速验证用）

面试中提一嘴加分。虽然 100GB 跑起来很慢，但逻辑上完全 work：

```bash
# 2. 排序 + 去重（同一个 driver+order 只保留一行）
# 4. 统计每个 driver_id 出现次数
# 6. 取前 10

cut -d',' -f1,2 huge_log.csv \
  | sort -u \
  | cut -d',' -f1 \
  | sort \
  | uniq -c \
  | sort -rn \
  | head -10
```

一行搞定逻辑，面试里能秀一下。但实际 100GB 文件，`sort` 命令的外部排序会很慢（不过 GNU sort 默认就支持外部排序，会自动用临时文件）。

---

## 各方案对比

| 方案 | 时间复杂度 | 内存占用 | 实现复杂度 | 适用场景 |
|------|----------|---------|----------|---------|
| **哈希分治** | O(n) 两遍扫 | ~1GB | 中 | 单机、面试首选 |
| **外部排序** | O(n log n) | O(1) 计数阶段 | 中 | 内存极度受限 |
| **MapReduce** | O(n) | 分布式 | 高 | 数据量更大、有集群 |
| **命令行** | O(n log n) | 系统管理 | 低 | 快速验证 |

---

## 面试回答要点

如果我是面试者，我会这么组织答案：

> 100GB 文件放不进内存，核心思路是**哈希分治**。
>
> 第一步，逐行读取文件，对 driver_id 做哈希取模，分散写入 N 个小文件。这样同一个车主的所有记录一定在同一个文件里。
>
> 第二步，逐个处理小文件。每个文件全部加载到内存，用 `HashMap<driver_id, HashSet<order_id>>` 做去重和计数。HashSet 保证相同 driver_id + order_id 的不同状态只算一次。
>
> 第三步，维护一个大小为 10 的小顶堆，所有小文件的结果都过一遍堆，最终堆里就是全局 Top 10。
>
> 如果某个小文件还是太大放不进内存（哈希倾斜），可以对这个文件换一个哈希函数再分一次。如果数据量更大、单机处理不了，就上 MapReduce，Mapper 输出 `<driverId, orderId>`，Reducer 里用 HashSet 去重后计数。

### 面试官可能追问的点

**Q：为什么用小顶堆不用大顶堆？**
> 小顶堆的堆顶是最小的，新元素比堆顶大就替换进来，保持堆大小始终为 K。最终堆里就是最大的 K 个。如果用大顶堆，你得把所有数据都放进去再取 K 个，空间不够。

**Q：如果哈希分片后某个文件特别大怎么办？**
> 说明某些 driver_id 特别集中（哈希倾斜），对这个大文件换一个哈希函数，按 `driver_id + order_id` 再分一次。或者直接增大分片数 N。

**Q：HashSet 存 orderId 太多内存不够呢？**
> 可以用两阶段方案：第一阶段按 `driver_id + order_id` 组合哈希分片后去重，只输出 driver_id；第二阶段再按 driver_id 分片计数。或者用布隆过滤器做近似去重（有极小误差但省内存）。

**Q：能不能用数据库？**
> 可以。把 100GB 文件用 LOAD DATA 导入 MySQL/ClickHouse，然后一条 SQL 搞定：`SELECT driver_id, COUNT(DISTINCT order_id) AS cnt FROM logs GROUP BY driver_id ORDER BY cnt DESC LIMIT 10`。但面试官考的是你对分治和数据结构的理解，不是让你用数据库。

---

## 小结

这道题的本质是三个子问题的组合：**大文件不能一次加载（→ 分治）**、**去重（→ HashSet / 排序相邻去重）**、**Top K（→ 小顶堆）**。把这三个单独拎出来都不难，组合在一起就是一道综合考查基本功的好题。面试时重点讲清楚"为什么按 driver_id 哈希分片"和"为什么用小顶堆"就够了。
