# 17｜uv 与 Uvicorn：后端依赖和运行

> 所属阶段：Python 后端工程化
>
> 本课合并知识点：Python 版本、虚拟环境、`pyproject.toml`、`uv.lock`、依赖组、uv 命令、ASGI、Uvicorn、进程与部署
>
> 前置知识：Python、FastAPI、Docker、CI
>
> 学习目标：能解释“依赖如何复现”和“Web 服务如何真正运行”，并识别本地开发与生产部署的差别。

[← Git、代码协作与 CI](./16-Git代码协作与持续集成.md) · [密码哈希、JWT 与权限 →](./18-密码哈希JWT与权限控制.md)

## 1. uv 和 Uvicorn 名字相似，职责完全不同

```text
uv       → 管 Python 版本、项目依赖、锁文件、虚拟环境、运行命令
Uvicorn  → 运行 ASGI 应用，监听端口，接收 HTTP/WebSocket 请求
FastAPI  → 定义路由、参数校验、依赖和响应
```

`uv run uvicorn app.main:app` 可以读成：“uv 准备/使用项目环境，然后启动 Uvicorn，Uvicorn 导入 `app.main` 中的 `app` 对象”。uv 不是 Web 服务器；Uvicorn 不是包管理器。

## 2. Python 项目的几个核心文件

| 文件/目录 | 作用 | 是否通常提交 |
| --- | --- | --- |
| `pyproject.toml` | 项目元数据、依赖声明、工具配置 | 是 |
| `uv.lock` | 解析后的准确依赖版本 | 是 |
| `.python-version` | 团队约定的 Python 版本 | 通常是 |
| `.venv/` | 本机安装出来的虚拟环境 | 否 |
| `src/app/` | 应用源码 | 是 |
| `alembic/` | 数据库迁移脚本 | 是 |

`pyproject.toml` 描述可接受的版本范围，`uv.lock` 记录一次解析得到的具体版本与来源。锁文件用于复现；不要手改锁文件。`.venv` 与操作系统、解释器路径有关，通常不提交。

## 3. 虚拟环境到底隔离了什么

系统 Python 安装的包可能服务于多个项目；项目 `.venv` 则为当前项目提供独立的解释器入口和安装包位置。它不是 Docker，也不隔离文件系统、网络或操作系统。

```bash
uv sync
uv run python -c "import sys; print(sys.executable)"
uv run python -c "import fastapi; print(fastapi.__version__)"
```

使用 `uv run` 不要求先手工激活 `.venv`。VS Code 仍应选择项目 `.venv` 的 Python 解释器，让补全、检查和调试与命令行一致。

## 4. 从空项目到可运行 FastAPI

下面假设 `backend/` 已是一个 uv 项目，且已有后文展示的 `src/app/main.py` 和正确的包安装配置。若从零开始，可先用 `uv init backend` 创建基础项目，再按第 7 节补齐目录和构建配置；`uv init` 本身不会自动生成 `src/app/main.py`。

```bash
cd backend
uv python pin 3.13
uv add fastapi "uvicorn[standard]"
uv add --dev pytest httpx
uv run uvicorn app.main:app --reload
```

`uv python pin` 将解释器版本写到项目配置文件；它不等于强制团队所有机器都已安装该解释器。`uv add` 更新项目依赖声明和锁文件。`--dev` 表示开发依赖，测试/检查工具不需要进入精简运行镜像。`uvicorn[standard]` 的可选依赖提供一些优化/开发能力；实际部署按项目锁文件安装即可。

## 5. `uv sync`、`uv lock`、`uv run` 的区别

```text
uv lock  → 更新/检查依赖解析结果并维护 uv.lock
uv sync  → 使项目环境与锁文件对应的依赖集合一致
uv run   → 在项目环境中运行命令，通常先确保锁和环境可用
```

`uv sync` 默认执行精确同步：可能移除环境中不在目标依赖集合里的额外包。`uv run` 的默认同步策略与 `uv sync` 不完全相同；团队 CI 需要确定版本时使用 `--locked`，让锁文件过期时报错，而不是静默改写。

```bash
uv lock --check
uv sync --locked
uv run --locked pytest
```

`--locked` 会验证锁文件与依赖声明一致；`--frozen` 使用现有锁文件而不做相同的一致性检查。不要把二者理解为“都会锁住当前运行版本”。升级依赖应有明确的变更、测试和锁文件提交。

## 6. 依赖与开发依赖放在哪里

`fastapi`、`uvicorn`、数据库驱动和运行期使用的库属于运行依赖；`pytest`、静态检查工具通常属于开发依赖。把运行必需包只放开发组，会导致生产镜像 `uv sync --no-dev` 后启动失败。

```toml
[project]
name = "todolab-backend"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
  "fastapi",
  "uvicorn[standard]",
  "sqlalchemy",
  "asyncpg",
  "alembic",
]

[dependency-groups]
dev = ["pytest", "httpx"]
```

