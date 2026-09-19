# 11｜FastAPI、Pydantic 与依赖注入

> 所属阶段：Python Web API 核心  
> 本课合并知识点：FastAPI 应用、APIRouter、请求参数、Pydantic v2、响应模型、异常处理、Depends、资源生命周期、异步端点、OpenAPI、测试  
> 前置知识：Python、HTTP、JSON、Axios  
> 学习目标：熟悉 FastAPI 的完整请求链，能够设计类型清晰的 API、指挥 AI 编写后端功能，并能在面试中说明框架原理与工程边界。

[← Python 后端基础](./10-Python现代语法与后端开发基础.md) · [学习首页](../README.md)

## 阅读方式

- 第 1～17 节掌握应用、路由、参数和 Pydantic Schema；
- 第 18～30 节理解响应、异常、依赖注入和资源生命周期；
- 第 31～39 节覆盖异步、配置、中间件、OpenAPI 和测试；
- 第 40～43 节用于 AI 编程、代码审核和面试复习；
- 不需要背诵装饰器参数，重点理解 HTTP 边界、数据校验和职责分层。

## 1. FastAPI 的定位

FastAPI 是基于 Python 类型注解构建 Web API 的框架。它整合了路由、参数解析、数据校验、依赖注入和 OpenAPI 文档。

```text
HTTP Request
    ↓
ASGI Server
    ↓
FastAPI Router
    ↓
参数提取 + Pydantic 校验 + Depends
    ↓
Endpoint
    ↓
Service / Repository
    ↓
Response Model 序列化与过滤
    ↓
HTTP Response
```

FastAPI 不负责替代：

- 业务层设计；
- 数据库事务策略；
- 身份和权限规则；
- 任务队列；
- 生产网关与 HTTPS；
- 自动解决所有异步性能问题。

## 2. 安装基础依赖

使用项目规定的依赖工具安装。venv + pip 示例：

```powershell
python -m pip install fastapi "uvicorn[standard]"
```

开发和测试常见依赖：

```powershell
python -m pip install pytest httpx
```

如果需要表单或文件上传，通常还需要：

```powershell
python -m pip install python-multipart
```

真实项目应把依赖写入 `pyproject.toml` 并使用既有锁定流程，不应只依赖本机临时安装结果。

## 3. 最小 FastAPI 应用

```python
# src/app/main.py
from fastapi import FastAPI

app = FastAPI(
    title="Todo API",
    version="1.0.0",
)


@app.get("/health")
async def health_check() -> dict[str, str]:
    return {"status": "ok"}
```

启动：

```powershell
uvicorn app.main:app --reload --app-dir src
```

导入字符串含义：

```text
app.main : Python 模块
app      : 模块中的 FastAPI 实例
```

`--reload` 只用于本地开发，生产环境不应直接照搬开发启动方式。

## 4. Path Operation

```python
@app.get("/todos")
async def list_todos() -> list[dict[str, object]]:
    return []


@app.post("/todos", status_code=201)
async def create_todo() -> dict[str, object]:
    return {"id": 1, "title": "Learn FastAPI"}
```

装饰器把三部分组合成路径操作：

```text
HTTP Method + Path + Endpoint Function
```

常见方法语义：

| 方法 | 常见用途 |
| --- | --- |
| GET | 查询资源，不应产生业务副作用 |
| POST | 创建资源或执行非幂等动作 |
| PUT | 整体替换资源表示 |
| PATCH | 部分更新 |
| DELETE | 删除资源 |

具体行为仍由 API 契约定义，不能只靠方法名称推断全部细节。

## 5. APIRouter

随着接口增多，应按业务域拆分 Router：

```python
# src/app/api/todos.py
from fastapi import APIRouter

router = APIRouter(prefix="/todos", tags=["todos"])


@router.get("")
async def list_todos() -> list[dict[str, object]]:
    return []
```

注册：

```python
# src/app/main.py
from fastapi import FastAPI

from app.api.todos import router as todo_router

app = FastAPI()
app.include_router(todo_router, prefix="/api/v1")
```

最终路径：

```text
/api/v1 + /todos = /api/v1/todos
```

`tags` 主要影响 OpenAPI 文档分组，不是权限或业务分层机制。

## 6. Path 参数

```python
from typing import Annotated

from fastapi import APIRouter, Path

TodoId = Annotated[int, Path(gt=0, description="Todo 主键")]


@router.get("/{todo_id}")
async def get_todo(todo_id: TodoId) -> dict[str, object]:
    return {"id": todo_id}
```

访问 `/todos/42` 时，FastAPI 会把字符串路径段转换为 int，并验证大于零。

