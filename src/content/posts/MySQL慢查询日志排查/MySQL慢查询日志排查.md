---
title: MySQL 慢查询日志排查：从开启到优化的完整套路
description: "之前线上出过一次接口响应变慢的问题，最后定位到是几条 SQL 拖慢了整个服务。排查过程中把慢查询日志这块系统地学了一遍，发现这东西是 MySQL 性能调优的第一步，不会用的话后面什么 EXPLAIN、加索引都无从下手。"
published: 2025-12-21
tags:
    - MySQL
    - 性能优化
    - 慢查询
category:
    - 数据库
draft: false
---
> 之前线上出过一次接口响应变慢的问题，最后定位到是几条 SQL 拖慢了整个服务。排查过程中把慢查询日志这块系统地学了一遍，发现这东西是 MySQL 性能调优的第一步，不会用的话后面什么 EXPLAIN、加索引都无从下手。

## 慢查询日志是什么？

简单说：**MySQL 自带的"监控摄像头"**，专门记录执行时间超过阈值的 SQL 语句。

默认是关闭的（怕影响性能），需要手动开启。一旦打开，所有执行超过 `long_query_time` 的查询都会被记录下来，包括 SQL 文本、执行时间、扫描行数等信息。

## 第一步：开启慢查询日志

### 临时开启（重启失效）

```sql
-- 查看当前状态
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';

-- 设置阈值（单位：秒），超过这个时间的 SQL 会被记录
SET GLOBAL long_query_time = 1;

-- 设置日志文件位置
SET GLOBAL slow_query_log_file = '/var/log/mysql/mysql-slow.log';

-- 额外：把没用到索引的查询也记录下来（很有用！）
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

> 注意：`SET GLOBAL` 修改后，**当前已有的连接不会立即生效**，需要新建连接才能看到效果。这个坑我踩过——改完之后测试发现没记录，还以为哪里配错了，其实是因为用的还是旧连接。

### 永久开启（写配置文件）

编辑 `my.cnf`（Linux 通常在 `/etc/mysql/my.cnf` 或 `/etc/my.cnf`）：

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

改完重启 MySQL 服务生效。

### 阈值设多少合适？

没有标准答案，看场景：

| 场景 | 建议阈值 | 说明 |
|------|---------|------|
| 一般业务 | 2 秒 | 大部分 OLTP 查询应该在毫秒级完成 |
| 高并发系统 | 1 秒 | 稍微慢一点就会拖垮连接池 |
| 深度排查 | 0.1 秒 | 激进模式，能抓到更多潜在问题 |
| 全量分析 | 0 秒 | 记录所有 SQL，仅用于短时间抓取，别长开 |

> `long_query_time = 0` 的时候要特别小心，高并发场景下日志文件会暴涨

## 第二步：看懂慢查询日志

打开日志文件，一条慢查询记录长这样：

```
# User@Host: app_user[app_user] @ [192.168.1.100]  Id: 12345
SET timestamp=1710238625;
SELECT * FROM orders WHERE YEAR(created_time) = 2025 AND status = 'paid'
  ORDER BY amount DESC LIMIT 100;
```

每个字段的含义：

- **Query_time: 3.456789** — 执行耗时 3.45 秒，这就是被记录的原因
- **Lock_time: 0.000123** — 等待锁的时间，这里基本可以忽略
- **Rows_sent: 42** — 最终返回给客户端的行数
- **Rows_examined: 1523678** — MySQL 为了找到这 42 行，**实际扫描了 150 万行**

> **Rows_examined 和 Rows_sent 的比值是一个非常关键的指标。** 上面这条 SQL，扫了 150 万行才返回 42 行，比值大约 36000:1——基本可以断定缺索引了。理想情况下这个比值应该尽量接近 1:1。

## 第三步：用工具分析日志

生产环境的慢查询日志可能有成千上万条，一条条看不现实。需要工具帮忙汇总分析。

### mysqldumpslow（MySQL 自带）

```bash
mysqldumpslow -s at -t 10 /var/log/mysql/mysql-slow.log

mysqldumpslow -s c -t 10 /var/log/mysql/mysql-slow.log

