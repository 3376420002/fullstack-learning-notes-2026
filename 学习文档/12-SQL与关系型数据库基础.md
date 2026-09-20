# 12｜SQL 与关系型数据库基础

> 目标：能设计基本表结构、写安全查询、看懂事务和索引，并让 AI 不生成危险 SQL。

[← FastAPI](./11-FastAPI、Pydantic与依赖注入.md) · [学习首页](../README.md)

## 1. 关系模型

```text
Table   一类实体或关系
Row     一条记录
Column  一个属性
Primary Key  唯一标识一行
Foreign Key  关联并约束另一张表
```

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email VARCHAR(320) NOT NULL UNIQUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE todos (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    owner_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(100) NOT NULL,
    completed BOOLEAN NOT NULL DEFAULT FALSE,
    version INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT ck_todos_title CHECK (LENGTH(TRIM(title)) > 0)
);
```

## 2. 数据类型和约束

```text
BIGINT / INTEGER       整数和 ID
NUMERIC                精确金额
VARCHAR / TEXT         文本
BOOLEAN                真/假
DATE                   日期
TIMESTAMP WITH TIME ZONE 时间点
UUID                   UUID
```

```text
PRIMARY KEY   行身份
FOREIGN KEY   引用完整性
UNIQUE        唯一值或组合
NOT NULL      必须存在
CHECK         值满足规则
DEFAULT       未传值时的默认表达式
```

前端和 Pydantic 校验改善体验，数据库约束负责最终一致性。

## 3. NULL

NULL 表示未知、缺失或不适用，不等于 0、空字符串或 false。

```sql
WHERE due_at IS NULL
WHERE due_at IS NOT NULL
SELECT COALESCE(display_name, username) FROM users;
```

不要写 `column = NULL`。

## 4. INSERT 与 SELECT

```sql
INSERT INTO todos (owner_id, title)
VALUES (:owner_id, :title)
RETURNING id, owner_id, title, completed, created_at;
```

```sql
SELECT id, title, completed, created_at
FROM todos
WHERE owner_id = :owner_id
  AND completed = FALSE
ORDER BY created_at DESC, id DESC
LIMIT :page_size;
```

业务查询明确列名，不长期依赖 `SELECT *`。

## 5. UPDATE 与 DELETE

```sql
UPDATE todos
SET completed = TRUE,
    version = version + 1
WHERE id = :todo_id
  AND owner_id = :owner_id
RETURNING id, title, completed;
```

```sql
DELETE FROM todos
WHERE id = :todo_id
  AND owner_id = :owner_id;
```

UPDATE 和 DELETE 执行前必须核对 WHERE。应用还应检查受影响行数，区分成功和资源不存在。

## 6. 排序与分页

页码分页：

```sql
ORDER BY created_at DESC, id DESC
LIMIT :page_size
OFFSET :offset;
```

游标分页：

```sql
WHERE owner_id = :owner_id
  AND (created_at, id) < (:cursor_created_at, :cursor_id)
ORDER BY created_at DESC, id DESC
LIMIT :page_size;
```

排序加入稳定 ID。大数据量深页通常更适合游标分页。

## 7. INNER JOIN 与 LEFT JOIN

```sql
SELECT t.id, t.title, u.email
FROM todos AS t
INNER JOIN users AS u
    ON u.id = t.owner_id;
```

INNER JOIN 只保留匹配行。

```sql
SELECT u.id, COUNT(t.id) AS todo_count
FROM users AS u
LEFT JOIN todos AS t
    ON t.owner_id = u.id
