## mysql相关优化（索引维度）

### 建表测试

先设计几张表，用户表 `users`，商品表 `products`，订单表 `orders`,订单明细表 `order_items`

用户表 `users`

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '用户ID',
    username VARCHAR(64) NOT NULL UNIQUE COMMENT '用户名',
    email VARCHAR(128) NOT NULL COMMENT '邮箱',
    phone VARCHAR(20) DEFAULT NULL COMMENT '手机号',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '注册时间'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


```

商品表 `products`

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '商品ID',
    name VARCHAR(128) NOT NULL COMMENT '商品名称',
    category VARCHAR(64) DEFAULT NULL COMMENT '分类',
    price DECIMAL(10, 2) NOT NULL COMMENT '价格',
    stock INT NOT NULL DEFAULT 0 COMMENT '库存',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '上架时间',
    INDEX idx_category_price (category, price)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


```

订单表 `orders`

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '订单ID',
    user_id BIGINT NOT NULL COMMENT '下单用户ID',
    order_no VARCHAR(64) NOT NULL UNIQUE COMMENT '订单号',
    status TINYINT NOT NULL DEFAULT 0 COMMENT '订单状态 0-待支付 1-已支付 2-已发货 3-完成',
    total_amount DECIMAL(10,2) NOT NULL COMMENT '订单总额',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '下单时间',
    INDEX idx_user_created (user_id, created_at),
    FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

订单明细表 `order_items`

```sql
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '明细ID',
    order_id BIGINT NOT NULL COMMENT '订单ID',
    product_id BIGINT NOT NULL COMMENT '商品ID',
    quantity INT NOT NULL COMMENT '购买数量',
    price DECIMAL(10,2) NOT NULL COMMENT '单价',
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    INDEX idx_order_product (order_id, product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 生成大量数据以供测试

生成测试的数据，可以使用sql脚本，python脚本等

批量生成 `users` 数据（10 万条）

```sql
START TRANSACTION;

INSERT INTO users (username, email, phone, created_at)
SELECT
  CONCAT('user_', LPAD(id, 6, '0')),
  CONCAT('user_', LPAD(id, 6, '0'), '@example.com'),
  CONCAT('13', LPAD(FLOOR(RAND()*100000000), 8, '0')),
  NOW() - INTERVAL FLOOR(RAND()*365) DAY
FROM (
  SELECT @rownum := @rownum + 1 AS id
  FROM information_schema.COLUMNS a, information_schema.COLUMNS b, (SELECT @rownum := 0) r
  LIMIT 100000
) AS temp;

COMMIT;

```

批量生成 `products` 数据（1000 条）

```sql
START TRANSACTION;

INSERT INTO products (name, category, price, stock, created_at)
SELECT
  CONCAT('商品_', LPAD(id, 4, '0')),
  ELT(FLOOR(1 + (RAND() * 5)), '数码', '图书', '服饰', '家居', '食品'),
  ROUND(50 + RAND() * 950, 2),
  FLOOR(100 + RAND() * 900),
  NOW() - INTERVAL FLOOR(RAND()*180) DAY
FROM (
  SELECT @pid := @pid + 1 AS id
  FROM information_schema.COLUMNS a, (SELECT @pid := 0) r
  LIMIT 1000
) AS temp;

COMMIT;

```

批量生成 `orders` 数据（每用户 1~3 个订单）

```sql
-- 模拟循环插入
DELIMITER $$

CREATE PROCEDURE generate_orders()
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < 200000 DO
    INSERT INTO orders (user_id, order_no, status, total_amount, created_at)
    SELECT
      FLOOR(1 + RAND() * 100000),
      CONCAT('ORD_', LPAD(FLOOR(RAND() * 10000000), 7, '0')),
      FLOOR(RAND() * 4),
      ROUND(50 + RAND() * 1000, 2),
      NOW() - INTERVAL FLOOR(RAND() * 60) DAY;
    SET i = i + 1;
  END WHILE;
END$$

DELIMITER ;

CALL generate_orders();

```

批量生成 `order_items` 数据（每订单 1~3 个商品）

```sql
START TRANSACTION;

