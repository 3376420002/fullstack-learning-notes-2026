# 12｜SQL 与关系型数据库基础

> 所属阶段：数据库核心  
> 本课合并知识点：关系模型、DDL、约束、CRUD、JOIN、聚合、子查询、CTE、窗口函数、事务、隔离级别、锁、索引、执行计划、范式与建模  
> 前置知识：Python、FastAPI、HTTP 数据契约  
> 学习目标：熟悉关系型数据库和 SQL 的核心能力，能够读懂、指挥 AI 编写安全查询，并能在面试中准确说明查询、事务和索引原理。

[← FastAPI、Pydantic 与依赖注入](./11-FastAPI、Pydantic与依赖注入.md) · [学习首页](../README.md)

## 阅读方式

- 第 1～18 节掌握关系模型、表结构和增删改查；
- 第 19～32 节理解 JOIN、聚合、子查询和窗口函数；
- 第 33～43 节重点理解事务、隔离、锁、索引和执行计划；
- 第 44～50 节用于数据建模、AI 编程、代码审核和面试复习；
- 示例以 PostgreSQL 兼容 SQL 为主，具体数据库差异会在下一课展开。

## 1. 数据库在全栈项目中的位置

```text
Vue
 ↓ HTTP
FastAPI Router
 ↓
Service
 ↓
Repository / SQLAlchemy
 ↓ SQL
PostgreSQL
```

关系型数据库负责：

- 持久保存数据；
- 执行查询、排序、聚合和关联；
- 通过约束维护一致性；
- 通过事务处理并发修改；
- 通过索引提高特定查询效率；
- 为备份、恢复和审计提供基础。

ORM 可以帮助生成和组合 SQL，但不能替代对 SQL、索引和事务的理解。

## 2. 关系模型

关系型数据库以表组织数据：

```text
Table   表，表示一类实体或关系
Row     行，一条记录
Column  列，一个属性
Key     键，用来唯一标识或关联记录
```

Todo 表：

| id | owner_id | title | completed | created_at |
| ---: | ---: | --- | :---: | --- |
| 1 | 7 | 学习 SQL | false | 2026-09-20 10:00:00+00 |
| 2 | 7 | 学习索引 | true | 2026-09-20 11:00:00+00 |

表不是 Excel 工作表。数据库还提供类型、约束、事务、权限、索引和查询优化器。

## 3. 主键

主键唯一标识一行：

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(320) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

主键特点：

- 唯一；
- 不能为 NULL；
- 被其他表的外键引用；
- 应稳定，不应随普通业务属性变化。

常见选择：

```text
BIGINT 自增 ID    紧凑、高效、简单
UUID              分布式生成方便，体积和索引局部性需评估
自然键            具有业务含义，但可能变化或过长
```

用户邮箱通常不适合作为所有关联表的主键，因为邮箱可能变更。

## 4. 外键

```sql
CREATE TABLE todos (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    owner_id BIGINT NOT NULL REFERENCES users(id),
    title VARCHAR(100) NOT NULL
);
```

外键保证 `todos.owner_id` 引用的用户确实存在。

外键动作：

```sql
REFERENCES users(id) ON DELETE CASCADE
REFERENCES users(id) ON DELETE RESTRICT
REFERENCES users(id) ON DELETE SET NULL
```

| 策略 | 含义 |
| --- | --- |
| CASCADE | 删除父记录时自动删除子记录 |
| RESTRICT / NO ACTION | 存在引用时阻止删除 |
| SET NULL | 删除父记录后把外键置空，列必须允许 NULL |

级联规则是业务决策。用户删除时是否同时删除 Todo，不能只按方便程度选择。

## 5. 常见数据类型

| 类型 | 适合内容 |
| --- | --- |
| `INTEGER` / `BIGINT` | 整数、主键、计数 |
| `NUMERIC(p,s)` | 精确金额和小数 |
| `REAL` / `DOUBLE PRECISION` | 允许浮点误差的科学计算 |
| `VARCHAR(n)` / `TEXT` | 文本 |
| `BOOLEAN` | 真/假 |
| `DATE` | 日期 |
| `TIMESTAMP WITH TIME ZONE` | 明确时间点 |
| `UUID` | UUID 标识 |
| `JSON` / `JSONB` | 半结构化数据，能力依数据库而异 |

类型选择应表达真实语义：

- 金额使用 NUMERIC，不使用浮点；
- 时间点使用带时区的类型并统一 UTC；
- 布尔状态使用 BOOLEAN；
- 不要把所有内容都塞进 TEXT 或 JSON。

## 6. NULL

NULL 表示未知、缺失或不适用，不等于：

```text
0
空字符串
false
空数组
```

判断 NULL：

