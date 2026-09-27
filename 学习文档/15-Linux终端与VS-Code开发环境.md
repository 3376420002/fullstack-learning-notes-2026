# 15｜Linux、终端与 VS Code 开发环境

> 所属阶段：开发环境与运维基础
>
> 本课合并知识点：Linux 文件系统、Shell、路径、权限、进程、端口、日志、SSH、环境变量和 VS Code 远程开发
>
> 前置知识：Docker 与 Compose、HTTP、FastAPI
>
> 学习目标：能看懂常见 Linux 命令，按“文件→进程→端口→日志→配置”定位服务故障，并能指挥 AI 安全地操作开发环境。

[← Docker 与 Compose](./14-Docker与Compose应用容器化.md) · [Git、代码协作与 CI →](./16-Git代码协作与持续集成.md)

## 阅读方式

本文继续用 TodoLab：Vue 前端、FastAPI API、PostgreSQL 数据库。命令只是教学示例；生产机器上的路径、用户名、服务名必须先核实。重点理解命令在检查什么，而不是死记参数。

## 1. 为什么全栈开发需要 Linux

本地是 Windows，部署的 Docker 容器和服务器却常是 Linux。服务部署后，最常遇到的不是“代码不会写”，而是：进程没启动、端口没监听、环境变量缺失、目录没权限、反向代理指向错误地址。Linux 基础让你能看懂这些现场证据。

```text
浏览器请求失败
  → 域名/网络
  → 服务器端口
  → Nginx / API 进程
  → 日志与环境变量
  → 数据库连接和文件权限
```

Shell 是输入命令的程序；终端是呈现输入输出的界面；Linux 是操作系统。PowerShell、Bash、zsh 的命令和引号规则不完全相同，复制命令前先看当前终端类型。

## 2. 文件系统与绝对、相对路径

Linux 只有一棵以 `/` 为根的目录树。常见目录：

| 路径 | 常见用途 | 备注 |
| --- | --- | --- |
| `/home/用户名` | 普通用户的个人目录 | `~` 通常指当前用户家目录 |
| `/etc` | 系统和服务配置 | 修改通常要管理员权限 |
| `/var/log` | 系统/服务日志 | 容器应用更多写标准输出 |
| `/tmp` | 临时文件 | 不保证长期保留 |
| `/usr` | 程序及共享资源 | 不要随手改系统文件 |
| `/opt` | 第三方软件或应用 | 具体部署方式不同 |
| `/proc` | 内核暴露的进程信息 | 虚拟文件系统 |

绝对路径从 `/` 开始；相对路径从当前工作目录开始。`./backend` 是当前目录下的 `backend`，`../backend` 是上一级目录下的 `backend`。同一条命令在不同工作目录执行可能得到不同结果。

```bash
pwd                 # 当前目录
ls -lah             # 列出当前目录；包含隐藏文件
cd /opt/todolab     # 切换目录
realpath compose.yaml
```

`/`、`~`、`./` 和 `../` 不是一回事。脚本中应尽量使用确定的路径，避免在错误目录执行危险命令。

## 3. 查找文件与阅读内容

```bash
find /opt/todolab -name 'compose.yaml' -type f
rg --files /opt/todolab | head
rg 'DATABASE_URL' /opt/todolab/backend
sed -n '1,80p' /opt/todolab/backend/pyproject.toml
tail -n 100 /var/log/nginx/error.log
```

`rg` 适合快速搜代码；`find` 适合按文件属性搜；`sed -n` 和 `tail` 适合截取内容。读生产配置时，输出可能含密钥，不要随意贴到群聊、AI 对话或公开 issue。

有空格的路径需要引号：`cd "/opt/todo lab"`。Linux 文件名大小写敏感，`App.py` 与 `app.py` 是两个文件。

## 4. 管道、重定向与退出码

```bash
rg 'ERROR' /var/log/todolab/app.log | tail -n 20
curl -i http://127.0.0.1:8000/health > health.txt
command_that_may_fail 2> error.log
echo $?
```

`|` 把前一条命令的标准输出交给下一条；`>` 覆盖文件；`>>` 追加；`2>` 重定向错误输出。退出码 `0` 通常表示成功，非零表示失败。自动化脚本不能只看屏幕有无文字，要看退出码。