这只是结构示例，实际项目的范围、数据库驱动及构建后端应由现有代码和测试决定。库项目常把可选功能放在 extras（`[project.optional-dependencies]`）；开发组和用户可安装的 extras 不是同一个概念。

## 7. `src` 布局与导入路径

```text
backend/
├─ pyproject.toml
├─ uv.lock
└─ src/
   └─ app/
      ├─ __init__.py
      └─ main.py
```

`uvicorn app.main:app` 要求 Python 能导入 `app.main`。`src` 布局通常靠项目的构建/可编辑安装让包可导入；如果 `pyproject.toml` 没定义构建系统、项目没有安装，可能需要调整配置或命令，而不是一律修改 `PYTHONPATH` 掩盖布局问题。

```python
# src/app/main.py
from fastapi import FastAPI

app = FastAPI(title="TodoLab API")


@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

`module:attribute` 中冒号左边是可导入模块，右边是该模块中的 ASGI 应用对象。`main.py` 文件名和变量名都要与实际项目一致。

## 8. WSGI 和 ASGI 的位置

WSGI 是 Python Web 应用与服务器之间的同步接口；ASGI 支持异步连接与 HTTP、WebSocket 等协议。FastAPI 是 ASGI 应用，Uvicorn 是 ASGI 服务器。

```text
浏览器 / Nginx
    ↓ HTTP
Uvicorn（监听端口、协议处理）
    ↓ ASGI 调用
FastAPI（路由、依赖、业务代码）
    ↓
数据库 / 外部服务
```

ASGI 不表示所有代码都会自动并行，也不保证慢数据库会变快。阻塞调用如果在异步路径中直接运行，仍可能卡住事件循环。是否使用同步或异步数据库驱动，要与调用链一致。

## 9. 开发时为什么常用 `--reload`

```bash
uv run uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

热重载监视源码变化并重启工作进程，适合本地开发；生产环境会带来额外复杂性，不应默认启用。`--reload` 和 `--workers` 不能同时使用。开发服务器看到“启动成功”不代表数据库、迁移和外部接口都可用。

## 10. 监听地址与容器

本地只给自己访问可监听 `127.0.0.1`；在容器内要让其他容器/宿主机通过映射端口访问，Uvicorn 通常要监听 `0.0.0.0`：

```bash
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

`0.0.0.0` 是“监听所有 IPv4 网卡”，不是浏览器应该输入的目的地址。对外暴露还取决于 Docker `ports`、防火墙、反向代理和网络策略。

## 11. 多 worker 与进程内状态

```bash
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2
```

多个 worker 是多个独立进程，各有自己的内存、连接池和应用生命周期。把登录状态、任务队列或全局计数只存 Python 全局变量，多个 worker 会彼此看不到。跨进程共享状态应使用数据库、缓存或可靠队列，并按业务一致性要求设计。

worker 数不是越多越好：会增加内存和数据库连接数。应根据 CPU、内存、请求类型、容器副本数与压测结果决定。多个容器副本和单容器多 worker 也是两个独立维度。

## 12. 应用启动与关闭生命周期

FastAPI 可用 lifespan 在启动时创建资源、关闭时释放资源。例如数据库连接池、HTTP 客户端应按应用生命周期管理，而非每次请求都重建，也不应在模块导入时偷偷建立生产连接。

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    # 在这里初始化应用级资源；真实项目应保存并在退出时关闭。
    yield
    # 在这里释放应用级资源。


app = FastAPI(lifespan=lifespan)
```

每个 worker 都会执行自己的生命周期，因此一次性数据库迁移不应塞进 lifespan 后让所有 worker 同时执行。迁移更适合独立部署步骤或第 14 课的单独 Compose 服务。

## 13. 反向代理、HTTPS 与可信代理头

常见部署是 Nginx/负载均衡器接收 HTTPS，再以内部 HTTP 转给 Uvicorn。应用可能需要知道原始协议和客户端地址，代理会传 `X-Forwarded-Proto`、`X-Forwarded-For` 等头。但这些头可被客户端伪造，Uvicorn 只应信任已知代理来源。

不要为了修复重定向 URL 错误就无条件信任所有代理头。先确认网络拓扑、反向代理是否覆盖客户端传入的相关头，再配置允许的代理 IP/网段。TLS 终止、域名、CORS 和 Cookie 安全属性要一起考虑。

## 14. 配置文件、环境变量与秘密

应用配置应区分开发、测试、生产，但不要靠散落的硬编码 `if environment == ...` 控制所有行为。数据库 URL、签名密钥等运行时秘密由部署环境提供，不进源码和镜像；前端 `VITE_*` 变量是公开配置，不可存后端秘密。

