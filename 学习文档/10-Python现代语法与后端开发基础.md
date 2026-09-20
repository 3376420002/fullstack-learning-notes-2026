# 10｜Python 现代语法与后端开发基础

> 目标：能读懂 FastAPI 项目的 Python 代码，让 AI 编写类型清楚、异常明确且异步边界正确的后端功能。

[← Axios](./09-Axios与前后端接口层.md) · [学习首页](../README.md) · [FastAPI →](./11-FastAPI、Pydantic与依赖注入.md)

## 1. 环境与虚拟环境

```powershell
python --version
python -c "import sys; print(sys.executable)"
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install fastapi
```

虚拟环境隔离项目依赖，不等于 Docker。通常不提交 `.venv/`，而是提交 `pyproject.toml` 和项目使用的锁文件。

项目可能使用 uv、Poetry、PDM 或 pip，不要混用多个依赖流程。

## 2. 变量与类型

```python
title: str = "学习 Python"
count: int = 3
completed: bool = False
score: float = 95.5
missing: None = None
```

Python 是动态类型语言，名称可以绑定不同类型对象；同时它不会任意混合不兼容类型：

```python
1 + "2"  # TypeError
```

类型注解主要服务编辑器、类型检查器和框架，普通 Python 默认不会自动执行注解校验。

字符串常用操作：

```python
raw_title = "  Learn FastAPI  "
title = raw_title.strip()
slug = title.lower().replace(" ", "-")
message = f"Todo: {title}"
parts = title.split(" ")
```

字符串不可变，方法会返回新字符串。处理文件和 HTTP 文本时明确使用 UTF-8；不要手工拼接 SQL、JSON 或 URL，分别使用参数化查询、`json` 模块和 URL 工具。

## 3. 容器

```python
titles: list[str] = ["Vue", "Python"]
point: tuple[int, int] = (10, 20)
todo_by_id: dict[int, str] = {1: "学习 SQL"}
roles: set[str] = {"user", "admin"}
```

```text
list   有序、可修改
tuple  有序、不可修改
dict   键值映射
set    唯一元素集合
```

赋值不会自动复制可变对象：

```python
first = [1, 2]
second = first
second.append(3)
assert first == [1, 2, 3]
```

`==` 比较值是否相等，`is` 比较是否为同一个对象：

```python
if todo is None:       # 单例 None 用 is
    ...

if todo.status == "completed":  # 普通值用 ==
    ...
```

## 4. 条件与循环

```python
if not title.strip():
    raise ValueError("title must not be empty")

for index, todo in enumerate(todos, start=1):
    print(index, todo.title)
```

```python
active_titles = [
    todo.title
    for todo in todos
    if not todo.completed
]
```

复杂逻辑使用普通循环，不要为了短而写难读的多层推导式。

## 5. 函数

```python
def normalize_title(title: str) -> str:
    normalized = title.strip()
    if not normalized:
        raise ValueError("title must not be empty")
    return normalized
```

关键字参数：

```python
def list_todos(*, page: int = 1, page_size: int = 20) -> list[Todo]:
    ...

list_todos(page=2, page_size=50)
```

参数较多时使用名称比依赖顺序更清楚。

可变数量参数：

```python
def log_event(event: str, *tags: str, **context: object) -> None:
    print(event, tags, context)

log_event("todo.created", "audit", todo_id=42, user_id=7)
```

`*args` 收集额外位置参数为 tuple，`**kwargs` 收集额外关键字参数为 dict。它们常见于框架和包装器，普通业务函数仍应优先使用明确参数。

## 6. 可变默认参数

错误：

```python
def add_tag(tag: str, tags: list[str] = []) -> list[str]:
    tags.append(tag)
    return tags
```

正确：

```python
def add_tag(tag: str, tags: list[str] | None = None) -> list[str]:
    result = [] if tags is None else list(tags)
    result.append(tag)
    return result
```

默认参数在函数定义时创建，多次调用会共享可变默认对象。

## 7. 模块与包

```text
app/
├─ __init__.py
├─ main.py
├─ services/
│  └─ todos.py
└─ schemas/
   └─ todo.py
```

```python
from app.services.todos import create_todo
```

不要通过随意修改 `sys.path` 解决导入问题。模块顶层主要放定义，不应在 import 时执行耗时任务或连接外部系统。

## 8. 异常

```python
class TodoNotFoundError(Exception):
    def __init__(self, todo_id: int) -> None:
        self.todo_id = todo_id
        super().__init__(f"Todo {todo_id} was not found")
```

```python
try:
    return repository.get(todo_id)
except DatabaseError as exc:
    raise TodoRepositoryError() from exc
```

`raise ... from exc` 保留底层原因。

不要使用：

```python
try:
    run()
except Exception:
    pass
```

它会吞掉业务故障和程序错误。

## 9. 类型注解

```python
def find_todo(todo_id: int) -> Todo | None:
    ...
```

```python
todo = find_todo(42)
if todo is None:
    raise TodoNotFoundError(42)

print(todo.title)
```

None 检查后，类型工具能把 todo 缩小为 Todo。

尽量避免无边界 Any、无理由 cast 和 `type: ignore`。

## 10. dataclass、Enum 与 Protocol

```python
from dataclasses import dataclass
from enum import StrEnum
from typing import Protocol


@dataclass(frozen=True, slots=True)
class Todo:
    id: int
    title: str
    completed: bool
```

```python
class TodoStatus(StrEnum):
    ACTIVE = "active"
    COMPLETED = "completed"
```

```python
class TodoRepository(Protocol):
    def get(self, todo_id: int) -> Todo:
        ...

    def save(self, todo: Todo) -> Todo:
        ...
```