INSERT INTO order_items (order_id, product_id, quantity, price)
SELECT
  FLOOR(1 + (RAND() * 200000)), -- 订单 ID 范围
  FLOOR(1 + (RAND() * 1000)),   -- 商品 ID 范围
  FLOOR(1 + RAND()*5),
  ROUND(50 + RAND()*950, 2)
FROM (
  SELECT @oiid := @oiid + 1 AS id
  FROM information_schema.COLUMNS a, information_schema.COLUMNS b, (SELECT @oiid := 0) r
  LIMIT 500000
) AS temp;

COMMIT;

```

## 开始测试相关优化项

我们尝试执行一下

```sql
EXPLAIN SELECT * FROM users WHERE phone = '1302706965';
```

发现其执行计划如下：

![Image](https://github.com/user-attachments/assets/260cd39e-cf04-44d9-99b0-ea111a5f0533)

###  `EXPLAIN` 字段详细解释

| 字段名            | 含义                                             | 优化建议                               |
| ----------------- | ------------------------------------------------ | -------------------------------------- |
| **id**            | 查询中语句块的标识编号。值越大，优先执行。       | 用于区分子查询和多表操作               |
| **select_type**   | 查询类型，如 SIMPLE、PRIMARY、SUBQUERY 等        | 标识 SQL 中各部分结构                  |
| **table**         | 当前正在访问的表名或别名                         | 逐行显示各表的处理方式                 |
| **partitions**    | 查询涉及的分区（如果启用了分区表）               | 不涉及分区则为空                       |
| **type**          | 表访问类型（连接类型），衡量性能优劣的关键字段   | 越靠近 `ALL` 性能越差                  |
| **possible_keys** | 可能用于优化该查询的索引列表                     | 表示 MySQL 考虑的索引                  |
| **key**           | 实际使用到的索引名                               | 为空则表示没有使用索引                 |
| **key_len**       | 使用的索引长度（字节）                           | 越短越快，完整匹配通常更优             |
| **ref**           | 哪些列或常量被用于查找索引值                     | 通常是 `const` 或 `column_name`        |
| **rows**          | 预估扫描的行数（不是结果行）                     | 值越小，性能越好                       |
| **filtered**      | 表中行经过 WHERE 条件过滤后的估算百分比          | 越高说明条件越精准                     |
| **Extra**         | 额外信息，如是否使用文件排序、临时表、索引覆盖等 | 查看是否使用了临时表、回表、排序等操作 |



------

###  `type`（连接类型）由好到差排序

| 类型       | 含义与性能                         | 示例场景                          |
| ---------- | ---------------------------------- | --------------------------------- |
| **system** | 表只有一行（system 表）            | 少见                              |
| **const**  | 主键或唯一索引等值匹配             | `WHERE id = 1`                    |
| **eq_ref** | 多表 JOIN 且唯一索引匹配           | `JOIN ON id = ...` 且唯一索引     |
| **ref**    | 普通索引等值匹配                   | `WHERE user_id = 100`             |
| **range**  | 范围查询，如 `>`, `<`, `BETWEEN`   | `WHERE created_at > '2025-01-01'` |
| **index**  | 全索引扫描（比全表好，但仍不理想） | 覆盖索引，但无 WHERE              |
| **ALL**    | 全表扫描（最差）                   | 无索引匹配或 LIKE 前缀查询        |



------

### `Extra` 常见取值说明与优化建议

| Extra 信息        | 含义                                          | 优化建议                   |
| ----------------- | --------------------------------------------- | -------------------------- |
| Using index       | 使用了覆盖索引（无需回表）                    | 非常好                     |
| Using where       | 使用了 WHERE 过滤条件                         | 可接受                     |
| Using temporary   | 使用了临时表，通常出现在 GROUP BY、ORDER BY   | 尽量避免                   |
| Using filesort    | 使用了文件排序（非索引排序）                  | 应优化 ORDER BY 字段加索引 |
| Using join buffer | 联合查询使用了连接缓存，说明没有索引支持 JOIN | 加索引                     |
| Impossible WHERE  | 条件恒为假，SQL 可优化                        | 检查条件逻辑               |
| NULL              | 未使用任何附加操作                            | 通常表示性能较好           |

### EXPLAIN 优化目标：

- `type` 至少 `ref` 以上；
- `rows` 越小越好；
- `Extra` 避免 `Using filesort`, `Using temporary`；
- 覆盖索引时出现 `Using index` 是好事。

### 分析

我们根据以上的相关字符含义得出，我们这条sql效率非常差，无论消耗资源还是其他，cpu占用等。

#### MySQL 常见索引类型对比表

| 索引类型                    | 定义                       | 是否唯一       | 支持多列           | 匹配方式              | 适用场景                   | 限制 / 注意事项                                              |
| --------------------------- | -------------------------- | -------------- | ------------------ | --------------------- | -------------------------- | ------------------------------------------------------------ |
| **普通索引 (INDEX)**        | 默认索引类型，无唯一性限制 | ❌ 否           | ✅ 是               | 精确、范围、LIKE 前缀 | 查询加速，最常用           | 可重复值，需关注选择性                                       |
| **唯一索引 (UNIQUE)**       | 不允许列值重复             | ✅ 是           | ✅ 是               | 同普通索引            | 唯一性约束，如邮箱、手机号 | 插入重复会报错；NULL 可重复                                  |
| **主键索引 (PRIMARY)**      | 唯一 + 非空的索引列        | ✅ 是           | ❌ 否               | 精确匹配              | 主键查找，如 ID 主键       | 每表只能有一个，自动唯一                                     |
| **复合索引 (Multi-column)** | 多列组成一个索引           | ❌/✅ 取决于定义 | ✅ 是               | 最左前缀匹配原则      | 多条件查询联合加速         | 必须按“最左匹配”使用顺序优化                                 |
| **全文索引 (FULLTEXT)**     | 基于文本内容倒排索引       | ❌ 否           | ✅ 是（MySQL 5.7+） | 自然语言 / 布尔模式   | 文本搜索，如文章标题、内容 | 仅支持 `MyISAM`/`InnoDB`（5.6+），适用于英文                 |
| **空间索引 (SPATIAL)**      | 用于地理空间数据           | ❌ 否           | ✅ 是               | 空间距离等计算        | GIS 类型字段查询           | 仅适用于 `MyISAM`（旧版）或 `InnoDB`（新版），字段类型需为 `GEOMETRY` |

我们试着给phone添加普通索引再次尝试

```sql
ALTER TABLE users ADD INDEX idx_phone (phone);
EXPLAIN SELECT * FROM users WHERE phone = '1302706965';
```

添加各种索引的相关语句

```sql
ALTER TABLE users ADD UNIQUE INDEX idx_email (phone); -- 唯一索引
ALTER TABLE users ADD PRIMARY KEY (id);-- 主键索引
ALTER TABLE articles ADD FULLTEXT INDEX idx_content (content); -- 全文索引
ALTER TABLE locations ADD SPATIAL INDEX idx_position (position); -- 空间索引
ALTER TABLE users ADD INDEX idx_username_prefix (username(10)); -- 前缀索引
ALTER TABLE orders ADD INDEX idx_user_product (user_id, product_id); -- 复合索引
```



可以看到执行计划如下：

![Image](https://github.com/user-attachments/assets/1d48c2fb-7c2d-4cd5-9f60-b6dced4619b7)

可以看到其效率有一个质的提升。

我们接下来试着修改相关查询改为模糊查询，以手机号后四位进行模糊查询：

```sql
explain SELECT * FROM users WHERE phone LIKE '%6965'
```

执行计划为：

![Image](https://github.com/user-attachments/assets/695a28ad-1575-4c45-a85c-23b6683c1541)

可以看到又再次退化到没有索引的状态，我们试着添加全文索引尝试

```sql
ALTER TABLE users ADD FULLTEXT INDEX idx_content_phone(phone);
SELECT phone, MATCH(phone) AGAINST('6965') as score
FROM users
WHERE MATCH(phone) AGAINST('6965' IN NATURAL LANGUAGE MODE);
```

可能会出现查询不到的情况，下面我们来介绍一下全文索引

#### **全文索引详解：**

全文索引（FULLTEXT INDEX）在 MySQL 中是一种特殊索引，专门用于处理**自然语言文本搜索**（如文章内容、评论等）。但它并不是通用优化工具，对于手机号、编码、ID 等结构化字段并不合适。

以下是它的 **使用限制、性能消耗** 及场景总结：

------

#####  一、全文索引的使用限制

| 限制项                  | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| 支持字段类型            | 仅支持 `CHAR`、`VARCHAR`、`TEXT` 类型字段（不能用于 `INT`）  |
| 支持存储引擎            | 仅支持 `InnoDB` 和 `MyISAM`（MyISAM 支持较早，但已过时）     |
| 查询语法限制            | 只能配合 `MATCH(...) AGAINST(...)` 使用，**不能用于 `LIKE` 查询** |
| 最小索引词长度          | 默认 `InnoDB` 为 3，`MyISAM` 为 4，小于该长度的词不会被索引（可调） |
| 停用词限制（Stopwords） | 默认会忽略常见词（如 “the”, “this” 等），可以被自定义或清空  |
| 中文支持差              | 默认按空格分词，不支持中文，需要第三方分词插件（如 [Mroonga](https://mroonga.org/)） |
| 查询限制                | 不能用于 `%xxx`、正则表达式、范围查询                        |
| 事务支持                | 仅 `InnoDB` 支持事务，`MyISAM` 不支持                        |

**示例：无效的 FULLTEXT 使用**

```
sql


