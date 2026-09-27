# 13｜PostgreSQL、SQLAlchemy 与 Alembic

> 所属阶段：数据库工程实践
>
> 本课合并知识点：PostgreSQL 类型与运行机制、SQLAlchemy 2.x ORM、Session 与事务、关系加载、异步数据库访问、Alembic 迁移、FastAPI 集成
>
> 前置知识：SQL 与关系型数据库、Python、FastAPI、Pydantic
>
> 学习目标：看懂数据库层代码，能准确指挥 AI 为现有项目实现持久化、迁移和排错，并能在面试中讲清楚每一层的职责。

[← SQL 与关系型数据库基础](./12-SQL与关系型数据库基础.md)

## 阅读方式

- 先读第 1～8 节，知道 PostgreSQL 提供什么、与通用 SQL 有何区别；
- 再读第 9～23 节，掌握 SQLAlchemy 模型、查询、Session、事务、关系加载；
- 接着读第 24～31 节，理解 Alembic 迁移与安全发布；
- 最后读第 32 节后的完整 Todo 链路、排错方法、AI 指令和面试表达。

本文代码采用 SQLAlchemy 2.x 风格、Pydantic v2 和 PostgreSQL。示例中的依赖版本、数据库地址和迁移模板以实际项目配置为准，不必照搬版本号或目录。

## 1. 三项技术各自负责什么

```text
PostgreSQL   真正保存数据，执行 SQL、约束、事务和索引
SQLAlchemy   用 Python 构造 SQL，并把表行映射成 Python 对象
Alembic      用有版本的脚本管理数据库结构变化
```

放入上一课的请求链：

```text
Vue → HTTP → FastAPI Router → Service → Repository
                                         ↓
                                   SQLAlchemy ORM
                                         ↓ SQL
                                    PostgreSQL

Alembic ───────────→ 管理 PostgreSQL 的表、列、索引等结构
```

例如新增任务时，Router 解析 HTTP，请求 Schema 校验数据，Service 判断用户能否创建，Repository 通过 SQLAlchemy 发出 `INSERT`，PostgreSQL 最终写入并检查约束。Alembic 不参与每一次请求，只在部署或维护时执行迁移。

> 记忆：PostgreSQL 是数据库，SQLAlchemy 是访问数据库的库，Alembic 是数据库结构的版本管理工具。

## 2. PostgreSQL 中的数据库、Schema、表与角色

这些名称容易混淆：

```text
PostgreSQL 服务实例
├─ 数据库 todolab
│  ├─ Schema public
│  │  ├─ 表 users
│  │  └─ 表 todos
│  └─ Schema audit
│     └─ 表 events
└─ 数据库 analytics

Role：连接身份和权限主体；可以拥有数据库对象
```

- 数据库之间通常不能像同一数据库内的表那样直接 `JOIN`；
- Schema 是数据库内部的命名空间，不等于 Pydantic Schema；
- `public.todos` 表示 `public` Schema 下的 `todos` 表；
- `search_path` 决定不写 Schema 名称时从哪里查找表；
- 应用使用最小权限角色，不应长期使用超级用户连接。

在 `psql` 中可以观察：

```sql
SELECT current_database(), current_user, current_schema();
SHOW search_path;
```

`psql` 的 `\l`、`\dn`、`\dt` 是客户端快捷命令，不是可以直接放进 SQLAlchemy 执行的 SQL。

## 3. 连接地址与环境变量

异步 SQLAlchemy 配合 `asyncpg` 时，数据库 URL 形如：

```text
postgresql+asyncpg://app_user:password@127.0.0.1:5432/todolab
```

其中 `postgresql` 是数据库方言，`asyncpg` 是驱动，后面依次是用户、密码、地址、端口和数据库名。密码中的特殊字符需要 URL 编码。真实地址放服务端环境变量，不能提交到 Git，也不能放入前端 `VITE_` 变量。

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    database_url: str


