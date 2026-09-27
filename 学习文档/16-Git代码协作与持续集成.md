# 16｜Git、GitHub/Gitee 与持续集成

> 所属阶段：团队协作与工程质量
>
> 本课合并知识点：Git 三个区域、提交、分支、合并、冲突、远程仓库、PR、代码评审和 CI
>
> 前置知识：Linux/终端、项目工程化、基本测试
>
> 学习目标：能把 AI 产生的代码变成可审查、可回退的提交；理解团队 PR 流程与 CI 的边界。

[← Linux、终端与 VS Code](./15-Linux终端与VS-Code开发环境.md) · [uv、Uvicorn 与后端运行 →](./17-uv与Uvicorn后端运行.md)

## 1. Git、GitHub 和 Gitee 分别是什么

Git 是本地版本控制工具；GitHub/Gitee 是托管 Git 仓库并提供 PR、Issue、权限和自动化能力的平台。没有网络时 Git 仍可查看历史、创建分支和提交。远程平台不是 Git 的“保存按钮”，本地提交也不会自动出现在远程。

```text
工作区 → 暂存区 → 本地提交 → 远程仓库
         git add   git commit    git push
```

TodoLab 修改一个 API 时，建议先读差异、只暂存相关文件，再提交；不把日志、`.env`、数据库文件或 AI 临时产物一起推送。

## 2. 工作区、暂存区、提交与 HEAD

工作区是磁盘上正在编辑的文件；暂存区（index）是“下一次提交准备包含的内容”；提交是一次带父节点的快照。`HEAD` 通常指当前分支最新提交。

```bash
git status --short
git diff                  # 未暂存的改动
git diff --staged         # 已暂存、将进入下次提交的改动
git log --oneline -5
```

`git add` 不等于提交，`git commit` 不等于推送。提交后的内容仍可在本地分支；`git push` 才更新对应远程分支。提交对象有哈希标识，可以追溯具体版本。

## 3. 一次安全的最小提交

```bash
git status --short
git diff -- backend/src/app/api/todos.py
git add -- backend/src/app/api/todos.py
git diff --staged --check
git diff --staged
git commit -m "fix: validate todo title before saving"
```

先读差异，再用明确文件路径暂存。`git add .` 会把当前目录下所有符合规则的变化一并暂存，方便但容易夹带无关修改。多人的共享工作区尤其不能假定未识别文件可以随意提交或丢弃。

提交信息用“类型 + 意图”便于审查，格式是否采用 Conventional Commits 以团队规范为准。一个提交最好表达一件可以理解的变化，不一定只改一个文件。

## 4. `.gitignore` 与“已经被 Git 跟踪”

`.gitignore` 只影响未跟踪文件的默认忽略行为；已提交过的秘密或缓存不会因新增忽略规则而从历史消失。典型忽略项包括 `.env`、`.venv/`、`node_modules/`、`dist/`、日志和本机编辑器缓存。`uv.lock`、`pnpm-lock.yaml` 通常需要提交，以复现依赖版本。

```bash
git status --ignored --short
git check-ignore -v backend/.env
git ls-files backend/.env
```

若真实密钥曾进入仓库，首先轮换/撤销密钥；删除工作区文件或追加 `.gitignore` 不能撤销已泄漏的凭据。

## 5. 分支是什么

分支本质上是指向某个提交的可移动引用。一个功能分支让你在不直接改 `main` 的情况下完成修改与评审。

```bash
git switch main
git pull --ff-only
git switch -c feature/todo-filter
git branch --show-current
```

分支名最好表达工作目的。`main` 是否受保护、是否允许直接推送由仓库规则决定；团队通常要求 PR、CI 通过和评审后合并。

## 6. `fetch`、`pull`、`push` 的区别

- `git fetch`：下载远程引用和对象，不直接改当前工作区；
- `git pull`：通常相当于 fetch 后再合并或变基，具体行为受配置影响；
- `git push`：把本地分支提交发往远程；
- `git pull --ff-only`：只在可快进时更新，避免意外自动合并提交。

```bash
git remote -v
git fetch origin
git log --oneline --left-right main...origin/main
```

“远程有变化”不等于“本地文件已更新”。先 fetch 和看差异，再决定合并方式。

## 7. 合并、变基与冲突

