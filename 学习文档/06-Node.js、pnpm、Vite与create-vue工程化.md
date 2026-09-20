# 06｜Node.js、pnpm、Vite 与 create-vue

> 目标：能创建和运行 Vue 工程、看懂配置、让 AI 在现有工具链内开发并完成验证。

[← TypeScript](./05-TypeScript基础与类型系统.md) · [学习首页](../README.md) · [Vue 3 →](./07-Vue3核心基础与组件化.md)

## 1. 四项工具的职责

```text
Node.js      运行前端开发工具
pnpm         安装依赖、锁定版本、执行 scripts
Vite         开发服务器和生产构建
create-vue   生成官方 Vue 项目结构
```

```text
create-vue 生成项目
  ↓
pnpm 安装依赖
  ↓
Node.js 运行 Vite、TypeScript、ESLint
  ↓
浏览器运行构建后的 HTML、CSS、JavaScript
```

## 2. 创建项目

```powershell
node --version
pnpm --version
pnpm create vue@latest
```

课程推荐选择：

```text
TypeScript       是
Vue Router       是
Pinia            是
ESLint           是
Prettier         是
单元测试         项目需要时选择
JSX              初学阶段否
```

创建后：

```powershell
Set-Location .\todo-web
pnpm install
pnpm dev
```

## 3. Node.js 与浏览器不同

| 能力 | 浏览器 | Node.js |
| --- | --- | --- |
| DOM、window | 有 | 默认没有 |
| 文件系统 | 受限 | 可访问 |
| 页面渲染 | 有 | 默认没有 |
| 构建工具 | 不直接运行 | 运行 Vite 等工具 |

浏览器 Vue 代码不能随意使用：

```ts
import { readFile } from 'node:fs/promises'
```

这是 Node.js API。

## 4. package.json

```json
{
  "name": "todo-web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "type-check": "vue-tsc --build",
    "lint": "eslint ."
  },
  "dependencies": {
    "vue": "^3.5.0"
  },
  "devDependencies": {
    "vite": "^6.0.0",
    "typescript": "^5.0.0"
  }
}
```

版本仅用于理解结构，实际项目不要强行覆盖脚手架生成的兼容版本。

```text
dependencies      应用代码运行所需依赖
devDependencies   构建、检查和测试工具
scripts           团队统一命令入口
```

常见语义化版本写作 `主版本.次版本.修订版本`：

```text
3.5.2
│ │ └─ 修复兼容问题
│ └── 新增向后兼容功能
└──── 可能包含不兼容变化
```

`^3.5.0` 通常允许升级到同一主版本的较新版本，`~3.5.0` 通常只允许修订版本升级。`package.json` 表达允许范围，`pnpm-lock.yaml` 记录本次真正解析出的完整依赖图；二者都应提交。

## 5. pnpm 常用命令

```powershell
pnpm install
pnpm add axios
pnpm add -D eslint
pnpm remove axios
pnpm dev
pnpm build
pnpm type-check
pnpm lint
pnpm why vite
pnpm list --depth 0
```

不要提交 `node_modules/`。应用项目通常应提交 `pnpm-lock.yaml`，让本地和 CI 安装同一依赖图。

CI 常用：

```powershell
pnpm install --frozen-lockfile
```

pnpm 会把下载内容集中存储，并通过链接组成项目的 `node_modules`，减少重复占用。它对“幽灵依赖”也更严格：代码直接 import 的包就应写入当前项目的 dependencies，而不是碰巧从另一个包的依赖中找到。

## 6. Vite 的两种工作

```text
开发：快速启动、模块转换、热更新
构建：打包、压缩、代码拆分、生成 dist
```

```powershell
pnpm dev
pnpm build
pnpm preview
```

`preview` 只用于本地查看构建结果，不是生产服务器。

页面能在 dev 打开不代表类型检查和生产构建一定通过。

构建后常见产物：

```text
dist/
├─ index.html
└─ assets/
   ├─ index-[hash].js
   └─ index-[hash].css
```

hash 便于长期缓存并在内容变化时更新。`dist` 是可部署的静态产物，通常由 CI 重新生成，不应与源代码混淆，也不要手工编辑。

## 7. 项目入口

```text
index.html
└─ src/main.ts
   └─ App.vue
      └─ views / components
```

`main.ts`：

```ts
import { createApp } from 'vue'
import App from './App.vue'
import './assets/main.css'

createApp(App).mount('#app')
```

现代 Vite 项目使用 ESM：

```ts
import { createApp } from 'vue'
export function createTodo() {}
```