转换失败通常进入请求验证错误流程，而不是调用 Endpoint 后再报错。

静态路径要注意声明顺序和歧义：

```python
@router.get("/me")
async def get_my_todos():
    ...


@router.get("/{todo_id}")
async def get_todo(todo_id: int):
    ...
```

应让 `/me` 被识别为静态路径，而不是尝试转换成 todo_id。

## 7. Query 参数

```python
from typing import Annotated, Literal

from fastapi import Query

Page = Annotated[int, Query(ge=1)]
PageSize = Annotated[int, Query(ge=1, le=100)]


@router.get("")
async def list_todos(
    page: Page = 1,
    page_size: PageSize = 20,
    status: Literal["active", "completed"] | None = None,
    q: Annotated[str | None, Query(max_length=100)] = None,
) -> dict[str, object]:
    return {
        "page": page,
        "page_size": page_size,
        "status": status,
        "q": q,
    }
```

对应：

```text
/todos?page=2&page_size=50&status=active&q=fastapi
```

有默认值的 Query 参数通常可选，没有默认值的参数通常必填。

分页上限应在后端强制执行，不能只依赖前端限制。

## 8. Header 与 Cookie

Header：

```python
from typing import Annotated

from fastapi import Header


@router.get("")
async def list_todos(
    request_id: Annotated[
        str | None,
        Header(alias="X-Request-ID"),
    ] = None,
) -> dict[str, str | None]:
    return {"request_id": request_id}
```

Cookie：

```python
from fastapi import Cookie


@router.get("/session")
async def read_session(
    session_id: Annotated[str | None, Cookie()] = None,
) -> dict[str, bool]:
    return {"authenticated": session_id is not None}
```

读取 Cookie 不等于验证会话。真实认证依赖必须校验签名、过期、撤销和用户状态。

## 9. 请求 Body 与 Pydantic

```python
from pydantic import BaseModel, Field


class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)


@router.post("", status_code=201)
async def create_todo(payload: TodoCreate) -> dict[str, object]:
    return {
        "id": 1,
        "title": payload.title,
        "completed": False,
    }
```

FastAPI 会根据类型注解：

```text
读取 JSON Body
    ↓
调用 Pydantic 解析与校验
    ↓
成功后把 TodoCreate 对象传给 Endpoint
```

请求体不是合法 JSON、字段缺失或不符合约束时，Endpoint 通常不会执行。

## 10. Schema 的职责

Pydantic Schema 用于 API 边界：

- 解析输入；
- 校验字段；
- 类型转换；
- 序列化输出；
- 生成 JSON Schema；
- 参与 OpenAPI 文档；
- 过滤响应字段。

```text
Pydantic Schema    API 请求与响应契约
SQLAlchemy Model   Python 对象与数据库表的映射
Domain Object      可选的内部业务对象
```

三者可能有相似字段，但生命周期、依赖方向和安全边界不同。

## 11. 创建、更新与响应 Schema

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class TodoCreate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    title: str = Field(min_length=1, max_length=100)


class TodoUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    title: str | None = Field(default=None, min_length=1, max_length=100)
    completed: bool | None = None


class TodoOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    completed: bool
    created_at: datetime
```

`extra="forbid"` 会拒绝契约之外的输入字段，适合需要尽早发现前后端拼写错误的请求模型。

`from_attributes=True` 允许响应模型从对象属性读取数据，常用于 ORM 对象到响应 Schema 的转换。

## 12. Field 约束

```python
from decimal import Decimal

from pydantic import BaseModel, Field


class ProductCreate(BaseModel):
    name: str = Field(min_length=1, max_length=120)
    price: Decimal = Field(gt=0, decimal_places=2)
    stock: int = Field(ge=0, le=1_000_000)
    description: str | None = Field(default=None, max_length=2_000)
```

常见约束：

```text
gt / ge        大于 / 大于等于
lt / le        小于 / 小于等于
min_length     最小长度
max_length     最大长度
pattern        正则模式
multiple_of    倍数
```

接口校验不能替代数据库约束。唯一性、外键和并发一致性仍应由数据库保障。

## 13. 字段验证器

Pydantic v2：

```python
from pydantic import BaseModel, Field, field_validator


class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)

    @field_validator("title")
    @classmethod
    def normalize_title(cls, value: str) -> str:
        normalized = value.strip()
        if not normalized:
            raise ValueError("title must not be blank")
        return normalized
```

验证器适合字段格式和输入规范化。

不要在字段验证器中查询数据库、发送网络请求或执行复杂业务流程。唯一性、当前用户权限和资源状态属于 Service 或专门依赖的职责。

## 14. 模型验证器

```python
from datetime import date
from typing import Self