```sql
WHERE completed_at IS NULL
WHERE completed_at IS NOT NULL
```

错误：

```sql
WHERE completed_at = NULL
```

SQL 使用三值逻辑：TRUE、FALSE、UNKNOWN。与 NULL 的普通比较通常得到 UNKNOWN，不会被 WHERE 选中。

提供默认值：

```sql
SELECT COALESCE(display_name, username) AS display_name
FROM users;
```

## 7. 约束

```sql
CREATE TABLE todos (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    owner_id BIGINT NOT NULL REFERENCES users(id),
    title VARCHAR(100) NOT NULL,
    completed BOOLEAN NOT NULL DEFAULT FALSE,
    priority SMALLINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT ck_todos_title_not_blank
        CHECK (LENGTH(TRIM(title)) > 0),
    CONSTRAINT ck_todos_priority
        CHECK (priority BETWEEN 0 AND 3)
);
```

后续查询示例假设项目又通过迁移逐步增加了 `completed_at`、`due_at`、`deleted_at` 和 `version` 等字段。示例用于解释 SQL 能力，不应整段连续复制后重复创建同名表。

常见约束：

```text
PRIMARY KEY   行身份
FOREIGN KEY   引用完整性
UNIQUE        值或组合唯一
NOT NULL      不允许缺失
CHECK         值必须满足表达式
DEFAULT       未提供值时的默认表达式
```

前端和 Pydantic 校验改善体验，数据库约束负责最终一致性。并发请求无法只靠前端校验保证唯一性。

## 8. UNIQUE

单列唯一：

```sql
ALTER TABLE users
ADD CONSTRAINT uq_users_email UNIQUE (email);
```

组合唯一：

```sql
ALTER TABLE todo_tags
ADD CONSTRAINT uq_todo_tags_pair UNIQUE (todo_id, tag_id);
```

组合约束表示同一个 Todo 不能重复关联同一个 Tag。

NULL 与 UNIQUE 的行为存在数据库差异和可配置特性，设计前应确认目标数据库语义。

## 9. CREATE、ALTER 与 DROP

创建：

```sql
CREATE TABLE tags (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);
```

修改：

```sql
ALTER TABLE todos
ADD COLUMN due_at TIMESTAMP WITH TIME ZONE;
```

删除列：

```sql
ALTER TABLE todos
DROP COLUMN due_at;
```

删除表：

```sql
DROP TABLE tags;
```

DDL 会改变数据库结构，生产环境应通过迁移工具审查和执行。删除表或列可能不可恢复，执行前必须确认备份、依赖和回滚方案。

## 10. INSERT

```sql
INSERT INTO todos (owner_id, title)
VALUES (7, '学习 SQL');
```

插入多行：

```sql
INSERT INTO todos (owner_id, title)
VALUES
    (7, '学习 JOIN'),
    (7, '学习事务'),
    (8, '学习索引');
```

PostgreSQL 可以返回新记录：

```sql
INSERT INTO todos (owner_id, title)
VALUES (:owner_id, :title)
RETURNING id, owner_id, title, completed, created_at;
```

`:owner_id` 表示参数概念，具体占位符语法由驱动或 ORM 决定。

## 11. SELECT

```sql
SELECT id, title, completed
FROM todos;
```

别名：

```sql
SELECT
    id AS todo_id,
    title AS todo_title
FROM todos;
```

业务查询中优先明确列名，而不是长期使用：

```sql
SELECT * FROM todos;
```

明确列名可以减少无关数据传输、字段冲突和表结构变化带来的影响。

## 12. WHERE

```sql
SELECT id, title
FROM todos
WHERE owner_id = :owner_id
  AND completed = FALSE;
```

常见条件：

```sql
priority >= 2
status IN ('active', 'paused')
created_at BETWEEN :start_at AND :end_at
title LIKE 'SQL%'
deleted_at IS NULL
```

`BETWEEN` 通常包含两端。时间范围更常使用半开区间：

```sql
WHERE created_at >= :start_at
  AND created_at < :end_at
```

这样更适合连续日期区间，也能避免结束时刻精度问题。

## 13. LIKE 与文本搜索

```sql
WHERE title LIKE 'SQL%'
```

模式字符：

```text
%   任意长度字符
_   单个字符
```

PostgreSQL 提供不区分大小写的 `ILIKE`：

```sql
WHERE title ILIKE '%' || :keyword || '%'
```

包含式搜索以 `%` 开头时，普通 B-tree 索引通常难以直接支持。大规模文本搜索应评估全文索引或专用搜索方案。

用户输入仍必须参数化，不能因为使用 LIKE 就拼接 SQL。

## 14. ORDER BY

