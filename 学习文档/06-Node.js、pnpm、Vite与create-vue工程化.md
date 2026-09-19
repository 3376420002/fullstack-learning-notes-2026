# 06｜Node.js、pnpm、Vite 与 create-vue 工程化

> 所属阶段：Vue 前端工程底座  
> 本课合并知识点：Node.js、pnpm、package.json、Vite、环境变量、开发代理、create-vue、构建与排错  
> 前置知识：HTML、CSS、TypeScript  
> 本课目标：能够创建、运行、理解和排查一个标准的 Vue 3 + TypeScript 前端工程。

[← TypeScript](./05-TypeScript基础与类型系统.md) · [学习首页](../README.md)

## 1. 四项技术分别解决什么问题

```text
Node.js      运行前端工程工具
pnpm         安装、锁定和执行项目依赖
Vite         提供开发服务器与生产构建
create-vue   按官方推荐配置生成 Vue 工程
```

它们的协作关系：

```text
create-vue 生成项目文件
        ↓
pnpm 根据 package.json 安装依赖
        ↓
Node.js 运行 Vite、TypeScript、ESLint 等工具
        ↓
Vite 启动开发服务器或构建 dist
        ↓
浏览器运行生成的 JavaScript、CSS 与 HTML
```

Node.js 在这里主要运行**开发工具**。Vue 应用的客户端代码最终仍在浏览器中运行，不能因为工程使用 Node.js 就在浏览器源码中随意调用文件系统等 Node API。

## 2. 版本策略

课程大纲以 Node.js 22 LTS 为基线。实际项目应遵守三个原则：

1. 使用仍受支持的 LTS 版本；
2. 满足当前项目 package.json 中的 engines 要求；
3. 团队与 CI 使用一致的主版本。

检查版本：

```powershell
node --version
pnpm --version
```

不要只写“使用最新版”。工具版本会变化，项目必须有可复现的版本约束和 lockfile。

## 3. Node.js 是什么

Node.js 是 JavaScript 运行环境。在前端工程中，它负责执行：

- Vite 开发服务器；
- TypeScript 类型检查器；
- ESLint、Prettier；
- 测试工具；
- 构建脚本；
- 依赖安装过程中需要执行的工具代码。

浏览器与 Node.js 的运行环境不同：

| 能力 | 浏览器 | Node.js |
| --- | --- | --- |
| DOM | 有 | 默认没有 |
| `window` | 有 | 没有 |
| 文件系统 | 受限 | 可通过 Node API 访问 |
| 页面渲染 | 有 | 默认没有 |
| npm 生态工具 | 不直接运行 | 可以运行 |

例如下面的代码不能直接放进浏览器 Vue 组件：

```ts
import { readFile } from 'node:fs/promises'
```

这是 Node.js API，不是浏览器 API。

## 4. 安装和切换 Node.js 的原则

建议使用 Node 版本管理器，而不是长期依赖一次性的全局安装。版本管理器的具体选择因操作系统而异，但目标一致：

- 能安装多个 Node 主版本；
- 能为项目选择版本；
- 能快速复现团队环境；
- 升级时不破坏其他项目。

项目可以通过 `.nvmrc` 或其他版本文件记录主版本：

```text
22
```

package.json 也可以声明：

```json
{
  "engines": {
    "node": ">=22"
  }
}
```

注意：engines 通常只是声明约束，是否强制拒绝安装取决于包管理器和项目配置。

## 5. pnpm 是什么

pnpm 是 Node.js 包管理器，主要负责：

- 读取 package.json；
- 安装 dependencies 和 devDependencies；
- 生成并使用 `pnpm-lock.yaml`；
- 执行 package.json scripts；
- 管理本机内容寻址存储；
- 为项目建立严格的依赖链接。

与复制每个包的完整文件相比，pnpm 会复用本机存储中的包内容，通常更节省磁盘空间。

pnpm 的严格依赖结构还能帮助发现“代码使用了未直接声明的包”这类问题。

## 6. Corepack 与 pnpm

在包含 Corepack 的 Node 环境中，可以使用：

```powershell
corepack enable
pnpm --version
```

