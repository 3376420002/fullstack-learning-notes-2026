# 19｜pytest、httpx、Apifox 与 OpenAPI 接口协作

> 所属阶段：质量保障与前后端联调
>
> 本课合并知识点：测试层次、pytest、fixture、FastAPI TestClient、httpx、测试数据库、OpenAPI、Apifox、接口契约和 CI
>
> 前置知识：HTTP、JSON、FastAPI、Axios、数据库、认证
>
> 学习目标：能判断接口的输入、输出、错误和权限是否符合约定，能指挥 AI 补有效测试，而不只验证“200 成功”。

[← 密码哈希、JWT 与权限](./18-密码哈希JWT与权限控制.md) · [Vue 工具、Element Plus 与 UnoCSS →](./20-Vue工具ElementPlus与UnoCSS.md)

## 1. 为什么已有 Swagger 页面还要测试

FastAPI 的 `/docs` 和 `/openapi.json` 能展示接口定义，但“文档能打开”不代表业务正确。比如 `PATCH /todos/42` 返回 200，也可能错误地修改了别人的 Todo。测试需要验证状态码、响应结构、数据库状态、权限边界以及错误场景。

```text
接口契约：应该接收什么、返回什么
测试代码：给定条件下，实际是否符合预期
调试工具：人工观察某次真实请求
```

OpenAPI、pytest 和 Apifox 解决的是相关但不同的问题，三者可以配合。

## 2. 测试的几个层次

| 层次 | 典型对象 | 优点 | 局限 |
| --- | --- | --- | --- |
| 单元测试 | Service 纯逻辑、校验函数 | 快，定位精确 | 不证明 HTTP/数据库接线正确 |
| API 测试 | FastAPI 路由与依赖 | 覆盖状态码、序列化、认证 | 可用替身数据库，未必覆盖真实 SQL |
| 数据库集成测试 | API/Repository + 测试 PostgreSQL | 发现 SQL、约束、事务问题 | 环境较慢，隔离更复杂 |
| 端到端测试 | 浏览器 → API → DB | 覆盖用户旅程 | 慢、容易受环境影响 |

不必每个行为都写完整浏览器测试；应优先保证关键业务和权限链路被相应层次覆盖。测试是否有效取决于断言和隔离，不取决于文件数量。

## 3. pytest 的基本结构

```python
def normalize_title(raw: str) -> str:
    title = raw.strip()
    if not title:
        raise ValueError("title must not be empty")
    return title


def test_normalize_title_trims_spaces() -> None:
    assert normalize_title("  buy milk  ") == "buy milk"
```

pytest 发现符合命名规则的 `test_*.py` 文件和 `test_*` 函数。普通 `assert` 即可；失败时报告实际值与预期值。测试名应表达行为，而不是只叫 `test_1`。

```bash
uv run pytest
uv run pytest tests/test_todos.py -q
```

`-q` 减少输出，不改变测试内容。CI 应显示失败日志而不是静默吞掉错误。

## 4. Arrange–Act–Assert 让测试可读

```text
Arrange：准备用户、数据、配置
Act：    调用函数或发 HTTP 请求
Assert： 检查状态、响应和副作用
```

一个测试最好有清晰的业务意图。只写 `assert response.status_code == 200` 常常不足以证明数据正确；创建接口还应检查返回的 ID、字段、数据库是否真正写入。

## 5. fixture：可复用的测试准备

```python
import pytest
from fastapi.testclient import TestClient
from app.main import app


@pytest.fixture
def client():
    with TestClient(app) as test_client:
        yield test_client


def test_health(client: TestClient) -> None:
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

fixture 由参数名注入。`with TestClient(app)` 会处理应用生命周期；若真实 lifespan 会连接生产资源，测试必须先替换配置/依赖，不能让测试意外连接生产数据库。共享 fixture 放 `conftest.py` 时应控制作用域，避免测试间互相污染。

## 6. FastAPI 依赖覆盖

如果路由通过 `Depends(get_repository)` 获取仓库，测试可以提供内存替身：

```python
from fastapi.testclient import TestClient
from app.main import app
from app.dependencies import get_repository


def override_repository():
    return FakeTodoRepository()


app.dependency_overrides[get_repository] = override_repository
try:
    with TestClient(app) as client:
        response = client.get("/api/v1/todos")
        assert response.status_code == 200
finally:
    app.dependency_overrides.clear()
```

`FakeTodoRepository` 是项目需提供的测试替身。覆盖后一定清理，否则下个测试可能意外继续使用替身。若项目还有其他依赖覆盖，最好只移除本次覆盖，不机械 `clear()` 全局已有配置。

替身适合验证路由、序列化和部分业务逻辑；它无法发现真实 PostgreSQL 的索引、约束、事务或 SQL 兼容问题。关键持久化流程仍需要真实数据库集成测试。

## 7. 参数化：同一规则的多组输入

```python
import pytest