管道可能让人只看到最后一条命令的退出状态。Bash 脚本中可按需启用 `set -euo pipefail`，但先理解脚本如何处理预期失败，不要机械加入每个脚本。

## 5. 文件权限：谁能读、写、执行

```text
-rwxr-x--- 1 deploy app  824 Sep 1 10:00 start.sh
│││││││││
│└┬┘└┬┘└┬┘
│ owner group others
└ 文件类型（- 普通文件，d 目录）
```

`r` 读、`w` 写、`x` 执行。目录的 `x` 意味着可进入/遍历，目录可读不一定能进入。上例文件所有者可读写执行，同组用户可读执行，其他用户无权限。

```bash
id
ls -ld /opt/todolab /opt/todolab/backend
stat /opt/todolab/backend/start.sh
chmod u+x /opt/todolab/backend/start.sh
```

权限应按最小权限原则设置。`chmod 777` 看似快速解决问题，却把所有用户的写权限也放开，通常是错误修复。`chown` 会改变所有者，必须先确定目标路径和运行用户；不要对未知目录递归操作。

## 6. 用户、`sudo` 与服务身份

Linux 服务通常不应该一直用 root 运行。`sudo` 让有权限的用户临时以更高权限执行命令，但它不是“命令失败就加上”的万能开关。

```bash
whoami
id
ps -o user,pid,cmd -C nginx
```

例如 Nginx 主进程可能以 root 启动以绑定低端口，再由低权限 worker 处理请求。一个应用若需要写目录，就检查目录所有者、组和容器用户，而不是先把整个项目改成 777。

## 7. 进程、PID、前台与后台

运行 `uvicorn app.main:app` 会产生进程。PID 是进程编号；进程退出，服务就停止。终端关掉后前台进程可能收到终止信号，因此长期运行服务通常由 systemd、容器平台或其他进程管理器托管。

```bash
ps aux | rg 'uvicorn|postgres'
pgrep -af uvicorn
top
```

`kill PID` 默认发送终止信号，允许程序清理；`kill -9` 强制结束，可能跳过清理，应作为最后手段。先确认 PID 的命令行和用户，避免误停其他服务。

## 8. 端口与监听地址

```bash
ss -lntp
ss -lntp | rg ':8000|:5432'
curl -i http://127.0.0.1:8000/health
```

`ss` 查看监听端口；`curl` 发真实 HTTP 请求。`127.0.0.1:8000` 只对本机可达；`0.0.0.0:8000` 表示进程监听所有 IPv4 接口，但外部是否能访问仍取决于防火墙、容器端口映射和网络路由。容器中的 `localhost` 是该容器自己。

排查顺序可以是：进程在不在 → 端口是否监听 → 本机请求是否成功 → 容器/代理是否转发 → 外部网络是否允许。

## 9. DNS、HTTP 与网络定位

```bash
getent hosts db
curl -I https://example.com
curl -v http://127.0.0.1:8000/health
```

`getent hosts db` 在 Compose 网络内部可验证服务名解析；宿主机上未必能解析 `db`。`curl -v` 可以看到连接、请求和响应细节。连接被拒绝通常与端口未监听相关；超时可能是网络或防火墙；收到 404 则说明已到达某个 HTTP 服务，接下来查路由/代理路径。

不要把“Ping 不通”直接等同于“HTTP 服务不可用”，ICMP 与 TCP/HTTP 是不同协议，环境可能禁用 ICMP。

## 10. 环境变量与配置生效范围

```bash
printenv | rg '^APP_'
APP_ENV=development uv run uvicorn app.main:app
```

环境变量属于进程及其子进程环境。你在一个终端设置的变量，不会自动进入已经运行的容器或 systemd 服务。`export DATABASE_URL=...` 会把变量传给该 Shell 启动的子进程，但输入命令可能进入历史记录；秘密应使用受控的配置/秘密管理机制。

容器中的环境变量可由 Compose `environment:` 指定；systemd 服务可由服务单元配置注入。改配置后通常要重建/重启对应进程才能生效。

## 11. 日志：先看时间线和错误上下文

```bash
tail -n 100 /var/log/nginx/error.log
journalctl -u todolab-api --since '30 minutes ago' --no-pager
docker compose logs --tail 100 api
```

三条命令分别对应传统文件日志、systemd 服务日志和容器标准输出。排错时先找“首次错误”和其前后的日志，不要只复制最后一行堆栈。注意时区、请求 ID、服务名、退出码和重启次数。