不同 Node 发行版本对 Corepack 的捆绑策略可能不同。如果找不到 corepack，应按照 pnpm 官方安装方式安装，而不是从陌生来源复制脚本。

团队项目应记录 pnpm 版本，并让本地与 CI 使用一致版本。不要让每位成员长期使用完全不同的主版本。

## 7. package.json

package.json 是 Node 工程的核心清单：

```json
{
  "name": "todo-web",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build",
    "preview": "vite preview",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "vue": "^3.5.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "vite": "^6.0.0"
  },
  "engines": {
    "node": ">=22"
  }
}
```

上面的版本仅用于解释字段，不应复制后强行覆盖脚手架生成的实际兼容版本。

常见字段：

| 字段 | 作用 |
| --- | --- |
| `name` | 包或项目名称 |
| `private` | 防止应用项目被误发布到包仓库 |
| `type` | `module` 表示 `.js` 默认使用 ESM |
| `scripts` | 项目命令入口 |
| `dependencies` | 应用运行所需依赖 |
| `devDependencies` | 开发、检查、测试、构建工具 |
| `engines` | 声明运行环境要求 |

不要手工删除 package.json 中不理解的配置。先确认是哪个工具生成、谁在读取它。

## 8. dependencies 与 devDependencies

运行依赖：

```powershell
pnpm add vue
pnpm add axios
```

开发依赖：

```powershell
pnpm add -D typescript
pnpm add -D eslint prettier
```

判断方法：

- 应用运行代码会直接 import：通常放 dependencies；
- 只用于开发、检查、测试或构建：通常放 devDependencies。

前端应用构建后不会把整个 node_modules 部署到浏览器。打包器会分析入口和 import，把实际需要的客户端代码写入构建产物。

## 9. 常用 pnpm 命令

```powershell
# 安装 package.json 中声明的依赖
pnpm install

# 安装运行依赖
pnpm add axios

# 安装开发依赖
pnpm add -D eslint

# 删除依赖
pnpm remove axios

# 执行 scripts 中的 dev
pnpm dev

# 执行本地依赖提供的命令
pnpm exec tsc --noEmit

# 查看直接依赖
pnpm list --depth 0

# 查询某个包为什么被安装
pnpm why vite

# 检查可更新依赖
pnpm outdated
```

不要把“全部依赖升级到最新”当作日常修复手段。升级应阅读变更、分批进行并运行测试与构建。

## 10. pnpm-lock.yaml

package.json 描述允许的版本范围，`pnpm-lock.yaml` 记录实际解析出的依赖图和版本。

应用项目通常应该提交 lockfile：

```text
package.json       声明希望安装什么
pnpm-lock.yaml     记录实际安装什么
```

团队和 CI 使用 lockfile 可以减少“我的电脑能运行，你的电脑不能运行”。

CI 常用严格安装：

```powershell
pnpm install --frozen-lockfile
```

如果 package.json 和 lockfile 不一致，该命令会失败，而不是悄悄修改依赖图。

不要手工编辑 lockfile，也不要因为冲突就直接删除它。依赖变更应通过 pnpm 命令产生。

## 11. 语义化版本范围

版本通常采用：

```text
主版本.次版本.修订版本
major.minor.patch
```

例如 `3.5.12`：

- major：可能包含不兼容变化；
- minor：通常增加向后兼容功能；
- patch：通常包含兼容修复。

常见范围：

```text
3.5.12   精确版本
^3.5.12  允许兼容的次版本和修订版本变化
~3.5.12  通常只允许修订版本变化
```

版本范围只是包作者和生态遵守的约定，不是绝对安全保证。lockfile、测试和构建仍然必不可少。