```sql
SELECT id, title, created_at
FROM todos
WHERE owner_id = :owner_id
ORDER BY created_at DESC, id DESC;
```

加入稳定的第二排序键很重要。当多行 `created_at` 相同，只按时间排序可能导致分页顺序不稳定。

NULL 排序行为存在数据库差异。PostgreSQL 可以明确指定：

```sql
ORDER BY due_at ASC NULLS LAST;
```

不要直接把用户提供的任意字符串拼进 ORDER BY。允许的排序字段应通过白名单映射。

## 15. LIMIT 与 OFFSET

```sql
SELECT id, title, created_at
FROM todos
WHERE owner_id = :owner_id
ORDER BY created_at DESC, id DESC
LIMIT :page_size
OFFSET :offset;
```

页码分页易于理解：

```text
offset = (page - 1) × page_size
```

问题：

- 深页需要跳过大量记录；
- 并发插入或删除可能导致重复或遗漏；
- 大 OFFSET 查询可能越来越慢。

小型后台列表仍可使用页码分页，关键是限制 page_size。

## 16. 游标分页

以 `(created_at, id)` 作为稳定游标：

```sql
SELECT id, title, created_at
FROM todos
WHERE owner_id = :owner_id
  AND (created_at, id) < (:cursor_created_at, :cursor_id)
ORDER BY created_at DESC, id DESC
LIMIT :page_size;
```

游标分页优势：

- 大数据量深页性能通常更稳定；
- 面对新增数据时更不容易重复或跳过；
- 与匹配索引配合良好。

限制：

- 不适合直接跳到任意页码；
- 游标必须编码排序字段；
- 排序条件变化后游标通常失效。

## 17. UPDATE

```sql
UPDATE todos
SET completed = TRUE,
    completed_at = CURRENT_TIMESTAMP
WHERE id = :todo_id
  AND owner_id = :owner_id;
```

必须确认 WHERE。遗漏 WHERE 会更新整张表。

PostgreSQL 返回更新结果：

```sql
UPDATE todos
SET title = :title
WHERE id = :todo_id
RETURNING id, title, completed;
```

应用应检查受影响行数或 RETURNING 结果，用于区分更新成功和资源不存在。

## 18. DELETE 与软删除

硬删除：

```sql
DELETE FROM todos
WHERE id = :todo_id
  AND owner_id = :owner_id;
```

软删除：

```sql
UPDATE todos
SET deleted_at = CURRENT_TIMESTAMP
WHERE id = :todo_id;
```

软删除并不简单：

- 所有查询都要过滤已删除数据；
- 唯一约束可能受影响；
- 外键和级联更复杂；
- 数据仍占空间；
- 恢复和最终清理需要规则。

只有确实需要恢复、审计或法规保留时才选择软删除，不要把它当成默认安全方案。

## 19. SQL 查询的逻辑处理顺序

简化的逻辑顺序：

```text
FROM / JOIN
WHERE
GROUP BY
HAVING
SELECT
DISTINCT
ORDER BY
LIMIT / OFFSET
```

这解释了为什么同一层级中，WHERE 通常不能直接使用 SELECT 才创建的别名：

```sql
SELECT price * quantity AS total
FROM order_items
WHERE total > 100; -- 很多数据库中不可用
```

可以使用子查询或重复表达式。实际执行计划由优化器决定，不一定机械遵循书写或逻辑顺序。

## 20. INNER JOIN

```sql
SELECT
    t.id,
    t.title,
    u.username
FROM todos AS t
INNER JOIN users AS u
    ON u.id = t.owner_id
WHERE t.completed = FALSE;
```

INNER JOIN 只保留两侧能匹配的行。

连接条件应写清楚：

```sql
ON u.id = t.owner_id
```

遗漏连接条件可能产生笛卡尔积，导致行数暴增。

## 21. LEFT JOIN

```sql
SELECT
    u.id,
    u.username,
    COUNT(t.id) AS todo_count
FROM users AS u
LEFT JOIN todos AS t
    ON t.owner_id = u.id
GROUP BY u.id, u.username;
```

LEFT JOIN 保留左表全部行。没有 Todo 的用户仍然出现，Todo 列为 NULL。

这里使用 `COUNT(t.id)`，因为没有匹配时 `t.id` 为 NULL，不会计数。`COUNT(*)` 会统计 LEFT JOIN 后的用户行，结果可能至少为 1。

## 22. LEFT JOIN 条件位置

保留所有用户，只关联未完成 Todo：

```sql
SELECT u.id, t.title
FROM users AS u
LEFT JOIN todos AS t
    ON t.owner_id = u.id
   AND t.completed = FALSE;
```

如果把条件写在 WHERE：