`uv run --env-file` 与 Uvicorn 的 `--env-file` 不是同一层的配置机制；团队应约定谁读取环境文件、哪些变量传给应用。Docker Compose 的 `.env` 用于模板替换，不会自动传给所有容器，见第 14 课。

## 15. 健康检查、就绪与优雅退出

- 存活检查：进程能回应一个轻量请求；
- 就绪检查：当前副本可处理正常流量；
- 依赖检查：数据库/外部服务是否可用。

这些检查可以分开。若 `/health` 每次都查询数据库，数据库短暂抖动可能让所有实例被判不健康；若完全不检查依赖，又无法表示真正就绪。根据部署平台与恢复策略设计，不要在健康检查里做重查询。

优雅退出时停止接收新请求、等待进行中的请求完成并关闭资源；进程被强制杀死则可能中断处理。发布滚动更新时尤其要考虑超时和长请求。

## 16. 本地、CI、容器的同一条命令链

```text
本地：uv sync → uv run pytest → uv run uvicorn --reload
CI：  uv sync --locked → uv run --locked pytest
镜像：uv sync --locked --no-dev → uvicorn（不热重载）
```

关键不是所有环境命令一模一样，而是依赖声明和锁文件一致、启动入口明确、差异有意设计。CI 可以额外跑 lint/类型检查；生产镜像则移除开发依赖。

## 17. 一次具体排错：容器内报 `ModuleNotFoundError: app`

按顺序检查：

1. Dockerfile 的 `WORKDIR`、`COPY` 是否把 `src/app` 放进镜像；
2. `pyproject.toml` 是否定义构建系统并正确发现 `src` 包；
3. `uv sync` 是否安装项目本身，是否用了 `--no-install-project` 后忘了第二次同步；
4. Uvicorn 的导入字符串是否与真实包名一致；
5. 运行的是不是镜像中的 `.venv` Python。

盲目加 `sys.path.append()` 往往只是在绕过真正的打包配置错误。

## 18. 一次具体排错：接口一并发就很慢

检查慢请求耗时分布、数据库查询、连接池与外部 HTTP 调用。异步路由里若用阻塞 `time.sleep()` 或同步网络请求，会阻塞事件循环；应改用合适的异步库，或把同步工作放在合适的线程/任务执行边界。增加 worker 可缓解部分情况，但不替代修复慢查询和阻塞调用。

## 19. 可以交给 AI 的工程指令

```text
请审查当前 FastAPI 项目的 pyproject.toml、uv.lock、Dockerfile 和 CI。
确认运行依赖与开发依赖分组正确，src 包可导入，
本地、CI、容器均使用相同锁文件。
先列出发现的问题和依据，再做最小修改；
不要无理由升级全部依赖，也不要把真实 .env 放入镜像。
修改后运行项目已有的测试与构建检查，并报告实际结果。
```

```text
请分析这个 Uvicorn 启动失败日志（已脱敏）。
区分导入错误、端口冲突、数据库连接、迁移失败和代理配置问题。
按只读检查优先给出定位顺序，再提供最小修复和验证步骤。
```

## 20. 面试表达速记

> uv 管项目依赖、锁文件和虚拟环境；Uvicorn 是 ASGI 服务器；FastAPI 是应用框架。`uv run uvicorn app.main:app` 会在项目环境中启动服务器并导入应用对象。

> `pyproject.toml` 声明依赖需求，`uv.lock` 固定解析结果，`.venv` 是当前机器安装后的环境。CI 通过 `--locked` 检查锁文件与声明一致，避免构建时意外改版本。

> `--reload` 用于开发，`--workers` 启动多个独立进程。worker 不共享 Python 全局变量，多实例状态应放数据库/缓存等外部系统。

> 容器内服务通常监听 `0.0.0.0`，但是否对公网开放还取决于端口发布、代理和防火墙；代理头只能信任已知反向代理。

## 21. 技术地图与官方资料

```text
pyproject.toml + uv.lock
        ↓ uv sync / uv run
项目 .venv 与 app 包
        ↓ Uvicorn 导入 app.main:app
ASGI 请求 → FastAPI → Service → PostgreSQL
```

- [uv 项目指南](https://docs.astral.sh/uv/guides/projects/)
- [uv 锁定与同步](https://docs.astral.sh/uv/concepts/projects/sync/)
- [uv 在 Docker 中使用](https://docs.astral.sh/uv/guides/integration/docker/)
- [Uvicorn 设置](https://www.uvicorn.org/settings/)
- [Uvicorn 部署](https://www.uvicorn.org/deployment/)
- [FastAPI 生命周期](https://fastapi.tiangolo.com/advanced/events/)

## 22. 下一步

下一课学习登录安全：密码如何保存，JWT 如何签发/验证，接口如何判断“是谁”和“有没有权限”。

[进入下一课：密码哈希、JWT 与权限控制 →](./18-密码哈希JWT与权限控制.md)