## 12. scripts 是统一命令入口

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "type-check": "vue-tsc --build",
    "lint": "eslint .",
    "format": "prettier --write src/"
  }
}
```

执行：

```powershell
pnpm dev
pnpm build
pnpm type-check
pnpm lint
```

scripts 的价值：

- 团队不需要记住长命令；
- 本地与 CI 使用相同入口；
- 依赖命令自动从当前项目解析；
- 可以在不改变调用方式的情况下调整底层工具。

运行前应先阅读项目自己的 scripts，因为不同脚手架版本生成的脚本名称可能不同。

## 13. ESM 与 CommonJS

现代前端工程主要使用 ESM：

```ts
import { createApp } from 'vue'
export function createTodo() {}
```

传统 CommonJS：

```js
const moduleValue = require('./module')
module.exports = moduleValue
```

`"type": "module"` 会影响 Node.js 如何解释 `.js` 文件。现代 Vite 配置通常使用 ESM，不要在同一个配置文件中随意混用 `import` 和 `require`。

第三方依赖的模块格式兼容问题应先查看工具文档和报错上下文，不要盲目修改 package.json 的 type。

## 14. Vite 是什么

Vite 主要提供两个阶段：

```text
开发阶段：快速启动开发服务器、按需转换模块、热更新
构建阶段：分析依赖、打包、压缩、拆分并生成静态产物
```

开发：

```powershell
pnpm dev
```

构建：

```powershell
pnpm build
```

本地预览构建结果：

```powershell
pnpm preview
```

`vite preview` 只用于本地检查构建产物，不等同于生产级托管方案。

## 15. 开发服务器与热更新

Vite 开发服务器通常会输出类似地址：

```text
Local: http://localhost:5173/
```

修改源文件后，Vite 会进行热更新：

- 只更新受影响的模块；
- 尽量保留当前应用状态；
- 显示编译或转换错误；
- 不需要每次手工刷新整个页面。

开发服务器提供的是开发体验，不应该直接暴露为正式生产服务。

## 16. Vite 项目的 HTML 入口

Vite 把项目根目录的 `index.html` 视为入口之一：

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Todo Web</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

加载链路：

```text
index.html
└─ src/main.ts
   └─ App.vue
      └─ 其他组件与样式
```

不要把所有页面内容继续堆在 index.html；Vue 应用内容从 `#app` 挂载点进入。

## 17. create-vue

create-vue 是 Vue 官方脚手架，用于生成标准的 Vue + Vite 工程。

```powershell
pnpm create vue@latest
```

典型交互选项包括：

- 项目名称；
- TypeScript；
- JSX；
- Vue Router；
- Pinia；
- 单元测试；
- 端到端测试；
- ESLint；
- Prettier。

本课程的推荐学习配置：

| 选项 | 建议 | 原因 |
| --- | --- | --- |
| TypeScript | 是 | 本课程使用严格类型 |
| JSX | 否 | 先学习标准 Vue Template |
| Vue Router | 是 | 后续学习页面路由 |
| Pinia | 是 | 后续学习全局状态 |
| 单元测试 | 是 | 建立可测试工程底座 |
| 端到端测试 | 初学阶段可暂缓 | 后续完整项目再加入 |
| ESLint | 是 | 检查代码问题 |
| Prettier | 是 | 统一格式 |

选择会随脚手架版本调整。创建时应阅读每个问题，不要只机械复制按键顺序。

## 18. 创建项目的完整流程

```powershell
# 1. 确认环境
node --version
pnpm --version

# 2. 运行官方脚手架
pnpm create vue@latest

# 3. 进入脚手架生成的目录
Set-Location .\todo-web

# 4. 安装依赖
pnpm install

# 5. 启动开发服务器
pnpm dev
```

然后在浏览器打开终端显示的本地地址。

首次创建后应立即验证：

```powershell
pnpm type-check
pnpm lint
pnpm build
```

具体脚本以生成的 package.json 为准。

## 19. 标准目录结构

```text
todo-web/
├─ public/                 原样复制的公共静态文件
├─ src/
│  ├─ api/                 HTTP 请求封装
│  ├─ assets/              会被构建工具处理的资源
│  ├─ components/          可复用组件
│  ├─ composables/         组合式逻辑
│  ├─ router/              Vue Router
│  ├─ stores/              Pinia Store
│  ├─ types/               共享 TypeScript 类型
│  ├─ views/               路由页面
│  ├─ App.vue              根组件
│  └─ main.ts              应用入口
├─ .env.example            可提交的变量示例
├─ .gitignore
├─ index.html
├─ package.json
├─ pnpm-lock.yaml
├─ tsconfig.json
├─ tsconfig.app.json
└─ vite.config.ts
```