from pydantic import BaseModel, model_validator


class DateRange(BaseModel):
    start_date: date
    end_date: date

    @model_validator(mode="after")
    def validate_order(self) -> Self:
        if self.end_date < self.start_date:
            raise ValueError("end_date must not be before start_date")
        return self
```

模型验证器适合多个字段之间的结构关系。

跨模型、跨用户或依赖数据库状态的规则仍应放在业务层，而不是让 Schema 承担全部业务逻辑。

## 15. 严格转换与宽松解析

Pydantic 默认会进行部分合理转换，例如 JSON 字符串形式的数字可能被解析为 int，具体行为取决于类型和配置。

需要严格值时可以使用严格类型或配置：

```python
from pydantic import BaseModel, ConfigDict


class StrictPayload(BaseModel):
    model_config = ConfigDict(strict=True)

    count: int
```

严格模式不一定适合所有 HTTP 输入。Query 和表单值天然来自字符串，设计时要明确希望允许的转换，而不是无条件开启或关闭。

## 16. 部分更新与 `exclude_unset`

PATCH 请求需要区分：

```text
字段没有发送
字段明确发送为 null
字段发送了新值
```

```python
@router.patch("/{todo_id}", response_model=TodoOut)
async def update_todo(
    todo_id: TodoId,
    payload: TodoUpdate,
) -> TodoOut:
    changes = payload.model_dump(exclude_unset=True)
    return service.update(todo_id, changes)
```

`exclude_unset=True` 只保留客户端实际提供的字段。

如果字段类型允许 None，那么明确发送 `{"title": null}` 会与完全不发送 title 区分。业务层仍要决定是否允许清空字段。

## 17. Alias 与命名风格

前端可能使用 camelCase，Python 内部使用 snake_case：

```python
from pydantic import BaseModel, ConfigDict


def to_camel(value: str) -> str:
    first, *rest = value.split("_")
    return first + "".join(word.capitalize() for word in rest)


class ApiModel(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
    )
```

团队也可以选择 API 全部使用 snake_case。重点是契约一致，并避免每个组件零散转换。

Alias 策略需要同时验证输入、输出和 OpenAPI 文档表现，不应只看 Python 属性访问是否方便。

## 18. 响应模型

```python
@router.get("/{todo_id}", response_model=TodoOut)
async def get_todo(todo_id: TodoId) -> object:
    return service.get(todo_id)
```

响应模型会：

- 验证和序列化返回数据；
- 生成 OpenAPI 响应 Schema；
- 过滤未声明字段；
- 暴露返回契约。

例如数据库 User 含有 `password_hash`，公开 UserOut 不声明该字段，就能降低误返回风险。

不能只依赖响应模型修复设计错误。敏感数据仍不应被随意传到表现层或记录到日志。

## 19. 返回类型与 `response_model`

两种常见写法：

```python
@router.get("/{todo_id}")
async def get_todo(todo_id: TodoId) -> TodoOut:
    ...
```

```python
@router.get("/{todo_id}", response_model=TodoOut)
async def get_todo(todo_id: TodoId) -> Todo:
    ...
```

返回类型适合函数真实返回值与 API Schema 一致的情况。`response_model` 适合内部返回 ORM 或领域对象，但对外使用另一响应契约。

项目应采用一致风格，保证类型检查器和 FastAPI 都能准确理解。

## 20. 状态码与 Response

创建：

```python
from fastapi import status


@router.post("", response_model=TodoOut, status_code=status.HTTP_201_CREATED)
async def create_todo(payload: TodoCreate) -> TodoOut:
    ...
```

删除后无响应体：

```python
from fastapi import Response, status


@router.delete("/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_todo(todo_id: TodoId) -> Response:
    service.delete(todo_id)
    return Response(status_code=status.HTTP_204_NO_CONTENT)
```

204 不应返回 JSON Body。创建资源时还可以通过 `Location` Header 指向新资源地址。

## 21. 请求验证错误与 422

以下情况通常返回 422：

- Path 不能转换为声明类型；
- Query 超出范围；
- Body 缺少必填字段；
- 字段类型或约束失败；
- Pydantic 验证器拒绝输入。

默认响应包含 `detail` 问题列表。

400 可用于其他通用错误请求，但不能简单说 FastAPI 的 Pydantic 验证默认返回 400。

团队可以自定义验证错误结构，但前端、OpenAPI、日志和测试必须同步契约。

## 22. HTTPException

```python
from fastapi import HTTPException, status


@router.get("/{todo_id}", response_model=TodoOut)
async def get_todo(todo_id: TodoId) -> TodoOut:
    todo = repository.find(todo_id)
    if todo is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Todo not found",
        )
    return todo
