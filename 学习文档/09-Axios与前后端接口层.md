# 09｜Axios 与前后端接口层

> 所属阶段：Vue 前后端通信  
> 本课合并知识点：Axios 实例、请求配置、TypeScript 类型、错误模型、拦截器、认证、刷新流程、取消请求、上传下载、并发、API 分层、Pinia 集成  
> 前置知识：HTTP、JSON、TypeScript、Vue 3、Vue Router、Pinia  
> 学习目标：熟悉 Vue 调用 FastAPI 的完整接口层，能够指挥 AI 编写可靠请求代码，并能在面试中说明常用机制、安全边界和异常处理。

[← Vue Router 与 Pinia](./08-Vue-Router与Pinia.md) · [学习首页](../README.md)

## 阅读方式

- 第 1～13 节掌握 Axios 请求、响应和类型基础；
- 第 14～25 节理解错误处理、拦截器和认证；
- 第 26～34 节覆盖取消、超时、上传、并发和测试；
- 第 35～38 节用于 AI 编程、代码审核和面试复习；
- 不需要背诵全部配置项，重点掌握接口层职责和失败场景。

## 1. Axios 在项目中的位置

Axios 是运行在浏览器或 Node.js 中的 HTTP 客户端。在当前技术栈中，它负责把前端请求发送给 FastAPI：

```text
Vue Component
      ↓ 用户交互
Pinia Action / Composable
      ↓ 调用业务接口函数
API Module
      ↓ 使用 Axios 实例
HTTP + JSON / FormData
      ↓
FastAPI Router
```

Axios 负责 HTTP 通信能力，不负责：

- Vue 页面渲染；
- Pinia 全局状态管理；
- 后端业务规则；
- 数据库持久化；
- 自动验证所有运行时 JSON；
- 自动决定登录和刷新策略。

## 2. Axios 与 fetch

浏览器已经内置 `fetch`，Axios 是第三方库。二者都能完成 HTTP 请求。

| 能力 | Axios | fetch |
| --- | --- | --- |
| JSON 响应 | 自动解析常见 JSON 响应 | 手工调用 `response.json()` |
| 非 2xx | 默认拒绝 Promise | 默认仍然成功，需要检查 `response.ok` |
| 超时 | 提供 `timeout` | 通常结合 AbortController |
| 请求与响应拦截 | 内置 Interceptors | 需要自行封装 |
| 请求取消 | 支持标准 `AbortSignal` | 支持标准 `AbortSignal` |
| 上传进度 | 浏览器适配器可用 | 原生 fetch 支持受环境限制 |
| 依赖体积 | 需要安装依赖 | 浏览器内置 |

项目选择原则：

- 请求简单且希望减少依赖，可以使用 fetch；
- 需要统一实例、拦截器、复杂认证和一致错误处理，Axios 通常更方便；
- 团队应统一主要封装方式，避免不同页面各自实现一套错误和认证逻辑。

Axios 不是 fetch 的“高级版本”，两者是不同 API 设计下的工具选择。

## 3. 安装 Axios

```powershell
pnpm add axios
```

确认依赖：

```powershell
pnpm list axios
```

Axios 自带 TypeScript 类型，现代项目通常不需要额外安装 `@types/axios`。

## 4. 最小请求

GET：

```ts
import axios from 'axios'

const response = await axios.get('/api/v1/todos')
console.log(response.data)
```

POST：

```ts
const response = await axios.post('/api/v1/todos', {
  title: '学习 Axios',
})
```

常用方法：

```ts
axios.get(url, config)
axios.post(url, data, config)
axios.put(url, data, config)
axios.patch(url, data, config)
axios.delete(url, config)
```

注意 POST、PUT、PATCH 的第二个参数是请求体，而 GET 和 DELETE 的第二个参数通常是配置对象。

## 5. 创建统一 Axios 实例

不要在每个组件中重复写域名、超时和公共配置。创建独立实例：

```ts
// src/api/http.ts
import axios from 'axios'

export const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api/v1',
  timeout: 10_000,
  headers: {
    Accept: 'application/json',
  },
})
```

调用：

```ts
await http.get('/todos')
```

最终地址可能是：

```text
baseURL /api/v1 + url /todos = /api/v1/todos
```

