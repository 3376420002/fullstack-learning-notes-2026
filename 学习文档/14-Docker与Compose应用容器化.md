# 14｜Docker 与 Compose 应用容器化

> 所属阶段：全栈工程交付
>
> 本课合并知识点：镜像、容器、Dockerfile、构建缓存、网络、端口、数据卷、Compose、健康检查、数据库迁移和排错
>
> 前置知识：Vue 3 + Vite、FastAPI、PostgreSQL、Alembic
>
> 学习目标：看懂并修改常见 Docker 配置，能指挥 AI 把前端、后端和数据库组成可复现环境，并在面试中解释容器化的边界。

[← PostgreSQL、SQLAlchemy 与 Alembic](./13-PostgreSQL、SQLAlchemy与Alembic.md)

## 阅读方式

- 第 1～10 节建立镜像、容器、卷、网络与端口的心智模型；
- 第 11～17 节阅读前后端 Dockerfile、Nginx 与构建缓存；
- 第 18～25 节用 Compose 连接三个服务，理解启动、配置与数据库持久化；
- 第 26 节以后用于排错、AI 编程指令和面试复习。

本课示例围绕 TodoLab：Vue 页面、FastAPI 接口、PostgreSQL 16 数据库。示例是可迁移的工程模板；实际项目应优先复用已有的依赖管理、路由和部署配置。

## 1. Docker 解决什么问题

在没有统一运行环境时，同一项目可能在甲的电脑能运行，在乙的电脑缺少 PostgreSQL、Python 版本不匹配，或前端构建工具版本不同。Docker 把运行应用所需的文件、依赖和启动方式描述成镜像，再从镜像启动容器。

```text
源码 + Dockerfile ──构建──→ 镜像 ──运行──→ 容器
                                     ↓
                            网络、环境变量、数据卷
```

Docker 不替代代码测试、数据库备份和权限设计。容器让“运行条件是什么”变得明确，也让开发、CI 与部署更容易复现。

## 2. 镜像、容器、Docker Engine、Docker Desktop

| 名称 | 一句话理解 | TodoLab 对应物 |
| --- | --- | --- |
| 镜像 Image | 可复用的运行模板 | `postgres:16`、后端镜像 |
| 容器 Container | 镜像运行后的独立进程与文件系统 | 正在运行的 `api` 容器 |
| Docker Engine | 管理镜像、容器、网络和卷的服务 | 接收 `docker` 命令 |
| Dockerfile | 构建镜像的配方 | 安装依赖、复制代码、设置启动命令 |
| Compose | 描述和管理多个服务 | 同时运行 `web`、`api`、`db` |
| Registry | 存放和分发镜像 | Docker Hub、公司私有仓库 |

同一个镜像可以启动多个容器。修改本地源码不会自动改变已经构建好的镜像；通常需要重新构建，或者开发环境使用受控的文件同步/挂载。

`docker` 命令行是客户端；它通过 Docker Engine 执行操作。Windows 上即使命令行能运行，也可能因为 Docker Desktop 尚未启动而无法连接 Engine。

## 3. 容器与虚拟机的区别

```text
虚拟机：应用 + 完整来宾操作系统 + 虚拟硬件
容器：应用进程 + 文件/网络隔离；通常共享宿主机内核
```

容器启动通常更轻，但不等于“完全隔离的安全沙箱”。在 Windows/macOS 的 Docker Desktop 中，Linux 容器仍通过虚拟化的 Linux 环境运行；因此不能简单理解为 Linux 内核直接运行在 Windows 上。

分享或面试时，重点说明容器是进程级运行与隔离方式，镜像是用于创建容器的模板。

## 4. 安装后先看三个状态

```powershell
docker --version
docker compose version
docker info
```

前两个命令检查客户端和 Compose 插件；`docker info` 能进一步确认是否连接到正在运行的 Engine。Docker Desktop 在 Windows 上通常需要启用 WSL2 后端和硬件虚拟化。

遇到“Cannot connect to the Docker daemon/Engine”时，先检查 Docker Desktop 状态和当前 Docker context，而不是立即改项目代码。

## 5. 一条 `docker run` 看懂容器参数

本地开发可先单独启动 PostgreSQL：

```powershell
docker run -d `
  --name todolab-db `
  -e POSTGRES_DB=todolab `
  -e POSTGRES_PASSWORD=dev-only-change-me `
  -v todolab-pgdata:/var/lib/postgresql/data `
  -p 127.0.0.1:15432:5432 `
  postgres:16
```