```

在简单 Endpoint 中使用 HTTPException 很直接。

复杂项目中更推荐：

```text
Service 抛领域异常
    ↓
FastAPI 异常处理器映射 HTTP
```

这样业务逻辑不会长期绑定 FastAPI。

## 23. 自定义异常处理器

领域异常：

```python
class TodoNotFoundError(Exception):
    def __init__(self, todo_id: int) -> None:
        self.todo_id = todo_id
        super().__init__(f"Todo {todo_id} was not found")
```

统一错误 Schema：

```python
from pydantic import BaseModel


class ErrorResponse(BaseModel):
    code: str
    message: str
    request_id: str | None = None
```

异常处理器：

```python
from fastapi import Request
from fastapi.responses import JSONResponse


@app.exception_handler(TodoNotFoundError)
async def handle_todo_not_found(
    request: Request,
    exc: TodoNotFoundError,
) -> JSONResponse:
    return JSONResponse(
        status_code=404,
        content={
            "code": "TODO_NOT_FOUND",
            "message": "Todo does not exist",
            "request_id": getattr(request.state, "request_id", None),
        },
    )
```

对外错误不要包含堆栈、SQL、密钥或内部路径。完整异常应写入安全日志和监控。

## 24. Depends 依赖注入

```python
from typing import Annotated

from fastapi import Depends


def get_current_user() -> CurrentUser:
    ...


CurrentUserDep = Annotated[CurrentUser, Depends(get_current_user)]


@router.get("/me", response_model=UserOut)
async def get_me(current_user: CurrentUserDep) -> CurrentUser:
    return current_user
```

FastAPI 根据函数签名解析依赖图：

```text
Endpoint
└─ get_current_user
   └─ get_token
      └─ Authorization Header
```

依赖的返回值会传给参数。依赖也可以继续声明其他依赖。

## 25. Service 依赖

```python
def get_todo_repository() -> TodoRepository:
    return SqlAlchemyTodoRepository()


TodoRepositoryDep = Annotated[
    TodoRepository,
    Depends(get_todo_repository),
]


def get_todo_service(
    repository: TodoRepositoryDep,
) -> TodoService:
    return TodoService(repository)


TodoServiceDep = Annotated[
    TodoService,
    Depends(get_todo_service),
]


@router.get("/{todo_id}", response_model=TodoOut)
async def get_todo(
    todo_id: TodoId,
    service: TodoServiceDep,
) -> Todo:
    return service.get(todo_id)
```

Endpoint 依赖抽象的业务入口，不直接创建数据库连接或堆叠查询。

## 26. 依赖缓存

同一次请求中，相同依赖默认会复用结果：

```python
Depends(get_current_user)
```

这意味着多个下游依赖共同需要当前用户时，认证解析通常只执行一次。

需要每次重新执行可以使用：

```python
Depends(get_value, use_cache=False)
```

不要随意关闭缓存。数据库会话、当前用户和请求上下文通常希望在单个请求内保持一致。

这个缓存只存在于当前请求依赖解析过程，不是跨请求应用缓存。

## 27. `yield` 依赖与资源清理

同步示例：

```python
from collections.abc import Iterator


def get_session() -> Iterator[Session]:
    with SessionFactory() as session:
        yield session
```

异步示例：

```python
from collections.abc import AsyncIterator


async def get_session() -> AsyncIterator[AsyncSession]:
    async with AsyncSessionFactory() as session:
        yield session
```

概念流程：

```text
yield 之前   获取资源
yield 值     提供给 Endpoint 和下游依赖
yield 之后   清理资源
```

即使 Endpoint 失败，清理部分仍会执行。事务提交和回滚策略应明确设计，不能只假设关闭 Session 就等于正确处理事务。

## 28. 认证依赖

```python
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> CurrentUser:
    payload = verify_access_token(token)
    user = await user_repository.find(payload.subject)

    if user is None or not user.is_active:
        raise InvalidCredentialsError()

    return user
```

`OAuth2PasswordBearer` 主要负责从 Authorization Header 提取 Bearer Token，并为 OpenAPI 描述安全方案。它不会自动验证签名、过期、撤销或用户权限。

授权依赖可以继续检查角色或资源归属，后端必须对每个受保护接口独立执行权限判断。

## 29. Form 与文件上传

```python
from typing import Annotated

from fastapi import File, Form, UploadFile


@router.post("/avatar")
async def upload_avatar(
    display_name: Annotated[str, Form(min_length=1, max_length=100)],
    file: Annotated[UploadFile, File()],
) -> dict[str, str | None]:
    return {
        "display_name": display_name,
        "filename": file.filename,
        "content_type": file.content_type,
    }