```sql
SELECT u.id, t.title
FROM users AS u
LEFT JOIN todos AS t
    ON t.owner_id = u.id
WHERE t.completed = FALSE;
```

没有 Todo 的用户会因 `t.completed` 为 NULL 被过滤，效果接近 INNER JOIN。

ON 定义怎样匹配，WHERE 过滤连接后的结果。对外连接尤其要理解这个差异。

## 23. 多对多关系

```sql
CREATE TABLE tags (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE todo_tags (
    todo_id BIGINT NOT NULL
        REFERENCES todos(id) ON DELETE CASCADE,
    tag_id BIGINT NOT NULL
        REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (todo_id, tag_id)
);
```

查询某 Todo 的标签：

```sql
SELECT tag.id, tag.name
FROM tags AS tag
INNER JOIN todo_tags AS tt
    ON tt.tag_id = tag.id
WHERE tt.todo_id = :todo_id
ORDER BY tag.name;
```

关联表还可以保存关系自身属性，例如添加时间、添加者或排序位置。

## 24. 聚合函数

```sql
SELECT
    COUNT(*) AS total_count,
    COUNT(completed_at) AS completed_with_time,
    MIN(created_at) AS first_created_at,
    MAX(created_at) AS last_created_at
FROM todos;
```

常见聚合：

```text
COUNT
SUM
AVG
MIN
MAX
```

区别：

```text
COUNT(*)       统计行
COUNT(column)  统计该列非 NULL 的行
COUNT(DISTINCT column) 统计不同的非 NULL 值
```

## 25. GROUP BY

按用户统计：

```sql
SELECT
    owner_id,
    COUNT(*) AS total_count,
    SUM(CASE WHEN completed THEN 1 ELSE 0 END) AS completed_count
FROM todos
GROUP BY owner_id;
```

SELECT 中未聚合的列通常必须出现在 GROUP BY 中。

分组会改变结果粒度：

```text
原始结果：每行一个 Todo
分组结果：每行一个 owner_id
```

写聚合前先明确最终每行代表什么。

## 26. WHERE 与 HAVING

```sql
SELECT owner_id, COUNT(*) AS todo_count
FROM todos
WHERE deleted_at IS NULL
GROUP BY owner_id
HAVING COUNT(*) >= 10;
```

```text
WHERE   分组前过滤原始行
HAVING  分组后过滤聚合结果
```

能在 WHERE 过滤的条件优先放 WHERE，可以减少进入聚合的数据量，也更清楚地表达语义。

## 27. CASE

```sql
SELECT
    id,
    title,
    CASE
        WHEN completed THEN 'completed'
        WHEN due_at < CURRENT_TIMESTAMP THEN 'overdue'
        ELSE 'active'
    END AS display_status
FROM todos;
```

条件聚合：

```sql
SELECT
    COUNT(*) AS total_count,
    SUM(CASE WHEN completed THEN 1 ELSE 0 END) AS completed_count
FROM todos;
```

PostgreSQL 还支持 `FILTER` 语法，下一课会进一步介绍。

## 28. 子查询

查找任务数超过 10 的用户：

```sql
SELECT id, username
FROM users
WHERE id IN (
    SELECT owner_id
    FROM todos
    GROUP BY owner_id
    HAVING COUNT(*) > 10
);
```

子查询可以出现在：

- WHERE；
- FROM；
- SELECT；
- INSERT、UPDATE、DELETE 等语句中。

选择 JOIN、子查询或 CTE 时优先保证语义清楚，再通过执行计划确认性能，不要只凭“子查询一定慢”等经验判断。

## 29. EXISTS

查询至少有一个未完成 Todo 的用户：

```sql
SELECT u.id, u.username
FROM users AS u
WHERE EXISTS (
    SELECT 1
    FROM todos AS t
    WHERE t.owner_id = u.id
      AND t.completed = FALSE
);
```

EXISTS 关心是否存在匹配行，不关心子查询具体返回列。

查找没有 Todo 的用户：

```sql
SELECT u.id, u.username
FROM users AS u
WHERE NOT EXISTS (
    SELECT 1
    FROM todos AS t
    WHERE t.owner_id = u.id
);
```

`NOT EXISTS` 通常比带 NULL 风险的 `NOT IN` 更容易正确表达反连接。

## 30. CTE

```sql
WITH todo_stats AS (
    SELECT
        owner_id,
        COUNT(*) AS total_count,
        SUM(CASE WHEN completed THEN 1 ELSE 0 END) AS completed_count
    FROM todos
    GROUP BY owner_id
)
SELECT
    u.id,
    u.username,
    s.total_count,
    s.completed_count
FROM users AS u
INNER JOIN todo_stats AS s
    ON s.owner_id = u.id
WHERE s.total_count >= 10;
```