mysqldumpslow -s t -t 10 /var/log/mysql/mysql-slow.log
```

`-s` 的排序选项：
- `t` — 按总查询时间
- `at` — 按平均查询时间
- `c` — 按出现次数
- `r` — 按返回行数
- `ar` — 按平均返回行数

`mysqldumpslow` 会把具体参数值替换成 `S`（字符串）和 `N`（数字），方便合并相同模式的查询。输出大概长这样：

```
Count: 3421  Time=2.35s (8039s)  Lock=0.00s (1s)  Rows=12.3 (42076)
  SELECT * FROM orders WHERE status = 'S' AND user_id = N ORDER BY created_time DESC
```

意思是：这条查询模式出现了 3421 次，平均每次 2.35 秒，总共耗时 8039 秒。**这种高频慢查询是优先要干掉的**。

## 第四步：用 EXPLAIN 分析具体 SQL

找到了"嫌疑犯"之后，用 EXPLAIN 看它的执行计划，搞清楚慢在哪。

```sql
EXPLAIN SELECT * FROM orders
WHERE YEAR(created_time) = 2025 AND status = 'paid'
ORDER BY amount DESC LIMIT 100;
```

输出：

```
+----+-------------+--------+------+---------------+------+---------+------+---------+-----------------------------+
| id | select_type | table  | type | possible_keys | key  | key_len | rows | filtered | Extra                       |
+----+-------------+--------+------+---------------+------+---------+------+---------+-----------------------------+
|  1 | SIMPLE      | orders | ALL  | NULL          | NULL | NULL    |1523678| 10.00  | Using where; Using filesort |
+----+-------------+--------+------+---------------+------+---------+------+---------+-----------------------------+
```

### 重点看哪几列？

**`type` 列 —— 访问类型（最重要！）**

从好到差排列：

```
const > eq_ref > ref > range > index > ALL
  最快                                  最慢
```

- **const**：通过主键或唯一索引直接定位一行，最快
- **eq_ref**：JOIN 时通过唯一索引匹配，每次只读一行
- **ref**：用非唯一索引查找，可能返回多行
- **range**：索引范围扫描（如 `BETWEEN`、`>`、`<`、`IN`）
- **index**：扫描整个索引树（比 ALL 好一点，但还是慢）
- **ALL**：全表扫描，最慢。**看到 ALL 基本就是要优化的信号**

上面那条 SQL 的 `type = ALL`，全表扫描 150 万行，难怪慢。

**`key` 列 —— 实际用了哪个索引**

`key = NULL` 说明没有用到任何索引。结合 `possible_keys = NULL` 来看，是 MySQL 压根就找不到能用的索引。

**`Extra` 列 —— 额外信息**

- `Using where`：用了 WHERE 条件过滤（正常）
- `Using index`：覆盖索引，不需要回表（好事）
- `Using filesort`：需要额外排序操作（**危险信号**，可能触发磁盘排序）
- `Using temporary`：用了临时表（**危险信号**，常见于 GROUP BY、DISTINCT）

**`rows` 列 —— 预估扫描行数**

这是优化器的估算值，跟实际可能有差距。MySQL 8.0+ 可以用 `EXPLAIN ANALYZE` 看到真实执行数据：

```sql
EXPLAIN ANALYZE SELECT * FROM orders
WHERE YEAR(created_time) = 2025 AND status = 'paid'
ORDER BY amount DESC LIMIT 100;
```

> `EXPLAIN ANALYZE` 会**真正执行**这条 SQL（不只是估算），所以对于非常慢的查询要谨慎使用，别在生产高峰期跑。

## 第五步：常见慢查询模式及优化

### 模式一：对索引列使用函数

```sql
-- 慢！YEAR() 函数导致索引失效
SELECT * FROM orders WHERE YEAR(created_time) = 2025;

-- 优化：改成范围查询，能走索引
SELECT * FROM orders
WHERE created_time >= '2025-01-01' AND created_time < '2026-01-01';
```

> 这是最常见的索引失效场景之一。**在 WHERE 条件中对字段使用函数，会导致该字段上的索引无法使用。** 包括 `YEAR()`、`DATE()`、`SUBSTR()`、`UPPER()` 这些都一样。

### 模式二：隐式类型转换

```sql
-- phone 字段是 varchar，但传了个数字进来
-- MySQL 会对整列做隐式转换，索引失效
SELECT * FROM users WHERE phone = 13800138000;

