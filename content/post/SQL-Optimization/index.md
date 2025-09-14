---
title: SQL优化
description: 索引设计（联合索引、覆盖索引）、执行计划分析、慢查询优化
date: 2024-09-01T01:02:34+08:00
lastmod: 2024-09-01T01:02:34+08:00
slug: SQL-Optimization

tags:
  - sql
categories:
  - 系统优化
---

## 索引设计

索引是加速查询的“高速公路”，但设计不当反而成为“堵车源头”。索引设计主要围绕两个核心概念：**联合索引（Composite Index）** 和 **覆盖索引（Covering Index）**。

---

### 联合索引（Composite Index / Multi-column Index）

#### 什么是联合索引？

在多个字段上建立的一个索引。例如：

```sql
CREATE INDEX idx_user_age_city ON users(age, city);
```

这个索引可以加速以下查询：

```sql
SELECT * FROM users WHERE age = 25 AND city = 'Beijing';
SELECT * FROM users WHERE age = 25;  --   可用（最左前缀原则）
SELECT * FROM users WHERE city = 'Beijing'; --  不可用（跳过了age）
```

#### 最左前缀原则（Leftmost Prefix Principle）

MySQL（InnoDB）的 B+树索引要求查询必须从索引的最左列开始，不能跳过中间列。

有效使用：

- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b = ? AND c = ?`

无效使用：

- `WHERE b = ?`
- `WHERE c = ?`
- `WHERE b = ? AND c = ?`

#### 常见问题

> “如果我建了 (a, b, c) 联合索引，WHERE b = ? AND c = ? 能用上索引吗？”

→ **不能！** 因为跳过了最左列 a，违反最左前缀原则。

> “那如果 WHERE a > 10 AND b = 5 呢？”

→ **a 可用，b 部分可用**。因为 a 是范围查询（>），b 虽然在索引中，但无法高效利用（因为 B+树在 a>10 后是无序的）。

**优化建议**：把等值条件放前面，范围条件放后面。如：`(status, created_at)` 比 `(created_at, status)` 更高效，如果 status 是等值筛选。

---

### 覆盖索引（Covering Index）

#### 什么是覆盖索引？

如果一个索引包含了查询所需的所有字段，数据库就**不需要回表**（即不需要再去主键索引查完整行数据），这种索引叫“覆盖索引”。

#### 举例：

```sql
-- 表结构
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    city VARCHAR(50),
    email VARCHAR(100)
);

-- 建立联合索引
CREATE INDEX idx_age_city_name ON users(age, city, name);
```

查询：

```sql
--   覆盖索引：所有字段都在索引里，无需回表
SELECT name FROM users WHERE age = 25 AND city = 'Beijing';

--  非覆盖索引：email 不在索引中，必须回表
SELECT email FROM users WHERE age = 25 AND city = 'Beijing';
```

#### 为什么“回表”影响性能？

- InnoDB 中，普通索引叶子节点存的是**主键值**，查到主键后还要去**聚簇索引（主键索引）** 中查完整行 → 多一次 I/O。
- 覆盖索引避免了这次 I/O，极大提升性能，尤其在高并发场景。

#### 常见问题

> “什么是覆盖索引？有什么好处？”

→ 覆盖索引是包含查询所有字段的索引，避免回表，减少 I/O，提升查询性能。

> “如何判断一个查询是否使用了覆盖索引？”

→ 用 `EXPLAIN` 查看执行计划，如果 `Extra` 列显示 **“Using index”**，说明使用了覆盖索引。

---

## 执行计划分析（EXPLAIN）

这是 SQL 优化的“显微镜”，让你知道 MySQL**到底怎么执行你的 SQL**。

### 基本用法：

```sql
EXPLAIN SELECT * FROM users WHERE age = 25 AND city = 'Beijing';
```

### 关键字段解读：

| 字段名          | 含义说明                                                                                                                                                                              |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`            | 查询序号，越大越先执行（子查询时有用）                                                                                                                                                |
| `select_type`   | 查询类型：SIMPLE / PRIMARY / SUBQUERY 等                                                                                                                                              |
| `table`         | 操作的表名                                                                                                                                                                            |
| `type`          | **访问类型，最重要！** 从优到劣：system > const > eq_ref > ref > range > index > ALL                                                                                                  |
| `possible_keys` | 可能用到的索引                                                                                                                                                                        |
| `key`           | **实际使用的索引**                                                                                                                                                                    |
| `key_len`       | 使用索引的长度（可用于判断联合索引用了几列）                                                                                                                                          |
| `rows`          | 预估扫描行数 → 越小越好                                                                                                                                                               |
| `Extra`         | 额外信息，重点关注：<br> - **Using index**（覆盖索引）<br> - **Using where**（服务器层过滤）<br> - **Using filesort**（需排序，性能差）<br> - **Using temporary**（用临时表，性能差） |

