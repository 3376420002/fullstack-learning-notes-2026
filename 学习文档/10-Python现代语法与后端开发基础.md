# 10｜Python 现代语法与后端开发基础

> 所属阶段：Python 后端基础  
> 本课合并知识点：运行环境、虚拟环境、基础语法、容器、函数、模块、异常、类型注解、dataclass、Protocol、异步、上下文管理器、项目结构、测试  
> 前置知识：编程基础、HTTP、JSON、前端接口调用  
> 学习目标：熟悉 FastAPI 项目中高频使用的 Python 能力，能够读懂和指挥 AI 编写后端代码，并能在面试中准确说明核心机制。

[← Axios 与接口层](./09-Axios与前后端接口层.md) · [学习首页](../README.md)

## 阅读方式

- 第 1～16 节掌握 Python 语法、函数、模块和异常；
- 第 17～27 节理解类型、对象、迭代器和上下文管理器；
- 第 28～35 节重点理解异步、工程结构、依赖和测试；
- 第 36～39 节用于 AI 编程、代码审核和面试复习；
- 不需要背诵冷门语法，重点理解数据类型、边界、异常和异步执行模型。

## 1. Python 在当前技术栈中的位置

当前项目的后端链路：

```text
Vue + Axios
    ↓ HTTP / JSON
FastAPI
    ↓ Python 函数与对象
Service
    ↓ SQLAlchemy
PostgreSQL
```

Python 负责运行 FastAPI 应用、表达业务规则、调用数据库、执行后台任务和编写测试。

学习目标不是记住所有内置函数，而是掌握后端开发最常见的能力：

```text
数据结构 + 函数 + 类型 + 异常 + 模块 + 异步 + 工程化
```

## 2. Python 的执行环境

Python 是语言规范，CPython 是最常见的 Python 实现。

概念链路：

```text
.py 源代码
   ↓ 解析与编译
Python 字节码
   ↓
Python 虚拟机执行
```

字节码缓存可能出现在 `__pycache__/`，不应作为业务源码手工维护。

检查环境：

```powershell
python --version
python -c "import sys; print(sys.executable)"
```

Windows 还可能提供 Python Launcher：

```powershell
py --list
py -3.12 --version
```

具体项目应使用其声明并支持的 Python 版本，而不是盲目切换到本机最新版本。

## 3. 虚拟环境

虚拟环境为项目提供独立的 Python 可执行文件和依赖目录：

```powershell
python -m venv .venv
```

PowerShell 激活：

```powershell
.\.venv\Scripts\Activate.ps1
```

检查当前解释器：

```powershell
python -c "import sys; print(sys.executable)"
```

也可以不激活，直接使用虚拟环境解释器：

```powershell
.\.venv\Scripts\python.exe -m pip install fastapi
```

虚拟环境隔离的是 Python 包和解释器选择，不是 Docker 容器，也不会自动隔离数据库、端口和操作系统资源。

通常不提交 `.venv/`，而是提交依赖声明和锁定文件。

## 4. 安装依赖的基本原则

优先使用当前解释器调用 pip：

```powershell
python -m pip install fastapi
```

相比直接执行 `pip`，这种方式更明确地绑定当前 `python`。

项目可能使用不同工具：

```text
venv + pip
uv
Poetry
PDM
pip-tools
```

不要在同一个项目里随意混用多个锁文件和依赖工作流。先阅读 `pyproject.toml`、README 和 CI 配置，再使用项目规定的工具。

## 5. 变量、对象与动态类型

```python
title = "学习 Python"
count = 3
completed = False
```

Python 变量更准确地说是绑定到对象的名称：

```python
value = 1
value = "now a string"
```

Python 是动态类型语言，名称可以重新绑定到不同类型对象；同时它也是强类型语言，不会任意把不兼容类型静默混合：

```python
1 + "2"  # TypeError
```

类型注解可以让工具提前发现不合理绑定，但 Python 运行时默认不会因为注解自动阻止赋值。

## 6. 常见基础类型

```python
name: str = "Alice"
age: int = 20
score: float = 95.5
enabled: bool = True
missing: None = None
```

常见注意点：