这条命令中的密码只是教学用本地占位值；真实密码不应写进命令历史、文档或仓库。逐项理解：

```text
-d              后台运行
--name          给容器一个稳定名称
-e              设置容器内环境变量
-v              将命名卷挂到 PostgreSQL 数据目录
-p              把宿主机 15432 映射到容器 5432
postgres:16     镜像名称和标签
```

示例选择宿主机 `15432`，避免本机 PostgreSQL 已占用 `5432`。`127.0.0.1` 限制在本机访问。容器内部 PostgreSQL 仍监听 `5432`。

查看和连接：

```powershell
docker ps
docker logs todolab-db
docker exec -it todolab-db psql -U postgres -d todolab
```

若容器中初始化了其他用户，`psql -U` 要与 `POSTGRES_USER` 一致。

## 6. 容器生命周期与镜像生命周期

```text
pull/build 镜像
   ↓
create 容器
   ↓
start → running → stop
            ↘ restart
   ↓
remove 容器
```

常见命令：

```powershell
docker ps
docker ps -a
docker stop todolab-db
docker start todolab-db
docker logs --tail 100 todolab-db
docker inspect todolab-db
```

停止容器不会删除它；删除容器也不会自动删除命名卷。删除镜像和删除容器也是两件事。部署时通常重新创建容器而不是进入容器手工修改文件。

## 7. 镜像层与构建缓存

Dockerfile 中的构建指令形成可复用的层。若依赖声明没有变化，之前安装依赖的层可能被缓存；若在安装依赖前就 `COPY . .`，源码每次改变都容易让后续安装步骤重新执行。

```dockerfile
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev --no-install-project
COPY . .
RUN uv sync --locked --no-dev
```

把变化少的依赖文件放在前面，是为了利用缓存。缓存不是正确性的保证：基础镜像、依赖锁文件或构建参数变化时应重新验证产物。

## 8. 容器的可写层为什么不能放数据库唯一副本

容器可以写文件，但其可写层随容器删除而消失。数据库的唯一数据副本不能只放在那里。命名卷由 Docker 独立管理，重新创建容器时可以继续挂载同一卷。

```text
PostgreSQL 容器
└─ /var/lib/postgresql/data  ←→  Docker 命名卷 pgdata
```

卷使数据跨容器生命周期保留，但卷本身不是备份。误删记录、损坏数据库或删除卷后，仍需要独立备份与恢复方案。`docker compose down --volumes` 会删除 Compose 定义的命名卷，数据库资料可能随之丢失；在有真实数据的项目中不要把它当普通清理命令。

## 9. 命名卷与 Bind Mount

| 挂载方式 | 来源 | 常见用途 |
| --- | --- | --- |
| 命名卷 | Docker 管理的数据区域 | PostgreSQL 数据、持久缓存 |
| Bind Mount | 宿主机明确路径 | 开发时同步源码、导入配置文件 |

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

```yaml
volumes:
  - ./backend:/app
```

开发时 bind mount 源码方便热重载；生产镜像通常把确定版本的源码构建进镜像，避免运行结果受宿主机随意修改影响。Windows 上 bind mount 还可能遇到路径转换、权限和文件监控性能问题。

## 10. 容器网络与 `localhost` 的位置

Compose 默认会为应用创建网络，服务可用服务名互相访问：

```text
web  →  http://api:8000
api  →  postgresql+asyncpg://...@db:5432/todolab
```

`localhost` 永远指“当前网络环境自己”：

- 在 `api` 容器里，`localhost:5432` 指 `api` 容器，不是 `db`；
- 在浏览器里，`http://api:8000` 通常无法解析，因为 `api` 是 Compose 内部服务名；
- 在宿主机里，`localhost:15432` 才是第 5 节映射后的数据库端口。

容器重建后 IP 可能改变；服务之间应使用稳定的服务名，不要写死内部 IP。

## 11. `EXPOSE`、`ports` 与容器间访问

```dockerfile
EXPOSE 8000
```

`EXPOSE` 描述镜像预期监听端口，本身不会把端口开放到宿主机。Compose 的 `ports` 才将容器端口发布出去：

```yaml
ports:
  - "127.0.0.1:8080:80"
```