```

`UploadFile` 通常比一次性读成 bytes 更适合较大文件，因为它提供文件式接口并可能使用临时存储。

后端必须验证：

- 实际大小；
- 允许类型；
- 文件内容；
- 文件名安全；
- 存储权限；
- 恶意文件风险。

客户端提供的 `content_type` 和文件扩展名都不应被完全信任。

## 30. Body、Form 和 File 的边界

普通 JSON 接口：

```http
Content-Type: application/json
```

文件和表单字段：

```http
Content-Type: multipart/form-data; boundary=...
```

同一个请求不能同时把整个请求体既当成普通 JSON Body，又当成 multipart 表单解析。

需要上传文件并携带结构化元数据时，可以：

- 把简单字段作为 Form；
- 把 JSON 字符串放入某个 Form 字段后明确解析；
- 先创建资源，再单独上传文件；
- 使用对象存储预签名上传。

选择应写入 API 契约。

## 31. 同步与异步 Endpoint

异步 I/O 链路：

```python
@router.get("/{todo_id}")
async def get_todo(todo_id: TodoId) -> TodoOut:
    return await async_service.get(todo_id)
```

同步阻塞链路：

```python
@router.get("/{todo_id}")
def get_todo(todo_id: TodoId) -> TodoOut:
    return sync_service.get(todo_id)
```

选择原则：

```text
异步数据库/HTTP 客户端 → async def + await
同步阻塞库             → 普通 def，交给 FastAPI 线程池处理
CPU 密集任务           → 任务队列、进程或独立服务
```

不要只把 Endpoint 改为 `async def`，却继续在里面执行同步数据库查询、`requests.get` 或 `time.sleep`。

## 32. Lifespan

应用级资源使用 lifespan 管理：

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    app.state.http_client = await create_http_client()
    try:
        yield
    finally:
        await app.state.http_client.aclose()


app = FastAPI(lifespan=lifespan)
```

适合应用启动和关闭范围的资源：

- HTTP 连接池；
- 模型或配置缓存；
- 消息系统客户端；
- 监控和追踪资源。

数据库表迁移通常应由部署流程显式执行，而不是每个 Web 进程启动时自动并发运行。

## 33. BackgroundTasks

```python
from fastapi import BackgroundTasks


@router.post("/{todo_id}/notify", status_code=202)
async def notify_owner(
    todo_id: TodoId,
    background_tasks: BackgroundTasks,
) -> dict[str, str]:
    background_tasks.add_task(send_notification, todo_id)
    return {"status": "accepted"}
```

BackgroundTasks 适合响应返回后执行的小型、短时、进程内任务。

它不是可靠任务队列：

- 进程崩溃会丢任务；
- 没有天然持久化和分布式重试；
- 长任务会占用 Web 进程资源；
- 多副本协调能力有限。

邮件批量发送、视频处理、数据导入等重要任务通常使用 Celery、Dramatiq、RQ 或云任务系统。

## 34. 配置与 Pydantic Settings

```powershell
python -m pip install pydantic-settings
```

```python
from functools import lru_cache

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    app_name: str = "Todo API"
    database_url: str
    allowed_origins: list[str] = Field(default_factory=list)


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

配置可以作为依赖：

```python
SettingsDep = Annotated[Settings, Depends(get_settings)]
```

不要提交真实 `.env`。提交 `.env.example` 说明变量名称。测试修改环境变量时要清理 `lru_cache`，避免旧配置污染测试。

## 35. CORS 中间件

开发环境中，Vue 和 FastAPI 可能不同源：

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
)
```

注意：

- Origin 包含协议、主机和端口；
- 带凭据时应配置明确可信 Origin；
- CORS 只影响浏览器跨源访问；
- curl 和服务端请求不受浏览器 CORS 机制约束；
- CORS 不能代替认证、授权或 CSRF 防护。

如果使用 Vite 开发代理，浏览器看到的可能是同源请求，但生产部署仍需明确网关和跨域结构。

## 36. 自定义 Middleware

请求 ID 示例：

```python
from time import perf_counter
from uuid import uuid4

from fastapi import Request


@app.middleware("http")
async def request_context(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID") or str(uuid4())
    request.state.request_id = request_id
    started_at = perf_counter()

    response = await call_next(request)

    response.headers["X-Request-ID"] = request_id
    response.headers["Server-Timing"] = (
        f"app;dur={(perf_counter() - started_at) * 1000:.2f}"
    )
    return response
```

Middleware 适合请求级横切能力：

- 请求 ID；
- 访问日志；
- 性能计时；
- 安全响应头；
- 某些统一上下文。