- `bool` 是 `int` 的子类，但业务上应保持语义清楚；
- 浮点数不适合直接表示精确金额；
- `None` 表示缺失或无值，不等于空字符串和数字零；
- Python 整数不会像固定宽度整数那样轻易溢出，但仍受内存限制。

金额通常使用 `Decimal`：

```python
from decimal import Decimal

price = Decimal("19.99")
```

使用字符串构造能避免先经过二进制浮点造成误差。

## 7. 字符串

```python
title = "学习 FastAPI"
message = f"任务：{title}"
```

常见操作：

```python
normalized = title.strip()
lower_name = name.lower()
parts = "a,b,c".split(",")
path = "/".join(["api", "v1", "todos"])
```

字符串是不可变对象。所谓修改通常会创建新字符串：

```python
title = title.strip()
```

大量片段拼接优先使用 `"".join(parts)`，而不是在循环中反复 `+=`。

面向数据库时不要使用 f-string 拼接 SQL。应使用 SQLAlchemy 或参数化查询，避免 SQL 注入。

## 8. List、Tuple、Dict 与 Set

### List

有顺序、可修改：

```python
todos: list[str] = ["学习 Vue", "学习 Python"]
todos.append("学习 FastAPI")
```

### Tuple

有顺序、不可修改：

```python
point: tuple[int, int] = (10, 20)
```

### Dict

键值映射：

```python
todo: dict[str, object] = {
    "id": 1,
    "title": "学习 Python",
    "completed": False,
}
```

现代 Python 的 dict 保持插入顺序，但业务逻辑不应依赖数据库或外部 JSON 恰好使用某种字段顺序。

### Set

唯一元素集合：

```python
roles: set[str] = {"user", "editor"}
roles.add("admin")
```

Set 适合去重和成员判断，不保证可用于表达需要稳定业务顺序的列表。

## 9. 可变与不可变

常见不可变对象：

```text
int、float、bool、str、tuple、frozenset、None
```

常见可变对象：

```text
list、dict、set、大多数自定义对象
```

赋值不会自动复制对象：

```python
first = [1, 2]
second = first
second.append(3)

assert first == [1, 2, 3]
```

浅复制：

```python
copy_of_list = first.copy()
copy_of_dict = todo.copy()
```

浅复制只复制外层容器，内部嵌套可变对象仍可能共享。深复制应谨慎使用，通常更应先明确数据所有权。

## 10. 条件、真值与比较

```python
if not title.strip():
    raise ValueError("title must not be empty")
```

常见假值：

```text
False、None、0、0.0、""、[]、{}、set()
```

相等与身份：

```python
left == right   # 值是否相等
left is right   # 是否为同一个对象
```

判断 None 使用：

```python
if value is None:
    ...
```

不要使用 `is` 比较普通字符串或数字值。

链式比较：

```python
if 1 <= page <= 100:
    ...
```

## 11. 循环与推导式

```python
for todo in todos:
    print(todo)
```

同时获得索引：

```python
for index, todo in enumerate(todos, start=1):
    print(index, todo)
```

并行遍历：

```python
for key, value in zip(keys, values, strict=True):
    print(key, value)
```

列表推导式：

```python
active_titles = [
    todo.title
    for todo in todos
    if not todo.completed
]
```

推导式适合简洁转换和过滤。包含多层嵌套、异常处理或复杂副作用时，普通循环更可读。

## 12. 函数

```python
def normalize_title(title: str) -> str:
    normalized = title.strip()
    if not normalized:
        raise ValueError("title must not be empty")
    return normalized
```

函数应有清晰输入、输出和异常契约。

关键字参数：

```python
def list_todos(*, page: int = 1, page_size: int = 20) -> list[str]:
    ...

list_todos(page=2, page_size=50)
```

`*` 后的参数只能通过名称传递，适合参数较多、布尔参数或顺序容易混淆的函数。

位置参数与关键字参数边界也可以使用 `/` 和 `*` 明确表示，但业务代码中应以可读性为主。

## 13. 可变默认参数陷阱

错误写法：

```python
def add_tag(tag: str, tags: list[str] = []) -> list[str]:
    tags.append(tag)
    return tags
```