复制编辑
SELECT * FROM users WHERE phone LIKE '%6965'; -- ❌ FULLTEXT 无法支持此写法
sql


复制编辑
SELECT * FROM users WHERE phone = '13912345678'; -- ❌ FULLTEXT 也不适合精确匹配
```

合法用法示例

```
sql复制编辑CREATE TABLE articles (
  id INT PRIMARY KEY,
  title VARCHAR(200),
  body TEXT,
  FULLTEXT(title, body)
) ENGINE=InnoDB;

SELECT * FROM articles
WHERE MATCH(title, body) AGAINST ('mysql fulltext search' IN NATURAL LANGUAGE MODE);
```

------

#####  **二、全文索引的性能消耗**

| 项目         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| 写入成本     | 每次插入/更新带有 FULLTEXT 字段的数据，会触发词拆分与索引构建，CPU 占用高 |
| 空间占用     | FULLTEXT 索引构建倒排索引，占用磁盘空间远高于 BTREE 索引     |
| 查询成本     | `MATCH ... AGAINST()` 查询在大文本上较快，但需要词匹配，有时结果不精确 |
| 不能覆盖索引 | FULLTEXT 索引**无法使用 `Using index` 优化**，每次查询仍需访问原数据行 |
| 查询不可控   | 查询分数是相关度评分，排序不稳定，可能导致分页复杂化         |
| 查询缓存失效 | 查询中 `MATCH ... AGAINST()` 的特性导致不容易被查询缓存命中  |



------

#####  三、性能优化建议（使用 FULLTEXT 时）

| 优化项                                | 说明                                 |
| ------------------------------------- | ------------------------------------ |
| 设置较小的 `innodb_ft_min_token_size` | 支持更短关键词（默认 3）             |
| 清空 `innodb_ft_stopword_table`       | 避免常见词被忽略                     |
| 使用 `BOOLEAN MODE`                   | 精确控制搜索（如 `+关键词 -排除词`） |
| 控制字段内容长度                      | 避免过长字段导致索引膨胀             |
| 增加 `WITH PARSER ngram` 插件         | 支持中文分词（如 Mroonga、ngram）    |



------

#####  四、使用场景建议总结

| 场景                             | 是否推荐使用 FULLTEXT | 替代方案                     |
| -------------------------------- | --------------------- | ---------------------------- |
| 长文本搜索（如文章、描述、评论） | 推荐                  | -                            |
| 手机号、邮箱、ID 模糊匹配        | 不推荐                | 后缀字段 + 索引              |
| 中文全文搜索                     | 默认不支持            | 推荐 Mroonga / ElasticSearch |
| 数据量大、查询实时性强           | 适用                  | 配合 BOOLEAN MODE 和分词调优 |
| 短词查询（<3 个字符）            | 默认不支持            | 调整参数后才可支持           |



------

#####  五、配置参考

```
ini复制编辑[mysqld]
# 最小词长度（InnoDB）
innodb_ft_min_token_size = 2
# 关闭停用词列表
innodb_ft_enable_stopword = OFF
```

执行后需：

```
sql复制编辑-- 删除旧索引
ALTER TABLE articles DROP INDEX ft_title;