@pytest.mark.parametrize(
    ("raw", "expected"),
    [(" buy milk ", "buy milk"), ("read", "read")],
)
def test_title_normalization(raw: str, expected: str) -> None:
    assert normalize_title(raw) == expected
```

参数化避免复制多个结构完全相同的测试，但不要把不相关的业务场景塞进一张难以理解的大表。失败用例也应检查异常类型、错误码和字段。

## 8. 接口测试要覆盖哪些类别

以创建 Todo 为例：

| 类别 | 例子 | 应检查 |
| --- | --- | --- |
| 正常 | 合法标题 | 201/200、字段和持久化结果 |
| 输入边界 | 空标题、超长标题 | 422/约定错误、无脏数据 |
| 认证 | 没有令牌、令牌过期 | 401，不创建数据 |
| 授权 | A 修改 B 的 Todo | 403/404，不改变 B 的数据 |
| 资源状态 | ID 不存在、已删除 | 404 或约定状态 |
| 并发/重复 | 重复提交、版本冲突 | 幂等/冲突策略符合契约 |
| 分页排序 | 边界页码、排序字段 | 稳定次序、总数/游标正确 |

不要为了让测试通过而让测试依赖数据插入顺序、数据库自增 ID 具体值或上一测试留下的记录。

## 9. 测试数据库的隔离

测试不能连接生产数据库。应有独立连接串和数据库，例如 CI 中单独启动 PostgreSQL。每个测试的隔离方式可选：事务回滚、清空数据/重建 schema、为测试生成独立数据库等；要与应用实际连接池和事务边界匹配。

如果应用在测试请求中另开数据库连接，测试外层事务回滚未必覆盖其写入。需要按当前 SQLAlchemy 会话依赖设计隔离，并验证“测试结束后数据确实消失”。数据库迁移应在测试开始前应用，避免测试 schema 与生产发布 schema 不一致。

## 10. httpx 同步与异步客户端

`FastAPI TestClient` 适合普通同步测试；测试异步调用链时可使用 HTTPX `AsyncClient` 和 `ASGITransport`：

```python
import httpx

transport = httpx.ASGITransport(app=app)
async with httpx.AsyncClient(transport=transport, base_url="http://test") as client:
    response = await client.get("/health")
    assert response.status_code == 200
```

片段需放在异步测试函数内，并配置 pytest 的异步测试支持。`ASGITransport` 不负责自动触发 ASGI lifespan；若应用启动/关闭需要资源，测试要显式管理 lifespan，或使用适合的测试客户端/工具。不要因为本地同步示例通过，就假定异步 DB 的生命周期也被覆盖。

## 11. Mock 外部服务，不 Mock 自己的业务真相

调用第三方短信、支付或天气服务时，可用 Mock/测试替身避免真实收费或网络不稳定；但不要把最关键的业务规则也全部 Mock 掉。比如“Todo 只能由所有者修改”应在有真实授权逻辑的层面测试。Mock 的断言也要包含请求参数、错误处理与超时路径。

## 12. OpenAPI 是什么

OpenAPI 是机器可读的 HTTP API 描述格式，包含路径、方法、参数、请求体、响应、认证方案和 Schema。FastAPI 可根据路由、类型和 Pydantic 模型生成 `/openapi.json`，并在 `/docs` 展示交互式文档。

```text
FastAPI 路由 + Pydantic 模型
       ↓ 生成
/openapi.json
       ↓ 导入/消费
文档、Apifox、类型生成器、契约检查
```

生成文档只是“代码声称自己如何工作”。若响应模型写错或示例与实际业务不一致，文档一样会错。关键接口需用自动测试校验实际行为和契约。

## 13. 让 OpenAPI 描述更有用

```python
from fastapi import APIRouter, status
from pydantic import BaseModel, Field

router = APIRouter(prefix="/api/v1/todos", tags=["todos"])


class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=120)


class TodoRead(BaseModel):
    id: int
    title: str
    completed: bool


@router.post("", response_model=TodoRead, status_code=status.HTTP_201_CREATED)
async def create_todo(payload: TodoCreate) -> TodoRead:
    ...  # 教学片段：此处应调用 Service 并返回真实持久化结果