默认值在函数定义时创建，多次调用会共享同一个 list。

正确写法：

```python
def add_tag(
    tag: str,
    tags: list[str] | None = None,
) -> list[str]:
    result = [] if tags is None else list(tags)
    result.append(tag)
    return result
```

不可变默认值通常安全：

```python
def list_todos(page: int = 1) -> list[str]:
    ...
```

## 14. `*args` 与 `**kwargs`

```python
def log_event(event: str, *tags: str, **details: object) -> None:
    print(event, tags, details)

log_event(
    "todo.created",
    "todo",
    "write",
    todo_id=42,
    actor_id=7,
)
```

它们适合包装器、框架适配和参数转发，但业务函数若大量依赖任意参数，类型与契约会变得模糊。

参数解包：

```python
options = {"page": 2, "page_size": 50}
list_todos(**options)
```

传入前应确保键名符合函数签名。

## 15. 模块与包

一个 `.py` 文件通常是模块，包含模块的目录可以构成包：

```text
app/
├─ __init__.py
├─ main.py
├─ services/
│  ├─ __init__.py
│  └─ todos.py
└─ models/
   ├─ __init__.py
   └─ todo.py
```

导入：

```python
from app.services.todos import create_todo
```

推荐使用清晰的绝对导入。避免通过修改 `sys.path` 或依赖当前工作目录碰巧导入成功。

模块导入时，顶层代码会执行。顶层应主要放定义和轻量配置，不应在 import 时启动服务器、连接外部系统或执行耗时任务。

## 16. `__name__` 与程序入口

```python
def main() -> None:
    print("application started")


if __name__ == "__main__":
    main()
```

直接执行文件时，`__name__` 为 `"__main__"`；作为模块导入时，条件内代码不会运行。

FastAPI 常由 ASGI 服务器按导入路径加载应用：

```powershell
uvicorn app.main:app --reload
```

其中第一个 `app.main` 是模块路径，第二个 `app` 是模块中的 FastAPI 实例名称。

## 17. 异常处理

```python
try:
    todo = repository.get(todo_id)
except DatabaseError as exc:
    logger.exception("failed to load todo", extra={"todo_id": todo_id})
    raise ServiceUnavailableError() from exc
else:
    return todo
finally:
    metrics.record_query()
```

结构：

```text
try       可能失败的代码
except    处理指定异常
else      没有异常时执行
finally   无论成功失败都执行
```

不要使用空的广泛捕获：

```python
try:
    run()
except Exception:
    pass
```

它会隐藏编程错误和真实故障。

## 18. 自定义业务异常

```python
class TodoError(Exception):
    """Todo 领域异常基类。"""


class TodoNotFoundError(TodoError):
    def __init__(self, todo_id: int) -> None:
        self.todo_id = todo_id
        super().__init__(f"todo {todo_id} was not found")


class DuplicateTodoError(TodoError):
    pass
```

Service 抛业务异常，FastAPI 边界再映射为 HTTP：

```text
TodoNotFoundError    → 404
DuplicateTodoError   → 409
PermissionError      → 403
```

不要让业务层到处直接抛 `HTTPException`，否则核心逻辑会长期依赖 HTTP 表现层。

异常链：

```python
raise TodoRepositoryError() from exc
```

它保留底层原因，便于日志和调试，同时允许上层暴露更稳定的异常类型。

## 19. 类型注解

```python
def find_todo(todo_id: int) -> Todo | None:
    ...
```

常见类型：

```python
names: list[str]
todo_by_id: dict[int, Todo]
result: Todo | None
pair: tuple[int, str]
callback: Callable[[Todo], bool]
```

类型注解主要服务：

- 编辑器补全；
- 静态类型检查；
- 框架读取元数据；
- API 和函数契约说明；
- 安全重构。

Python 默认不会仅凭注解自动校验运行时数据。FastAPI 和 Pydantic 会主动读取注解并执行额外行为，这是框架能力。

## 20. 类型缩小与 `None`

```python
todo = repository.find(todo_id)

if todo is None:
    raise TodoNotFoundError(todo_id)

return todo.title
```

`if todo is None` 之后，类型检查器可以把后续 `todo` 缩小为非 None 类型。