具体业务校验和资源权限更适合依赖或 Service，不应全部塞进 Middleware。

## 37. OpenAPI 与接口文档

默认地址通常是：

```text
/docs           Swagger UI
/redoc          ReDoc
/openapi.json   OpenAPI 文档
```

FastAPI 根据路由、类型和 Pydantic Schema 生成：

- 路径与方法；
- 参数位置和约束；
- 请求体 Schema；
- 响应 Schema；
- 状态码；
- 安全方案；
- 标签和说明。

OpenAPI 可以用于：

- 前端生成 TypeScript 类型或客户端；
- 自动化契约测试；
- API 网关配置；
- 文档和调试。

自动文档不能替代业务语义说明。分页、排序、错误码、幂等和权限仍应明确记录。

## 38. 分页响应

```python
from typing import Generic, TypeVar

from pydantic import BaseModel, Field

T = TypeVar("T")


class Page(BaseModel, Generic[T]):
    items: list[T]
    page: int = Field(ge=1)
    page_size: int = Field(ge=1)
    total: int = Field(ge=0)
```

使用：

```python
@router.get("", response_model=Page[TodoOut])
async def list_todos(...) -> Page[TodoOut]:
    ...
```

页码分页容易理解，但深页性能可能下降；游标分页适合大数据量和持续变化列表。选择取决于查询模式和产品体验。

不要先加载全部数据再在 Python 中切片，数据库查询应执行 limit、offset 或游标条件。

## 39. 完整 Todo 请求链

Schema：

```python
class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)


class TodoOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    completed: bool
```

Service：

```python
class TodoService:
    def __init__(self, repository: TodoRepository) -> None:
        self._repository = repository

    async def create(self, command: CreateTodoCommand) -> Todo:
        if await self._repository.exists_with_title(command.title):
            raise DuplicateTodoError(command.title)
        return await self._repository.create(command)
```

Router：

```python
@router.post("", response_model=TodoOut, status_code=201)
async def create_todo(
    payload: TodoCreate,
    service: TodoServiceDep,
) -> Todo:
    command = CreateTodoCommand(title=payload.title)
    return await service.create(command)
```

完整链路：

```text
Axios JSON
   ↓
TodoCreate 校验
   ↓
Router 转换为 Command
   ↓
Service 执行业务规则
   ↓
Repository 持久化
   ↓
Todo 领域/ORM 对象
   ↓
TodoOut 过滤并序列化
   ↓
HTTP 201 JSON
```

小项目可以让 Router 直接把 Pydantic Schema 传给 Service，但应理解这是简化选择，不是唯一架构。

## 40. FastAPI 项目结构

```text
src/app/
├─ main.py
├─ api/
│  ├─ dependencies.py
│  └─ v1/
│     ├─ router.py
│     ├─ auth.py
│     └─ todos.py
├─ core/
│  ├─ config.py
│  ├─ errors.py
│  ├─ logging.py
│  └─ security.py
├─ schemas/
│  ├─ auth.py
│  └─ todo.py
├─ services/
│  └─ todos.py
├─ repositories/
│  └─ todos.py
├─ models/
│  └─ todo.py
└─ db/
   ├─ session.py
   └─ migrations/
```

依赖方向：

```text
API → Service → Repository → Database
 ↓       ↓
Schema  Domain Types
```

不要让 Repository 依赖 FastAPI Request，也不要让 SQLAlchemy Session 散落在每个 Endpoint。

## 41. 测试 FastAPI

同步测试客户端：

```python
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_health_check() -> None:
    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

异步测试常使用 HTTPX ASGITransport：

```python
import pytest
from httpx import ASGITransport, AsyncClient

from app.main import app


@pytest.mark.anyio
async def test_health_check_async() -> None:
    transport = ASGITransport(app=app)
    async with AsyncClient(
        transport=transport,
        base_url="http://test",
    ) as client:
        response = await client.get("/health")

    assert response.status_code == 200
```

测试工具的 lifespan 行为随组合方式和版本配置而不同。依赖应用启动资源的测试，应使用项目明确的 lifespan 测试方案。

## 42. Dependency Override

```python
def override_todo_service() -> FakeTodoService:
    return FakeTodoService(
        todos=[Todo(id=1, title="Test", completed=False)]
    )


def test_get_todo() -> None:
    app.dependency_overrides[get_todo_service] = override_todo_service

    try:
        with TestClient(app) as client:
            response = client.get("/api/v1/todos/1")
    finally:
        app.dependency_overrides.clear()

    assert response.status_code == 200
    assert response.json()["title"] == "Test"