-- 重建索引
ALTER TABLE articles ADD FULLTEXT(ft_title);
```

可见全文索引限制非常多，性能消耗也不小，对于我们上面的查询，我们可以使用一下方法。

#### 曲线救国

```sql
-- 添加冗余字段
ALTER TABLE users ADD COLUMN phone_suffix_right VARCHAR(10);
-- 设置该字段指，截取phone后四位
UPDATE users SET phone_suffix_right = RIGHT(phone, 4);
-- 添加索引
ALTER TABLE users ADD INDEX idx_phone_right (phone_suffix_right);
-- 查看执行计划
explain SELECT * FROM users WHERE phone_suffix_right = '6965'
```

执行计划如下：

![Image](https://github.com/user-attachments/assets/a8e172e7-b302-4ced-9151-921b3b83f599)

### 范围查询（部分使用索引）

```sql
SELECT * FROM users WHERE created_at >= '2025-06-01';
```

- 命中索引（`created_at` 上有索引时）；
- 查询大范围时仍可能带来大量回表或临时表。

**优化建议**：

- 限制字段数避免回表；
- 创建联合索引：`(created_at, id)`；
- 覆盖索引最理想：只查索引中的字段。

### 多条件组合（联合索引使用顺序）

```sql
SELECT * FROM users WHERE created_at = '2025-06-01' AND phone = '13900001111';
```

- 若索引为 `(created_at, phone)` 则命中；
- 若顺序反了 `(phone, created_at)`，可能不命中。

**优化建议**：

- 索引顺序遵循最左前缀匹配原则；
- 使用 `EXPLAIN` 检查 `key` 和 `rows`。

### JOIN 查询（关联效率分析）

```sql
SELECT o.id, u.username, o.total_amount
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at >= '2025-06-01';
```

- 外键字段上需有索引（`orders.user_id`）。
- `EXPLAIN` 检查 `type = ref` 或 `eq_ref`，`rows` 尽量小。

**优化建议**：

- 两边关联字段均加索引；
- 确保 `ON` 和 `WHERE` 条件可过滤。

### 聚合查询（索引覆盖优化）

```sql
SELECT COUNT(*) FROM orders WHERE status = 1;
```

- 若 `status` 有索引，能大幅减少扫描。
- 更佳：`COUNT(status)` + 覆盖索引。

### 分页查询（慢查询常见源头）

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10000, 10;
```