不要用断言掩盖真实可空情况：

```python
todo = repository.find(todo_id)
assert todo is not None
```

`assert` 主要用于开发期不变量，运行时可能被优化模式关闭，也不适合处理用户可触发的正常错误分支。

## 21. Type Alias 与 NewType

类型别名：

```python
from typing import TypeAlias

TodoId: TypeAlias = int
```

它提升可读性，但静态类型上仍与 int 等价。

需要静态区分不同 ID 时：

```python
from typing import NewType

TodoId = NewType("TodoId", int)
UserId = NewType("UserId", int)
```

`NewType` 主要帮助静态检查，运行时开销很小，也不会自动验证数据库值。

项目应根据复杂度选择，不要为了形式给每个字段创建新类型。

## 22. dataclass

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True, slots=True)
class Todo:
    id: int
    title: str
    completed: bool
    created_at: datetime
```

dataclass 自动生成初始化、表示和比较等样板方法。

参数含义：

```text
frozen=True   阻止普通属性重新赋值，表达值对象语义
slots=True    使用 slots，减少动态属性并可能降低内存开销
```

dataclass 不是 Pydantic 替代品。普通 dataclass 不会自动解析和验证外部 JSON。它常适合作为内部命令对象、值对象和领域对象。

可变默认字段使用工厂：

```python
from dataclasses import dataclass, field


@dataclass
class TodoGroup:
    todos: list[Todo] = field(default_factory=list)
```

## 23. Enum

```python
from enum import StrEnum


class TodoStatus(StrEnum):
    ACTIVE = "active"
    COMPLETED = "completed"
```

使用：

```python
status = TodoStatus.ACTIVE
assert status.value == "active"
```

Enum 适合有限、稳定的取值集合，例如状态、角色和排序方向。

如果项目需要兼容不提供 `StrEnum` 的 Python 版本，可以使用：

```python
from enum import Enum


class TodoStatus(str, Enum):
    ACTIVE = "active"
    COMPLETED = "completed"
```

数据库和 API 如何保存 Enum 值必须形成明确约定。

## 24. 类与对象

```python
class TodoService:
    def __init__(self, repository: TodoRepository) -> None:
        self._repository = repository

    def complete(self, todo_id: int) -> Todo:
        todo = self._repository.get(todo_id)
        if todo.completed:
            return todo
        return self._repository.mark_completed(todo_id)
```

`self` 表示当前实例，由 Python 在实例方法调用时传入。

后端使用类的合理场景：

- 组合依赖；
- 保存明确生命周期状态；
- 表达实体和值对象；
- 实现可替换接口；
- 把相关业务行为组织在一起。

简单、无状态的转换函数不必强行放入工具类。Python 项目通常同时使用函数和类。

## 25. 组合优先于复杂继承

```python
class TodoService:
    def __init__(
        self,
        repository: TodoRepository,
        notifier: TodoNotifier,
    ) -> None:
        self._repository = repository
        self._notifier = notifier
```

组合通过持有协作者构建能力，比多层继承更容易替换和测试。

继承适合确实存在的 is-a 关系和框架扩展点。不要创建 `BaseManager`、`AbstractBaseService` 等多层结构，只为了看起来“面向对象”。

## 26. Protocol 与依赖倒置

```python
from typing import Protocol


class TodoRepository(Protocol):
    def get(self, todo_id: int) -> Todo:
        ...

    def save(self, todo: Todo) -> Todo:
        ...
```

任何提供兼容方法的对象都可以满足 Protocol，不必显式继承：

```python
class InMemoryTodoRepository:
    def get(self, todo_id: int) -> Todo:
        ...

    def save(self, todo: Todo) -> Todo:
        ...
```

这称为结构化类型。Service 可以依赖行为契约，而不是依赖 SQLAlchemy 的具体实现。

小项目不必为每个类创建 Protocol。只有需要替换实现、隔离基础设施或增强测试边界时才有明显价值。

## 27. 迭代器与生成器

可迭代对象可以用于 `for`：

```python
for todo in todos:
    ...
```

生成器使用 `yield` 按需产生值：

```python
from collections.abc import Iterator