```text
dataclass  内部数据对象，减少样板代码
Enum       有限稳定取值
Protocol   结构化行为契约，便于替换和测试
Pydantic   外部数据解析、校验和序列化
```

## 11. 类与组合

```python
class TodoService:
    def __init__(self, repository: TodoRepository) -> None:
        self._repository = repository

    def complete(self, todo_id: int) -> Todo:
        todo = self._repository.get(todo_id)
        return self._repository.mark_completed(todo.id)
```

类适合组合依赖和组织业务行为。简单无状态转换使用函数即可。

后端项目通常优先组合，而不是建立多层抽象继承树。

装饰器接收函数并返回包装后的函数，FastAPI 的 `@router.get(...)` 就是在注册路由：

```python
from functools import wraps

def audit(action: str):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            print("audit", action)
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

自定义装饰器要保留元数据，并正确处理同步/异步函数。业务项目不要用层层装饰器隐藏核心控制流。

## 12. 上下文管理器与生成器

```python
with path.open("r", encoding="utf-8") as file:
    content = file.read()
```

with 确保文件、事务、锁和客户端在异常时也能清理。

```python
from collections.abc import Iterator


def iter_active(todos: list[Todo]) -> Iterator[Todo]:
    for todo in todos:
        if not todo.completed:
            yield todo
```

生成器惰性产生值，但底层数据库是否真正流式仍取决于驱动和 ORM 配置。

## 13. async 与 await

```python
async def load_todo(todo_id: int) -> Todo:
    return await repository.get(todo_id)
```

```text
async def   定义协程函数
await       等待 I/O 并把控制权交还事件循环
event loop  调度其他可运行协程
```

异步适合网络和数据库 I/O，不会自动加速 CPU 计算。

错误：

```python
async def endpoint() -> None:
    time.sleep(5)  # 阻塞事件循环
```

底层库是同步的，外层改成 async 也不会自动变成非阻塞。

## 14. 日期、金额和环境变量

```python
from datetime import UTC, datetime
from decimal import Decimal

created_at = datetime.now(UTC)
price = Decimal("19.99")
```

```python
import os

database_url = os.environ["DATABASE_URL"]
```

时间点使用带时区 UTC，金额使用 Decimal。秘密来自服务端环境，不写进源码或日志。

常用标准库边界：

```python
from pathlib import Path
import json
import logging

logger = logging.getLogger(__name__)
config_path = Path("config") / "defaults.json"
payload = json.loads('{"title":"学习 Python"}')
logger.info("todo created", extra={"todo_id": 42})
```

`Path` 比手工拼路径更跨平台；`json.loads/dumps` 处理字符串，`json.load/dump` 处理文件。正式服务使用 logging，不用 print 记录业务日志，并避免写入密码和 Token。

## 15. 项目结构与工具

```text
backend/
├─ pyproject.toml
├─ src/app/
│  ├─ api/
│  ├─ services/
│  ├─ repositories/
│  ├─ schemas/
│  └─ models/
└─ tests/
```

```powershell
python -m ruff check .
python -m ruff format --check .
python -m mypy src
python -m pytest
```

具体工具以项目 pyproject 和 CI 为准。

## 16. pytest 示例

```python
def test_normalize_title_strips_whitespace() -> None:
    assert normalize_title("  Learn Python  ") == "Learn Python"


def test_normalize_title_rejects_blank_value() -> None:
    with pytest.raises(ValueError, match="must not be empty"):
        normalize_title("   ")
```

测试验证可观察行为、边界值和异常，不要只断言内部函数调用次数。

## 17. 并发、并行与 GIL

```text
asyncio       单线程中协调大量 I/O 等待
线程          适合同步 I/O；共享内存要注意竞争
多进程        适合 CPU 密集任务；进程间数据需通信
```

CPython 的 GIL 使同一进程中多个线程通常不能同时执行大量 Python CPU 字节码，但线程仍能在网络、文件等 I/O 等待期间发挥作用。Web 服务常用多个进程加异步 I/O；图片处理、模型计算等 CPU 任务放进进程池或任务队列。

## 18. 常见错误

- 提交 `.venv` 或本机秘密；
- 使用可变默认参数；
- 用 `is` 比较普通字符串或数字；
- 广泛捕获后静默吞错；
- 把类型注解当运行时校验；
- 在 async 函数中调用同步阻塞 I/O；
- 为简单函数建立复杂类层次；
- 日志包含密码、Token 和数据库连接信息。

## 19. 给 AI 的开发指令

```text
请在现有 Python 项目中实现 TodoService。
先读取 pyproject、现有类型、异常和 Repository 写法。
使用完整类型注解，不使用可变默认参数或无理由 Any。
Service 抛领域异常，不直接依赖 FastAPI HTTPException。
确认同步和异步调用链一致，不要只把 def 改成 async def。
完成后运行 Ruff、类型检查和 pytest。
```

## 20. 面试表达

> Python 是动态类型语言，类型注解主要用于静态检查和框架元数据，默认不自动执行运行时校验。

> 可变默认参数在函数定义时创建，会跨调用共享，通常使用 None 或 default_factory。

> dataclass 适合内部数据对象，Pydantic 适合外部数据校验，Protocol 用于表达结构化行为契约。

> async/await 适合 I/O 并发，不会自动加速 CPU 任务，也不能把同步库自动变成异步。

> with 管理资源生命周期，生成器通过 yield 惰性产生值。

[进入下一课：FastAPI、Pydantic 与依赖注入 →](./11-FastAPI、Pydantic与依赖注入.md)