CTE 使用 `WITH` 命名中间结果，适合拆解复杂查询和递归查询。

CTE 是否物化、是否影响优化取决于数据库和版本。不能仅凭使用 WITH 判断性能更好或更差。

## 31. 集合运算

```sql
SELECT email FROM active_users
UNION
SELECT email FROM invited_users;
```

```text
UNION          合并并去重
UNION ALL      合并并保留重复，通常更快
INTERSECT      交集
EXCEPT         左侧减去右侧
```

两侧列数和类型必须兼容。

如果业务不需要去重，优先明确使用 UNION ALL，避免无意义的排序或哈希去重成本。

## 32. 窗口函数

窗口函数计算跨行结果，但不会像 GROUP BY 那样把多行折叠成一行。

每个用户的 Todo 排名：

```sql
SELECT
    id,
    owner_id,
    title,
    created_at,
    ROW_NUMBER() OVER (
        PARTITION BY owner_id
        ORDER BY created_at DESC, id DESC
    ) AS row_number
FROM todos;
```

累计数量：

```sql
SELECT
    created_at,
    COUNT(*) OVER (
        ORDER BY created_at, id
    ) AS running_count
FROM todos;
```

常见窗口函数：

```text
ROW_NUMBER、RANK、DENSE_RANK
LAG、LEAD
SUM、COUNT、AVG OVER (...)
```

## 33. 事务

事务把多条语句组成一个逻辑工作单元：

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

失败时：

```sql
ROLLBACK;
```

两个余额更新必须一起成功或一起失败，否则会造成资金凭空减少。

## 34. ACID

```text
Atomicity    原子性：事务整体成功或整体失败
Consistency  一致性：约束和业务不变量从一个有效状态到另一个
Isolation    隔离性：并发事务的中间状态相互隔离到指定程度
Durability   持久性：提交结果在故障后仍能恢复
```

数据库的一致性约束只能保护已声明规则。应用没有声明的业务不变量，数据库不会自动理解。

持久性也不等于无需备份。误删除、账号入侵和灾难恢复仍需要备份与恢复策略。

## 35. 自动提交与事务边界

数据库客户端可能默认自动提交每条语句，也可能由 ORM Session 管理事务。

Web 请求常见边界：

```text
开始请求
  ↓
创建 Session / Transaction
  ↓
执行完整业务操作
  ↓ 成功
COMMIT
  ↓ 失败
ROLLBACK
  ↓
关闭 Session
```

不要在 Service 的每个 Repository 方法内部随意 commit，否则一个业务操作可能被切成无法整体回滚的多个事务。

事务也不应无意义地跨越很慢的外部 API 请求，否则会长期持有连接和锁。

## 36. 隔离级别与并发现象

常见隔离级别：

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

常见并发现象：

| 现象 | 含义 |
| --- | --- |
| 脏读 | 读到其他事务尚未提交的数据 |
| 不可重复读 | 同一事务重复读取一行得到不同值 |
| 幻读 | 同一条件重复查询出现或消失行 |
| 丢失更新 | 并发修改相互覆盖 |

更高隔离通常减少并发异常，但可能增加等待、冲突或重试。

不同数据库对标准隔离级别的具体实现并不完全相同，必须结合 PostgreSQL 的 MVCC 语义理解。

## 37. 丢失更新与乐观锁

错误流程：

```text
请求 A 读取 version=1
请求 B 读取 version=1
请求 A 更新并提交
请求 B 用旧数据覆盖 A
```

乐观锁：

```sql
UPDATE todos
SET title = :title,
    version = version + 1
WHERE id = :todo_id
  AND version = :expected_version;
```

如果受影响行数为 0，表示记录不存在或版本已变化，应用可以返回 409 冲突或重新读取后处理。

乐观锁适合冲突较少的场景。它需要客户端或业务层正确传递版本信息。

## 38. 悲观锁

```sql
BEGIN;

SELECT id, balance
FROM accounts
WHERE id = :account_id
FOR UPDATE;

UPDATE accounts
SET balance = balance - :amount
WHERE id = :account_id;

COMMIT;
```

`FOR UPDATE` 会对选中行取得写锁，其他冲突事务可能等待。

注意：

- 锁必须位于事务内；
- 事务要尽量短；
- 多资源加锁顺序应一致；
- 要设置合理超时；
- 锁不能替代数据库约束。

## 39. 死锁

```text
事务 A 锁住资源 1，等待资源 2
事务 B 锁住资源 2，等待资源 1
```

数据库通常会检测死锁并中止其中一个事务。

减少死锁：