不要默认给所有请求设置 `Content-Type: application/json`。没有请求体的 GET 不需要它，FormData 上传还需要浏览器生成包含 boundary 的 Content-Type。

## 6. baseURL 与开发代理

推荐开发请求使用相对地址：

```dotenv
VITE_API_BASE_URL=/api/v1
```

Vite 代理：

```ts
// vite.config.ts
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

浏览器请求 `/api/v1/todos`，Vite 在开发服务器侧转发给 FastAPI。

需要明确：

- Vite proxy 只在开发服务器中生效；
- 生产环境需要 Nginx、网关或平台配置；
- `VITE_` 变量会进入浏览器构建产物，不能存放秘密；
- baseURL 应由环境配置，不应散落硬编码在组件中。

## 7. Axios 响应对象

```ts
const response = await http.get('/todos')
```

主要字段：

| 字段 | 含义 |
| --- | --- |
| `data` | 响应体 |
| `status` | HTTP 状态码 |
| `statusText` | 状态文本，具体环境可能不同 |
| `headers` | 响应头 |
| `config` | 本次请求配置 |
| `request` | 底层请求对象，随环境不同 |

API 模块通常只向上层返回 `data`：

```ts
export async function getTodos(): Promise<TodoDto[]> {
  const { data } = await http.get<TodoDto[]>('/todos')
  return data
}
```

如果上层需要分页响应头或状态码，可以返回完整响应或定义专门结果类型，不应为了统一而丢掉真实需要的信息。

## 8. TypeScript 泛型

```ts
interface TodoDto {
  id: number
  title: string
  completed: boolean
}

const response = await http.get<TodoDto[]>('/todos')

response.data[0]?.title
```

泛型让 TypeScript 在编译期理解 `data` 的结构，但不会检查服务器实际返回值。

```ts
http.get<TodoDto[]>('/todos')
```

这句话的准确含义是：

```text
开发者告诉 TypeScript，预期响应是 TodoDto[]
```

而不是：

```text
Axios 已经在运行时验证响应一定符合 TodoDto[]
```

关键数据需要结合 Zod、Valibot 等运行时 Schema，或在边界编写明确解析逻辑。

## 9. 请求数据类型

同一个 Todo 在创建、更新和响应阶段通常使用不同类型：

```ts
export interface TodoCreate {
  title: string
}

export interface TodoUpdate {
  title?: string
  completed?: boolean
}

export interface TodoDto {
  id: number
  title: string
  completed: boolean
  created_at: string
}
```

API 函数：

```ts
export async function createTodo(input: TodoCreate): Promise<TodoDto> {
  const { data } = await http.post<TodoDto>('/todos', input)
  return data
}

export async function updateTodo(
  id: number,
  input: TodoUpdate,
): Promise<TodoDto> {
  const { data } = await http.patch<TodoDto>(`/todos/${id}`, input)
  return data
}
```

不要使用一个包含 `id`、`created_at` 的完整响应类型作为创建输入，否则前端可能错误发送只应由服务器生成的字段。

## 10. Params 与 Data

Query 参数放在 `params`：

```ts
await http.get('/todos', {
  params: {
    status: 'active',
    page: 2,
    page_size: 20,
  },
})
```

请求体放在 `data`，通常通过方法的第二个参数传入：

```ts
await http.post('/todos', {
  title: '学习接口层',
})
```

生成的请求概念上是：

```http
GET /todos?status=active&page=2&page_size=20
```

与：

```http
POST /todos
Content-Type: application/json