- 大偏移量分页会导致“跳过”前 `10000` 行记录；
- `EXPLAIN` 显示全表扫描 + filesort。

**优化建议**：

- 使用“延迟分页”：

```sql
SELECT * FROM orders
WHERE id > (SELECT id FROM orders ORDER BY created_at DESC LIMIT 10000, 1)
ORDER BY created_at DESC
LIMIT 10;
```

分页的相关优化可参考：[MySQL分页优化指南：告别LIMIT OFFSET的性能噩梦](https://mp.weixin.qq.com/s/iyXdKa4mOkqJWN0spN0DLA)

[京东面试：mysql深度分页 严重影响性能？根本原因是什么？如何优化？](https://mp.weixin.qq.com/s/LXvQMQFf_SyLWh1zI6onkA)

[史上最全MySQL各种锁详解](https://mp.weixin.qq.com/s/3PV5yF0ObuSfnw6pHXtKzA)

# 回表与索引覆盖

## 一、什么是回表？

### 回表（Row Lookup）指的是：

在使用**二级索引（非聚簇索引）\**查询时，若查询的字段\**不完全在索引中**，MySQL 需要：

1. 先根据索引定位到对应的主键值（`id`）；
2. 再用主键去主表（聚簇索引）中**“回表”**读取其它字段。

这一步叫**回表操作**。

------

##  二、什么是覆盖索引？

查询所需的所有字段都**包含在一个索引中**，则称为“覆盖索引”，此时**不需要回表**，可以直接从索引中返回结果。

##  三、如何查看是否发生了回表？

### 使用 `EXPLAIN` 看执行计划的 `Extra` 字段：

```sql
EXPLAIN SELECT name FROM users WHERE phone = '13912345678';
```

| id   | select_type | table | type | key       | key_len | rows | Extra             |
| ---- | ----------- | ----- | ---- | --------- | ------- | ---- | ----------------- |
| 1    | SIMPLE      | users | ref  | idx_phone | 23      | 1    | **Using index** ✅ |



#### `Extra` 字段说明：

| Extra 描述                          | 是否回表     | 说明                                                         |
| ----------------------------------- | ------------ | ------------------------------------------------------------ |
| `Using index`                       | ❌ 不回表     | 使用了**覆盖索引**，数据来自索引树                           |
| `Using where`                       | ✅ 通常会回表 | 用于过滤，但数据可能来自主表                                 |
| `Using index condition`             | ✅ 是         | 使用 **Index Condition Pushdown (ICP)**，部分条件在索引中处理，仍需回表 |
| 无 `Using index` 且无 `Using where` | ✅            | 更可能是全表或主键扫描                                       |

终极手段

MySQL 5.7+ 支持优化器 trace：

```
sql复制编辑SET optimizer_trace="enabled=on",end_markers_in_json=on;
SELECT * FROM users WHERE phone = '1353526525';
SELECT * FROM information_schema.optimizer_trace\G
```

可以在 `index_condition_pushdown` 和 `rowid lookup` 等字段中看到是否有回表行为。

## 四、如何避免回表？

| 方法                                   | 说明                               |
| -------------------------------------- | ---------------------------------- |
| 覆盖索引                               | 查询字段尽量包含在已存在的索引列中 |
| 使用 `SELECT 索引字段` 而非 `SELECT *` | 减少不必要字段，优化覆盖率         |
| 组合索引                               | 将多个常用字段放在一个联合索引中   |
| 业务查询尽量限制在索引字段范围内       | 提高命中率和可覆盖性               |

## 五、补充：查看慢查询是否因回表导致

虽然慢查询日志不直接显示“是否回表”，但你可以通过以下方法判断：

1. 使用 `EXPLAIN` 检查慢 SQL 是否命中 `Using index`；
2. 对慢查询运行 `SHOW PROFILE` 来观察 IO 消耗（需要先开启）；
3. 使用 `pt-query-digest` 识别高 IO 查询，看是否涉及非覆盖索引。

## 六、实用建议：是否优化为覆盖索引？

| 场景                            | 是否建议优化为覆盖索引            |
| ------------------------------- | --------------------------------- |
| 高频查询、字段较少              | ✅ 是，减少回表 IO                 |
| 查询字段仅主键                  | ❌ 不必优化                        |
| 查询字段不可能进索引（如 TEXT） | ❌ 不适合做覆盖索引                |
| 大数据量分页或排序场景          | ✅ 尽可能覆盖，避免回表 + filesort |

## 七、覆盖索引的“滥用”风险与误区

| **问题**                 | **说明**                                                     |
| ------------------------ | ------------------------------------------------------------ |
| 索引膨胀、占用空间       | 包含多个字段的复合索引体积大，可能占用大量磁盘和内存。       |
| 写入性能下降             | 每次 `INSERT/UPDATE/DELETE` 操作都需维护更多索引，增加写负担。 |
| 过度依赖，增加维护复杂度 | 查询变动会频繁要求修改索引结构，维护成本增加。               |
| 使用场景不匹配反而无效   | 查询字段未按最左前缀使用时，覆盖索引无效或部分失效。         |

## 八、如何“正确使用”覆盖索引？

使用建议：

| 场景                     | 建议                                     |
| ------------------------ | ---------------------------------------- |
| 高频 `SELECT` 且列固定   | 使用覆盖索引效果最佳。                   |
| 表数据量大、单表读多写少 | 用覆盖索引显著提升读性能。               |
| 查询字段有限             | 使用包含少字段的复合索引可减少空间开销。 |

 设计技巧：

1. **最左前缀原则：**
    覆盖索引也遵循最左匹配规则，字段顺序必须优化。

   ```
   sql复制编辑CREATE INDEX idx_email_username ON users(email, username);
   -- 支持：email = ? AND username = ?
   -- 不支持：username = ? （email 未使用，无法命中）
   ```

2. **将常用的查询字段加入联合索引末尾**：

   ```
   CREATE INDEX idx_email_username_id ON users(email, username, id);
   ```

3. **避免过多字段索引：**
    如你有查询 `SELECT *`，不要尝试将所有字段都放进索引，会严重浪费资源。

4. **利用 `EXPLAIN` 验证覆盖索引是否生效：**
    查看 `Extra` 字段是否含有 `Using index`。