日志里不应记录明文密码、完整 Token、密钥和敏感请求体。发送给 AI 前先脱敏，保留能定位问题的状态码、路径、错误类型和时间。

## 12. systemd 与容器服务的区别

传统 Linux 主机可能用 systemd 管理 API：

```bash
systemctl status todolab-api
journalctl -u todolab-api -n 100 --no-pager
```

Docker/Compose 则由容器引擎管理：

```bash
docker compose ps
docker compose logs --tail 100 api
```

先查项目采用哪种部署方式，再选命令。不要在容器里尝试 `systemctl` 来管理由 Compose 启动的 API。修改 systemd 单元文件后通常需 `daemon-reload`，修改应用配置仍需按服务流程重启。

## 13. SSH：远程连接不是“复制命令到服务器”

```bash
ssh deploy@example.com
ssh -i /path/to/key deploy@example.com
```

SSH 用于加密远程登录；密钥认证的私钥必须保密并设置合理权限。首次连接会看到主机指纹，应通过可信渠道核对，不要无条件接受；服务器主机密钥突然变化可能是重装，也可能意味着连接对象有问题。

`ssh` 连接成功后，终端命令是在远端执行。尤其是编辑文件、运行迁移、删除容器前，要确认当前主机、项目目录和环境，避免把开发操作落到生产。

## 14. VS Code 的本地与远程工作区

VS Code 可以在本地打开项目，也可通过 Remote - SSH 或 Dev Containers 在远端/容器中开发。关键是分清“编辑器界面在哪”和“终端、语言服务、文件实际在哪”：

```text
本地窗口             → 终端和文件在本机
Remote - SSH 窗口    → 终端和工作区文件在远端
Dev Container 窗口   → 终端和工具链在开发容器
```

打开工作区后先看左下角远程标识、集成终端 `pwd` 和 Python/Node 解释器路径。插件可能分为本地 UI 插件与远端工作区插件；智能提示不工作时检查插件装在了哪侧。

## 15. VS Code 里最有用的开发能力

- 文件搜索定位接口、组件和配置；
- 全局搜索和引用查找理解调用链；
- 终端运行 `pnpm`、`uv`、`docker compose`；
- 调试器设置断点、查看变量和调用栈；
- Git 视图检查改动、差异和提交；
- Problems 面板集中查看类型与语法错误。

Vue 3 建议使用 Vue - Official 扩展；Python 开发要让 VS Code 选择项目 `.venv` 的解释器。保存时格式化和类型检查可以自动化，但仍需看清它修改了哪些文件。

## 16. 开发容器与运行容器不是同一概念

Dev Container 主要为“开发者在一致工具环境中写代码”服务；第 14 课的后端/前端镜像主要为“交付应用”服务。开发容器可装调试器、Git、测试工具，并挂载源码；生产镜像应尽量精简且版本固定。不要把含个人 SSH 密钥和编辑器缓存的开发容器直接当生产镜像。

## 17. Shell 脚本基础

```bash
#!/usr/bin/env bash
set -euo pipefail

project_dir=/opt/todolab
cd "$project_dir"
docker compose config --quiet
docker compose ps
```

第一行指定解释器；变量引用通常写 `"$project_dir"`，避免路径中的空格被拆开。`set -euo pipefail` 帮助尽早发现意外失败，但遇到 `rg` 未匹配（返回非零）等“预期失败”时要显式处理。

脚本不要通过拼接未经验证的输入构造删除命令或远程命令。要改变生产状态，应有明确目标、备份与回滚路径。

## 18. 软件安装与版本

不同发行版包管理器不同：Debian/Ubuntu 常用 `apt`，Fedora 常用 `dnf`，Alpine 常用 `apk`。容器镜像往往很精简，缺少 `curl`、`bash`、`ps` 并不代表应用错误；不要为了排错直接在正在运行的生产容器里永久安装工具。

查环境时先看 `cat /etc/os-release`、`command -v python`、`python --version`。项目依赖由 `uv.lock`/`pnpm-lock.yaml` 管理，不应靠“机器上碰巧安装了某版本”。

## 19. 一次具体排错：API 容器运行却访问不到

假设浏览器访问 `/api/v1/todos` 返回 502：