{"title":"学习接口层"}
```

数组和嵌套对象的 Query 序列化没有唯一行业格式。前端 Axios 配置必须与 FastAPI 的接口约定一致。

## 11. Header

单次请求 Header：

```ts
await http.get('/todos', {
  headers: {
    'X-Request-ID': crypto.randomUUID(),
  },
})
```

常见 Header：

| Header | 用途 |
| --- | --- |
| `Accept` | 客户端希望接收的媒体类型 |
| `Content-Type` | 请求体的媒体类型 |
| `Authorization` | Bearer 等认证凭据 |
| `If-None-Match` | 条件请求和缓存验证 |
| `X-Request-ID` | 链路追踪，是否使用由系统约定 |

不要在前端伪造本应由可信网关设置的身份 Header，例如 `X-User-ID`，后端也不应信任普通浏览器可以随意修改的 Header。

## 12. PUT 与 PATCH

常见语义：

```text
PUT     用提交内容整体替换资源表示，通常要求完整字段
PATCH   对资源进行部分修改，只提交需要变化的字段
```

示例：

```ts
await http.patch(`/todos/${id}`, {
  completed: true,
})
```

实际行为必须以 API 契约为准。不能只凭方法名称假定后端一定实现严格的完整替换或部分更新。

## 13. API 模块的组织

```text
src/api/
├─ http.ts          Axios 实例和底层拦截器
├─ errors.ts        错误类型与标准化
├─ auth.ts          登录、退出、会话刷新
├─ todos.ts         Todo 接口函数
└─ files.ts         上传和下载
```

`src/api/todos.ts`：

```ts
import { http } from './http'
import type { TodoCreate, TodoDto, TodoUpdate } from '@/types/todo'

export const todoApi = {
  async list(): Promise<TodoDto[]> {
    const { data } = await http.get<TodoDto[]>('/todos')
    return data
  },

  async get(id: number): Promise<TodoDto> {
    const { data } = await http.get<TodoDto>(`/todos/${id}`)
    return data
  },

  async create(input: TodoCreate): Promise<TodoDto> {
    const { data } = await http.post<TodoDto>('/todos', input)
    return data
  },

  async update(id: number, input: TodoUpdate): Promise<TodoDto> {
    const { data } = await http.patch<TodoDto>(`/todos/${id}`, input)
    return data
  },

  async remove(id: number): Promise<void> {
    await http.delete(`/todos/${id}`)
  },
}
```

组件不需要知道 baseURL、认证 Header 和底层错误识别方式，只调用语义清楚的业务接口函数。

## 14. DTO 与前端领域类型

后端常使用 snake_case：

```ts
interface TodoDto {
  id: number
  title: string
  completed: boolean
  created_at: string
}
```

前端可能希望使用 camelCase 和 Date：

```ts
interface Todo {
  id: number
  title: string
  completed: boolean
  createdAt: Date
}
```

统一转换：

```ts
function toTodo(dto: TodoDto): Todo {
  return {
    id: dto.id,
    title: dto.title,
    completed: dto.completed,
    createdAt: new Date(dto.created_at),
  }
}
```

转换应集中在 API 边界，不要让每个组件重复处理 `created_at`。

如果团队决定 API 和前端统一使用同一命名格式，也可以不转换。关键是形成明确约定。

## 15. Axios 错误的主要类别

```text
Axios 请求失败
├─ HTTP 错误       已收到非 2xx 响应
├─ 网络错误       没有收到 HTTP 响应
├─ 超时           超过客户端等待时间
├─ 主动取消       AbortController 终止请求
├─ 配置或代码错误 请求尚未正确发出
└─ 业务错误       HTTP 成功但业务结果不满足要求
```

这些错误对用户提示和程序处理不同：

- 401 可能触发会话恢复或登录；
- 403 表示当前身份无权限；
- 404 表示资源不存在；
- 409 表示当前状态冲突；
- 422 通常表示请求字段验证失败；
- 429 需要遵守限流策略；
- 5xx 表示服务端或网关异常；
- 网络错误也可能是断网、DNS、CORS 拦截或服务未启动。

## 16. 使用 `axios.isAxiosError`

```ts
import axios from 'axios'

try {
  await todoApi.list()
} catch (error: unknown) {
  if (axios.isAxiosError(error)) {
    console.log(error.response?.status)
    console.log(error.response?.data)
    console.log(error.code)
  } else {
    console.error('非 Axios 错误', error)
  }
}
```

`catch` 中使用 `unknown` 比 `any` 更安全，因为它要求代码先缩小类型再读取属性。

Axios 错误中：

```text
error.response   服务器返回了 HTTP 响应
error.request    请求已创建，但未获得正常响应
error.config     触发错误的请求配置
error.code       Axios 或底层错误代码
```

日志中不要直接输出完整配置，因为其中可能包含 Authorization Header、Cookie 相关信息或敏感请求数据。

## 17. 建立统一前端错误模型

```ts
export type ApiErrorKind =
  | 'http'
  | 'network'
  | 'timeout'
  | 'canceled'
  | 'validation'
  | 'unknown'