这里是“宿主机本机 8080 → web 容器 80”。同一 Compose 网络中的 `web` 访问 `api:8000` 不要求 `api` 发布宿主机端口。数据库也无需为了让 API 访问而暴露到宿主机。

若宿主机其他电脑也要访问，需要网络和安全配置；不要把 `0.0.0.0` 绑定误认为“只在本机”。

## 12. Dockerfile 常用指令

```text
FROM        选择基础镜像或开启新构建阶段
WORKDIR     设置后续命令的工作目录
COPY        将构建上下文中的文件复制进镜像
RUN         构建镜像时执行命令
ENV         设置镜像内环境变量
ARG         声明构建时参数
EXPOSE      文档化监听端口
CMD         容器启动时的默认命令
ENTRYPOINT  容器默认入口程序
```

`RUN` 发生在构建时；`CMD` 发生在容器启动时。数据库迁移通常属于部署流程，不宜写进 Dockerfile 的 `RUN`，因为构建镜像时不应依赖真实生产数据库。

`COPY` 只能从构建上下文读取文件。命令 `docker build -f backend/Dockerfile backend` 的上下文是 `backend/`，不能在 Dockerfile 里 `COPY ../frontend`。

## 13. 后端 Dockerfile：FastAPI + uv

假设后端已有 `pyproject.toml`、`uv.lock`、`src/app/` 和 `alembic/`，可以从这个模板理解构建步骤：

```dockerfile
# backend/Dockerfile
FROM python:3.13-slim

# 教学示例使用 latest；正式项目应固定已验证的 uv 版本或镜像 digest。
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app
ENV UV_LINK_MODE=copy
ENV PATH="/app/.venv/bin:$PATH"

COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev --no-install-project

COPY . .
RUN uv sync --locked --no-dev

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`uv sync --locked` 要求锁文件与依赖声明一致；`--no-dev` 排除开发依赖。因此 `uvicorn`、数据库驱动、Alembic 等运行时所需包必须在适当的运行依赖中。

`--no-install-project` 先装依赖，源码复制后再装项目，提高构建缓存命中率。若项目是 uv workspace 或构建后端依赖本地文件，需要按真实项目调整。`ENV PATH` 让启动命令使用镜像内 `.venv` 的程序。

`--host 0.0.0.0` 让 Uvicorn 在容器网络接口监听；若只监听 `127.0.0.1`，其他容器无法连接。镜像标签和运行依赖应按项目锁定，示例不是版本升级建议。

## 14. 后端 `.dockerignore`

```gitignore
# backend/.dockerignore
.venv/
__pycache__/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.env
.env.*
*.pyc
.git/
```

`.dockerignore` 控制哪些文件不会进入构建上下文。它和 Git 的 `.gitignore` 作用不同：前者影响 Docker build，后者影响版本控制。不要把 `uv.lock`、`alembic/versions/` 和应用源码排除，否则镜像无法完整运行。

若项目有需要复制的 `.env.example`，要检查 `.env.*` 规则是否将其一并排除；按实际构建需要调整。真实 `.env` 不应复制进镜像。

## 15. 前端 Dockerfile：构建与运行分开

浏览器最终只需要 Vite 产出的静态文件，通常不需要在运行镜像里保留 Node.js 和 `node_modules`：

```dockerfile
# frontend/Dockerfile
FROM node:22-alpine AS build
WORKDIR /app

RUN corepack enable
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

FROM nginx:stable-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

第一阶段使用 Node/pnpm 构建，第二阶段只带 Nginx 和 `dist` 静态产物。项目 `package.json` 应锁定 `packageManager`；正式部署还要固定验证过的基础镜像版本或 digest。

Vite 的 `VITE_` 变量通常在构建时写入前端产物，改运行时容器环境变量不会自动修改已构建 JS。前端变量都是公开配置，不能存数据库密码、JWT 签名密钥或私钥。

## 16. 前端 `.dockerignore`

```gitignore
# frontend/.dockerignore
node_modules/
dist/
.env.local
.env.*.local
.git/
```

只复制构建需要的源码和配置。`node_modules` 应在镜像内根据锁文件重新安装，不应从 Windows 宿主机复制 Linux 不兼容的依赖目录。

## 17. Nginx：静态页面、SPA 回退和 API 代理