目录不是越多越专业。只有出现对应职责时再创建文件夹，并保持同类代码位置一致。

## 20. public 与 src/assets

`public/` 中的资源：

- 通常按原文件名复制；
- 通过根路径引用；
- 不经过模块 import 和常规资源分析；
- 适合必须保留固定文件名的内容。

```html
<img src="/favicon.svg" alt="" />
```

`src/assets/` 中的资源：

- 通过 import 或组件模板引用；
- 会参与构建处理；
- 文件名可能带内容哈希；
- 更适合组件实际使用的图片和字体。

```ts
import todoIllustration from '@/assets/todo-illustration.png'
```

不要把所有图片无差别放入 public。

## 21. vite.config.ts

最小配置：

```ts
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})
```

`defineConfig` 为配置提供类型提示。`@` 别名让导入路径更稳定：

```ts
import type { Todo } from '@/types/todo'
```

路径别名需要同时被 Vite 和 TypeScript 理解。create-vue 通常会生成匹配配置，不要只改一边。

## 22. 开发代理连接 FastAPI

开发阶段常见地址：

```text
Vue / Vite     http://localhost:5173
FastAPI        http://localhost:8000
```

可以让 Vite 把 `/api` 请求转发到 FastAPI：

```ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
    },
  },
})
```

前端请求使用相对地址：

```ts
fetch('/api/v1/todos')
```

开发时浏览器只访问 Vite 地址，Vite 在服务器侧转发请求，因此可以简化本地跨源配置。

注意：

- Vite proxy 只在开发服务器运行时生效；
- 生产环境需要由 Nginx、网关或部署平台配置反向代理；
- 代理不是关闭权限或安全校验；
- 修改 Vite 配置后通常需要重启开发服务器。

## 23. 环境变量文件

Vite 常见文件：

```text
.env                 所有模式加载
.env.local           本机私有覆盖，通常不提交
.env.development     开发模式
.env.production      生产构建模式
.env.example         可提交的变量说明
```

示例：

```dotenv
VITE_API_BASE_URL=/api/v1
VITE_APP_TITLE=TodoLab
```

TypeScript 中读取：

```ts
const apiBaseUrl = import.meta.env.VITE_API_BASE_URL
```

只有符合 Vite 暴露规则的变量才能进入客户端代码，通常使用 `VITE_` 前缀。

## 24. 前端环境变量不是秘密

任何进入浏览器构建产物的变量都可能被用户查看：

```dotenv
# 可以公开
VITE_API_BASE_URL=/api/v1

# 不可以放入前端
VITE_DATABASE_PASSWORD=secret
VITE_JWT_SIGNING_KEY=secret
```

前端环境变量适合：

- API 公共地址；
- 应用名称；
- 公开功能开关；
- 公开监控项目标识。

数据库密码、JWT 签名密钥、第三方私钥必须保留在后端或安全服务端环境中。

## 25. 为环境变量补充类型

```ts
// src/env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string
  readonly VITE_APP_TITLE: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

这能提供编辑器补全，但不会在运行时确认变量一定存在。

应用启动时仍可做明确检查：

```ts
const apiBaseUrl = import.meta.env.VITE_API_BASE_URL