```

依赖覆盖能替换数据库、认证或外部服务，使 API 测试快速且可控。

集成测试仍需要真实测试数据库或接近生产的基础设施，不能全部用 Fake 代替。

## 43. 常见错误与准确理解

### Schema 就是 ORM Model

错误。Schema 负责 API 数据契约，ORM Model 负责数据库映射。

### Pydantic 能验证所有业务规则

错误。字段和结构校验适合 Pydantic；权限、唯一性和资源状态属于 Service 与数据库。

### FastAPI Endpoint 全部写 async 更快

错误。同步阻塞调用放进 async Endpoint 会阻塞事件循环。

### Depends 是全局单例容器

错误。它是基于函数签名的请求依赖系统，默认缓存主要发生在单次请求依赖图中。

### 前端已校验，所以后端可以省略校验

错误。HTTP 输入不可信，客户端可以绕过或伪造。

### response_model 只用于生成文档

错误。它还参与响应验证、序列化和字段过滤。

### CORS 是权限系统

错误。CORS 是浏览器跨源读取策略，不能阻止 curl 或其他服务端客户端访问接口。

### BackgroundTasks 是可靠消息队列

错误。它是当前 Web 进程内的轻量后台执行机制。

### 204 可以返回成功 JSON

不应如此。204 表示没有响应体，应返回空响应。

### 应用启动时自动创建和迁移所有表

生产环境风险较高。数据库迁移应通过明确、可审计的部署步骤执行。

## 44. 指挥 AI 编写 FastAPI

### 创建 CRUD 接口指令

```text
请在现有 FastAPI + Pydantic v2 项目中实现 Todo CRUD。
先读取项目目录、现有 Router、Schema、Service、异常和依赖注入写法。
分别定义 TodoCreate、TodoUpdate、TodoOut，禁止把 ORM Model 直接当请求 Schema。
POST 返回 201，DELETE 返回空的 204，PATCH 使用 model_dump(exclude_unset=True)。
Router 只处理 HTTP 边界并调用 Service，不把业务规则和数据库查询堆进 Endpoint。
补充 200、201、204、404、409、422 场景测试，并运行 Ruff、类型检查和 pytest。
```

### 创建依赖指令

```text
请为当前 FastAPI 项目实现数据库 Session、当前用户和 TodoService 依赖。
使用 Annotated 与 Depends，按项目同步或异步数据库栈选择正确实现。
Session 使用 yield 依赖确保关闭，明确事务提交、回滚和异常边界。
认证依赖必须验证 Token 和用户状态，不能只提取 Authorization Header。
不要创建隐藏的全局可变 Session，不要在每个 Endpoint 重复构造依赖链。
为依赖缓存和测试 override 补充说明与测试。
```

### 统一错误处理指令

```text
请为现有 FastAPI 项目建立统一错误响应。
保留 FastAPI 默认 422 语义或明确记录自定义变化。
把领域异常映射为稳定 HTTP 状态码和业务 code，响应包含安全 message 与 request_id。
日志保留完整异常链，但响应不得泄露堆栈、SQL、文件路径、Token 或数据库信息。
前端已依赖的错误结构不能被无说明破坏，补充 OpenAPI responses 和测试。
```

### 异步审查指令

```text
请审查这条 FastAPI 请求链的同步和异步边界。
检查 Endpoint、Depends、Service、数据库驱动、HTTP 客户端和文件操作。
不要只把 def 改为 async def；确认底层调用是否真正非阻塞。
指出事件循环阻塞、连接池压力、缺失超时、取消处理和不必要并发。
只做与根因有关的修改，并使用并发测试或可观察指标验证。
```

### OpenAPI 契约指令

```text
请检查 FastAPI 生成的 OpenAPI 与前端 Axios 类型是否一致。
核对 Path、Query、Body、状态码、响应模型、分页、错误结构和安全方案。
不要用 TypeScript 类型断言掩盖契约差异；优先修复服务端 Schema 或统一生成流程。
保持现有 API 兼容，破坏性变化必须明确列出迁移方案。
```

### 排错指令

```text
请根据请求、响应、FastAPI 日志和 traceback 定位接口问题。
依次检查路由匹配、HTTP 方法、Content-Type、Path/Query/Body 来源、Pydantic 422、Depends、业务异常和数据库异常。
区分 404 路由不存在与资源不存在，区分 422 输入错误与 500 程序错误。
先提供可验证证据，再做最小修复；不要关闭校验、CORS 或异常日志掩盖问题。
```

## 45. 审核 AI 生成 FastAPI 代码的清单

```text
[ ] Router prefix 和最终 API 路径正确
[ ] HTTP 方法、状态码和响应体语义一致
[ ] Path、Query、Header、Cookie、Body、Form 来源明确
[ ] 请求、更新和响应 Schema 已分离
[ ] 使用 Pydantic v2 API，没有混用旧版写法
[ ] PATCH 能区分未发送字段与显式 null
[ ] 响应模型不会暴露内部或敏感字段
[ ] 领域规则没有全部塞进 Pydantic 验证器
[ ] Service 没有无必要地依赖 HTTPException
[ ] Depends 生命周期、缓存和测试替换方式清楚
[ ] 数据库 Session 能关闭，事务能提交或回滚
[ ] async Endpoint 中没有同步阻塞 I/O
[ ] CORS 使用可信 Origin，未被当成权限机制
[ ] BackgroundTasks 没有承担必须可靠执行的长任务
[ ] 错误响应不泄露堆栈、SQL、密钥和内部路径
[ ] OpenAPI、前端契约和自动化测试保持一致
[ ] Ruff、类型检查和 pytest 按项目配置通过
```

## 46. 面试表达速记

### FastAPI 的核心机制

> FastAPI 读取 Python 函数签名和类型注解，完成路由匹配、参数提取、Pydantic 校验、依赖解析、响应序列化和 OpenAPI 生成。类型注解是框架契约的一部分，而不仅是编辑器提示。

### Pydantic Schema 与 ORM Model

> Pydantic Schema 描述 API 输入输出并执行解析、校验和序列化；ORM Model 映射数据库表和关系。数据库字段不应自动等于公开 API 字段，尤其要避免暴露密码哈希等内部数据。

### 422

> FastAPI 的请求参数或 Pydantic Body 校验失败时默认常返回 422，通常 Endpoint 尚未执行。400 可以表示其他错误请求，但不能把框架默认验证错误统一说成 400。

### response_model

> response_model 不只生成文档，还会验证、序列化和过滤返回数据。它允许内部返回 ORM 或领域对象，对外保持稳定且安全的响应契约。

### Depends

> Depends 根据函数签名构建依赖图，可以组合认证、数据库 Session、配置和 Service。同一请求中的相同依赖默认可复用结果，yield 依赖还能表达资源获取与清理。

### 同步与异步 Endpoint

> 异步驱动和异步 HTTP 客户端适合 async def；同步阻塞调用适合普通 def 交给线程池。只改变 Endpoint 关键字而不改变底层 I/O，不会获得异步收益，反而可能阻塞事件循环。

### 依赖注入与分层

> Router 负责 HTTP 边界，Service 负责业务规则，Repository 负责数据访问。FastAPI Depends 负责组装它们，使 Endpoint 不需要直接创建 Session 或具体基础设施对象。

### Lifespan 与 yield 依赖

> Lifespan 管理应用启动到关闭范围的共享资源，yield 依赖管理单次请求范围的资源，例如数据库 Session。二者生命周期不同。

### BackgroundTasks

> BackgroundTasks 适合响应返回后执行短小的进程内工作，不保证持久化和分布式重试。重要长任务应使用真正的任务队列。

### OpenAPI

> FastAPI 根据路由和 Schema 自动生成 OpenAPI，可用于文档、客户端生成和契约测试。但业务错误码、分页语义、幂等和权限仍需显式设计。

## 47. 本课技术地图

```text
FastAPI Application
├─ ASGI / Lifespan
├─ Router
│  ├─ Path / Query
│  ├─ Header / Cookie
│  ├─ Body / Form / File
│  └─ Status / Response
├─ Pydantic v2
│  ├─ Request Schema
│  ├─ Update Schema
│  ├─ Response Schema
│  ├─ Field / Model Validator
│  └─ Serialization / OpenAPI
├─ Depends
│  ├─ Settings
│  ├─ Current User
│  ├─ Session
│  └─ Service
├─ Error Boundary
│  ├─ 422 Validation
│  ├─ HTTPException
│  └─ Domain Exception Handler
└─ Engineering
   ├─ Middleware / CORS
   ├─ Background Tasks
   ├─ Dependency Override
   └─ API Tests
```

## 48. 下一步

下一课学习 **SQL 与关系型数据库基础**：

- 表、行、列、主键与外键；
- 数据类型、NULL 与约束；
- SELECT、INSERT、UPDATE、DELETE；
- WHERE、ORDER BY、GROUP BY 与聚合；
- INNER JOIN、LEFT JOIN 与多表关系；
- 索引、执行计划与查询性能；
- 事务、隔离级别与并发一致性；
- 数据建模、范式与 Todo 数据库设计；
- 可直接交给 AI 的 SQL 开发指令；
- 面试中的 SQL 核心表达。