- 按一致顺序访问资源；
- 保持事务短小；
- 为查询提供合适索引，减少锁定范围和时间；
- 不在事务中等待用户输入或慢外部服务；
- 对可重试事务实现有限次数和退避。

死锁是并发系统可能出现的正常故障类型，应用需要正确处理，而不是假设永远不会发生。

## 40. 索引

```sql
CREATE INDEX ix_todos_owner_id
ON todos (owner_id);
```

索引类似为特定查询建立的快速定位结构，常见 B-tree 索引适合：

- 等值查询；
- 范围查询；
- 排序；
- 前缀匹配；
- JOIN 键查找。

代价：

- 占用磁盘和缓存；
- INSERT、UPDATE、DELETE 需要维护；
- 创建索引可能消耗大量资源；
- 索引过多会让优化和维护更复杂。

不是每个列都应创建索引。

## 41. 组合索引

查询：

```sql
SELECT id, title, created_at
FROM todos
WHERE owner_id = :owner_id
  AND completed = FALSE
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

可能的组合索引：

```sql
CREATE INDEX ix_todos_owner_completed_created_id
ON todos (owner_id, completed, created_at DESC, id DESC);
```

列顺序应根据实际 WHERE、范围和 ORDER BY 设计。

组合索引 `(a, b, c)` 通常更容易支持从左侧开始的条件，但并不意味着所有只涉及 b 或 c 的查询都能高效使用它。

应通过真实数据和执行计划验证，而不是只靠口诀。

## 42. 可索引查询与常见失效

更容易利用普通索引：

```sql
WHERE created_at >= :start_at
  AND created_at < :end_at
```

可能妨碍普通索引直接定位：

```sql
WHERE DATE(created_at) = :target_date
```

可以改写范围，或根据数据库能力建立表达式索引。

其他常见问题：

- 隐式类型转换；
- 前导 `%` 模糊匹配；
- 对索引列执行函数；
- OR 条件跨多个低选择性列；
- 返回表中大部分行；
- 统计信息陈旧。

优化器可能正确判断全表扫描比走索引更便宜。看到顺序扫描不代表一定有问题。

## 43. EXPLAIN 与 EXPLAIN ANALYZE

```sql
EXPLAIN
SELECT id, title
FROM todos
WHERE owner_id = 7;
```

真实执行并统计：

```sql
EXPLAIN ANALYZE
SELECT id, title
FROM todos
WHERE owner_id = 7;
```

重点观察：

- 扫描类型；
- 估算行数与实际行数；
- 过滤掉的行数；
- JOIN 算法；
- 排序和聚合；
- 执行时间；
- 是否重复循环大量次数。

`EXPLAIN ANALYZE` 会真正执行语句。不要在生产环境对有副作用的 UPDATE、DELETE 或高成本查询随意使用。

## 44. N+1 查询

```text
1 次查询 Todo 列表
N 次分别查询每个 Todo 的 owner
总计 N + 1 次查询
```

解决方向：

- JOIN；
- ORM eager loading；
- 批量 `WHERE id IN (...)`；
- DataLoader 类批处理；
- 调整响应结构。

不要无条件把所有关系都 eager load。加载策略应根据接口真正需要的数据和基数决定。

N+1 是查询次数模式，不是某个 ORM 独有问题。

## 45. 数据库范式

### 第一范式 1NF

列值保持原子性，不在一个字段中保存逗号分隔的多个 Tag。

### 第二范式 2NF

在组合主键场景中，非键字段依赖完整主键，而不是只依赖其中一部分。

### 第三范式 3NF

非键字段不应通过另一个非键字段间接依赖主键。

范式的目标是减少重复和更新异常，不是追求越多表越好。

报表和高性能读取场景可能有意识地反范式化，但必须明确同步和一致性成本。

## 46. Todo 数据模型

```text
users
├─ id PK
├─ username UNIQUE
└─ created_at

todos
├─ id PK
├─ owner_id FK → users.id
├─ title
├─ completed
├─ due_at NULL
├─ version
└─ created_at

tags
├─ id PK
└─ name UNIQUE