if (!apiBaseUrl) {
  throw new Error('缺少 VITE_API_BASE_URL')
}
```

修改 `.env` 后通常需要重启 Vite 开发服务器。

## 26. 开发、构建和预览的区别

### dev

```powershell
pnpm dev
```

- 启动开发服务器；
- 快速热更新；
- 读取 development 模式环境变量；
- 不是最终优化产物。

### build

```powershell
pnpm build
```

- 生成优化后的静态文件；
- 默认输出到 dist；
- 文件通常压缩并带哈希；
- 读取 production 模式环境变量。

### preview

```powershell
pnpm preview
```

- 在本地查看 dist；
- 用于发现开发模式与构建模式差异；
- 不是生产服务器。

## 27. Vite 不等于完整类型检查

Vite 的转换过程追求速度，通常不会单独完成完整 TypeScript 类型检查。

因此工程应明确运行：

```powershell
pnpm type-check
pnpm build
```

Vue 项目通常使用 `vue-tsc` 检查 `.vue` 文件中的 TypeScript 和模板类型。

不要看到页面能启动就认为没有类型错误。CI 应同时运行 type-check、lint、test 和 build 中项目实际配置的步骤。

## 28. ESLint 与 Prettier 的分工

```text
ESLint     发现潜在代码问题和不合理模式
Prettier   统一空格、换行、引号等格式
TypeScript 检查类型关系
```

三者有交集，但不能互相完全替代。

常见命令：

```powershell
pnpm lint
pnpm format
pnpm type-check
```

不要在没有理解团队规则时同时安装多套相互冲突的格式插件。优先使用 create-vue 生成的兼容配置。

## 29. .gitignore 与提交边界

常见不提交内容：

```gitignore
node_modules/
dist/
.env.local
.env.*.local
*.log
```

通常应该提交：

```text
package.json
pnpm-lock.yaml
vite.config.ts
tsconfig*.json
.env.example
src/
public/
```

不要提交 node_modules。其他成员通过 package.json 和 lockfile 复原依赖。

## 30. .env.example

`.env.example` 不放真实秘密，只说明需要哪些变量：

```dotenv
VITE_API_BASE_URL=/api/v1
VITE_APP_TITLE=TodoLab
```

新成员可以复制为本机文件并填入适当值。注释应说明用途和示例，不要把生产令牌复制进去。

## 31. 构建产物 dist

典型 dist：

```text
dist/
├─ assets/
│  ├─ index-abc123.js
│  └─ index-def456.css
└─ index.html
```

文件名哈希用于缓存失效：内容改变后文件名也会改变。

生产部署通常需要：

- 静态文件服务器；
- 正确的缓存头；
- SPA 路由回退到 index.html；
- `/api` 反向代理到 FastAPI；
- HTTPS；
- gzip 或 Brotli 压缩；
- 安全响应头。

这些属于后续部署与 Docker 阶段。

## 32. 代码拆分的基本认识

Vite 构建时会分析 ESM import。动态导入可以形成按需加载边界：

```ts
const SettingsView = () => import('@/views/SettingsView.vue')
```

路由页面常使用这种方式降低首次加载体积。

不要在项目初期手工配置复杂分包。先使用合理的路由懒加载，并通过构建报告确认真实问题。

## 33. 浏览器兼容目标

Vite 会根据项目配置和工具链处理现代语法，但不是所有运行时 API 都会自动获得 polyfill。

例如：

```ts
structuredClone(value)
```

能否使用取决于目标浏览器是否支持，或项目是否明确提供兼容方案。

团队应定义浏览器支持范围，不要只在自己的最新版浏览器中验证。

## 34. 依赖安全与维护

基础习惯：

- 只安装真正需要的包；
- 安装前确认包名、维护状态和官方来源；
- 不盲目运行来源不明的安装命令；
- 提交并审查 lockfile 变化；
- 定期运行项目测试和构建；
- 使用审计命令作为线索，而不是机械强制升级所有依赖。

```powershell
pnpm audit
```

审计结果需要结合依赖是否进入生产代码、漏洞利用条件和升级影响判断。

## 35. 常见问题：命令找不到

### node 不是内部或外部命令

检查：

```powershell
Get-Command node
node --version
```

常见原因：

- Node 尚未安装；
- PATH 未刷新；
- 版本管理器尚未选择版本；
- 终端在安装前已经打开。

重新打开终端后再次检查，不要在未知目录复制 node.exe。

### pnpm 找不到

检查：

```powershell
Get-Command pnpm
pnpm --version
```

确认 Corepack 或 pnpm 安装是否完成，并检查当前 Node 版本是否切换后丢失了对应配置。

## 36. PowerShell 脚本执行问题

Windows 有时会提示无法加载 `pnpm.ps1`。可以先检查：

```powershell
Get-Command pnpm -All
```

在某些环境中可明确调用：

```powershell
pnpm.cmd --version
```

不要为了运行一个命令就随意把系统执行策略永久改为完全不受限制。若需要调整，应理解组织安全策略并使用最小范围。

## 37. 常见问题：运行目录错误

错误：

```text
ERR_PNPM_NO_IMPORTER_MANIFEST_FOUND
```

通常表示当前目录没有 package.json。

检查：

```powershell
Get-Location
Get-ChildItem
```

进入正确项目目录后再执行 pnpm 命令。

## 38. 常见问题：端口被占用

Vite 可能自动选择另一个端口，也可以手工指定：

```powershell
pnpm dev -- --port 5174
```

Windows 检查端口：

```powershell
Get-NetTCPConnection -LocalPort 5173 -ErrorAction SilentlyContinue
```

不要看到端口占用就直接结束未知进程。先确认进程身份和用途。

## 39. 常见问题：页面能开但 API 失败

按顺序检查：

1. FastAPI 是否正在 8000 端口运行；
2. 浏览器 Network 中请求 URL 是否正确；
3. 请求是否以 `/api` 开头并命中代理；
4. vite.config.ts 修改后是否重启；
5. FastAPI 路由是否真的包含 `/api/v1`；
6. 返回的是 404、422、500 还是网络错误；
7. 生产环境是否错误依赖了开发代理。

不要把所有问题都归因于 CORS。使用 Vite proxy 时，浏览器看到的往往是同源请求。

## 40. 常见问题：环境变量是 undefined

检查：

- 是否以 `VITE_` 开头；
- 是否使用 `import.meta.env`；
- 文件名是否与当前模式匹配；
- 是否重启开发服务器；
- 是否在错误目录创建 `.env`；
- 是否错误地在客户端尝试读取 `process.env`。

正确：

```ts
import.meta.env.VITE_API_BASE_URL
```

普通 Vite 客户端源码中不要假设存在 Node 的 `process.env`。

## 41. 常见问题：依赖已安装但无法 import

检查：

```powershell
pnpm list --depth 0
pnpm why package-name
```

可能原因：

- 只作为其他包的间接依赖存在，但当前项目未直接声明；
- 包名拼写错误；
- 包的导出路径改变；
- 类型定义与模块解析配置不匹配；
- 编辑器仍使用旧 TypeScript 服务状态。

当前源码直接 import 的第三方包通常应该成为当前项目的直接依赖。

## 42. 常见问题：开发正常，构建失败

可能原因：

- 开发服务器没有执行完整类型检查；
- 文件名大小写在 Windows 与 Linux 表现不同；
- production 环境变量缺失；
- 动态路径无法被构建器静态分析；
- 第三方包只在开发环境偶然可用；
- 代码依赖浏览器或 Node 的特定环境。

因此提交前至少执行：

```powershell
pnpm type-check
pnpm lint
pnpm build
```

## 43. Windows 与跨平台路径

源码 import 使用 URL 风格斜杠：

```ts
import { createTodo } from '@/api/todos'
```

不要在前端源码中硬编码 Windows 绝对路径：

```ts
// 错误
import data from 'D:\\project\\data.json'
```

构建和 CI 可能在 Linux 中运行。文件名大小写也应与 import 完全一致。

## 44. 一个建议的工程初始化清单

```text
[ ] Node 版本符合项目要求
[ ] pnpm 版本已确认
[ ] 使用 create-vue 创建工程
[ ] TypeScript strict 开启
[ ] package.json scripts 可以运行
[ ] pnpm-lock.yaml 已生成
[ ] .gitignore 正确
[ ] .env.example 已创建
[ ] 环境变量不包含秘密
[ ] @ 路径别名有效
[ ] Vite /api 开发代理有效
[ ] type-check 通过
[ ] lint 通过
[ ] build 通过
[ ] preview 可以打开 dist
```

## 45. 推荐的日常开发流程

```text
1. 拉取代码
2. pnpm install --frozen-lockfile
3. 根据 .env.example 准备本机环境变量
4. pnpm dev
5. 开发与浏览器调试
6. pnpm type-check
7. pnpm lint
8. pnpm test
9. pnpm build
10. 提交源码与 lockfile 变化
```

本地已有 node_modules 时，普通开发不需要每天重复安装。只有 package.json 或 lockfile 发生变化时才需要同步依赖。

## 46. 本课实操

### 练习 A：识别职责

说明下面工作由哪个工具主要负责：

1. 执行 Vite CLI；
2. 安装 Axios；
3. 生成 Vue 项目；
4. 启动热更新服务器；
5. 记录精确依赖图；
6. 生成 dist。

### 练习 B：创建工程

使用 create-vue 创建 `todo-web`，选择：

- TypeScript；
- Vue Router；
- Pinia；
- 单元测试；
- ESLint；
- Prettier。

安装后依次运行开发服务器、类型检查、lint 和构建。

### 练习 C：阅读 package.json

记录：

- 每个 script 的实际命令；
- Vue、Vite、TypeScript 的版本范围；
- dependencies 与 devDependencies 的差别；
- Node engines 是否存在；
- 使用的包管理器版本如何记录。

### 练习 D：配置代理

让 `/api` 转发到 `http://localhost:8000`，在前端请求：