```nginx
# frontend/nginx.conf
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    location /api/ {
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Vue Router 使用 History 模式时，用户直接刷新 `/todos/42`，Nginx 要返回 `index.html`，再由前端路由匹配页面。`/api/` 则转发到 Compose 网络中的 `api:8000`。

这里 `proxy_pass http://api:8000;` 没有尾部路径斜杠，保留原始 `/api/...` 路径。若 FastAPI 路由前缀为 `/api/v1`，前端也应请求 `/api/v1/...`。代理与后端路径不一致会造成 404。

浏览器只需访问同源的 `/api/...`，无需知道 `api` 服务名；同源访问也简化开发与部署时的跨域配置。若反向代理终止 HTTPS，生产环境还要正确处理 TLS、可信代理头和公开域名。

## 18. Compose：用一份文件描述完整 TodoLab

项目目录可组织为：

```text
todolab/
├─ compose.yaml
├─ .env.example
├─ backend/
│  ├─ Dockerfile
│  ├─ .dockerignore
│  ├─ pyproject.toml
│  ├─ uv.lock
│  ├─ alembic.ini
│  ├─ alembic/versions/
│  └─ src/app/
└─ frontend/
   ├─ Dockerfile
   ├─ .dockerignore
   ├─ nginx.conf
   ├─ package.json
   ├─ pnpm-lock.yaml
   └─ src/
```

Compose 文件描述服务，而不是把所有进程塞进一个容器：

```yaml
# compose.yaml
name: todolab

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: todolab
      POSTGRES_USER: todolab
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10

  migrate:
    build: ./backend
    environment:
      DATABASE_URL: ${DATABASE_URL:?set DATABASE_URL}
    command: ["alembic", "upgrade", "head"]
    depends_on:
      db:
        condition: service_healthy
    restart: "no"

  api:
    build: ./backend
    environment:
      DATABASE_URL: ${DATABASE_URL:?set DATABASE_URL}
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
    depends_on:
      migrate:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3)"]
      interval: 5s
      timeout: 4s
      retries: 10

  web:
    build: ./frontend
    ports:
      - "127.0.0.1:8080:80"
    depends_on:
      api:
        condition: service_healthy

volumes:
  pgdata:
```

这里 `db` 没有 `ports`，只供 Compose 内部访问；浏览器访问 `http://127.0.0.1:8080`。`migrate` 复用后端构建配置，先完成数据库升级，然后 `api` 才启动；`web` 等 API 的健康检查通过后再启动。

`api` 健康检查假设后端已有 `/health` 路由。例子可写：

```python
@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

健康端点应快速、轻量。若它只返回进程存活，就不能证明数据库可用；要不要额外做依赖就绪检查，应根据服务实际需求决定。

## 19. Compose `.env` 与容器环境变量不是一回事

在 `compose.yaml` 旁边放本地 `.env` 供变量替换：

```dotenv
# .env.example；复制后在本机填写真实本地值
POSTGRES_PASSWORD=local-development-value
DATABASE_URL=postgresql+asyncpg://todolab:local-development-value@db:5432/todolab
```

Compose 读取 `.env` 用于替换 `${...}`。`.env` 中的变量不会自动进入所有容器；上面的 `environment:` 才明确把它传给指定服务。URL 中的密码若含 `@`、`:`、`/` 等字符，需要 URL 编码，或者由应用从独立字段安全地构造 URL。

`.env.example` 可提交，真实 `.env` 应忽略。`docker compose config` 会解析并展示配置，其中可能含密码；排错时不要把完整输出贴到公开日志。构建镜像若需要私有仓库令牌，应使用 Docker build secret，不把秘密放进 `ARG`、`ENV` 或镜像层。

## 20. Compose 启动顺序与“就绪”的区别

`depends_on` 默认只处理启动顺序，不保证依赖服务能处理请求。PostgreSQL 容器已运行，但数据库初始化可能尚未完成。

```text
db running
   ↓ healthcheck 通过
db healthy
   ↓ Alembic 成功退出
migrate completed
   ↓ API 健康检查通过
api healthy
   ↓