### 常见问题

> “EXPLAIN 中 type 为 ALL 是什么意思？”

→ 全表扫描，性能最差，通常说明**没走索引**，需要优化。

> “Extra 出现 Using filesort 怎么办？”

→ 表示 MySQL 无法利用索引完成排序，需额外排序操作。优化方法：

- 建立合适的联合索引，把 `ORDER BY` 字段包含进去
- 避免 SELECT \*，只选必要字段减少排序开销

> “key_len 怎么看用了联合索引的哪几列？”

→ 举例：`idx(a, b, c)`，a 是 int(4 字节)，b 是 varchar(50) utf8mb4（50\*4 + 2 = 202 字节），c 是 tinyint(1 字节)

- 如果 key_len = 4 → 只用了 a
- 如果 key_len = 4 + 202 = 206 → 用了 a 和 b
- 如果 key_len = 207 → 用了 a、b、c

（注意：变长字段如 varchar 有长度前缀 2 字节，允许 NULL 的字段会多 1 字节）

---

## 慢查询优化

慢查询是系统性能杀手。优化流程一般为：

> **发现慢查询 → 分析执行计划 → 优化索引或 SQL → 验证效果**

### 1. 如何发现慢查询？

#### 开启慢查询日志：

```sql
-- 查看是否开启
SHOW VARIABLES LIKE 'slow_query_log';

-- 设置阈值（秒）
SET GLOBAL long_query_time = 1;

-- 日志文件位置
SHOW VARIABLES LIKE 'slow_query_log_file';
```

也可使用 **pt-query-digest** 工具分析慢日志。

### 2. 常见慢查询原因及优化方案

| 问题类型                | 原因                                    | 优化方案                                 |
| ----------------------- | --------------------------------------- | ---------------------------------------- |
| 全表扫描（type=ALL）    | 无索引或索引失效                        | 建合适索引，避免函数操作字段             |
| Using filesort          | ORDER BY 无法用索引                     | 联合索引包含排序字段                     |
| Using temporary         | GROUP BY / DISTINCT 无合适索引          | 建联合索引，或调整 SQL 逻辑              |
| 索引失效                | 对字段使用函数、隐式类型转换、OR 条件等 | 避免函数、保持类型一致、用 UNION 替代 OR |
| 扫描行数过多（rows 大） | 索引选择性差（如性别字段）              | 调整索引顺序，或建立更精准的筛选条件索引 |
| JOIN 无索引             | 关联字段无索引                          | 在 JOIN 字段上建立索引                   |
| 子查询性能差            | 相关子查询逐行执行                      | 改写为 JOIN 或派生表                     |

### 实战案例：

#### 低效写法：

```sql
SELECT * FROM orders
WHERE DATE(create_time) = '2024-06-01';
```

→ 对字段使用函数，索引失效！

#### 优化写法：

```sql
SELECT * FROM orders
WHERE create_time >= '2024-06-01 00:00:00'
  AND create_time < '2024-06-02 00:00:00';
```

→ 走索引，效率提升数十倍！

---

### 高级优化技巧

- **索引下推（ICP, Index Condition Pushdown）**：MySQL 5.6+，在存储引擎层过滤，减少回表。
- **Multi-Range Read（MRR）**：优化回表时的随机 I/O，转为顺序 I/O。
- **调整 join_buffer_size / sort_buffer_size**：适当增大减少磁盘临时表。
- **分页优化**：避免 `LIMIT 1000000, 10`，改用游标或延迟关联：

```sql
--  慢
SELECT * FROM users ORDER BY id LIMIT 1000000, 10;

--  快（前提是 id 有序且连续）
SELECT * FROM users WHERE id > 1000000 ORDER BY id LIMIT 10;
```

## 总结

> **索引三宝：最左前缀、覆盖索引、避免回表**  
> **执行计划：看 type、看 key、看 Extra**  
> **慢查优化：开日志、看执行、建索引、改 SQL**