`merge` 把两个分支的历史汇合；`rebase` 把当前分支的提交重新应用在另一基线之上。两者都可能产生冲突。共享分支上已有他人依赖的提交时，不要未经协调地变基并强推，因为提交哈希会变化。

```text
main:     A ─ B ─ C
feature:      └─ D ─ E

merge：保留分叉历史，可能产生 M
rebase：把 D/E 重新应用在 C 后，得到新提交 D'/E'
```

冲突不是 Git 丢失代码，而是两边修改无法自动决定取舍。处理时理解两边意图，编辑文件并运行测试；别机械地“全部接受当前/传入”。

```bash
git status
git diff --check
```

如果正在合并或变基，先看 `git status` 给出的下一步；中止命令会撤销该操作过程中的合并/变基状态，使用前确认当前未提交改动如何保留。

## 8. 回退、恢复与危险操作的边界

不同需求对应不同操作：

| 目标 | 常用方式 | 风险点 |
| --- | --- | --- |
| 撤销未暂存的某个文件改动 | `git restore -- path` | 会丢失该文件未提交修改 |
| 从暂存区移出但保留工作区改动 | `git restore --staged -- path` | 不删除工作区内容 |
| 为已推送提交做反向提交 | `git revert COMMIT` | 保留历史，适合共享分支 |
| 改写本地提交历史 | rebase/reset | 已共享时要谨慎 |

遇到用户或同事留下的改动，不先执行 `restore`、`reset --hard`、`clean -fd`。先看 `status`、`diff`，必要时备份和确认所有权。`git revert` 也可能产生冲突，仍需验证。

## 9. GitHub/Gitee 上的 PR 工作流

通常过程：

```text
需求/Issue
 → 从最新 main 创建功能分支
 → 小步提交并本地自检
 → push 到远程
 → 创建 Pull Request（Gitee 常称 Pull Request）
 → CI 与代码评审
 → 根据反馈修改
 → 合并 main
```

PR 描述至少交代：改了什么、为什么改、怎么验证、风险和回滚方式。截图或接口响应可帮助评审前端/接口改动，但需脱敏。评审关注正确性、边界、可维护性和安全，不只看格式。

## 10. 分支保护与评审责任

受保护分支可要求 CI 检查通过、指定人数评审、禁止直接推送等。具体能力与平台、仓库配置相关。CI 通过并不等于业务正确；评审通过也不代表不需测试。自动化承担重复性检查，人负责需求理解和风险判断。

不要让 AI 擅自合并 PR、覆盖别人的分支或跳过受保护分支策略。AI 生成代码应由提交者对结果负责。

## 11. CI 是什么，CD 又是什么

CI（持续集成）在提交/PR 后自动安装依赖、检查格式、做类型检查和运行测试，尽早发现“只在我电脑能跑”的问题。CD 可指持续交付/持续部署，进一步构建产物和发布环境；具体含义以团队流程为准。

```text
push / PR
  → checkout 源码
  → 安装锁定依赖
  → lint / typecheck / tests
  → 构建产物
  → （人工批准/自动）部署
```

构建、测试和部署最好分清权限边界。PR 来自不受信任的代码时，测试作业不要拥有生产秘密或写仓库的高权限 Token。

## 12. GitHub Actions 的核心结构

工作流放在 `.github/workflows/*.yml`；`on` 决定触发事件，`jobs` 定义作业，`steps` 定义步骤。每个作业在一个 runner 环境中执行，作业间文件不会天然共享。