1. `docker compose ps`：确认 `api` 是否 running/healthy；
2. `docker compose logs --tail 100 api`：看启动异常和数据库错误；
3. 在 `web` 容器检查 `api` 服务名和 `api:8000` 是否可达；
4. 检查 API 是否监听 `0.0.0.0:8000`，而非只监听容器内的 `127.0.0.1`；
5. 看 Nginx `proxy_pass` 与 FastAPI 路由前缀是否一致；
6. 若请求已到 API 再查数据库连接串中的主机是否为 `db`。

502 是代理无法从上游拿到有效响应；与 API 返回 404/500 的排查起点不同。先保存现场日志再改配置。

## 20. Windows PowerShell 与 Bash 的常见差异

| 目的 | Bash/Linux | PowerShell |
| --- | --- | --- |
| 当前目录 | `pwd` | `Get-Location` |
| 列文件 | `ls -lah` | `Get-ChildItem -Force` |
| 搜文本 | `rg pattern .` | `rg pattern .`（安装后） |
| 环境变量 | `$DATABASE_URL` | `$env:DATABASE_URL` |
| 路径分隔 | `/` | 通常 `\`，很多工具也接受 `/` |

`rm`、`curl` 等名称在 PowerShell 中可能有别名或不同语义；教程标记为 Bash 的命令应在 Bash/WSL/远端 Linux 终端中执行。不要在 Windows PowerShell 里不经判断地粘贴 `export`、`sudo`。

## 21. 安全操作边界

- 先用 `pwd`、`ls`、`git status`、`docker compose ps` 确认目标；
- 数据库文件和日志先备份或保留证据，不把“删了重来”作为默认修复；
- 远程机器上先确认环境标识，尤其是生产/测试同名目录；
- 只提升所需权限，不用 `chmod 777` 或广域 `sudo` 掩盖配置问题；
- 不把真实 `.env`、私钥、Token 放进 AI 提示词或提交到仓库。

## 22. 可以交给 AI 的排错指令

```text
你是我的 Linux/容器排错助手。当前项目是 Vue + FastAPI + PostgreSQL，
部署方式是 Docker Compose。现象：浏览器调用 /api/v1/todos 返回 502。
请先给只读诊断顺序，说明每条命令要验证的假设和可能输出；
区分宿主机、web 容器、api 容器、db 容器中的 localhost；
不要删除容器、数据卷、数据库或改权限。等我提供脱敏日志后再提出修复。
```

```text
请检查我贴出的 Bash/PowerShell 命令是否适用于当前系统，
标出会写入、覆盖、删除或暴露秘密的步骤；
若需要修改配置，请给出最小改动、验证方法和回滚方法。
```

让 AI 写命令时，要求它注明执行位置、权限要求、影响范围和预期结果，这比“给我一条能修好的命令”更可靠。

## 23. 面试表达速记

> Linux 排错我会先确认服务部署方式，再按进程、监听端口、本机请求、代理链路、应用日志和依赖服务逐层定位。502 常看代理到上游的连接，500 则先看后端异常日志。

> 文件权限分所有者、用户组和其他用户的读写执行权限。目录的执行权限代表可进入或遍历；不应使用 777 作为通用修复。

> 容器里的 localhost 指容器自身。Compose 服务之间应使用服务名，比如 API 访问数据库用 db:5432；宿主机访问则用发布端口。

> VS Code Remote - SSH/Dev Containers 让编辑器连接远端环境，但文件、解释器和终端可能在远端，必须确认当前工作区和环境后再执行命令。

## 24. 技术地图与官方资料

```text
Linux 排障
├─ 文件：pwd / ls / rg / 权限
├─ 进程：ps / systemctl / compose ps
├─ 网络：ss / getent / curl
├─ 日志：tail / journalctl / compose logs
└─ 环境：变量 / 配置 / 解释器 / 容器边界
```

- [GNU Bash 手册](https://www.gnu.org/software/bash/manual/)
- [Ubuntu Server 文档](https://documentation.ubuntu.com/server/)
- [VS Code：Remote - SSH](https://code.visualstudio.com/docs/remote/ssh)
- [VS Code：开发容器](https://code.visualstudio.com/docs/devcontainers/containers)

## 25. 下一步

下一课学习 Git、GitHub/Gitee 与 CI：把本地修改转成可审查的提交与合并请求，并让检查在每次提交后自动运行。

[进入下一课：Git、代码协作与 CI →](./16-Git代码协作与持续集成.md)
