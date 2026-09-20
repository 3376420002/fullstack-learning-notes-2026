# 11｜FastAPI、Pydantic 与依赖注入

> 目标：能实现清晰的 API 路由、Schema、依赖和测试，让 AI 不把业务、HTTP 与数据库混在一起。

[← Python](./10-Python现代语法与后端开发基础.md) · [学习首页](../README.md) · [SQL →](./12-SQL与关系型数据库基础.md)

## 1. 请求链

```text
HTTP Request
  ↓
FastAPI Router
  ↓ 参数解析、Pydantic 校验、Depends
Endpoint
  ↓
Service → Repository → Database
  ↓
Response Model 序列化和过滤
```

## 2. 最小应用与 Router

```python
from fastapi import FastAPI

from app.api.todos import router as todo_router

app = FastAPI(title="Todo API")
app.include_router(todo_router, prefix="/api/v1")
```

```python
router = APIRouter(prefix="/todos", tags=["todos"])


@router.get("/health")
async def health_check() -> dict[str, str]:
    return {"status": "ok"}
```

启动：

```powershell
uvicorn app.main:app --reload --app-dir src
```

`--reload` 只用于开发。

## 3. Path、Query、Header

```python
TodoId = Annotated[int, Path(gt=0)]


@router.get("/{todo_id}")
async def get_todo(
    todo_id: TodoId,
    page: Annotated[int, Query(ge=1)] = 1,
    request_id: Annotated[
        str | None,
        Header(alias="X-Request-ID"),
    ] = None,
) -> dict[str, object]:
    return {"id": todo_id, "page": page, "request_id": request_id}
```

FastAPI 根据函数签名判断参数来源、转换类型并执行约束。

请求数据来源可明确标注：

```python
@router.post("/import")
async def import_todos(
    metadata: Annotated[str, Form()],
    file: Annotated[UploadFile, File()],
) -> dict[str, str]:
    return {"filename": file.filename or ""}
```

```text
Path     URL 路径中的资源标识
Query    筛选、分页、排序
Header   认证、追踪、内容协商
Body     通常是 JSON 对象
Form     表单文本字段
File     multipart 文件
```

JSON Body 与 multipart Form/File 的编码不同，同一个请求不能同时把整体当作普通 JSON Body 和文件上传；复杂 metadata 可作为 JSON 字符串字段后再解析。

## 4. Pydantic Schema

```python
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

```text
Create   客户端创建时允许提交什么
Update   客户端允许修改什么
Out      接口允许返回什么
ORM      数据库怎样保存
```

Schema 与 ORM Model 不是同一种模型。

## 5. 验证器边界

```python
@field_validator("title")
@classmethod
def normalize_title(cls, value: str) -> str:
    normalized = value.strip()
    if not normalized:
        raise ValueError("title must not be blank")
    return normalized
```

验证器适合格式和字段关系，不适合数据库唯一性、当前用户权限和复杂业务流程。

`Field` 描述单字段约束，`field_validator` 处理一个或多个字段的规范化，`model_validator` 处理跨字段关系：

```python
class DateRange(BaseModel):
    start_at: datetime
    end_at: datetime

    @model_validator(mode="after")
    def validate_range(self) -> "DateRange":
        if self.end_at <= self.start_at:
            raise ValueError("end_at must be after start_at")
        return self
```

验证器应该确定、快速且无外部副作用。需要访问数据库的规则放到 Service，并在数据库中保留最终约束。

## 6. CRUD 路由

```python
@router.post("", response_model=TodoOut, status_code=201)
async def create_todo(
    payload: TodoCreate,
    service: TodoServiceDep,
) -> Todo:
    return await service.create(payload)


@router.patch("/{todo_id}", response_model=TodoOut)
async def update_todo(
    todo_id: TodoId,
    payload: TodoUpdate,
    service: TodoServiceDep,
) -> Todo:
    changes = payload.model_dump(exclude_unset=True)
    return await service.update(todo_id, changes)


@router.delete("/{todo_id}", status_code=204)
async def delete_todo(todo_id: TodoId, service: TodoServiceDep) -> Response:
    await service.delete(todo_id)
    return Response(status_code=204)
```

PATCH 使用 `exclude_unset=True` 区分未发送字段和显式 null。204 不返回 JSON Body。

## 7. response_model

响应模型会：

- 生成 OpenAPI Schema；
- 验证和序列化返回值；
- 过滤未声明字段；
- 保护 API 契约。

ORM User 有 password_hash，不代表 API 可以返回它。

## 8. 422 与异常

Path、Query 或 Body 校验失败时，FastAPI 默认通常返回 422，Endpoint 不会继续执行。

简单路由可以抛 HTTPException；分层项目更适合 Service 抛领域异常，由 FastAPI 处理器映射：

```text
TodoNotFoundError   → 404
DuplicateTodoError  → 409
PermissionError     → 403
```

对外错误使用稳定结构，不返回堆栈、SQL 或内部路径。

## 9. Depends

```python
def get_todo_service(
    repository: Annotated[TodoRepository, Depends(get_repository)],
) -> TodoService:
    return TodoService(repository)