下面是结构示意，Action 版本应按项目审查并固定为已验证版本/提交 SHA；路径和脚本名也要按项目实际调整：

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    steps:
      - uses: actions/checkout@<verified-version-or-commit>
      - uses: astral-sh/setup-uv@<verified-version-or-commit>
      - run: uv sync --locked --dev
      - run: uv run pytest

  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@<verified-version-or-commit>
      - uses: pnpm/action-setup@<verified-version-or-commit>
      - uses: actions/setup-node@<verified-version-or-commit>
        with:
          node-version-file: frontend/.node-version
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm build
```

这个片段**不能原样执行**：占位版本、`.node-version`、`typecheck` 脚本等都要按仓库确认。它表达的顺序是：受限权限 → 按锁文件安装 → 检查与构建。数据库集成测试还需声明数据库服务或专用测试环境。

## 13. CI 常见失败怎么区分

| 现象 | 先看什么 |
| --- | --- |
| 本地通过、CI 找不到包 | 锁文件是否提交；安装命令是否 frozen/locked |
| CI 找不到文件 | Linux 大小写敏感；工作目录与构建上下文 |
| 数据库连接失败 | CI 是否启动 PostgreSQL；连接串主机/端口 |
| 只有某一步失败 | 查看该步骤实际命令、退出码、前一段日志 |
| PR 通过、部署失败 | 环境变量、权限、迁移、镜像/运行环境差异 |

先读第一处有效错误，而不是只看最后一句 “Process completed with exit code 1”。CI 的红灯是证据，不等于一定要改应用代码。

## 14. Actions 的权限与供应链安全

第三方 Action 本质是会在 runner 上执行的代码。使用前检查维护者、权限、版本和输入；正式流程尽量固定可信版本或完整提交 SHA。默认 `GITHUB_TOKEN` 只赋予所需权限。不要把生产密钥放入普通 PR 测试流程；更不要在高权限 `pull_request_target` 上检出并执行不受信任 PR 的代码。

部署凭据应放在平台 Secrets/受保护环境，并限制谁能触发部署。日志中不要输出秘密。依赖升级和 Action 更新要经过评审，不应让 AI 自动替换成“最新”。

## 15. GitHub 与 Gitee 的共同点和差异

两者都可托管 Git 仓库、管理分支与合并请求；账户、权限、网页入口和 CI 产品各有差别。Git 命令对远程协议和仓库内容工作，不依赖平台界面。若同时维护两个远程，明确哪个是主仓库、谁负责同步和冲突处理。

```bash
git remote -v
git remote add mirror <approved-gitee-url>
```

示例只是展示命令，别将私有仓库推送到另一个平台，除非组织政策与用户授权允许。

## 16. 团队协作里 AI 应扮演什么角色

AI 适合解释 diff、生成小范围实现、补测试、整理 PR 描述、分析 CI 日志；不适合未经核查地覆盖他人改动、把秘密贴到外部服务、自动合并高风险变更。要求 AI 先列出将修改的文件，再按现有架构落地，最后报告实际验证结果与未验证部分。

可直接使用：

```text
请基于当前 TodoLab 仓库实现“按完成状态筛选待办”。
先检查工作区状态、现有路由/接口/类型和测试；只修改该功能相关文件。
保持现有 API 契约，必要时补前后端测试。
完成后给出逐文件变更摘要、实际运行的检查及结果、未验证风险。
不要提交、推送、合并或处理无关的用户改动。
```

```text
这是 CI 失败日志（已脱敏）。请识别第一个真正失败的步骤与错误，
区分依赖、路径大小写、环境配置、测试断言和应用代码问题。
先给只读检查，再给最小修复建议，不要建议跳过测试或扩大 Token 权限。
```

## 17. 面试表达速记

> Git 的工作区、暂存区和提交分别是正在编辑的文件、下一次提交的内容和已记录的快照。我会先看 `git diff`，只暂存相关文件，再看 `git diff --staged` 并提交。

> 合并保留分叉历史；变基把提交重新应用到新基线，会改变提交哈希。公共分支不应随意变基强推，冲突需要理解两边改动并重新测试。

> CI 在 PR/推送时自动执行依赖安装、静态检查、测试和构建；它不能替代需求评审与生产验证。工作流权限应最小化，不能让不可信 PR 获得部署秘密。

> 已泄漏的秘密需要轮换，单靠 `.gitignore` 或删除当前文件无法从 Git 历史和已克隆副本中撤回。

## 18. 技术地图与官方资料

```text
本地 Git
  ├─ 工作区 → 暂存区 → commit
  ├─ branch / merge / rebase
  └─ diff / log / revert
        ↓ push
远程平台
  ├─ PR / Review / Branch Protection
  └─ CI：lint → tests → build
```

- [Pro Git 免费在线书](https://git-scm.com/book/en/v2)
- [GitHub Actions 工作流语法](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [GitHub Actions Token 权限](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token)
- [uv 在 GitHub Actions 中的使用](https://docs.astral.sh/uv/guides/integration/github/)

## 19. 下一步

下一课专门讲 uv 与 Uvicorn：Python 依赖、锁文件、虚拟环境、ASGI 启动、开发热更新和部署运行是如何接起来的。

[进入下一课：uv、Uvicorn 与后端运行 →](./17-uv与Uvicorn后端运行.md)