def iter_active_titles(todos: list[Todo]) -> Iterator[str]:
    for todo in todos:
        if not todo.completed:
            yield todo.title
```

生成器特点：

- 惰性执行；
- 不需要一次性构建完整结果列表；
- 只能按迭代过程消费；
- 异常可能在迭代时而不是创建时出现。

数据库查询是否真正流式取决于 ORM、驱动和查询配置，不能看到生成器语法就断定不会占用大量内存。

## 28. 上下文管理器

```python
from pathlib import Path

path = Path("data.json")

with path.open("r", encoding="utf-8") as file:
    content = file.read()
```

离开 `with` 代码块时，即使发生异常，也会执行资源清理。

它适合：

- 文件；
- 数据库事务；
- 锁；
- 临时目录；
- 网络客户端；
- 跟踪和计时范围。

自定义上下文管理器：

```python
from contextlib import contextmanager
from collections.abc import Iterator


@contextmanager
def transaction() -> Iterator[None]:
    begin()
    try:
        yield
    except Exception:
        rollback()
        raise
    else:
        commit()
```

真实数据库事务应使用数据库库提供的事务上下文，不要自行模拟底层一致性。

## 29. 装饰器

装饰器包装函数或类：

```python
from collections.abc import Callable
from functools import wraps
from time import perf_counter
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")


def timed(func: Callable[P, R]) -> Callable[P, R]:
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        started_at = perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            print(func.__name__, perf_counter() - started_at)

    return wrapper
```

FastAPI 的路由写法就是装饰器：

```python
@router.get("/todos")
async def list_todos() -> list[TodoOut]:
    ...
```

装饰器适合日志、缓存、授权声明和框架注册等横切能力，但多层装饰会增加调用链调试难度。

## 30. 同步、异步与协程

同步函数：

```python
def calculate_total(values: list[int]) -> int:
    return sum(values)
```

异步函数：

```python
async def load_todo(todo_id: int) -> Todo:
    result = await repository.get(todo_id)
    return result
```

调用异步函数会创建协程对象，只有被 `await`、创建任务或交给事件循环后才会推进执行。

```text
async def   定义协程函数
await       暂停当前协程，等待可等待对象
event loop  调度可运行的协程和 I/O 事件
```

异步的主要价值是让一个线程在等待网络或数据库 I/O 时处理其他任务，不是让单个 CPU 计算自动变快。

## 31. 并发执行异步任务

Python 3.11 及以上可以使用 `TaskGroup`：

```python
import asyncio


async def load_dashboard(user_id: int) -> tuple[User, list[Todo]]:
    async with asyncio.TaskGroup() as group:
        user_task = group.create_task(load_user(user_id))
        todos_task = group.create_task(load_todos(user_id))

    return user_task.result(), todos_task.result()
```

也可以使用：

```python
user, todos = await asyncio.gather(
    load_user(user_id),
    load_todos(user_id),
)
```

只有相互独立的任务才适合并发。并发会增加数据库连接、外部 API 和内存压力，需要限流和超时策略。

## 32. 异步中的阻塞问题

错误：

```python
import time


async def endpoint() -> None:
    time.sleep(5)
```

`time.sleep` 会阻塞事件循环线程。

异步等待：

```python
import asyncio


async def endpoint() -> None:
    await asyncio.sleep(5)
```

重要边界：

- 异步函数应调用异步数据库驱动和异步 HTTP 客户端；
- 同步阻塞库不能因为外面写了 `async def` 就变成异步；
- FastAPI 可以用线程池运行普通同步端点，但线程资源也有限；
- CPU 密集任务更适合进程池、任务队列或独立计算服务；
- 不要在请求中执行长时间视频处理、模型训练等任务。

## 33. 日期、时间与时区

```python
from datetime import UTC, datetime

created_at = datetime.now(UTC)
```

后端建议使用带时区的 UTC 时间保存和传输：

```text
2026-09-19T03:00:00Z
```

避免混用 naive datetime 和 aware datetime：

```python
datetime.now()      # 通常无时区信息
datetime.now(UTC)   # 明确 UTC
```

显示给用户时再根据其时区转换。数据库列、Pydantic Schema 和前端日期解析需要保持同一契约。

## 34. Path、JSON 与环境变量

文件路径：

```python
from pathlib import Path