```

路由前缀、标签、请求/响应模型和成功状态码会进入文档。上面的函数体是省略号，不是完整实现；业务异常的响应格式也要通过代码和文档约定，不能只描述成功路径。

## 14. 接口契约最容易不一致的地方

- `null`、缺省字段和空数组含义不同；
- 日期时间应约定时区与字符串格式；
- 分页参数、排序字段与默认顺序要明确；
- 4xx 错误体要一致，不能每个接口返回不同结构；
- `PATCH` 是部分更新，不能把未传字段当成“设为 null”；
- 前端 TypeScript 类型与后端 Pydantic 模型可能独立漂移；
- OpenAPI 安全方案要反映真实认证方式。

接口变更先讨论兼容性：新增可选字段通常比删除/改类型容易兼容，但客户端仍可能有严格解析。版本号并不能自动解决兼容问题。

## 15. Apifox 的位置

Apifox 可管理接口定义、环境变量、调试请求、Mock 和团队协作。一个常见流程是：后端从 FastAPI 导出 OpenAPI → 导入 Apifox → 前后端核对接口路径、Schema、错误码 → 在开发/测试环境调试。也可能团队先在 Apifox 定义契约再实现后端；关键是明确“哪里是当前权威版本”和同步责任。

Apifox 中要分开环境的 Base URL 与接口相对路径。不要把生产 Token、数据库密码或用户真实数据写进共享示例、公开文档或截图。手工调试成功不能取代可重复执行的 pytest/CI 检查。

## 16. 前后端联调的具体对照

Todo 列表加载失败时，逐项对照：

```text
Vue/Axios 请求方法和 URL
 → Nginx/Vite 代理是否保留 /api 前缀
 → FastAPI 路由是否存在
 → Authorization/Cookie 是否发送
 → 响应状态码和 JSON 结构
 → 前端 TypeScript 类型与解析代码
```

浏览器 Network 面板确认真实请求；Apifox 可帮助单独验证 API；pytest 可固定已确认的行为。不要只看“页面空白”，那无法区分前端渲染、401、404 和后端 500。

## 17. 把测试放进 CI

CI 至少应在 PR 上运行关键单元/API 测试和构建。涉及 PostgreSQL 的集成测试需启动受控测试数据库，注入测试连接串并执行迁移；不能让 CI 使用开发者本机数据库或生产数据库。

失败时看第一个根因；不要用 `|| true`、整体跳过测试或把断言删掉来让 CI 变绿。偶发失败应先检查数据隔离、时间依赖、并发竞争和外部服务依赖。

## 18. 指挥 AI 补测试与契约

```text
请为 TodoLab 的 POST /api/v1/todos 和 PATCH /api/v1/todos/{id}
补 pytest API 测试。先阅读现有路由、Pydantic 模型、鉴权依赖和数据库测试 fixture。
覆盖成功、空标题/超长标题、未登录、过期令牌、跨用户修改、资源不存在；
每个测试检查状态码、响应体和必要的数据库副作用。
禁止连接生产数据库，不要改业务代码只为让测试通过。
完成后运行相关测试，报告实际通过/失败情况与未覆盖边界。
```

```text
请对照 FastAPI 的 /openapi.json、前端 Axios 类型和 Apifox 中的接口定义，
列出路径、字段、可空性、状态码、认证方式的差异。
先说明哪一份是团队约定的权威契约，再给最小同步方案；
不要凭空更改公共 API，也不要使用真实密钥做调试。
```

## 19. 面试表达速记

> 单元测试检查局部业务规则，API 测试覆盖路由、参数校验和响应，数据库集成测试验证真实 SQL/事务。关键权限链路不能只依赖 Mock。

> pytest 的 fixture 管理可复用准备和清理；参数化适合同一规则的多组输入。测试应断言响应和副作用，并保持数据隔离。

> OpenAPI 是接口机器可读契约，FastAPI 可生成文档；Apifox 用于团队管理与调试。文档、工具调试与自动测试互相补充，不互相替代。

> 前后端联调我会从浏览器真实请求出发，对照方法/路径、代理、认证、状态码和 JSON 结构，再用接口测试固定已确认的行为。

## 20. 技术地图与官方资料

```text
接口设计 → OpenAPI → Apifox / 前端类型
     ↓
FastAPI 实现 → pytest + TestClient/httpx
     ↓
测试 PostgreSQL → CI 检查 → PR 评审
```

- [pytest 文档](https://docs.pytest.org/en/stable/)
- [FastAPI：测试](https://fastapi.tiangolo.com/tutorial/testing/)
- [FastAPI：异步测试](https://fastapi.tiangolo.com/advanced/async-tests/)
- [HTTPX：ASGITransport](https://www.python-httpx.org/advanced/transports/)
- [OpenAPI 规范](https://spec.openapis.org/oas/)
- [Apifox 帮助文档](https://apifox.com/help/overview/start)

## 21. 下一步

最后一课回到前端开发体验：Vue 官方工具、Element Plus 和 UnoCSS 如何各司其职，组成可维护的后台界面。

[进入下一课：Vue 工具、Element Plus 与 UnoCSS →](./20-Vue工具ElementPlus与UnoCSS.md)