todo_tags
├─ todo_id PK/FK → todos.id
└─ tag_id PK/FK → tags.id
```

关系：

```text
User 1 ─── N Todo
Todo N ─── M Tag
```

表结构应从业务不变量和查询方式推导，而不是直接把前端 JSON 原样变成一张表。

## 47. SQL 注入与参数化

危险：

```python
sql = f"SELECT * FROM users WHERE email = '{email}'"
```

安全方向：

```python
sql = "SELECT id, email FROM users WHERE email = :email"
params = {"email": email}
```

参数化让驱动把 SQL 结构与数据值分开处理。

注意：

- 表名、列名和 ORDER BY 方向通常不能作为普通值参数；
- 动态标识符应使用白名单映射或库提供的安全标识符 API；
- ORM 也可能因手写文本拼接产生注入；
- 参数化不能修复越权查询，仍需 owner_id 等权限条件。

## 48. Schema Migration

数据库结构会随代码变化：

```text
新增表
新增列
修改约束
创建索引
迁移旧数据
删除旧字段
```

迁移原则：

- 文件进入版本控制；
- 顺序明确且可审查；
- 在接近生产的数据量上评估锁和耗时；
- 先兼容扩展，再迁移数据，最后清理旧结构；
- 备份和回滚策略明确；
- 不依赖开发者手工修改生产数据库。

当前技术栈会使用 Alembic 管理 SQLAlchemy 迁移，下一课会完整讲解。

## 49. 常见错误与准确理解

### ORM 可以代替 SQL

错误。ORM 最终仍执行 SQL，查询性能、事务和索引问题必须理解数据库。

### NULL 等于空字符串

错误。NULL 表示未知或缺失，比较要使用 IS NULL。

### LEFT JOIN 后在 WHERE 过滤右表没有影响

错误。它可能过滤 NULL 行，让查询实际接近 INNER JOIN。

### 建索引一定让查询更快

错误。索引有维护成本，低选择性或返回大量行时全表扫描可能更合理。

### 外键会自动创建所有所需索引

不应假设。被引用主键通常有索引，但引用方外键是否自动索引取决于数据库，需要按查询和删除行为设计。

### 事务越大越安全

错误。过长事务占用连接、版本和锁，增加冲突与清理压力。

### 更高隔离级别没有代价

错误。更强隔离可能增加等待、回滚或重试。

### 软删除总比硬删除安全

错误。软删除引入查询、唯一约束、隐私和清理复杂度。

### EXPLAIN ANALYZE 只做分析不会执行

错误。ANALYZE 会真正运行语句。

### 应用先查唯一性再插入就不会重复

错误。并发请求之间仍可能竞争，最终需要数据库 UNIQUE 约束并处理冲突。

## 50. 指挥 AI 编写 SQL

### 设计表结构指令

```text
请为 Todo 系统设计 PostgreSQL 关系模型。
先列出实体、关系、业务不变量和高频查询，再编写 DDL。
包含 users、todos、tags、todo_tags，明确主键、外键、NOT NULL、UNIQUE、CHECK、DEFAULT 和删除策略。
时间点使用带时区类型，金额类字段不得使用浮点。
不要把所有字段塞进 JSONB，不要只依赖应用校验。
解释每个索引对应的真实查询，并指出写入成本。
```

### 编写查询指令

```text
请为当前数据库编写参数化 SQL，查询某用户未完成的 Todo。
支持 status、keyword、due_before 和稳定游标分页，排序为 created_at DESC、id DESC。
只选择接口需要的列，包含 owner_id 权限条件，不拼接用户输入。
同时给出建议索引与 EXPLAIN 验证方向，不要声称未经实测一定更快。
```

### 优化查询指令

```text
请根据表结构、数据量、索引和 EXPLAIN ANALYZE 输出诊断慢查询。
比较估算行数与实际行数，检查扫描方式、JOIN、排序、循环次数、过滤行和 N+1。
先指出证据，再提出查询改写、索引或统计信息方案。
说明新增索引的磁盘和写入成本，不要机械为每个 WHERE 列建索引。
```

### 事务审查指令

```text
请审查这个业务流程的事务边界和并发风险。
识别丢失更新、重复创建、锁顺序、外部 API 等待、死锁和重试。
根据冲突概率选择数据库约束、乐观锁、SELECT FOR UPDATE 或更高隔离级别。
明确 commit、rollback、超时和可重试错误，不能只在代码注释中假设原子性。
```

### 数据库迁移指令

```text
请为现有大表设计一次向后兼容的迁移。
先评估表大小、NULL、默认值、锁、索引创建和旧代码兼容性。
采用扩展结构、分批回填、双版本兼容、增加约束、最后清理的阶段化方案。
提供验证、监控、备份和回滚步骤，不要直接删除生产列或一次更新全表。
```

## 51. 审核 AI 生成 SQL 的清单

```text
[ ] 表和列名称表达清楚且风格统一
[ ] 主键、外键、NOT NULL、UNIQUE、CHECK 已按业务声明
[ ] 外键删除策略符合真实数据生命周期
[ ] 金额、时间和布尔值使用合适类型
[ ] NULL 语义清楚，没有使用 = NULL
[ ] SELECT 只读取需要列
[ ] UPDATE 和 DELETE 包含经过核对的 WHERE
[ ] JOIN 条件完整，LEFT JOIN 过滤位置正确
[ ] 聚合粒度明确，COUNT(*) 与 COUNT(column) 使用正确
[ ] 分页排序稳定且 page_size 有上限
[ ] 所有数据值参数化，动态标识符经过白名单
[ ] 查询包含租户或资源所有者权限条件
[ ] 索引对应真实查询，列顺序有依据
[ ] 性能结论由 EXPLAIN 和真实数据验证
[ ] 事务边界、隔离、锁和重试策略明确
[ ] 迁移考虑大表锁、兼容、回填和回滚
[ ] 没有把 ORM 当成 SQL 与事务知识替代品
```

## 52. 面试表达速记

### 主键与外键

> 主键唯一标识表中的一行，必须唯一且非 NULL；外键引用另一张表的候选键，维护引用完整性。外键的级联、限制或置空策略应根据业务生命周期选择。

### WHERE 与 HAVING

> WHERE 在分组前过滤原始行，HAVING 在 GROUP BY 后过滤聚合结果。能提前过滤的条件通常放 WHERE，以减少参与分组的数据。

### INNER JOIN 与 LEFT JOIN

> INNER JOIN 只保留匹配行，LEFT JOIN 保留左表全部行。LEFT JOIN 后如果在 WHERE 对右表字段做普通过滤，NULL 行会被去掉，可能退化成 INNER JOIN 语义。

### COUNT

> COUNT(*) 统计结果行，COUNT(column) 只统计该列非 NULL 的行。LEFT JOIN 统计子表数量时常使用 COUNT(child.id)，避免无匹配的左表行被算成一条。

### UNION 与 UNION ALL

> UNION 合并后去重，UNION ALL 保留重复，通常少一步去重成本。业务允许重复时应明确使用 UNION ALL。

### 事务与 ACID

> 事务把多条语句组成原子工作单元。ACID 分别是原子性、一致性、隔离性和持久性。数据库只维护已声明约束，应用仍需定义业务不变量和事务边界。

### 隔离级别

> 更高隔离减少脏读、不可重复读、幻读或序列化异常，但可能增加等待和回滚。具体行为依赖数据库的 MVCC 实现，应用还要处理冲突与重试。

### 乐观锁与悲观锁

> 乐观锁通过 version 条件更新并检查受影响行数，适合冲突少的场景；悲观锁通过 SELECT FOR UPDATE 等机制提前锁行，适合必须串行处理但会增加等待的场景。

### 索引

> 索引用额外存储和写入维护成本换取特定查询的定位、排序和 JOIN 性能。组合索引列顺序要匹配实际过滤和排序，并通过执行计划验证。

### N+1

> N+1 指先查一批主记录，再为每条记录分别查询关联数据，产生一次加 N 次查询。可以通过 JOIN、批量查询或合适的 ORM eager loading 解决，但不应无条件加载所有关系。

### SQL 注入

> SQL 注入来自把不可信输入拼入 SQL 结构。数据值应使用参数化查询，动态表名、列名和排序方向则通过白名单或安全标识符 API 处理。

### 范式

> 范式通过拆分依赖减少重复和更新异常。反范式可以优化特定读取，但会增加同步和一致性成本，应以真实查询和性能证据为依据。

## 53. 本课技术地图

```text
Relational Database
├─ Schema
│  ├─ Table / Column / Type
│  ├─ Primary / Foreign Key
│  └─ Constraints
├─ SQL
│  ├─ INSERT / SELECT
│  ├─ UPDATE / DELETE
│  ├─ WHERE / ORDER / Pagination
│  ├─ JOIN
│  ├─ GROUP / Aggregate
│  ├─ Subquery / CTE
│  └─ Window Function
├─ Consistency
│  ├─ Transaction / ACID
│  ├─ Isolation / MVCC
│  ├─ Optimistic Lock
│  └─ Pessimistic Lock
├─ Performance
│  ├─ Index
│  ├─ EXPLAIN
│  └─ N+1
└─ Design
   ├─ Relationships
   ├─ Normalization
   ├─ Parameterization
   └─ Migration
```

## 54. 下一步

下一课合并学习 **PostgreSQL、SQLAlchemy 与 Alembic**：

- PostgreSQL 数据库、Schema、角色与连接；
- PostgreSQL 常用类型、JSONB、数组和时间；
- MVCC、VACUUM、锁与执行计划；
- SQLAlchemy 2.x Declarative Model；
- Session、事务、查询和关系加载；
- 同步与异步 SQLAlchemy；
- N+1、连接池和事务边界；
- Alembic 迁移、自动生成与生产发布；
- FastAPI 数据库依赖和 Todo 持久化；
- AI 编程指令、审核清单与面试表达。