project_root = Path(__file__).resolve().parents[1]
config_path = project_root / "config" / "settings.json"
```

JSON：

```python
import json

payload = json.loads('{"title": "学习 Python"}')
text = json.dumps(payload, ensure_ascii=False)
```

标准 JSON 不直接支持 datetime、Decimal、set 和任意 Python 对象，需要明确编码策略。

环境变量：

```python
import os

database_url = os.environ["DATABASE_URL"]
```

`os.environ[...]` 在缺失时立即报错，适合必需配置。不要把数据库密码和密钥硬编码进源码。后续 FastAPI 课程会使用 Pydantic Settings 统一解析配置。

## 35. 后端项目结构

推荐起点：

```text
backend/
├─ pyproject.toml
├─ README.md
├─ .env.example
├─ src/
│  └─ app/
│     ├─ __init__.py
│     ├─ main.py
│     ├─ api/
│     ├─ core/
│     ├─ models/
│     ├─ repositories/
│     ├─ schemas/
│     └─ services/
└─ tests/
   ├─ unit/
   └─ integration/
```

目录职责：

| 目录 | 职责 |
| --- | --- |
| `api/` | HTTP 路由和依赖 |
| `core/` | 配置、安全、日志等基础能力 |
| `schemas/` | Pydantic 请求和响应模型 |
| `models/` | SQLAlchemy 持久化模型 |
| `repositories/` | 数据访问抽象 |
| `services/` | 业务流程与规则 |
| `tests/` | 单元和集成测试 |

目录应随真实职责产生。小项目可以更简单，不需要一开始创建所有层。

## 36. pyproject.toml 与开发工具

`pyproject.toml` 是现代 Python 项目的核心配置入口之一：

```toml
[project]
name = "todo-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
  "fastapi",
  "uvicorn[standard]",
]

[project.optional-dependencies]
dev = [
  "pytest",
  "pytest-asyncio",
  "ruff",
  "mypy",
]

[tool.ruff]
line-length = 100

[tool.mypy]
python_version = "3.12"
strict = true
```

版本只作为结构示例，真实项目应锁定经过测试的兼容版本。

工具职责：

```text
Ruff       lint 和格式化
mypy       静态类型检查
Pyright    另一种常用静态类型检查器
pytest     自动化测试
coverage   测试覆盖率统计
```

常见命令：

```powershell
python -m ruff check .
python -m ruff format --check .
python -m mypy src
python -m pytest
```

具体命令应以项目依赖和配置为准。

## 37. pytest 基础

业务函数：

```python
def normalize_title(title: str) -> str:
    normalized = title.strip()
    if not normalized:
        raise ValueError("title must not be empty")
    return normalized
```

测试：

```python
import pytest

from app.services.todos import normalize_title


def test_normalize_title_strips_whitespace() -> None:
    assert normalize_title("  Learn Python  ") == "Learn Python"


def test_normalize_title_rejects_empty_value() -> None:
    with pytest.raises(ValueError, match="must not be empty"):
        normalize_title("   ")
```

测试重点：

- 正常结果；
- 边界值；
- 业务异常；
- 权限和状态变化；
- 外部依赖失败；
- 异步并发行为。

不要只测试代码行被执行，还要验证可观察行为和业务结果。

## 38. 日志

```python
import logging

logger = logging.getLogger(__name__)


def get_todo(todo_id: int) -> Todo:
    logger.info("loading todo", extra={"todo_id": todo_id})
    return repository.get(todo_id)
```

异常日志：

```python
try:
    return repository.get(todo_id)
except DatabaseError:
    logger.exception("database query failed", extra={"todo_id": todo_id})
    raise