-- 优化：确保类型一致
SELECT * FROM users WHERE phone = '13800138000';
```

### 模式三：LEFT JOIN 缺少索引

```sql
-- 如果 order_items.order_id 没有索引，这个 JOIN 会很慢
SELECT o.*, oi.product_name
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = 12345;
```

```sql
-- 确保关联字段上有索引
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

### 模式四：LIKE 左模糊

```sql
-- 左模糊走不了索引
SELECT * FROM products WHERE name LIKE '%手机%';

-- 如果必须做全文搜索，考虑全文索引或 ES
ALTER TABLE products ADD FULLTEXT INDEX ft_name(name);
SELECT * FROM products WHERE MATCH(name) AGAINST('手机');
```

### 模式五：SELECT * 回表浪费

```sql
-- 查了所有列，即使有索引也得回表取数据
SELECT * FROM orders WHERE status = 'paid';

-- 优化：只查需要的列，如果能用覆盖索引就更好
SELECT id, order_no, amount FROM orders WHERE status = 'paid';

-- 建一个覆盖索引，连回表都省了
CREATE INDEX idx_status_cover ON orders(status, id, order_no, amount);
```

### 模式六：ORDER BY 导致 filesort

```sql
-- status 有索引，但 amount 没有，排序得额外做 filesort
SELECT * FROM orders WHERE status = 'paid' ORDER BY amount DESC;

-- 优化：建联合索引，让查询和排序都走索引
CREATE INDEX idx_status_amount ON orders(status, amount);
```

## 第六步：加完索引，验证效果

```sql
-- 加索引
CREATE INDEX idx_created_status ON orders(created_time, status);

-- 重新 EXPLAIN
EXPLAIN SELECT * FROM orders
WHERE created_time >= '2025-01-01' AND created_time < '2026-01-01'
  AND status = 'paid'
ORDER BY amount DESC LIMIT 100;
```

优化后预期看到：
- `type` 从 `ALL` 变成 `range`
- `key` 显示用到了新建的索引
- `rows` 从百万级降到几千或更少
- `Extra` 里不再有 `Using filesort`（如果排序字段也在索引里的话）

> 加完索引后记得跑一下 `ANALYZE TABLE orders;` 更新统计信息，不然优化器可能还是用旧的统计数据做决策。

### MySQL 8.0 的隐藏索引（小技巧）

不确定索引有没有用？不用直接 DROP，可以先设成 invisible：

```sql
-- 把索引设为不可见（优化器不会用它，但索引还在）
ALTER TABLE orders ALTER INDEX idx_old_index INVISIBLE;

-- 观察一段时间没问题，再真正删掉
DROP INDEX idx_old_index ON orders;

-- 如果发现有影响，随时恢复
ALTER TABLE orders ALTER INDEX idx_old_index VISIBLE;
```

## 完整排查流程（一图流）

```
发现接口慢 / 收到告警
       │
       ▼
开启慢查询日志（long_query_time = 1）
       │
       ▼
收集一段时间（几小时~一天）
       │
       ▼
pt-query-digest 分析日志
       │
       ▼
找到 Top N 慢查询（按 Response time 排名）
       │
       ▼
逐条 EXPLAIN，看 type/key/rows/Extra
       ↓
  type=ALL? key=NULL? → 加索引 / 改写查询
       ↓
验证：重新 EXPLAIN + 实际查询耗时
       │
       ▼
ANALYZE TABLE 更新统计信息
       │
       ▼
持续监控（别把慢查询日志关了）
```

## 小结

慢查询排查的套路其实挺固定的：开日志 → 工具分析找到 Top N → EXPLAIN 定位原因 → 加索引或改写 SQL → 验证。核心就是 **Rows_examined 和 Rows_sent 的比值** + **EXPLAIN 里 type 列**这两个指标，大部分慢查询问题看这两个就能定位到方向。
