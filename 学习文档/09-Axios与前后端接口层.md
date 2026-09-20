# 09｜Axios 与前后端接口层

> 目标：能封装可靠的前端 API 层，让 AI 正确处理类型、错误、认证、取消和上传。

[← Router 与 Pinia](./08-Vue-Router与Pinia.md) · [学习首页](../README.md) · [Python →](./10-Python现代语法与后端开发基础.md)

## 1. Axios 的位置

```text
Vue View
  ↓
Pinia / Composable
  ↓
API Module
  ↓
Axios Instance
  ↓ HTTP
FastAPI
```

组件不应重复拼接 baseURL、认证 Header 和错误结构。

## 2. 创建实例

```ts
import axios from 'axios'

export const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api/v1',
  timeout: 10_000,
  headers: {
    Accept: 'application/json',
  },
})
```

不要给所有请求全局设置 `Content-Type: application/json`，FormData 需要浏览器生成 boundary。

## 3. 常用请求

```ts
http.get('/todos', { params: { page: 1, status: 'active' } })
http.post('/todos', { title: '学习 Axios' })
http.patch('/todos/42', { completed: true })
http.delete('/todos/42')
```

```text
params   Query String
data     JSON Body 或 FormData
headers  单次请求 Header
signal   取消请求
timeout  单次超时
```

## 4. API 模块

```ts
export const todoApi = {
  async list(params?: TodoListParams): Promise<TodoDto[]> {
    const { data } = await http.get<TodoDto[]>('/todos', { params })
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

API 模块描述端点、方法、DTO 和转换；Pinia 管理共享状态；页面负责展示。

## 5. Axios 泛型的边界

```ts
const { data } = await http.get<TodoDto[]>('/todos')
```

泛型只告诉 TypeScript 预期类型，不会验证运行时 JSON。

重要外部数据使用运行时 Schema 或由 OpenAPI 生成并保持契约测试。

## 6. DTO 转换

```ts
interface TodoDto {
  id: number
  title: string
  completed: boolean
  created_at: string
}

interface Todo {
  id: number
  title: string
  completed: boolean
  createdAt: Date
}

function toTodo(dto: TodoDto): Todo {
  return {
    id: dto.id,
    title: dto.title,
    completed: dto.completed,
    createdAt: new Date(dto.created_at),
  }
}
```

转换集中在 API 边界，不要散落在 Vue 组件中。

## 7. 错误分类

```text
HTTP Error    收到非 2xx 响应
Network Error 没有收到正常 HTTP 响应
Timeout       超过客户端等待时间
Canceled      主动终止请求
Validation    FastAPI 422 字段错误
Business      HTTP 成功但业务结果不满足需求
```

```ts
try {
  await todoApi.list()
} catch (error: unknown) {
  if (axios.isAxiosError(error)) {
    console.log(error.response?.status)
  }
}
```

catch 使用 unknown，再通过类型守卫缩小。

## 8. 统一错误模型

```ts
interface ApiError {
  kind: 'http' | 'network' | 'timeout' | 'canceled' | 'validation' | 'unknown'
  message: string
  status?: number
  fieldErrors?: Record<string, string>
}
```

```text
401   恢复会话或登录
403   当前身份无权限
404   资源不存在
409   状态或版本冲突
422   请求字段不符合 Schema
429   限流
5xx   服务端或网关故障
```

底层负责分类，页面决定具体提示方式。不要在全局拦截器中对所有错误弹同一个 Toast。

## 9. 请求拦截器

```ts
http.interceptors.request.use((config) => {
  const token = getAccessToken()
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})
```

一个实例只应服务可信 API，避免把 Token 发到其他域名。拦截器注册一次，不要在组件每次渲染时注册。

## 10. 401 刷新原则

```text
收到 401
  ↓
确认不是 refresh 请求且未重试
  ↓
多个并发 401 共享一个 refresh Promise
  ↓ 成功
更新 Token 并重放一次请求
  ↓ 失败
清理会话并进入登录流程
```

必须避免刷新递归、无限重试和并发刷新风暴。创建、支付等非幂等请求还要评估重放风险。

## 11. Cookie 与 Bearer

```text
Bearer Token    Authorization Header，存储需考虑 XSS
HttpOnly Cookie JavaScript 不可读取，需配合 SameSite、Secure、CORS、CSRF
```

跨源 Cookie：

```ts
axios.create({
  baseURL: 'https://api.example.com',
  withCredentials: true,
})
```

`withCredentials` 不会自动创建登录态，CORS 也不能替代认证和权限。

## 12. 取消与竞态

```ts
watch(keyword, async (value, _oldValue, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  try {
    todos.value = await todoApi.search(value, controller.signal)
  } catch (error) {
    if (axios.isCancel(error)) return
    throw error
  }
})
```

取消可以避免旧请求覆盖新结果。客户端取消不保证服务端已经停止业务操作。

## 13. 重试与并发

更适合有限重试：

- GET 等幂等读取；
- 502、503、504；
- 网络瞬时故障；
- 有最大次数、退避并尊重 Retry-After。

支付、创建订单等非幂等 POST 不能无条件自动重试。

独立请求可以：

```ts
const [profile, todos] = await Promise.all([
  userApi.getProfile(),
  todoApi.list(),
])
```

## 14. 文件上传

```ts
const formData = new FormData()
formData.append('display_name', displayName)
formData.append('file', file)

await http.post('/users/me/avatar', formData)
```

通常不要手工设置 multipart Content-Type，浏览器会生成 boundary。

前端文件类型和大小校验只改善体验，后端必须重新验证。

## 15. 常见错误

- 每个组件创建一个 Axios 实例；
- 全局强制 JSON Content-Type；
- 把泛型当成运行时验证；
- 所有错误都显示“网络异常”；
- 所有 401 都无限刷新；
- Token 发给不可信域名；
- 无条件重试 POST；
- 取消请求后认为服务端操作已撤销；
- 日志记录 Authorization、Cookie 或密码。

## 16. 给 AI 的开发指令

```text
请为现有 Vue 3 + TypeScript 项目建立 Axios 接口层。
复用 Vite 环境变量，创建单例实例和 todos API 模块。
分别定义 TodoCreate、TodoUpdate、TodoDto，不把泛型当运行时验证。
错误区分 HTTP、网络、超时、取消和 FastAPI 422。
认证刷新要防止递归、重复和并发风暴，不无条件重放非幂等请求。
完成后运行 type-check、测试和 build。
```

## 17. 面试表达

> Axios 是 HTTP 客户端，项目通常通过统一实例和 API 模块管理 baseURL、认证、超时和错误。

> Axios 泛型只提供编译期类型，不验证服务器 JSON。

> 拦截器适合公共认证和基础错误处理，页面业务提示不应全部塞入全局拦截器。

> 401 刷新要限制每个请求只重试一次，并让并发请求共享一个刷新任务。

> 客户端取消只停止等待和结果处理，不保证服务端事务停止。

[进入下一课：Python 现代语法与后端开发基础 →](./10-Python现代语法与后端开发基础.md)