```

日志应包含有助定位问题的上下文，例如请求 ID、用户的内部标识、资源 ID 和耗时。

禁止记录：

- 密码；
- Access Token 和 Refresh Token；
- Cookie；
- 数据库连接密码；
- 完整银行卡或高敏感个人信息；
- 不必要的完整请求体。

生产项目通常使用结构化日志，而不是散落的 `print()`。

## 39. 常见错误与准确理解

### 类型注解等于运行时校验

错误。普通 Python 默认不执行注解校验；Pydantic 等库会主动读取注解并校验。

### 默认参数每次调用都会重新创建

错误。默认值在函数定义时创建，可变默认值会跨调用共享。

### `is` 与 `==` 可以互换

错误。`==` 比较值，`is` 比较对象身份。None 使用 `is None`。

### `async def` 会让所有代码并行

错误。只有遇到真正可等待的非阻塞操作并把控制权交还事件循环，其他任务才有机会执行。

### 在 async 函数中调用同步数据库库没有影响

错误。阻塞调用可能卡住事件循环，需要同步端点、线程池或异步驱动等合适方案。

### 捕获 Exception 后返回空结果更稳定

错误。它会把数据库故障、编程错误和数据不存在混成同一结果，破坏可观察性。

### dataclass 可以代替 Pydantic Schema

不准确。dataclass 适合内部数据对象；Pydantic 主要解决外部数据解析、校验和序列化。

### 多层继承代表架构专业

错误。后端项目通常优先组合、清晰函数和小型对象，只有真实替换关系才使用继承。

### 虚拟环境保证依赖完全可复现

不完整。虚拟环境负责隔离，精确复现还需要依赖锁定、Python 版本、系统库和一致安装流程。

## 40. 指挥 AI 编写 Python 代码

### 创建后端模块指令

```text
请在现有 Python 后端项目中实现 Todo Service。
先读取 pyproject.toml、现有类型、异常、Repository 和代码风格。
使用项目支持的 Python 版本和完整类型注解，不使用可变默认参数或无理由的 Any。
Service 只表达业务规则，不直接依赖 FastAPI HTTPException。
通过构造参数接收 Repository，数据不存在时抛现有领域异常。
只修改相关文件，完成后运行 Ruff、类型检查和 pytest。
```

### 异步代码指令

```text
请审查这个 FastAPI 调用链的同步与异步边界。
识别阻塞数据库、文件、HTTP 请求、time.sleep 和 CPU 密集操作。
不要只把 def 改成 async def；确认底层库是否真正异步。
独立 I/O 可以使用 TaskGroup 或 gather，并说明异常、取消、超时和连接池压力。
保持事务边界正确，完成后增加能够覆盖并发失败的测试。
```

### 类型改进指令

```text
请在不改变运行行为的前提下完善这个 Python 模块的类型。
catch 之外避免无边界 Any，正确表达 None、容器、Callable 和异步返回值。
需要替换实现时使用小型 Protocol，不要为所有类机械创建接口。
不要用 cast 或 type: ignore 掩盖真实错误；每个忽略都必须有具体理由。
运行项目现有 mypy 或 Pyright 配置并修复根因。
```

### 测试指令

```text
请为现有 Python 业务函数补充 pytest 测试。
覆盖正常结果、边界值、领域异常和外部依赖失败。
使用项目已有 fixture 和 mock 方式，不 mock 被测函数自身，不断言内部实现细节。
异步测试遵守现有 pytest-asyncio 配置。
测试名称描述可观察行为，并运行相关测试与完整测试集。
```

### 排错指令

```text
请根据完整 traceback、输入和相关代码定位 Python 错误。
从异常的最底层原因开始分析，区分业务异常、类型错误、导入路径、环境依赖和异步阻塞。
先说明证据和可复现条件，再做最小修改。
不要用 except Exception: pass、关闭类型检查或修改 sys.path 掩盖问题。
修改后执行最小复现、相关测试、lint 和类型检查。
```

## 41. 审核 AI 生成 Python 代码的清单

```text
[ ] 使用项目声明的 Python 版本和依赖工具
[ ] 没有提交 .venv、缓存或本机秘密
[ ] 函数输入、返回值和可空性具有清晰类型
[ ] 没有可变默认参数
[ ] is 与 == 使用语义正确
[ ] 异常捕获具体且没有静默吞错
[ ] 领域层没有不必要地依赖 HTTPException
[ ] dataclass、Pydantic 和 ORM 模型职责没有混淆
[ ] async 调用链没有隐藏的阻塞 I/O
[ ] 并发任务考虑异常、取消、超时和资源上限
[ ] 文件、事务和客户端使用上下文管理器清理
[ ] 日志不包含 Token、Cookie、密码和敏感数据
[ ] 没有无理由的 Any、cast 或 type: ignore
[ ] 单元测试验证行为而不是内部实现
[ ] Ruff、类型检查和 pytest 按项目配置通过
```

## 42. 面试表达速记

### 动态类型与强类型

> Python 是动态类型语言，变量名可以绑定不同类型对象；同时它具有强类型语义，不会任意把不兼容类型静默相加。类型注解主要用于工具和框架，默认不等于运行时校验。

### List 与 Tuple

> List 有顺序且可修改，适合变化的集合；Tuple 有顺序且不可修改，适合固定结构或不可变语义。是否可哈希还取决于其中元素。

### 可变默认参数

> Python 的默认参数在函数定义时求值，因此 list 或 dict 默认值会被多次调用共享。通常使用 None，再在函数内部创建新对象，或 dataclass 中使用 default_factory。

### `is` 与 `==`

> `==` 比较值是否相等，`is` 比较是否引用同一个对象。判断 None 使用 `is None`，普通字符串和数字比较使用 `==`。

### 类型注解

> 类型注解提升补全、静态检查、重构和接口表达，但普通 Python 不自动执行校验。FastAPI 和 Pydantic 会利用注解生成参数解析、验证和文档。

### dataclass 与 Pydantic

> dataclass 主要减少内部数据对象的样板代码；Pydantic 主要处理不可信外部数据的解析、校验、序列化和 Schema。两者可以在不同边界同时使用。

### Protocol

> Protocol 表达结构化类型，只要对象提供兼容方法就能满足契约，不必显式继承。它适合让 Service 依赖 Repository 行为，而不是具体数据库实现。

### 生成器

> 生成器通过 yield 惰性地产生值，适合逐步处理数据和降低一次性内存占用。但底层数据库查询是否流式还取决于驱动和 ORM 配置。

### 上下文管理器

> with 语句把资源获取和释放组成明确作用域，即使中途异常也能清理，常用于文件、事务、锁和网络客户端。

### async 与 await

> async def 定义协程函数，await 在等待 I/O 时把控制权交还事件循环。异步适合高并发 I/O，不会自动加速 CPU 密集计算，也不能把同步阻塞库自动变成异步。

### GIL

> 在传统 CPython 构建中，GIL 通常限制同一进程内多个线程同时执行 Python 字节码，但 I/O 等待时线程仍有价值。CPU 密集任务常考虑多进程或外部任务系统。Python 新版本还存在可选的自由线程构建，因此结论要结合解释器和部署方式。

### 异常边界

> 底层异常可以使用 raise from 保留原因并转换为稳定的领域异常，HTTP 边界再把领域异常映射为状态码。不要在业务层到处抛 HTTPException，也不要广泛捕获后静默忽略。

## 43. 本课技术地图

```text
Python Backend
├─ Runtime
│  ├─ Interpreter / Virtualenv
│  └─ Dependencies / pyproject
├─ Language
│  ├─ Types / Containers
│  ├─ Functions / Modules
│  ├─ Exceptions
│  └─ Classes / Dataclass / Enum
├─ Type System
│  ├─ Annotations
│  ├─ Union / Callable
│  └─ Protocol
├─ Execution
│  ├─ Iterator / Generator
│  ├─ Context Manager
│  └─ Async / Event Loop
└─ Engineering
   ├─ Project Structure
   ├─ Ruff / Type Checker
   ├─ pytest
   └─ Logging
```

## 44. 下一步

下一课学习 **FastAPI、Pydantic 与依赖注入**：

- FastAPI 应用、Router 和路径操作；
- Path、Query、Header、Cookie 与 Body；
- Pydantic 请求和响应 Schema；
- 校验错误、异常处理与统一错误结构；
- Depends 依赖注入和生命周期；
- 同步与异步端点的选择；
- OpenAPI、Swagger UI 和接口契约；
- FastAPI 项目分层与测试；
- 可直接交给 AI 的 FastAPI 开发指令；
- 面试中的 FastAPI 与 Pydantic 表达。