ESM 的导入导出是静态结构，便于类型分析和按需打包。浏览器业务代码不要混用旧式 `require()`；配置文件若使用 Node API，要确认其执行环境是 Node 而不是浏览器。

## 8. 推荐目录

```text
src/
├─ api/           Axios 接口模块
├─ assets/        样式和构建资源
├─ components/    可复用组件
├─ composables/   可复用组合式逻辑
├─ router/        Vue Router
├─ stores/        Pinia
├─ types/         共享类型
├─ views/         路由页面
├─ App.vue
└─ main.ts
```

目录不是越多越好，有对应职责再创建。

## 9. public 与 assets

```text
public/       原样复制，用根路径引用
src/assets/   参与模块导入和构建处理
```

```html
<img src="/favicon.svg" alt="" />
```

```ts
import illustration from '@/assets/todo.png'
```

业务组件使用的图片通常放 `src/assets`，必须保留固定名称的公共文件可以放 public。

## 10. Vite 配置与别名

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

```ts
import type { Todo } from '@/types/todo'
```

别名要同时被 Vite 和 TypeScript 理解。优先保留 create-vue 生成的匹配配置。

## 11. FastAPI 开发代理

```ts
export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/api': {
        target: 'http://127.0.0.1:8000',
        changeOrigin: true,
      },
    },
  },
})
```

前端请求：

```ts
fetch('/api/v1/todos')
```

代理只在 Vite 开发服务器生效。生产环境需要 Nginx、网关或平台配置。

## 12. 环境变量

```dotenv
VITE_API_BASE_URL=/api/v1
VITE_APP_TITLE=TodoLab
```

读取：

```ts
const apiBaseUrl = import.meta.env.VITE_API_BASE_URL
```

任何进入前端构建产物的变量都能被用户查看。数据库密码、JWT 签名密钥和私钥不能使用 `VITE_` 变量保存。

提交 `.env.example`，不要提交真实 `.env.local`。

## 13. 开发检查

```powershell
pnpm type-check
pnpm lint
pnpm test
pnpm build
```

各项目脚本可能不同，先读 package.json。

```text
TypeScript / vue-tsc   类型
ESLint                 代码问题
Prettier               格式
Vitest                 测试
Vite build             生产构建
```

## 14. 常见排错

```text
命令找不到        检查 node、pnpm 和 PATH
找不到 package    确认当前目录有 package.json
端口占用          查看 Vite 实际启动端口
API 404           检查最终 URL、代理和 FastAPI prefix
变量 undefined    检查 VITE_ 前缀、文件位置并重启 Vite
开发正常构建失败  运行 type-check，检查大小写和生产变量
依赖能间接找到    当前代码直接 import 的包应直接声明
```

不要用删除 lockfile、升级全部依赖或关闭类型检查作为第一反应。

依赖本身也是供应链的一部分。添加包前确认维护状态、许可证和必要性；升级时阅读变更说明并运行完整检查。前端环境变量、源码映射和构建产物中都不应包含密钥，发现安全告警时也要先判断受影响版本和调用路径，不能只看告警数量。

## 15. 给 AI 的开发指令

```text
请在现有 Vue 3 + TypeScript + Vite 项目中实现功能。
先读取 package.json、vite.config、tsconfig 和现有目录，不升级依赖或更换工具链。
复用 @ 别名、环境变量和 /api 开发代理，不硬编码生产地址。
不提交 node_modules、dist 或本机 .env。
完成后按项目脚本运行 type-check、lint、test 和 build，并报告实际结果。
```

排错指令：

```text
请根据终端错误、package.json scripts 和浏览器 Network 定位问题。
区分 Node/pnpm 环境、Vite 转换、TypeScript、代理、环境变量和生产构建错误。
先说明根因证据，再做最小修改，不要顺带升级所有依赖。
```

## 16. 面试表达

> Node.js 在 Vue 工程中主要运行 Vite、TypeScript、ESLint 和测试工具，客户端代码最终仍在浏览器执行。

> package.json 声明依赖和脚本，pnpm-lock.yaml 锁定实际依赖图，node_modules 不应提交。

> Vite 开发阶段提供快速模块加载和热更新，构建阶段生成优化后的 dist。

> Vite 代理只解决本地开发转发，不能替代生产网关或后端权限校验。

> VITE_ 环境变量会进入浏览器代码，因此只能保存公开配置，不能保存秘密。

[进入下一课：Vue 3 核心基础 →](./07-Vue3核心基础与组件化.md)