web 启动
```

`service_healthy` 需要定义 `healthcheck`；`service_completed_successfully` 适合一次性迁移任务。健康检查只能证明被检查的条件成立，不能替代自动重试、告警和完整的业务监控。

上面的迁移服务适合学习和简单部署。多实例生产发布通常由 CI/CD 或发布作业统一执行迁移；不要让每个 API 副本在启动时同时迁移数据库。

## 21. 常用 Compose 命令

在含 `compose.yaml` 的目录中：

```powershell
docker compose config --quiet
docker compose up --build -d
docker compose ps
docker compose logs --tail 100 api
docker compose logs -f api
docker compose exec db psql -U todolab -d todolab
docker compose exec api python -c "import app; print('import ok')"
docker compose down
```

```text
config --quiet  解析并验证配置，不打印可能含秘密的完整内容
up --build -d  构建镜像并后台启动服务
ps             查看服务和健康状态
logs           查看容器标准输出/错误
exec           在正在运行的容器内执行命令
down           停止并删除本次 Compose 创建的服务容器与网络
```

`down` 默认保留命名卷；`down --volumes` 会删除相关命名卷。`docker compose restart` 不会把刚改过的 Dockerfile 或 Compose 配置重新应用；需要重新构建和创建时使用 `up --build`。

## 22. 开发环境与交付镜像如何区分

开发时常需要热更新：

```text
前端：pnpm dev + Vite HMR
后端：uvicorn --reload
数据库：PostgreSQL 容器
```

交付时更适合固定产物：

```text
前端：Vite build → dist → Nginx 镜像
后端：锁定依赖 + 已构建的应用镜像
数据库：独立容器或托管数据库 + 数据卷/备份
```

不要把 `--reload` 当生产启动参数。bind mount 适合本地开发；发布环境应以可追踪镜像版本和迁移脚本为准。Docker Compose 能管理单机多服务，但它本身不提供跨机器调度、自动数据库备份或完整的生产观测能力。

## 23. 构建参数与运行环境变量

```text
ARG                  只在构建阶段可用，适合非秘密构建选项
Dockerfile ENV       写入镜像配置并传给容器
Compose environment 运行时传给指定服务
VITE_*               前端构建时进入浏览器 JS，用户可见
```

构建时 `VITE_API_BASE_URL=/api/v1` 是公开配置；生产数据库密码绝不能放进 `VITE_` 变量。前端镜像构建后修改运行容器的 `VITE_API_BASE_URL` 通常不起作用，除非项目另行实现运行时配置注入。

如果 Build 阶段要访问私有包仓库，使用 BuildKit secret 或 SSH mount，避免令牌留在镜像历史中。运行时秘密也应由部署平台的秘密管理能力提供，不应提交到仓库。

## 24. 日志、健康检查和退出状态

容器主进程的标准输出和错误可通过 `docker logs` 或 `docker compose logs` 观察。FastAPI 日志应包含请求 ID、必要业务上下文和异常原因，同时避免记录密码、Token、完整连接串和敏感请求体。

```powershell
docker compose ps
docker compose logs --tail 100 migrate
docker compose logs --tail 100 db
```

如果 `migrate` 退出码非 0，先看迁移日志，不应通过删除 `alembic_version`、删库或 `stamp head` 掩盖错误。健康检查失败时，要区分应用进程没启动、端口没监听、路由路径错、数据库不可用或健康检查命令缺少依赖。

## 25. 数据持久化、迁移与备份三件事

```text
命名卷   让数据库文件跨容器重建保留
Alembic  让表结构按版本变化
备份     让误删、损坏、灾难后的数据可以恢复
```

三者不能互相替代。备份策略至少要考虑保留周期、存放位置、访问权限和定期恢复验证。升级 PostgreSQL 大版本前，需要阅读镜像和 PostgreSQL 官方的升级说明；单纯把 `postgres:16` 改成下一个主版本，并继续挂同一数据目录，通常不等于完成数据库升级。

## 26. 一张表定位常见故障

| 表现 | 先看哪里 | 常见原因 |
| --- | --- | --- |
| `docker` 无法连接 Engine | Docker Desktop、context、`docker info` | Engine 未启动 |
| `up` 提示端口占用 | `ports` 宿主机端口 | 本机已有程序使用 8080/15432 |
| API 连接数据库失败 | API 日志、`DATABASE_URL`、db 健康状态 | 容器里误用 `localhost`、密码不一致 |
| 页面能打开但 API 502 | Nginx 日志、api 健康状态 | API 未就绪或代理地址错误 |
| 前端刷新详情页 404 | Nginx `try_files` | 没有 SPA History 回退 |
| API 404 | 浏览器 Network、Nginx、FastAPI prefix | `/api/v1` 路径不一致 |
| 修改代码后页面没变化 | 镜像构建和容器状态 | 未重建镜像，或浏览器缓存 |
| 容器重建后数据库为空 | 卷挂载与 Compose 项目名 | 使用了不同命名卷或未挂卷 |
| 迁移失败 | migrate 日志、版本状态 | 模型和迁移不一致或已有数据冲突 |
| Windows 文件热更新不稳定 | bind mount、WSL2 路径 | 文件监控与跨文件系统性能问题 |

排错时沿浏览器 → Nginx → API → PostgreSQL 的实际请求路径走。`docker compose ps` 看状态，`logs` 看错误，`exec` 在正确容器内验证，而不是同时修改所有服务配置。

## 27. 安全和维护边界

- 镜像来源和版本要可追踪；生产环境尽量固定经过验证的标签或 digest；
- `.dockerignore` 排除真实 `.env`、本地虚拟环境、依赖目录和无关文件；
- 数据库不需要对外提供宿主机端口时就不要发布；
- 后端容器无需特权模式或宿主机 Docker socket；
- 构建产物只包含运行所需文件，避免把测试数据和密钥复制进去；
- 按官方发布说明更新基础镜像并重新构建、测试；
- 运行时用户、文件权限、只读文件系统和资源限制按部署环境逐步收紧；
- 监控镜像、容器状态、数据库备份及恢复能力。

容器化能改善交付一致性，但不是“镜像一构建就安全”。

## 28. 如何指挥 AI 完成容器化

### 为现有项目生成 Docker 配置

```text
请先阅读现有 Vue、FastAPI、PostgreSQL 项目结构、package.json、pnpm-lock.yaml、pyproject.toml、uv.lock、Alembic 配置和环境变量说明。
按现有依赖管理方式创建前后端 Dockerfile、对应 .dockerignore、Nginx 配置与 compose.yaml。
前端多阶段构建：Node/pnpm 生成 dist，Nginx 托管；History 路由刷新必须回退到 index.html，/api 转发给 api 服务。
后端使用现有 uv 锁文件，运行时监听 0.0.0.0:8000；数据库用 PostgreSQL 16 命名卷。
Compose 使用 db 健康检查和一次性 Alembic 迁移服务；API 仅在迁移成功后启动。
真实密码只从本地环境或部署秘密管理获取，不复制进镜像，不发布不必要的数据库端口。
完成后验证 Compose 配置、构建、服务健康、接口请求、页面刷新和数据库重建后的数据保留，报告实际结果。
```

### 排查容器化接口故障

```text
请根据浏览器 Network、Nginx/API/数据库日志与 docker compose ps 定位故障。
区分宿主机端口与容器端口、Compose 服务名与 localhost、API 路由 prefix、健康检查、数据库迁移和环境变量。
先找出请求在哪一跳失败，给出日志或响应证据，再做最小修改。
不要通过删除数据卷、删库、关闭健康检查或把数据库端口公开到所有网卡来掩盖问题。
```

### 审核现有 Dockerfile 和 Compose

```text
请审查当前 Dockerfile、.dockerignore、Nginx 配置和 Compose 文件。
重点检查依赖是否锁定、构建缓存是否合理、是否复制了密钥、VITE_ 变量是否被误当秘密、数据库卷是否正确、迁移是否只执行一次、服务是否等待真正就绪。
检查是否有不必要的宿主机端口、特权模式、开发 reload 和容器内手工修改的状态。
按风险从高到低给出修改，保留项目现有工具链和部署方式。
```

## 29. 审核 AI 生成配置的清单

```text
[ ] 每个服务职责清楚，前端、API、数据库分别运行
[ ] Dockerfile 的 build context 与 COPY 路径一致
[ ] packageManager、pnpm-lock.yaml、uv.lock 被正确使用
[ ] 真实 .env、node_modules、.venv 未复制进镜像
[ ] 前端产物从构建阶段复制到 Nginx 运行阶段
[ ] VITE_ 中没有秘密，前端请求路径与 Nginx/FastAPI 一致
[ ] SPA 路由直接刷新不会 404
[ ] 后端监听 0.0.0.0，而非仅容器内 localhost
[ ] API 使用 db 服务名，不把 db 写成 localhost
[ ] PostgreSQL 数据目录挂了明确的命名卷
[ ] 数据库端口没有无必要地发布到宿主机
[ ] db、migrate、api 的就绪与依赖顺序有依据
[ ] Alembic 没有在每个 API 进程启动时并发运行
[ ] 迁移失败时 API 不假装正常启动
[ ] Compose 配置、构建和实际请求经过验证
[ ] 不会将 down --volumes 当作普通停止命令
```

## 30. 面试表达速记

### 镜像与容器

> 镜像是包含应用和依赖的只读模板，容器是镜像启动后的隔离进程。修改 Dockerfile 或源码后要重新构建镜像并创建新容器，不能依赖进入容器手工改文件。

### 容器与虚拟机

> 容器通常共享宿主 Linux 内核，隔离的是进程、文件和网络；虚拟机运行独立来宾操作系统。Windows Docker Desktop 运行 Linux 容器时仍有 Linux 虚拟化层。

### 数据卷

> 容器可写层随容器删除而消失，数据库数据要挂在卷或可靠外部存储。卷解决持久化，不提供备份；备份还要能定期恢复验证。

### Compose 网络

> 同一 Compose 项目的服务通常可用服务名互相访问。API 连接数据库用 `db:5432`，浏览器访问的是宿主机发布的地址；每个容器的 `localhost` 都指它自己。

### `EXPOSE` 与 `ports`

> `EXPOSE` 只是声明容器预期使用的端口，`ports` 才把容器端口发布到宿主机。同一 Compose 网络内的服务通信不需要发布宿主机端口。

### 多阶段构建

> 多阶段构建在前一阶段安装工具并生成产物，再把必要产物复制到较小的运行镜像。Vue 常用 Node 构建、Nginx 托管 dist；锁文件和 .dockerignore 有助于可复现性与构建效率。

### 启动顺序与健康检查

> `depends_on` 的默认启动顺序不代表服务已就绪。数据库可以用 healthcheck，迁移可以作为一次性服务，API 在迁移成功后启动；健康检查也不能代替运行期重试和监控。

### Compose 的边界

> Compose 适合描述和运行一组相关服务。它不是数据库备份系统，也不自动提供跨机器调度、完整生产监控和零停机发布流程。

## 31. 本课技术地图

```text
TodoLab 容器化
├─ 镜像构建
│  ├─ backend/Dockerfile → FastAPI 镜像
│  ├─ frontend/Dockerfile → Node 构建 → Nginx 镜像
│  ├─ 锁文件与构建缓存
│  └─ .dockerignore 与秘密边界
├─ Compose 运行
│  ├─ web → api → db
│  ├─ migrate → Alembic upgrade
│  ├─ service name / network / port mapping
│  └─ healthcheck / depends_on
└─ 数据与运维
   ├─ pgdata 命名卷
   ├─ 备份和恢复
   ├─ logs / ps / exec / inspect
   └─ 镜像更新与发布