export interface ApiError {
  kind: ApiErrorKind
  message: string
  status?: number
  code?: string
  fieldErrors?: Record<string, string>
}
```

标准化：

```ts
import axios from 'axios'

export function normalizeApiError(error: unknown): ApiError {
  if (!axios.isAxiosError(error)) {
    return {
      kind: 'unknown',
      message: error instanceof Error ? error.message : '未知错误',
    }
  }

  if (error.code === 'ERR_CANCELED') {
    return { kind: 'canceled', message: '请求已取消', code: error.code }
  }

  if (error.code === 'ECONNABORTED' || error.code === 'ETIMEDOUT') {
    return { kind: 'timeout', message: '请求超时', code: error.code }
  }

  if (!error.response) {
    return { kind: 'network', message: '无法连接服务器', code: error.code }
  }

  return {
    kind: error.response.status === 422 ? 'validation' : 'http',
    status: error.response.status,
    message: `请求失败：${error.response.status}`,
    code: error.code,
  }
}
```

生产项目应从后端约定的错误结构中提取安全、可展示的信息，而不是直接把服务端堆栈或内部异常显示给用户。

## 18. FastAPI 的 422 验证错误

FastAPI 请求验证失败时常见响应：

```json
{
  "detail": [
    {
      "type": "string_too_short",
      "loc": ["body", "title"],
      "msg": "String should have at least 1 character",
      "input": ""
    }
  ]
}
```

前端类型：

```ts
interface ValidationIssue {
  type: string
  loc: Array<string | number>
  msg: string
  input?: unknown
}