settings = Settings()
```

本地 `.env.example` 可以写占位地址，真实 `.env` 应由项目忽略。生产环境通常由部署平台注入密钥。

## 4. PostgreSQL 常用数据类型

| 类型 | Todo 项目用途 | 注意 |
| --- | --- | --- |
| `BIGINT` / `INTEGER` | 自增 ID、计数 | 选择与预计规模相符的范围 |
| `UUID` | 外部可见 ID | 不要误以为 UUID 自动提供权限保护 |
| `TEXT` / `VARCHAR(n)` | 标题、说明 | 长度规则可由数据库约束或应用明确表达 |
| `BOOLEAN` | 是否完成 | `NULL` 和 `FALSE` 不同 |
| `TIMESTAMPTZ` | 创建、更新时间点 | 保存时间点，展示时转用户时区 |
| `DATE` | 只关心日期的截止日 | 不表示具体时刻 |
| `NUMERIC(p, s)` | 精确金额 | Python 中常配合 `Decimal` |
| `JSONB` | 结构灵活的附加信息 | 核心关联字段仍优先普通列 |
| `TEXT[]` | 小型字符串数组 | 复杂关系通常仍用关联表 |

`TIMESTAMP WITH TIME ZONE`（简称 `TIMESTAMPTZ`）保存时间点，输入时会按时区解释并归一化；数据库不会保留原始时区名称。Todo 的 `created_at` 适合这种类型，用户所在时区通常单独保存或从用户设置读取。

```sql
CREATE TABLE todos (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    owner_id BIGINT NOT NULL REFERENCES users(id),
    title VARCHAR(100) NOT NULL,
    completed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## 5. JSONB 与关系型列如何选择

假设每个 Todo 有一些扩展显示设置：

```sql
ALTER TABLE todos
ADD COLUMN display_options JSONB NOT NULL DEFAULT '{}'::jsonb;
```

`JSONB` 能保存结构化 JSON 并支持查询与索引，但没有理由把整个 Todo 都塞进一个 JSONB 字段。常用于筛选、排序、关联和约束的 `owner_id`、`completed`、`created_at` 应保留普通列。

```sql
SELECT id, title
FROM todos
WHERE display_options @> '{"color":"blue"}'::jsonb;
```

对真实高频的 JSONB 包含查询，可根据执行计划考虑 GIN 索引；不要预先为每个 JSONB 字段建立索引。JSONB 里的字段结构变化也需要契约、兼容和测试。

## 6. PostgreSQL 的 MVCC 与 VACUUM

多版本并发控制（MVCC）让读写事务可以在一定程度上互不阻塞。更新行时，数据库通常保留旧版本一段时间，其他事务按自己的可见性规则读取合适版本。

```text
事务 A 读取旧版本 ────────────────→ 提交
事务 B       更新并产生新版本 → 提交
```

旧版本不能无限堆积。PostgreSQL 的 `VACUUM` 会回收可重用空间，`ANALYZE` 更新优化器统计信息；通常由 autovacuum 自动执行。长期不结束的事务可能阻止旧版本清理，造成表膨胀、查询变慢。

面试不必背所有内部数据结构，但要能解释：为什么更新多、长事务多的表需要关注 autovacuum 和表膨胀。不要看到慢查询就直接在生产环境运行 `VACUUM FULL`，它可能需要更强的锁和额外空间。

## 7. 锁、隔离级别与冲突

PostgreSQL 默认隔离级别通常是 `READ COMMITTED`。同一事务的不同语句可能读到其他事务随后提交的新数据。更严格的隔离级别不能替代业务规则，发生冲突时仍可能需要重试。

需要对同一条记录串行修改时，可以在事务内锁行：

```sql
BEGIN;
SELECT id, completed FROM todos WHERE id = 42 FOR UPDATE;
UPDATE todos SET completed = TRUE WHERE id = 42;
COMMIT;
```

锁要与事务绑定，事务结束后释放。不要在持锁期间调用慢外部 API 或等待用户输入。两个事务以不同顺序锁定多条记录时可能死锁，应用应识别可重试错误，并且只重试可安全重试的整个事务。

## 8. 索引、统计信息与执行计划

常用列表查询：

```sql
SELECT id, title, completed, created_at
FROM todos
WHERE owner_id = 7 AND completed = FALSE
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

可考虑与查询形状匹配的组合索引：

```sql
CREATE INDEX ix_todos_owner_completed_created_id
ON todos (owner_id, completed, created_at DESC, id DESC);
```

索引会增加写入维护和磁盘成本。先用真实查询、数据量和执行计划判断收益：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, completed, created_at
FROM todos
WHERE owner_id = 7 AND completed = FALSE
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

`ANALYZE` 会真正执行语句；对生产写操作不要随意使用。观察实际行数、扫描方式、排序和缓冲区访问，而非只看到 `Seq Scan` 就判断一定有问题——小表顺序扫描可能更快。

## 9. SQLAlchemy Core、ORM 与驱动

SQLAlchemy 主要提供两层能力：

```text
Core   表达 SQL、连接和执行结果
ORM    在 Core 之上，把表行映射成 Python 对象并管理对象状态
```

`asyncpg` 是与 PostgreSQL 通信的驱动；SQLAlchemy 负责构造语句和管理 ORM/连接；Alembic 调用 SQLAlchemy 连接来运行迁移。

ORM 代码：

```python
stmt = select(Todo).where(Todo.owner_id == current_user_id)
todos = (await session.scalars(stmt)).all()
```

它仍会生成 SQL；必须理解 `WHERE`、索引、JOIN 和事务。SQLAlchemy 2.x 推荐以 `select()`、`Session.execute()` / `Session.scalars()` 等方式写查询，而不是在新代码中继续沿用旧式 `session.query()` 风格。

## 10. 选择同步还是异步访问

两条路线都可以正确工作，但一条调用链应保持一致：

| 路线 | 常见组合 | 适用情况 |
| --- | --- | --- |
| 同步 | `psycopg` + `create_engine` + `Session` | 项目现有同步库和普通 `def` Endpoint |
| 异步 | `asyncpg` + `create_async_engine` + `AsyncSession` | 已使用 async Endpoint 和非阻塞 I/O |

“异步”主要提升 I/O 等待期间的并发利用率，不会自动让单条 SQL 更快。不要只把 `def` 改成 `async def`，然后在里面继续直接调用同步数据库客户端。

下文用异步路线串联 FastAPI。加入已有项目时，先识别它当前采用哪一条路线。

## 11. 推荐的最小目录

```text
backend/
├─ pyproject.toml
├─ alembic.ini
├─ alembic/
│  ├─ env.py
│  └─ versions/
│     └─ <revision>_create_todos.py
├─ src/app/
│  ├─ main.py
│  ├─ api/todos.py
│  ├─ core/config.py
│  ├─ db/base.py
│  ├─ db/session.py
│  ├─ models/todo.py
│  ├─ schemas/todo.py
│  ├─ repositories/todos.py
│  └─ services/todos.py
└─ tests/
```

名称可按现有项目调整。关键关系是：`models/` 定义 ORM 表映射，`schemas/` 定义对外 API 契约，`repositories/` 管理查询，`services/` 管理业务规则，`alembic/versions/` 记录表结构演进。

## 12. 依赖、版本与配置

需要的能力通常包括：

```text
sqlalchemy          ORM 与 SQL 表达
asyncpg             PostgreSQL 异步驱动
alembic             数据库迁移
pydantic-settings   从服务端环境读取配置
```

项目使用 uv、Poetry 或 pip 时，按现有包管理方式安装，不要混用。迁移工具要与项目中 SQLAlchemy/Python 版本兼容；应用依赖和锁文件应进入版本控制。

例如 `pyproject.toml` 中会出现这些依赖，但不应照教程把现有项目依赖全部升级。若项目使用同步 `psycopg`，数据库 URL 与 engine 工厂也应相应改成同步写法。

## 13. DeclarativeBase：模型的共同起点

SQLAlchemy 2.x 的声明式模型可以这样建立：

```python
# app/db/base.py
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase


NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

`Base.metadata` 保存当前 Python 代码所描述的表结构，Alembic 的自动生成要拿它与目标数据库比较。命名约定让约束名称更稳定，便于迁移。已有项目若有自己的 Base，不要另建第二套未经关联的 Base。

## 14. Mapped 与 mapped_column：定义 Todo Model

```python
# app/models/todo.py
from datetime import datetime

from sqlalchemy import BigInteger, Boolean, DateTime, ForeignKey, Identity, String, func, text
from sqlalchemy.orm import Mapped, mapped_column

from app.db.base import Base


class Todo(Base):
    __tablename__ = "todos"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    owner_id: Mapped[int] = mapped_column(
        BigInteger,
        ForeignKey("users.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    title: Mapped[str] = mapped_column(String(100), nullable=False)
    completed: Mapped[bool] = mapped_column(
        Boolean,
        nullable=False,
        server_default=text("false"),
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        server_default=func.now(),
    )
```

这里假设 `users.id` 也是 `BIGINT`，且已有 User 模型/迁移。外键两端类型应匹配。`Mapped[int]` 是 Python 类型提示；`mapped_column` 的数据库类型、默认值和约束才决定实际表结构。

`default=` 是 SQLAlchemy 在写入时应用的 Python/SQL 表达式，`server_default=` 是数据库自身默认值。数据库约束和默认值不能只存在于 Pydantic 中，因为其他程序也可能直接写数据库。

## 15. ORM Model 与 Pydantic Schema 不同

```python
# app/schemas/todo.py
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)


class TodoUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=100)
    completed: bool | None = None


class TodoOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    completed: bool
    created_at: datetime
```

| 对象 | 负责什么 | 是否直接暴露给客户端 |
| --- | --- | --- |
| `Todo` ORM Model | 表、列、外键、关系 | 不应直接决定 API 字段 |
| `TodoCreate` | 创建请求允许哪些字段 | 是，作为输入契约 |
| `TodoUpdate` | 哪些字段允许局部修改 | 是，作为输入契约 |
| `TodoOut` | 客户端能看到哪些字段 | 是，作为输出契约 |

例如 `owner_id` 应来自已认证用户，不应接受客户端自由指定。数据库未来增加内部字段时，`TodoOut` 也不应自动把它们暴露出去。

## 16. Engine、连接池与 Session 工厂

```python
# app/db/session.py
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

from app.core.config import settings


engine = create_async_engine(
    settings.database_url,
    pool_pre_ping=True,
    echo=False,
)

SessionFactory = async_sessionmaker(
    bind=engine,
    expire_on_commit=False,
)


async def get_session() -> AsyncIterator[AsyncSession]:
    async with SessionFactory() as session:
        yield session
```

```text
Engine          数据库连接能力与连接池，通常每个应用进程创建一次
SessionFactory  创建独立 Session 的工厂
AsyncSession    一个工作单元中的 ORM 状态和事务上下文
```

每个请求通常获得自己的 Session。关闭 Session 会释放它占用的连接资源，但不会自动替业务决定何时提交。`pool_pre_ping` 有助于识别失效连接，不是所有数据库故障的自动重试器。

应用关闭时可在 FastAPI Lifespan 中 `await engine.dispose()`。连接池大小需结合应用进程数、数据库最大连接数和实际负载调整，不要照抄“越大越快”。

## 17. Session 不是数据库连接，也不是全局单例

Session 管理对象状态、身份映射和当前事务。它可能在需要时从连接池借用连接，再归还。不要在模块级创建一个 Session 给所有请求共同使用。

```text
Engine：共享的连接管理能力
Session：单次工作单元的状态容器
Transaction：一组需要一起成功或失败的数据库操作
```

同一个 `AsyncSession` 不能在多个并发任务之间共享。例如 `asyncio.gather()` 并发处理几个任务时，应给每个并发任务独立 Session，或者改为同一 Session 内顺序执行。

## 18. add、flush、commit、refresh 与 rollback

```python
todo = Todo(owner_id=user_id, title="学习数据库")
session.add(todo)        # 放入当前 Session，尚不代表写入已提交
await session.flush()    # 把待写操作发送到数据库；可取得生成的 ID
await session.refresh(todo)  # 从数据库重新读取该对象需要的字段
await session.commit()   # 提交事务，其他事务可按隔离规则看到结果
```

- `flush` 不是 `commit`；后续异常仍可能回滚；
- `commit` 前应确认整个业务操作都成功；
- `rollback` 用于撤销当前未提交事务；
- `refresh` 在需要数据库生成值时使用，不必对每个对象无条件调用；
- `expire_on_commit=False` 使提交后已加载字段保持可用，常见于异步项目。

更推荐用事务上下文管理器明确边界：

```python
async with SessionFactory.begin() as session:
    session.add(Todo(owner_id=user_id, title="学习数据库"))
# 无异常时提交并关闭；异常时回滚并关闭
```

“关闭 Session”和“提交事务”是两个不同动作。不要在每个 Repository 方法里各自 `commit()`，否则一个业务动作跨多个方法时难以保证原子性。

## 19. SQLAlchemy 2.x 的常用查询

按主键查询：

```python
todo = await session.get(Todo, todo_id)
```

按用户筛选并排序：

```python
from sqlalchemy import select

stmt = (
    select(Todo)
    .where(Todo.owner_id == user_id, Todo.completed.is_(False))
    .order_by(Todo.created_at.desc(), Todo.id.desc())
    .limit(20)
)
todos = (await session.scalars(stmt)).all()
```

`session.get()` 面向主键，可能利用 Session 的身份映射；普通条件查询用 `select()`。`scalars()` 返回单个 ORM 实体或值的序列；选择多列时用 `execute()` 取行。

```python
rows = (
    await session.execute(select(Todo.id, Todo.title).where(Todo.owner_id == user_id))
).all()
```

所有与当前用户有关的查询都应带上 `owner_id` 或等价授权条件。`todo_id` 唯一并不代表当前用户有权读取它。

## 20. Repository：集中数据访问

```python
# app/repositories/todos.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.todo import Todo


class TodoRepository:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def get_owned(self, todo_id: int, owner_id: int) -> Todo | None:
        stmt = select(Todo).where(Todo.id == todo_id, Todo.owner_id == owner_id)
        return await self.session.scalar(stmt)

    async def add(self, owner_id: int, title: str) -> Todo:
        todo = Todo(owner_id=owner_id, title=title)
        self.session.add(todo)
        await self.session.flush()
        return todo
```

Repository 不负责 HTTP 状态码，也不应决定“普通用户能否创建第 101 个任务”这类业务规则。它提供查询和持久化能力；Service 组合这些能力决定业务结果。

小项目可以先在 Service 中直接使用 Session，等查询逻辑确实重复或需要隔离时再抽 Repository。文件夹本身不会自动带来清晰架构。

## 21. 事务边界放在哪里

一个创建 Todo 的业务动作，通常应在一个事务里完成：

```python
async def create_todo(
    session: AsyncSession,
    owner_id: int,
    title: str,
) -> TodoOut:
    normalized = title.strip()
    if not normalized:
        raise ValueError("title must not be blank")

    async with session.begin():
        repository = TodoRepository(session)
        todo = await repository.add(owner_id, normalized)
        await session.refresh(todo)
        output = TodoOut.model_validate(todo)

    return output
```

这里 `session.begin()` 负责成功提交、异常回滚；Repository 的 `add()` 只 flush，不提前 commit。构造 `TodoOut` 发生在 Session 仍可用时，避免响应序列化时才触发隐式数据库读取。

注意：如果同一个 Session 已经因为之前的查询自动开启事务，再调用 `session.begin()` 可能报“事务已开始”。真实项目需要统一策略：在业务入口先开启事务，或由明确的请求工作单元统一管理，不要混合多种提交方式。

## 22. 关系映射与外键

数据库外键与 ORM 关系各有职责：

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    todos: Mapped[list["Todo"]] = relationship(back_populates="owner")


class Todo(Base):
    __tablename__ = "todos"

    id: Mapped[int] = mapped_column(BigInteger, Identity(), primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"), nullable=False)
    owner: Mapped["User"] = relationship(back_populates="todos")
```

这里省略了其他字段与 import，只展示关联。`ForeignKey` 让数据库检查引用完整性，`relationship()` 让 Python 代码能沿对象关系访问。只写 `relationship()` 而没有可靠数据库外键，不等于数据一定一致。

使用 `ondelete="CASCADE"` 时还要确认业务确实希望删除用户后任务跟着删除，并使 ORM 侧的关系处理与数据库策略协调。

## 23. N+1 与关系加载策略

假设列表查出 20 个 Todo，模板随后逐一访问 `todo.owner`。若每次访问都额外查数据库，就可能出现 1 次列表查询 + 20 次关联查询，这就是 N+1。

```python
from sqlalchemy.orm import selectinload

stmt = select(User).options(selectinload(User.todos))
users = (await session.scalars(stmt)).all()
```

`selectinload` 通常先查 User，再用一条或少量 `IN` 查询批量获取关联 Todo。`joinedload` 把关联数据放进同一个 JOIN 结果，集合关系可能产生重复父行，需结合结果去重：

```python
from sqlalchemy.orm import joinedload

result = await session.execute(select(User).options(joinedload(User.todos)))
users = result.unique().scalars().all()
```

异步 ORM 中避免依赖属性访问时发生隐式 I/O；查询时明确声明需要的关系。没有使用关系数据的接口，不应为了“防 N+1”无条件加载所有关系。可用 `lazy="raise"` 或 `raiseload()` 在开发中暴露意外的懒加载。

## 24. 连接池、超时与可观察性

一个慢接口可能慢在：获取连接、执行 SQL、等待锁、转换大量对象，或返回大响应。排查要分阶段观察。

```text
应用进程数 × 每进程可能占用的连接数 ≤ 数据库可用连接预算
```

实际连接数受 pool、溢出连接和部署实例数量影响。连接池太小会使请求等待，太大可能压垮数据库。`pool_pre_ping` 只检查借出的连接是否仍可用；查询超时、锁等待和整体请求超时需要另外配置并观察。

开发环境可通过 SQL 日志了解 ORM 发出什么语句，但不能在生产环境无筛选地记录完整参数或敏感值。慢查询应结合 PostgreSQL 日志、`pg_stat_activity`、执行计划与业务指标定位。

## 25. Alembic 为什么需要版本文件

修改 ORM 类只改变 Python 对表的描述，不会自动改变已经存在的数据库表。Alembic 迁移把结构变化记录为一个有顺序的版本图：

```text
初始数据库
  ↓ 001_create_users
users 表
  ↓ 002_create_todos
users + todos
  ↓ 003_add_due_at
todos 新增截止时间
```

每个迁移通常包含 `upgrade()` 和可选的 `downgrade()`，并声明上一个版本 `down_revision`。数据库中的 `alembic_version` 表记录当前版本。

生产发布应执行经过审查的迁移，而不是让每个 Web 进程启动时都运行 `Base.metadata.create_all()`。后者不管理历史变更，也不会替代迁移。

## 26. 创建 Alembic 迁移环境

在项目根目录执行一次：

```powershell
alembic init -t async alembic
```

生成的主要文件：

```text
alembic.ini
alembic/
├─ env.py
├─ script.py.mako
└─ versions/
```

异步模板适合应用使用异步驱动的例子。Alembic 命令本身仍是命令行流程，模板通过异步 Engine 建立连接，再用 `run_sync` 执行迁移逻辑。已有项目的 Alembic 环境通常需要沿用，不能重复初始化覆盖配置。

## 27. 让 Alembic 认识所有 ORM 模型

`alembic/env.py` 中最关键的连接是：

```python
from app.db.base import Base
import app.models.todo  # 导入模型，使 Todo 注册到 Base.metadata
import app.models.user  # 实际项目中还要导入所有需要迁移的模型

target_metadata = Base.metadata
```

具体导入路径要与项目结构匹配。若缺少模型导入，`Base.metadata` 可能没有相应表，自动生成结果可能为空，甚至误判数据库已有表需要删除。

数据库 URL 可以从服务端 Settings 注入 Alembic 配置，避免把真实密码写入 `alembic.ini`。配置完成后，先在独立测试数据库运行自动生成和升级，核对目标数据库绝不是生产库。

## 28. 常用迁移命令

```powershell
alembic current
alembic heads
alembic history
alembic revision --autogenerate -m "create todos"
alembic upgrade head
alembic check
```

这些命令分别用于查看数据库当前版本、代码当前头版本、历史、生成候选迁移、应用迁移，以及检查模型与数据库是否还有可自动检测的差异。

`revision --autogenerate` 是“生成候选代码”，`upgrade head` 才会实际修改目标数据库。务必确认连接的数据库和环境。`alembic check` 也继承自动检测的局限，不能证明所有结构或数据变化都已被正确迁移。

回退命令例如 `alembic downgrade -1` 会实际改数据库，仅在明确知道迁移可逆、已检查数据影响时使用。生产环境很多迁移更适合“前滚修复”，不能把 downgrade 当万能撤销。

## 29. 自动生成会漏掉什么

自动生成会比较模型元数据与现有数据库，通常能发现新表、新列、部分约束与索引变化，但不能可靠理解业务语义。例如把列 `title` 改名为 `name`，可能被识别为“删除旧列 + 新增新列”，直接执行会丢数据。

每次生成后都要人工检查：

- 是否指向正确数据库；
- 有没有意外的 `drop_table`、`drop_column`；
- 字段改名是否改为真正的 rename 操作；
- nullable、默认值、索引与外键是否正确；
- 已有数据能否通过新约束；
- 大表操作是否可能长时间持锁；
- 迁移是否与旧版、新版应用同时兼容。

生成出的迁移文件进入版本控制，和应用代码一起审查。自动生成不能替代发布计划。

## 30. 一个迁移文件长什么样

下面只展示教学形状，实际 revision ID 由 Alembic 生成，表定义需与模型和真实依赖一致：

```python
"""create todos"""

from alembic import op
import sqlalchemy as sa

revision = "002_create_todos"
down_revision = "001_create_users"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "todos",
        sa.Column("id", sa.BigInteger(), sa.Identity(), primary_key=True),
        sa.Column("owner_id", sa.BigInteger(), nullable=False),
        sa.Column("title", sa.String(100), nullable=False),
        sa.Column("completed", sa.Boolean(), server_default=sa.text("false"), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.ForeignKeyConstraint(["owner_id"], ["users.id"], ondelete="CASCADE"),
    )
    op.create_index("ix_todos_owner_id", "todos", ["owner_id"])


def downgrade() -> None:
    op.drop_index("ix_todos_owner_id", table_name="todos")
    op.drop_table("todos")
```

`downgrade()` 删除表会丢掉表中所有数据，因此这只是开发期示例，不代表生产上可以无代价回退。实际迁移 ID、约束名、索引名应保持和项目命名策略一致。

## 31. 带历史数据的安全字段变更

假设要给已有 Todo 增加必填 `priority`，不能一开始就无条件添加 `NOT NULL` 列，因为已有行没有值。常用阶段：

```text
第一步：新增 nullable 字段或安全的数据库默认值
第二步：应用兼容新旧结构，开始写入新字段
第三步：分批回填历史数据，验证没有遗漏
第四步：添加 NOT NULL / CHECK 等最终约束
第五步：后续版本删除兼容旧逻辑
```

小表可以在迁移内回填；大表通常需要单独的批处理、限速和可恢复方案。索引创建方式、锁和事务能力要按 PostgreSQL 版本与发布环境评估。对于不能在事务块中执行的操作，迁移脚本需要专门处理。

`stamp head` 只是把数据库标记为某版本，不会真正执行迁移；只有在已核实数据库结构完全对应版本时才可能使用，绝不能用它掩盖迁移失败。

## 32. FastAPI 中组装数据库依赖

```python
# app/api/todos.py
from typing import Annotated

from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.db.session import get_session
from app.schemas.todo import TodoCreate, TodoOut
from app.services.todos import create_todo

router = APIRouter(prefix="/todos", tags=["todos"])
SessionDep = Annotated[AsyncSession, Depends(get_session)]


@router.post("", response_model=TodoOut, status_code=status.HTTP_201_CREATED)
async def create_todo_endpoint(
    payload: TodoCreate,
    session: SessionDep,
    current_user: CurrentUserDep,
) -> TodoOut:
    return await create_todo(session, current_user.id, payload.title)
```

`CurrentUserDep` 是上一课认证依赖的占位名称，实际项目要导入已有定义。Router 只把已验证输入和当前用户交给业务层，不接收客户端传来的任意 owner ID。测试时可以替换 `get_session` 依赖，使用隔离的测试数据库。

## 33. 一次创建 Todo 的完整数据流

```text
1. Vue 表单提交 {"title":"学习 PostgreSQL"}
2. Axios 发送 POST /api/v1/todos
3. FastAPI + Pydantic 解析 TodoCreate
4. 认证依赖得到 current_user.id
5. Service 去空格并检查业务规则
6. Repository 添加 Todo ORM 对象
7. Session flush → PostgreSQL INSERT + 约束检查
8. 事务 commit，生成 TodoOut
9. FastAPI 返回 201 + JSON
10. Vue 把响应更新到页面状态
```

这些位置的故障不同：

```text
422 → 请求字段不符合 Pydantic Schema
401 → 未认证或凭据无效
403 → 已认证但无权限
409 → 唯一性或业务状态冲突（需由应用正确映射）
500 → 未处理异常；例如连接失败、迁移缺失或代码 Bug
```

数据库异常不能全部原样返回前端。应捕获已知约束冲突并映射为稳定错误；未知错误记录服务端日志，向客户端返回安全的通用信息。

## 34. 常见数据库错误怎样定位

| 表现 | 首先检查 | 常见原因 |
| --- | --- | --- |
| 连接被拒绝 | 地址、端口、服务状态 | 数据库未启动或网络不可达 |
| 身份验证失败 | 用户与凭据来源 | 环境变量错误、角色不存在 |
| 表不存在 | 当前数据库、Schema、迁移版本 | 连接到了错误库或没执行迁移 |
| 列不存在 | ORM 模型与实际表 | 代码和迁移未同步 |
| 外键错误 | 关联 ID 与删除策略 | 关联对象不存在或仍被引用 |
| 唯一约束错误 | 数据库约束名与业务输入 | 并发创建相同唯一值 |
| 请求挂住 | 活跃事务与锁等待 | 长事务、锁顺序冲突 |
| 列表慢 | 实际 SQL 与执行计划 | 缺索引、N+1、一次加载过多 |
| 异步懒加载错误 | 属性访问位置 | Session 已关闭或发生隐式 I/O |

按“请求参数 → 当前用户 → ORM 生成 SQL → 目标数据库 → 迁移版本 → 执行计划”逐层排查。不要看到数据库错误就立即删库重建。

## 35. 测试如何分层

```text
Service 单元测试：业务规则，不依赖真实数据库
Repository 集成测试：在测试数据库验证 SQL、约束和事务
API 测试：验证状态码、Schema、认证和响应
迁移测试：空库升级、已有数据升级、必要时验证回滚或前滚
```

PostgreSQL 特有类型、事务和索引行为，最终要用 PostgreSQL 测试；SQLite 不能完全模拟它。测试数据库应与开发/生产数据隔离。每个测试结束时回滚或重建必要数据，避免测试顺序影响结果。

值得覆盖的边界包括：

- 创建成功后再次查询能读到；
- 创建空标题被拒绝；
- 其他用户无法读取或修改该 Todo；
- 唯一约束冲突能映射成稳定错误；
- 出错事务不留下半成品；
- 一次列表请求不会随记录数量线性增加 SQL 次数。

## 36. 安全与发布边界

- 所有查询都应限制资源所有者或租户；仅凭“前端隐藏按钮”不能保证权限；
- SQL 参数值使用绑定参数，不拼接用户输入；动态排序列必须白名单；
- 数据库账号使用最小权限；迁移账号可能需要更高 DDL 权限，应单独管理；
- 密钥只保存在服务端配置系统，日志不打印完整连接 URL；
- 上线前备份并验证恢复流程，检查迁移预计锁时间和旧版应用兼容性；
- 多实例部署通常只由发布流程执行一次迁移，不让每个 Web 实例抢着执行。

需要遵守的是“明确谁能读写什么、什么操作必须在同一事务完成、结构变化如何发布”，不是盲目叠加工具。

## 37. 给 AI 的开发指令

### 在已有项目接入 Todo 持久化

```text
请先检查现有 FastAPI 项目的 Python 版本、依赖管理方式、SQLAlchemy 是同步还是异步、数据库配置、Base、Session 依赖和 Alembic 环境。
沿用现有架构实现 Todo 持久化，不重新建立第二套 Engine 或 Base。
使用 SQLAlchemy 2.x 的 Mapped、mapped_column 和 select；分别保留 ORM Model、TodoCreate、TodoUpdate、TodoOut。
Todo 的 owner_id 从认证用户取得，所有读取/修改/删除都按 owner_id 限制。
在业务工作单元明确 commit/rollback，Repository 不自行 commit。
新增或修改表结构必须提供并人工审查 Alembic 迁移，不在启动时 create_all。
补充成功、越权、校验、约束冲突和事务失败测试，报告实际运行结果。
```

### 排查慢查询与 N+1

```text
请定位 Todo 列表接口变慢的原因。
先记录实际 SQL 次数、SQL 文本形状、耗时、返回行数，再查看 PostgreSQL EXPLAIN (ANALYZE, BUFFERS)。
检查是否发生 N+1、缺少 owner_id 过滤、排序不稳定、加载了不需要的关系或分页上限缺失。
只在有证据时调整 selectinload/joinedload、查询列或索引，并解释写入成本。
用相同数据量对比优化前后的 SQL 次数与耗时，不要凭感觉宣布性能提升。
```

### 审查 Alembic 迁移

```text
请审查这次 Alembic autogenerate 生成的迁移文件和目标数据库状态。
核对每个 add/drop/alter 是否对应需求，特别检查重命名、NOT NULL、默认值、外键、索引和历史数据。
评估表大小、锁、旧版应用兼容、回填方式和失败恢复。
先在隔离数据库验证空库升级与已有数据升级；不要直接连接生产库执行。
给出可执行的发布顺序和验证点，不要用 stamp head 掩盖迁移失败。
```

### 审查事务与并发

```text
请检查当前 FastAPI + SQLAlchemy 代码的 Session 生命周期与事务边界。
找出共享全局 AsyncSession、Repository 内部 commit、异常后未回滚、长事务内调用外部 API、并发任务共用 Session 的问题。
核对版本冲突、唯一约束错误、死锁和可安全重试的范围。
先说明证据和最小修复，再修改代码，并用数据库集成测试验证。
```

## 38. 审核 AI 生成数据库代码的清单

```text
[ ] SQLAlchemy 同步/异步路线与现有项目一致
[ ] 只使用现有 Engine、Base、Settings 和 Alembic 环境
[ ] ORM Model 与 Pydantic 请求/响应 Schema 分离
[ ] 字段类型、NULL、默认值、主外键和删除策略符合业务
[ ] owner_id 等权限条件写入所有相关查询
[ ] Session 每个请求/工作单元独立，未跨并发任务共享
[ ] flush、commit、rollback 的位置和含义正确
[ ] Repository 没有私自提前 commit
[ ] 异步流程无同步阻塞数据库调用和隐式懒加载
[ ] 关系加载符合实际接口需求，没有 N+1
[ ] 查询列、排序、分页与索引相匹配
[ ] Alembic 迁移已人工审查，未出现意外删表/删列
[ ] 迁移能处理已有数据与新旧应用兼容
[ ] 密钥不进入代码、前端构建产物或日志
[ ] 有 PostgreSQL 集成测试和错误路径验证
```

## 39. 面试表达速记

### PostgreSQL 与 ORM

> PostgreSQL 保存并约束真实数据，SQLAlchemy 把 Python 模型和 SQL 操作连接起来。ORM 不能替代对 SQL、索引和事务的理解，性能问题最终仍要看实际 SQL 和执行计划。

### 数据库、Schema、表与角色

> 数据库是相对独立的数据集合，Schema 是数据库内部命名空间，表存放记录，Role 是权限身份。Pydantic Schema 描述接口数据，和 PostgreSQL Schema 不是同一个概念。

### MVCC 与 VACUUM

> PostgreSQL 用 MVCC 管理并发可见性，更新和删除后旧行版本不会立即消失；VACUUM 回收可重用空间，长事务可能阻碍清理并导致膨胀。

### Session 与事务

> SQLAlchemy Session 是一个工作单元的 ORM 状态容器，维护对象身份并协调数据库操作。flush 把待写 SQL 发给数据库但不提交，commit 才结束事务；业务原子性要求提交点在完整业务动作外层。

### 异步 Session

> AsyncSession 对应异步数据库 I/O，但同一个实例仍是有状态对象，不能在多个并发任务之间共享。应按请求或任务创建独立 Session，并避免异步代码中触发隐式懒加载。

### N+1

> N+1 是查询一批主对象后，访问每个对象的关系都再发一条查询。可以按需要使用 selectinload、joinedload 或明确 JOIN，并检查实际 SQL 数量与结果集大小。

### Alembic

> Alembic 用有版本的迁移脚本管理数据库结构。自动生成只提供候选修改，字段重命名、数据回填和高风险 DDL 必须人工审查并在测试数据库验证。

### 发布迁移

> 现有大表新增必填字段通常采用扩展、回填、加约束、清理旧逻辑的阶段化方式，避免旧版应用与新结构冲突，也要评估锁和回退或前滚方案。

## 40. 本课技术地图

```text
Todo 持久化
├─ PostgreSQL
│  ├─ Database / Schema / Role
│  ├─ Table / Type / Constraint / Index
│  ├─ MVCC / VACUUM / Lock
│  └─ EXPLAIN / Transaction
├─ SQLAlchemy 2.x
│  ├─ Engine / Pool
│  ├─ DeclarativeBase / Mapped / mapped_column
│  ├─ select / Session / flush / commit
│  ├─ relationship / selectinload / joinedload
│  └─ AsyncSession / FastAPI Depends
└─ Alembic
   ├─ env.py / target_metadata
   ├─ revision / upgrade / downgrade
   ├─ autogenerate + 人工审查
   └─ 兼容迁移 / 回填 / 发布验证
```

## 41. 官方参考资料

- [PostgreSQL 官方文档](https://www.postgresql.org/docs/current/)：数据类型、MVCC、VACUUM、索引与执行计划。
- [SQLAlchemy 2.0 ORM 文档](https://docs.sqlalchemy.org/en/20/orm/quickstart.html)：声明式模型与查询。
- [SQLAlchemy 异步文档](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)：AsyncSession、异步连接与隐式 I/O 边界。
- [Alembic 教程](https://alembic.sqlalchemy.org/en/latest/tutorial.html)：迁移环境和命令。
- [Alembic 自动生成说明](https://alembic.sqlalchemy.org/en/latest/autogenerate.html)：检测范围与人工审查要求。

## 42. 下一步

下一课适合学习 **Docker 与应用容器化**：把 Vue、FastAPI 和 PostgreSQL 放进可复现的开发环境，理解镜像、容器、数据卷、网络与 Compose，并说明为什么数据库持久数据不能只留在容器可写层。

[进入下一课：Docker 与 Compose 应用容器化 →](./14-Docker与Compose应用容器化.md)