TodoServiceDep = Annotated[TodoService, Depends(get_todo_service)]
```

FastAPI 根据函数签名构建依赖图。同一请求中的相同依赖默认会复用结果，但这不是跨请求全局缓存。

## 10. yield 依赖

```python
async def get_session() -> AsyncIterator[AsyncSession]:
    async with AsyncSessionFactory() as session:
        yield session
```

```text
yield 前   获取资源
yield      提供给 Endpoint
yield 后   清理资源
```

关闭 Session 不等于自动设计好事务。commit、rollback 和异常边界仍需明确。

## 11. 认证依赖

```python
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> CurrentUser:
    payload = verify_access_token(token)
    return await user_service.require_active(payload.subject)
```

OAuth2PasswordBearer 主要提取 Bearer Token 和生成文档，不会自动验证签名、过期和用户权限。

## 12. 同步与异步

```text
异步数据库/HTTP 客户端   async def + await
同步阻塞库               普通 def，由线程池处理
CPU 密集长任务           任务队列、进程或独立服务
```

不要在 async Endpoint 中直接运行同步数据库查询、`requests.get` 或 `time.sleep`。

## 13. Settings、CORS 与 Lifespan

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    database_url: str
    allowed_origins: list[str] = Field(default_factory=list)
```

真实 `.env` 不提交，使用 `.env.example` 说明变量。

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

带凭据时使用明确可信 Origin。CORS 不是权限系统。

Lifespan 管理应用级连接池和客户端，yield 依赖管理单次请求资源。

中间件包围每一个匹配的请求，可用于请求 ID、耗时、统一安全响应头等横切逻辑：

```python
@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID") or str(uuid4())
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response
```

中间件不要读取并消耗大文件 Body，也不要塞入具体业务规则。执行顺序和异常处理会影响日志与响应，增加多个中间件后应通过测试确认行为。

## 14. 文件与后台任务

```python
@router.post("/avatar")
async def upload_avatar(
    file: Annotated[UploadFile, File()],
) -> dict[str, str | None]:
    return {"filename": file.filename, "content_type": file.content_type}
```

文件名和 content_type 不可信，后端仍要验证大小、内容和权限。

BackgroundTasks 适合短小的进程内工作，不是可靠任务队列。重要长任务使用专门队列。

## 15. OpenAPI 与测试

```text
/docs
/redoc
/openapi.json
```

```python
def test_health_check() -> None:
    with TestClient(app) as client:
        response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

`app.dependency_overrides` 可以在测试中替换数据库、认证和 Service。集成测试仍应覆盖真实测试数据库。

OpenAPI 是接口的机器可读说明，可用于文档、客户端类型生成和契约检查，但它不会替代测试。测试可分为：

```text
Service 单元测试       不启动 HTTP，验证业务规则
API 测试               TestClient/AsyncClient 验证状态码和响应
数据库集成测试         验证查询、约束和事务
端到端测试             从浏览器到真实测试环境
```

至少覆盖成功、认证失败、资源不存在、冲突和 Pydantic 422；测试之间要隔离数据。

## 16. 常见错误

- 请求、更新和响应共用 ORM Model；
- 把数据库查询放进 Pydantic 验证器；
- 所有 Endpoint 都写 async 但内部仍阻塞；
- 把 Depends 当成全局单例容器；
- Service 到处抛 HTTPException；
- 204 返回 JSON；
- CORS 配成 `*` 并当作权限；
- 用 BackgroundTasks 执行必须可靠完成的长任务；
- 启动每个 Web 进程时自动运行数据库迁移。

## 17. 给 AI 的开发指令

```text
请在现有 FastAPI + Pydantic v2 项目中实现 Todo CRUD。
分别定义 TodoCreate、TodoUpdate、TodoOut，不把 ORM Model 当请求 Schema。
Router 只处理 HTTP 和 Service 调用，业务规则放 Service。
POST 返回 201，DELETE 返回空 204，PATCH 使用 exclude_unset。
数据库 Session 使用 yield 依赖并明确事务边界。
确认同步异步调用链一致，补充 404、409、422 和成功测试。
最后运行 Ruff、类型检查和 pytest。
```

## 18. 面试表达

> FastAPI 读取函数签名和类型注解，完成路由、参数解析、Pydantic 校验、依赖解析、响应序列化和 OpenAPI 生成。

> Pydantic Schema 是 API 契约，SQLAlchemy Model 是数据库映射，二者职责不同。

> response_model 不只生成文档，还会验证、序列化和过滤响应字段。

> Depends 构建请求依赖图，yield 依赖表达请求范围资源的获取与清理。

> 异步只有在底层 I/O 真正非阻塞时才有意义。

[进入下一课：SQL 与关系型数据库基础 →](./12-SQL与关系型数据库基础.md)