interface FastApiValidationError {
  detail: ValidationIssue[]
}
```

映射字段错误：

```ts
function mapValidationErrors(
  payload: FastApiValidationError,
): Record<string, string> {
  return Object.fromEntries(
    payload.detail.map((issue) => [
      issue.loc.slice(1).join('.'),
      issue.msg,
    ]),
  )
}
```

后端错误消息是否直接展示取决于产品语言和安全要求。正式产品通常会建立稳定错误码与本地化文案，而不是完全依赖框架默认英文消息。

## 19. 响应拦截器

```ts
http.interceptors.response.use(
  (response) => response,
  (error: unknown) => Promise.reject(error),
)
```

拦截器适合处理跨接口的基础能力：

- 统一认证恢复；
- 追踪 ID；
- 基础错误标准化；
- 统一日志钩子；
- 特定响应 Header。

不适合在全局拦截器中完成：

- 每个业务页面的提示文案；
- 所有错误都弹同一个 Toast；
- 把所有响应强制改成完全相同的业务对象；
- 遇到任何错误都自动重试；
- 直接操作具体 Vue 页面 DOM。

响应拦截器中必须返回响应或拒绝 Promise，否则调用方可能收到 `undefined` 或错误被吞掉。

## 20. 请求拦截器

```ts
http.interceptors.request.use((config) => {
  const accessToken = getAccessToken()

  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`
  }

  config.headers['X-Request-ID'] = crypto.randomUUID()
  return config
})
```

请求拦截器常用于公共 Header 和认证凭据，但要注意：

- 不要把 Token 发给不可信域名；
- 单个实例最好只服务预期 API；
- Token 读取方式应避免循环依赖 Store；
- 拦截器注册一次，不要在每次组件渲染时重复注册；
- 测试和热更新环境要留意重复拦截器。

## 21. Interceptor 的顺序与移除

注册会返回 ID：

```ts
const interceptorId = http.interceptors.request.use((config) => config)

http.interceptors.request.eject(interceptorId)
```

拦截器链存在执行顺序。认证刷新通常要先识别原始 401，再由更上层进行错误标准化。

如果功能只在某个应用生命周期存在，应保存 ID 并在适当时机移除。项目级单例 Axios 实例上的固定拦截器通常在模块初始化时注册一次。

## 22. Bearer Token 认证

请求形式：

```http
Authorization: Bearer <access-token>
```

Axios：

```ts
http.interceptors.request.use((config) => {
  const token = getAccessToken()
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})
```

Token 保存位置存在权衡：

| 位置 | 特点 |
| --- | --- |
| JavaScript 内存 | 刷新后丢失，减少长期暴露，但仍受当前页面 XSS 影响 |
| localStorage | 刷新可恢复，但任何同源 JavaScript 都能读取，XSS 风险更高 |
| HttpOnly Cookie | JavaScript 不能读取，需要服务端 Cookie、CSRF 与跨域策略配合 |

不存在只靠换一个存储位置就绝对安全的方案。认证设计必须同时考虑 XSS、CSRF、过期、撤销、跨域和后端能力。

## 23. Cookie 会话与 `withCredentials`

同源请求通常会按浏览器 Cookie 规则自动携带 Cookie。跨源且允许凭据时：

```ts
export const http = axios.create({
  baseURL: 'https://api.example.com',
  withCredentials: true,
})
```

FastAPI/CORS 侧也必须明确允许凭据和具体 Origin。使用凭据时不能简单返回：

```http
Access-Control-Allow-Origin: *
```

还要准确理解：

- `withCredentials` 只控制浏览器凭据行为，不会创建登录态；
- CORS 是浏览器跨源读取策略，不是认证和授权；
- Cookie 认证需要评估 CSRF；
- `SameSite`、`Secure`、`HttpOnly` 和 Domain/Path 都会影响行为。

## 24. Access Token 刷新

常见模式：

```text
请求 API
  ↓ access token 过期
收到 401
  ↓
调用 refresh endpoint
  ↓ 成功
保存新 access token
  ↓
重放一次原请求
```

必须防止：

- 刷新接口自身 401 后再次刷新；
- 同一个请求无限重试；
- 多个并发 401 同时发起多个刷新；
- 刷新失败后仍保留旧登录态；
- 重放非幂等请求产生重复副作用。

## 25. 单飞刷新示例

```ts
import type { InternalAxiosRequestConfig } from 'axios'

type RetryableRequest = InternalAxiosRequestConfig & {
  _retry?: boolean
}

let refreshPromise: Promise<string> | null = null

http.interceptors.response.use(
  (response) => response,
  async (error) => {
    const status = error.response?.status
    const originalRequest = error.config as RetryableRequest | undefined

    if (
      status !== 401
      || !originalRequest
      || originalRequest._retry
      || originalRequest.url?.includes('/auth/refresh')
    ) {
      return Promise.reject(error)
    }

    originalRequest._retry = true

    try {
      refreshPromise ??= refreshAccessToken().finally(() => {
        refreshPromise = null
      })

      const token = await refreshPromise
      originalRequest.headers.Authorization = `Bearer ${token}`
      return http(originalRequest)
    } catch (refreshError) {
      clearSession()
      return Promise.reject(refreshError)
    }
  },
)
```

这是结构示例，不是可以无条件复制的完整认证方案。项目还需确定刷新令牌如何存储、刷新响应格式、并发请求如何排队、失败如何跳转，以及后端是否允许安全重放请求。

## 26. 请求取消

Axios 支持标准 `AbortController`：

```ts
const controller = new AbortController()

const request = http.get('/todos', {
  signal: controller.signal,
})

controller.abort()
await request
```

Vue 中监听搜索关键字：

```ts
watch(keyword, async (value, _oldValue, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  try {
    const { data } = await http.get<TodoDto[]>('/todos', {
      params: { q: value },
      signal: controller.signal,
    })

    todos.value = data
  } catch (error) {
    if (axios.isCancel(error)) return
    throw error
  }
})
```

取消主要避免旧请求覆盖新结果和组件销毁后继续处理结果。浏览器取消不保证服务端一定停止已经开始的业务操作。

## 27. 超时

```ts
const http = axios.create({
  timeout: 10_000,
})
```

单次覆盖：

```ts
await http.post('/reports', input, {
  timeout: 60_000,
})
```

超时值要结合接口性质：

- 普通查询通常应快速返回；
- 大型导出可能适合异步任务，而不是无限延长请求；
- 文件上传需要结合大小和网络环境；
- 客户端超时不代表服务器自动回滚或停止执行；
- 网关和后端也有自己的超时配置。

## 28. 重试

Axios 默认不会替所有失败自动完成业务级重试。重试前需要判断：

```text
请求是否幂等
错误是否暂时性
服务端是否已处理请求
是否存在 Idempotency-Key
退避和最大次数
是否尊重 Retry-After
```

通常更适合自动重试：

- GET 等幂等读取；
- 明确的 502、503、504；
- 部分网络瞬时故障；
- 有最大次数和指数退避。

谨慎重试：

- 创建订单；
- 支付；
- 发送消息；
- 任何可能产生重复副作用的 POST。

不能把“失败就重试三次”写成所有接口的全局规则。

## 29. 并发请求

全部成功才继续：

```ts
const [profile, todos] = await Promise.all([
  userApi.getProfile(),
  todoApi.list(),
])
```

允许部分失败：

```ts
const results = await Promise.allSettled([
  userApi.getProfile(),
  todoApi.list(),
])
```

并发能减少串行等待，但也会增加服务器瞬时压力。存在依赖关系的请求仍应按顺序执行。

同一资源被多个组件同时请求时，可以考虑 Store 缓存、请求去重或专门的服务端状态库，而不是无条件发出重复请求。

## 30. FormData 文件上传

```ts
export async function uploadAvatar(file: File): Promise<AvatarDto> {
  const formData = new FormData()
  formData.append('file', file)

  const { data } = await http.post<AvatarDto>('/users/me/avatar', formData)
  return data
}
```

同时发送普通字段：

```ts
formData.append('display_name', displayName)
formData.append('file', file)
```

浏览器会自动生成：

```http
Content-Type: multipart/form-data; boundary=----...
```

通常不要手工写不含 boundary 的 `Content-Type: multipart/form-data`。Axios 和浏览器会根据 FormData 正确设置。

上传前端校验可以改善体验，但后端仍必须验证文件大小、真实类型、扩展名、内容安全和访问权限。

## 31. 上传进度

```ts
const progress = ref(0)

await http.post('/files', formData, {
  onUploadProgress(event) {
    if (!event.total) return
    progress.value = Math.round((event.loaded / event.total) * 100)
  },
})
```

注意：

- 能力取决于 Axios 运行环境和适配器；
- 100% 表示客户端上传完成，不表示后端处理完成；
- 大文件通常需要分片、断点续传或对象存储预签名方案；
- 上传取消应结合 AbortController。

## 32. 文件下载

```ts
export async function downloadReport(): Promise<void> {
  const { data } = await http.get<Blob>('/reports/latest', {
    responseType: 'blob',
  })

  const url = URL.createObjectURL(data)
  const anchor = document.createElement('a')
  anchor.href = url
  anchor.download = 'report.pdf'
  anchor.click()
  URL.revokeObjectURL(url)
}
```

正式实现还应考虑：

- 服务端 Content-Disposition 文件名；
- 非 ASCII 文件名编码；
- Blob 错误响应的解析；
- 大文件内存占用；
- 下载权限和过期地址；
- 浏览器弹窗与用户手势限制。

## 33. Pinia 与 API 层整合

Store：

```ts
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { todoApi } from '@/api/todos'
import { normalizeApiError } from '@/api/errors'
import type { TodoDto } from '@/types/todo'

export const useTodoStore = defineStore('todos', () => {
  const todos = ref<TodoDto[]>([])
  const loading = ref(false)
  const errorMessage = ref('')

  async function fetchTodos() {
    loading.value = true
    errorMessage.value = ''

    try {
      todos.value = await todoApi.list()
    } catch (error) {
      const apiError = normalizeApiError(error)

      if (apiError.kind !== 'canceled') {
        errorMessage.value = apiError.message
      }

      throw apiError
    } finally {
      loading.value = false
    }
  }

  return { todos, loading, errorMessage, fetchTodos }
})
```

如果 API 模块已经把 `TodoDto` 转换为前端领域类型 `Todo`，Store 则应保存转换后的 `Todo[]`。同一条数据链路只做一次统一转换。

职责：

```text
Axios 实例    baseURL、超时、通用拦截器
API 模块      URL、HTTP 方法、DTO 与数据转换
Pinia Store   共享状态、缓存和业务流程
Vue 页面      加载、空状态、错误状态和用户交互
```

不要让组件直接操作 Axios 全局默认值，也不要让 http.ts 直接修改具体页面状态。

## 34. 接口测试与可观察性

API 层适合独立测试：

- 请求方法和路径正确；
- Params 与 Body 映射正确；
- DTO 转换正确；
- 401 刷新只发生一次；
- 422 映射为字段错误；
- 取消和超时分类正确；
- 敏感 Header 不进入日志。

常见工具组合：

```text
Vitest             运行单元测试
Mock Service Worker 在网络边界模拟 HTTP
Vue Test Utils      验证组件面对各种接口状态的表现
浏览器 Network      检查真实请求、响应和时序
FastAPI OpenAPI     核对后端契约
```

日志可以记录方法、路径、状态码、耗时和追踪 ID，但应删除 Token、Cookie、密码和敏感请求体。

## 35. 常见错误与准确理解

### 只写响应泛型就认为数据安全

Axios 泛型只影响 TypeScript。外部 JSON 是否可信仍需运行时校验或可靠的契约生成流程。

### 在每个组件重复创建 Axios 实例

会造成 baseURL、超时、认证和错误行为不一致。实例应按后端或信任边界集中管理。

### 全局写死 JSON Content-Type

会影响 FormData 等非 JSON 请求。让 Axios 根据具体 data 处理，或在单个请求中明确设置。

### 所有错误统一提示网络异常

401、403、404、409、422、429、5xx、超时和断网的处理不同，应先分类。

### 所有 401 都刷新并重试

刷新接口自身失败、重复重试和并发刷新会形成循环。需要 `_retry`、刷新单飞和失败清理。

### 在拦截器里操作页面

拦截器应处理协议层公共逻辑，页面提示和交互由 View 或 Store 决定。

### 取消请求等于撤销服务端操作

取消通常只让客户端停止等待或处理结果，服务端可能已经开始执行。

### CORS 等于权限控制

CORS 约束浏览器脚本读取跨源响应，不能替代后端认证、授权或 CSRF 防护。

### 自动重试所有 POST

可能重复创建资源或执行支付等副作用。需要幂等设计和明确的重试条件。

## 36. 指挥 AI 编写接口层

### 创建 Axios 基础层指令

```text
请在现有 Vue 3 + TypeScript 项目中建立 Axios 接口层。
先读取现有 Vite 环境变量、API 路径、认证方式和错误响应格式。
创建单例 Axios 实例，配置相对 baseURL、合理超时和 Accept Header。
不要给所有请求强制设置 Content-Type，不要在前端环境变量保存秘密。
按 http、errors、auth、todos 模块拆分，组件不能直接修改 Axios 默认配置。
完成后运行 type-check、测试和 build，并总结接口层数据流。
```

### 创建 Todo API 指令

```text
请根据现有 FastAPI OpenAPI 契约实现 Todo API 模块。
分别定义 TodoCreate、TodoUpdate、TodoDto 和前端 Todo 类型。
实现 list、get、create、patch、delete，并在统一边界转换 snake_case 和日期。
Axios 泛型不能被描述为运行时验证；如项目已有 Schema 库，复用它验证响应。
不要在 Vue 组件中重复 URL 和 DTO 转换，不使用 any。
为成功、404、422 和网络错误补充现有测试体系中的测试。
```

### 认证拦截器指令

```text
请为现有 Axios 实例实现认证和 Token 刷新。
先确认项目使用 Bearer Token 还是 HttpOnly Cookie，不要自行更换认证方案。
Bearer 模式只向可信 API 添加 Authorization。
401 刷新需要阻止刷新接口递归、每个请求最多重试一次，并让并发 401 共享同一个刷新 Promise。
刷新失败后清理会话并交给现有路由流程处理，避免拦截器直接操作页面 DOM。
评估非幂等请求重放风险，并编写并发刷新与失败场景测试。
```

### 错误模型指令

```text
请建立类型安全的前端 ApiError 模型。
区分 HTTP、网络、超时、取消、FastAPI 422 验证和未知错误。
保留安全的 status、code、message 和 fieldErrors，不向用户展示后端堆栈。
日志必须移除 Authorization、Cookie、密码和敏感请求体。
页面决定具体提示方式，底层拦截器不要对所有错误统一弹 Toast。
```

### 排错指令

```text
请根据浏览器 Network、Axios 错误对象和 FastAPI 日志定位请求失败。
依次检查最终 URL、Vite 代理、HTTP 方法、Params、Body、Content-Type、Cookie 或 Authorization、CORS、状态码和响应体。
区分没有响应、非 2xx、422 字段错误、超时和主动取消。
先说明可验证的根因，再做最小修改；不要用关闭 CORS、删除类型或无限增加超时掩盖问题。
```

## 37. 审核 AI 生成接口代码的清单

```text
[ ] 使用统一且边界明确的 Axios 实例
[ ] baseURL 来自公开环境配置或相对路径
[ ] 没有把秘密写入 VITE_ 环境变量
[ ] Params、JSON Body 和 FormData 使用正确位置
[ ] 没有全局强制错误的 Content-Type
[ ] 请求与响应 DTO 分离，未发送只读字段
[ ] Axios 泛型没有被当成运行时验证
[ ] catch 参数使用 unknown 并正确缩小类型
[ ] 401 刷新防止递归、重复和并发风暴
[ ] Cookie 模式考虑 withCredentials、CORS 与 CSRF
[ ] 取消、超时、网络错误和 HTTP 错误能够区分
[ ] 非幂等请求没有被无条件自动重试
[ ] FormData boundary 交给浏览器处理
[ ] 敏感 Header、Cookie 和请求体不会进入日志
[ ] API、Store 和 View 的职责没有混合
[ ] type-check、test 和 build 按项目配置通过
```

## 38. 面试表达速记

### Axios 的作用

> Axios 是 HTTP 客户端，提供实例配置、请求和响应转换、拦截器、超时、取消以及统一错误对象。项目通常在 API 层创建实例，而不是让每个组件直接拼接 URL 和认证 Header。

### Axios 与 fetch

> fetch 是浏览器原生 API，依赖更少，但非 2xx 默认不会拒绝 Promise，JSON 和公共流程需要自行封装。Axios 对统一实例、拦截器、错误对象和进度处理更方便。选择应服从项目复杂度和团队约定。

### Axios 泛型

> `get<T>` 只告诉 TypeScript 预期响应类型，不会在运行时验证 JSON。可信度要求高的接口仍需 Schema 校验、契约生成或显式解析。

### 拦截器

> 拦截器适合认证 Header、Token 刷新、追踪和基础错误标准化等跨接口逻辑。具体页面提示和业务处理不应全部塞入全局拦截器，并且错误分支必须继续 reject。

### 401 刷新

> 刷新流程需要排除刷新接口自身、限制每个请求只重试一次，并让并发 401 共享一个刷新任务。刷新失败后清理会话。重放创建类请求还要考虑重复副作用和幂等性。

### Cookie 与 Bearer

> Bearer Token 常通过 Authorization Header 发送，存储位置要考虑 XSS。HttpOnly Cookie 不能被 JavaScript 读取，但需要正确的 SameSite、Secure、CORS 和 CSRF 策略。两者都必须配合后端认证和授权。

### 错误分类

> HTTP 错误表示收到了非成功响应，网络错误表示没有正常响应，超时和取消需要单独识别。FastAPI 的请求字段校验常返回 422，前端可以映射为字段级错误。

### 取消与超时

> Axios 可以使用 AbortSignal 取消请求，并通过 timeout 限制等待时间。客户端停止等待不代表服务端事务一定停止，因此重要写操作仍要有幂等和一致性设计。

### 接口层分工

> Axios 实例处理基础协议配置，API 模块描述具体端点和 DTO 转换，Pinia 维护共享状态与业务流程，Vue 页面负责加载、错误、空状态和交互展示。

## 39. 本课技术地图

```text
Vue / Pinia
    ↓
API Modules
├─ Endpoint / Method
├─ Params / Body
├─ DTO Mapping
└─ Runtime Validation
    ↓
Axios Instance
├─ baseURL / timeout
├─ Request Interceptor
├─ Response Interceptor
├─ Auth / Refresh
├─ Cancel / Retry
└─ Error Normalization
    ↓
HTTP
├─ JSON
├─ FormData
├─ Blob
└─ Cookie / Authorization
    ↓
FastAPI
```

## 40. 下一步

下一课进入后端基础，合并学习 **Python 现代语法与后端开发基础**：

- Python 运行环境、虚拟环境与依赖管理；
- 变量、容器、函数、模块和异常；
- 类型注解、dataclass、Enum 与 Protocol；
- 同步、异步、协程和 `await`；
- 上下文管理器与迭代器；
- 面向对象在后端项目中的实际边界；
- Python 项目结构、格式化、类型检查和测试；
- 可直接交给 AI 的 Python 开发指令；
- 面试中的 Python 核心表达。