GROUP BY u.id;
```

LEFT JOIN 保留左表全部行。统计子表使用 `COUNT(t.id)`，没有匹配时不会把空行算成一条。

右表过滤放 WHERE 可能移除 NULL 行，让 LEFT JOIN 接近 INNER JOIN。

## 8. 多对多

```sql
CREATE TABLE tags (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE todo_tags (
    todo_id BIGINT NOT NULL REFERENCES todos(id) ON DELETE CASCADE,
    tag_id BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (todo_id, tag_id)
);
```

中间表表达 Todo 与 Tag 的多对多关系，并阻止重复关联。

## 9. 聚合

```sql
SELECT
    owner_id,
    COUNT(*) AS total_count,
    SUM(CASE WHEN completed THEN 1 ELSE 0 END) AS completed_count
FROM todos
GROUP BY owner_id
HAVING COUNT(*) >= 10;
```

```text
WHERE   分组前过滤行
GROUP BY 改变结果粒度
HAVING  分组后过滤聚合结果
```

`COUNT(*)` 统计行，`COUNT(column)` 统计该列非 NULL 的行。

## 10. EXISTS、CTE 与窗口函数

```sql
SELECT u.id, u.email
FROM users AS u
WHERE EXISTS (
    SELECT 1
    FROM todos AS t
    WHERE t.owner_id = u.id
      AND t.completed = FALSE
);
```

```sql
WITH stats AS (
    SELECT owner_id, COUNT(*) AS total
    FROM todos
    GROUP BY owner_id
)
SELECT * FROM stats WHERE total >= 10;
```

```sql
SELECT
    id,
    owner_id,
    ROW_NUMBER() OVER (
        PARTITION BY owner_id
        ORDER BY created_at DESC, id DESC
    ) AS row_number
FROM todos;
```

CTE 拆解复杂查询，窗口函数跨行计算但不折叠原始行。

## 11. 事务与 ACID

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

失败时 `ROLLBACK`。

```text
Atomicity    整体成功或失败
Consistency  从有效状态到有效状态
Isolation    并发事务相互隔离到指定程度
Durability   提交结果可恢复
```

事务不要跨越慢外部 API 或用户等待，也不要在每个 Repository 方法中随意 commit。

## 12. 并发控制

乐观锁：

```sql
UPDATE todos
SET title = :title,
    version = version + 1
WHERE id = :todo_id
  AND version = :expected_version;
```

受影响行数为 0 表示版本冲突或资源不存在。

悲观锁：

```sql
SELECT id, balance
FROM accounts
WHERE id = :account_id
FOR UPDATE;
```

锁放在事务内，事务保持短小，多资源按一致顺序加锁以减少死锁。

## 13. 索引

```sql
CREATE INDEX ix_todos_owner_completed_created
ON todos (owner_id, completed, created_at DESC, id DESC);
```

索引适合等值、范围、排序和 JOIN，但会占空间并增加写入成本。

组合索引列顺序要匹配实际 WHERE 与 ORDER BY，不能为每个列机械建索引。

## 14. 执行计划

```sql
EXPLAIN
SELECT id, title FROM todos WHERE owner_id = 7;
```

```sql
EXPLAIN ANALYZE
SELECT id, title FROM todos WHERE owner_id = 7;
```

观察扫描方式、估算与实际行数、过滤、JOIN、排序、循环次数和时间。

`EXPLAIN ANALYZE` 会真正执行语句，不要对生产写操作随意使用。

## 15. N+1

```text
1 次查询 Todo 列表
N 次查询每个 Todo 的 owner
```

解决方式：JOIN、批量查询、ORM eager loading。不要无条件加载所有关系，要根据接口实际需要选择。

## 16. SQL 注入

危险：

```python
sql = f"SELECT * FROM users WHERE email = '{email}'"
```

安全：

```python
sql = "SELECT id, email FROM users WHERE email = :email"
params = {"email": email}
```

值使用参数化。表名、列名和排序方向通过白名单映射，不能直接拼接用户输入。

## 17. 迁移

```text
修改模型
  ↓
生成并人工审查迁移
  ↓
测试升级和数据兼容
  ↓
备份并在部署流程执行
```

大表变更要考虑锁、回填、旧代码兼容和回滚。不要手工修改生产库后忘记迁移文件。

## 18. 常见错误

- 用 ORM 代替 SQL 知识；
- 把 NULL 当空字符串；
- LEFT JOIN 后错误过滤右表；
- 先查唯一性但没有数据库 UNIQUE；
- 为每个列都建索引；
- 事务过长并包含外部请求；
- 忘记检查 UPDATE/DELETE WHERE；
- 拼接用户输入；
- 把 EXPLAIN ANALYZE 当成不会执行的分析命令。

## 19. 给 AI 的开发指令

```text
请为 Todo 系统设计 PostgreSQL 表和参数化查询。
明确主键、外键、NOT NULL、UNIQUE、CHECK、DEFAULT 和删除策略。
实现按用户筛选、稳定排序和游标分页，查询必须包含 owner_id 权限条件。
根据真实 WHERE、JOIN、ORDER BY 提出索引，并说明写入成本与 EXPLAIN 验证方式。
审查事务、并发更新和迁移风险，不拼接用户输入。
```

## 20. 面试表达

> 主键唯一标识一行，外键维护表之间的引用完整性，级联策略应根据数据生命周期选择。

> WHERE 在分组前过滤，HAVING 在 GROUP BY 后过滤聚合结果。

> INNER JOIN 只保留匹配行，LEFT JOIN 保留左表全部行；右表条件位置会影响结果。

> 事务通过 ACID 维护一致性，隔离越强通常冲突和重试成本越高。

> 索引用存储和写入成本换取特定查询性能，必须结合执行计划和真实数据验证。

> 参数化查询把 SQL 结构与数据分离，是防止 SQL 注入的基础。

下一课：**PostgreSQL、SQLAlchemy 与 Alembic**。