```

## 32. 官方参考资料

- [Docker：镜像与容器](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
- [Docker：Compose 启动顺序与健康检查](https://docs.docker.com/compose/how-tos/startup-order/)
- [Docker：Compose 服务名与网络](https://docs.docker.com/compose/how-tos/networking/)
- [Docker：数据卷与持久化](https://docs.docker.com/get-started/docker-concepts/running-containers/persisting-container-data/)
- [Docker：构建最佳实践](https://docs.docker.com/build/building/best-practices/)
- [uv：在 Docker 中使用](https://docs.astral.sh/uv/guides/integration/docker/)

## 33. 后续学习路线与剩余节数

原技术大纲列出技术和建议顺序，没有规定课程节数。按“相关技术合并成一节”的方式，完成本课后计划还有 **6 节**：

| 下一节 | 合并主题 | 核心用途 |
| --- | --- | --- |
| 15 | Linux、终端与 VS Code | 在本机和服务器定位文件、进程、端口与日志 |
| 16 | Git、GitHub/Gitee 与 CI | 版本协作、PR、自动检查与发布历史 |
| 17 | uv、Uvicorn 与后端运行 | Python 依赖复现、ASGI 服务和部署边界 |
| 18 | 密码哈希、JWT 与权限 | 登录、令牌校验、RBAC 与安全边界 |
| 19 | pytest/httpx、Apifox 与 OpenAPI | 自动化测试、接口契约和联调 |
| 20 | Vue 开发工具、Element Plus 与 UnoCSS | 类型化开发、后台组件与样式组织 |

接下来先学 **Linux、终端与 VS Code**：Docker 容器和服务器多数运行 Linux，掌握文件、权限、进程、端口及日志排查，才能解释容器“运行了但不可用”的问题。

[进入下一课：Linux、终端与 VS Code →](./15-Linux终端与VS-Code开发环境.md)