```ts
fetch('/api/v1/todos')
```

打开 Network 面板，解释浏览器看到的 URL 与 FastAPI 实际收到的路径。

### 练习 E：环境变量

创建：

```dotenv
VITE_API_BASE_URL=/api/v1
VITE_APP_TITLE=TodoLab
```

在 TypeScript 中读取并检查它们。解释为什么数据库密码不能使用相同方式传入。

### 练习 F：模拟构建故障

制造一个 TypeScript 类型错误，观察：

- pnpm dev 是否仍可能启动；
- pnpm type-check 的输出；
- pnpm build 是否包含类型检查取决于哪个 script；
- 浏览器运行结果是否能代表类型安全。

## 47. 练习 A 参考答案

1. Node.js 执行 Vite CLI；
2. pnpm 安装 Axios；
3. create-vue 生成项目；
4. Vite 提供热更新开发服务器；
5. pnpm-lock.yaml 记录由 pnpm 解析的依赖图；
6. Vite 的 build 阶段生成 dist。

## 48. 自测问题

进入下一课前，确保你能回答：

- Node.js 在 Vue 前端工程中主要做什么？
- 浏览器环境与 Node.js 环境有哪些重要差异？
- package.json 与 pnpm-lock.yaml 分别负责什么？
- dependencies 和 devDependencies 如何区分？
- 为什么应提交 lockfile 而不提交 node_modules？
- `pnpm dev`、`pnpm build`、`pnpm preview` 有什么区别？
- create-vue 与 Vite 的关系是什么？
- public 与 src/assets 有何区别？
- Vite 开发代理为什么不能代替生产反向代理？
- 为什么 `VITE_` 环境变量不能存放秘密？
- Vite 为什么不能自动代表完整 TypeScript 类型检查？
- 开发成功但生产构建失败时，应检查哪些方面？

## 49. 验收标准

完成本课后，你应当能够：

- 解释 Node.js、pnpm、Vite 和 create-vue 的职责；
- 检查 Node 与 pnpm 版本；
- 阅读 package.json scripts 和依赖分类；
- 使用 pnpm 安装、删除、查询和执行依赖；
- 理解并正确提交 pnpm-lock.yaml；
- 使用 create-vue 创建 Vue 3 + TypeScript 工程；
- 读懂标准 Vue 工程目录；
- 配置 Vite 路径别名与开发代理；
- 使用不同模式的环境变量且不泄露秘密；
- 区分开发服务器、生产构建和本地预览；
- 分别执行 type-check、lint、test 和 build；
- 排查命令、端口、路径、代理、环境变量和构建问题。

## 50. 下一步

下一课进入 **Vue 3 核心基础**，将集中学习：

- 单文件组件；
- `<script setup>`；
- 模板语法和指令；
- ref、reactive、computed 与 watch；
- props 与 emits；
- 生命周期；
- 列表、表单与异步状态；
- 组件拆分与组合式函数。

完成后，我们会把 Todo 页面正式改造成 Vue 组件应用。
