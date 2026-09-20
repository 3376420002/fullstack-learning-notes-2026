# 08｜Vue Router 与 Pinia

> 所属阶段：Vue 应用架构  
> 本课合并知识点：SPA 路由、动态参数、嵌套路由、路由守卫、懒加载、Pinia Store、状态边界、异步 Action、登录态  
> 前置知识：Vue 3 核心基础、TypeScript、Vite  
> 学习目标：熟悉页面导航和跨组件状态管理，能够读懂、指挥 AI 编写相关代码，并能在面试中说明核心原理与边界。

[← Vue 3 核心基础](./07-Vue3核心基础与组件化.md) · [学习首页](../README.md) · [Axios 与接口层 →](./09-Axios与前后端接口层.md)

## 阅读方式

- 第 1～18 节建立 Vue Router 的完整心智模型；
- 第 19～31 节理解 Pinia 和状态归属；
- 第 32～35 节把路由、登录态与 Todo Store 串成实际项目；
- 第 36～39 节用于 AI 编程、代码审核和面试复习；
- 不需要背诵全部 API，重点掌握数据流、使用边界和常见错误。

## 1. 两项技术分别解决什么问题

Vue 本身负责组件和界面更新，但完整 SPA 还需要处理两类问题：

```text
Vue Router   URL 对应哪个页面，页面之间怎样导航
Pinia        多个组件或页面怎样共享和修改业务状态
```

它们与 Vue 的关系：

```text
浏览器 URL
    ↓
Vue Router 选择 View
    ↓
View 组合 Components
    ↓
Components / View 使用 Pinia Store
    ↓
Store 调用 API 并维护共享业务状态
```

路由不是全局状态仓库，Pinia 也不负责替代 URL。可分享、可刷新恢复的页面位置和筛选条件通常适合 URL；跨页面共享的业务数据与登录态通常适合 Store。

## 2. SPA 路由的基本原理

传统页面导航通常向服务器请求一份新 HTML：

```text
/todos → 服务器返回 todos.html
/profile → 服务器返回 profile.html
```

SPA 使用同一份应用入口，前端路由根据 URL 切换组件：

```text
/todos       → TodoListView
/todos/42    → TodoDetailView
/profile     → ProfileView
```

Vue Router 会监听浏览器地址变化、匹配路由记录，再把对应组件渲染到 `<RouterView>`。

它不会自动获取业务数据。进入 `/todos/42` 后，页面仍需通过 API 或 Store 加载 ID 为 42 的 Todo。

## 3. 安装和注册

create-vue 创建项目时可以直接选择 Vue Router。已有项目也可以安装：

```powershell
pnpm add vue-router
```

`src/main.ts`：

```ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'

const app = createApp(App)

app.use(createPinia())
app.use(router)
app.mount('#app')
```

`app.use` 安装插件，使组件树能够访问 Router 和 Pinia。

## 4. 最小路由配置

`src/router/index.ts`：

```ts
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '@/views/HomeView.vue'
import TodoListView from '@/views/TodoListView.vue'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      name: 'home',
      component: HomeView,
    },
    {
      path: '/todos',
      name: 'todo-list',
      component: TodoListView,
    },
  ],
})

export default router
```

每条路由记录主要包含：

| 字段 | 作用 |
| --- | --- |
| `path` | URL 匹配规则 |
| `name` | 稳定的程序化导航名称 |
| `component` | 匹配后渲染的页面组件 |
| `children` | 嵌套路由 |
| `redirect` | 重定向目标 |
| `meta` | 权限、标题等附加信息 |
| `props` | 把路由参数转换为组件 Props |

## 5. RouterView 与 RouterLink

`App.vue`：

```vue
<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <header>
    <nav aria-label="主导航">
      <RouterLink to="/">首页</RouterLink>
      <RouterLink :to="{ name: 'todo-list' }">任务</RouterLink>
    </nav>
  </header>

  <RouterView />
</template>
```

`RouterLink` 生成导航链接，Vue Router 会拦截普通左键导航并在不重新加载整页的情况下切换路由。

`RouterView` 是当前匹配页面的出口。没有它，即使 URL 匹配成功，页面组件也没有渲染位置。

Vue Router 会为活动链接添加类名，例如：

```css
.router-link-active {
  font-weight: 700;
}
```

## 6. History 模式

最常见配置：

```ts
createWebHistory(import.meta.env.BASE_URL)
```

URL 形式自然：

```text
https://example.com/todos/42
```

生产服务器必须支持 SPA 回退。当用户直接刷新 `/todos/42` 时，服务器应返回应用的 `index.html`，再由 Vue Router 匹配页面。

否则会出现：

```text
点击链接正常
直接刷新 404
```

Hash 模式：

```ts
createWebHashHistory()
```

URL 带 `#`：

```text
https://example.com/#/todos/42
```

Hash 后面的内容通常不会作为普通路径发送到服务器，因此不依赖服务器回退配置，但 URL 不如 History 模式自然。

## 7. 命名路由

```ts
router.push({
  name: 'todo-detail',
  params: { id: 42 },
})
```

相比硬编码：

```ts
router.push('/todos/42')
```

命名路由的优势是路径改变时，调用方仍可使用稳定的业务名称。

名称应保持唯一并表达页面身份。大型项目可以集中定义名称常量，减少拼写错误：

```ts
export const RouteName = {
  Home: 'home',
  TodoList: 'todo-list',
  TodoDetail: 'todo-detail',
  Login: 'login',
} as const
```

## 8. 动态路由参数

路由配置：

```ts
{
  path: '/todos/:id',
  name: 'todo-detail',
  component: () => import('@/views/TodoDetailView.vue'),
}
```

访问 `/todos/42` 时，`id` 是路由参数。

组件中读取：

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

const todoId = computed(() => {
  const value = Number(route.params.id)
  return Number.isInteger(value) && value > 0 ? value : null
})
</script>
```

路由参数来源于 URL，本质上需要按字符串输入处理。即使 TypeScript 给出类型提示，也要验证能否转换为有效业务 ID。

## 9. 用 Props 解耦页面与路由

路由配置：

```ts
{
  path: '/todos/:id',
  name: 'todo-detail',
  component: () => import('@/views/TodoDetailView.vue'),
  props: (route) => ({ id: Number(route.params.id) }),
}
```

页面组件：

```vue
<script setup lang="ts">
const props = defineProps<{
  id: number
}>()
</script>
```

这样页面主要依赖 `id` Prop，而不是直接依赖整个 Router。组件更容易单独测试和复用。

转换后的数字仍要做有效性校验。`Number('abc')` 会得到 `NaN`，配置 Props 不会自动完成业务验证。

## 10. Query 查询参数

适合放在 Query 中的内容：

- 搜索关键字；
- 页码和每页数量；
- 排序方式；
- 筛选条件；
- 当前标签页。

示例 URL：

```text
/todos?status=active&page=2&q=vue
```

读取：

```ts
const route = useRoute()

const page = computed(() => {
  const value = Number(route.query.page ?? 1)
  return Number.isInteger(value) && value > 0 ? value : 1
})
```

更新 Query：

```ts
const router = useRouter()

await router.push({
  name: 'todo-list',
  query: {
    ...route.query,
    status: 'completed',
    page: '1',
  },
})
```

Query 值可能是字符串、字符串数组、`null` 或缺失值，不应直接断言成业务类型。

筛选条件放入 URL 的好处是刷新可恢复、浏览器前进后退有效、链接可以分享。

## 11. Path、Params 与 Query 的选择

| 数据 | 推荐位置 | 示例 |
| --- | --- | --- |
| 资源身份 | Path Params | `/todos/42` |
| 页面筛选 | Query | `/todos?status=active` |
| 分页与排序 | Query | `?page=2&sort=created_at` |
| 敏感令牌 | 不放 URL | 使用安全 Cookie 或适当认证头 |
| 大型表单草稿 | 组件或 Store | 不应塞入 URL |

URL 会进入浏览记录、日志、监控和分享内容，因此不应放密码或长期访问令牌。

## 12. 程序化导航

```ts
import { useRouter } from 'vue-router'

const router = useRouter()

await router.push({ name: 'todo-list' })
await router.replace({ name: 'login' })
router.back()
router.forward()
```

区别：

```text
push      添加一条历史记录
replace   替换当前历史记录
back      返回上一条记录
```

登录成功后的普通跳转常用 `replace`，避免用户按返回键立即回到登录页。具体选择仍要符合产品导航预期。

不要使用 `window.location.href` 完成普通 SPA 内部导航，它会让整个应用重新加载。访问外部网站、下载或明确要求整页刷新时才使用浏览器原生导航。

## 13. 监听同一页面中的参数变化

从 `/todos/1` 导航到 `/todos/2` 时，Vue Router 可能复用同一个页面组件实例，因此不能只依赖首次挂载。

```ts
const route = useRoute()

watch(
  () => route.params.id,
  (id) => {
    loadTodo(Number(id))
  },
  { immediate: true },
)
```

也可以使用组件内守卫：

```ts
import { onBeforeRouteUpdate } from 'vue-router'

onBeforeRouteUpdate(async (to) => {
  await loadTodo(Number(to.params.id))
})
```

只监听真正需要的字段，不建议深度监听整个 `route` 对象。

## 14. 嵌套路由

账户区域可能包含公共布局：

```ts
{
  path: '/settings',
  component: () => import('@/layouts/SettingsLayout.vue'),
  children: [
    {
      path: '',
      name: 'settings-profile',
      component: () => import('@/views/settings/ProfileSettingsView.vue'),
    },
    {
      path: 'security',
      name: 'settings-security',
      component: () => import('@/views/settings/SecuritySettingsView.vue'),
    },
  ],
}
```

`SettingsLayout.vue` 必须提供内部出口：

```vue
<template>
  <aside>设置导航</aside>
  <main>
    <RouterView />
  </main>
</template>
```

子路由的相对路径 `security` 会组合为 `/settings/security`。如果写成 `/security`，它会成为绝对路径。

## 15. 重定向、别名和 404

重定向：

```ts
{
  path: '/tasks',
  redirect: { name: 'todo-list' },
}
```

别名让多个 URL 渲染同一路由记录，但地址不会自动变化：

```ts
{
  path: '/todos',
  alias: '/tasks',
  component: TodoListView,
}
```

404 兜底通常放在最后：

```ts
{
  path: '/:pathMatch(.*)*',
  name: 'not-found',
  component: () => import('@/views/NotFoundView.vue'),
}
```

前端 404 表示没有匹配页面。API 返回的 Todo 不存在则是业务请求的 404，两者属于不同层次。

## 16. 路由懒加载

```ts
{
  path: '/todos',
  name: 'todo-list',
  component: () => import('@/views/TodoListView.vue'),
}
```

动态 `import()` 会让构建工具把页面拆成可按需加载的代码块，从而减少首次下载体积。

通常路由级页面适合懒加载，首屏关键页面或很小的共享组件不需要为了拆包而过度配置。

懒加载只拆分前端代码，不会自动延迟 API 数据请求。页面何时获取数据仍由组件、Store 或路由数据策略决定。

## 17. 路由元信息 Meta

```ts
{
  path: '/profile',
  name: 'profile',
  component: () => import('@/views/ProfileView.vue'),
  meta: {
    requiresAuth: true,
    title: '个人中心',
  },
}
```

TypeScript 项目可以扩展类型：

```ts
// src/router/route-meta.d.ts
import 'vue-router'

export {}

declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
    title?: string
    roles?: string[]
  }
}
```

Meta 是路由配置数据，不是安全边界。前端隐藏页面只能改善交互，后端 API 仍必须独立验证身份和权限。

## 18. 导航守卫

全局守卫：

```ts
router.beforeEach((to) => {
  const authStore = useAuthStore()

  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    return {
      name: 'login',
      query: { redirect: to.fullPath },
    }
  }

  if (to.name === 'login' && authStore.isAuthenticated) {
    return { name: 'home' }
  }

  return true
})
```

常见守卫层次：

| 守卫 | 范围 |
| --- | --- |
| `beforeEach` | 所有导航 |
| `beforeResolve` | 导航确认前，异步组件和组件内守卫完成后 |
| `afterEach` | 导航完成后的统计、标题等，不改变导航结果 |
| `beforeEnter` | 某条路由记录 |
| `onBeforeRouteLeave` | 当前组件离开 |
| `onBeforeRouteUpdate` | 当前组件被复用且路由变化 |

现代写法优先通过返回值控制导航，不要把旧式 `next()` 与返回值混用，否则容易出现重复调用或导航悬挂。

## 19. 未保存表单的离开保护

```ts
import { onBeforeRouteLeave } from 'vue-router'

const dirty = ref(false)

onBeforeRouteLeave(() => {
  if (!dirty.value) return true
  return window.confirm('内容尚未保存，仍要离开吗')
})
```

这只处理应用内路由导航。关闭标签页或刷新浏览器需要另外处理 `beforeunload`，并在组件卸载时清理监听器。

离开保护不能代替自动保存或草稿机制，产品应根据数据价值选择合适策略。

## 20. 页面标题与滚动行为

设置标题：

```ts
router.afterEach((to) => {
  document.title = to.meta.title
    ? `${to.meta.title} · TodoLab`
    : 'TodoLab'
})
```

滚动行为：

```ts
const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(to, _from, savedPosition) {
    if (savedPosition) return savedPosition
    if (to.hash) return { el: to.hash, behavior: 'smooth' }
    return { top: 0 }
  },
})
```

浏览器前进后退时恢复位置，普通新页面回到顶部，是常见体验策略。

## 21. Pinia 的定位

Pinia 是 Vue 官方推荐的状态管理库。它用 Store 表达一组共享业务状态及其派生值和修改动作。

```text
State     原始状态
Getter    根据 State 得出的派生值
Action    修改状态或执行异步业务流程
```

Composition API 风格对应：

```text
ref / reactive   → State
computed         → Getter
function         → Action
```

Pinia 的价值不只是“让任何组件访问变量”，更重要的是建立明确的状态所有权和业务操作入口。

## 22. 创建 Pinia

安装：

```powershell
pnpm add pinia
```

注册：

```ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)
app.mount('#app')
```

Store 应在 Pinia 安装后的应用上下文中使用。路由守卫通常在实际导航发生时调用 Store；应用外工具或测试中可以显式传入 Pinia 实例。

## 23. Setup Store

`src/stores/todos.ts`：

```ts
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'

export interface Todo {
  id: number
  title: string
  completed: boolean
}

export const useTodoStore = defineStore('todos', () => {
  const todos = ref<Todo[]>([])
  const loading = ref(false)
  const errorMessage = ref('')

  const completedCount = computed(
    () => todos.value.filter((todo) => todo.completed).length,
  )

  const remainingCount = computed(
    () => todos.value.length - completedCount.value,
  )

  function addTodo(todo: Todo) {
    todos.value.push(todo)
  }

  function removeTodo(id: number) {
    todos.value = todos.value.filter((todo) => todo.id !== id)
  }

  function reset() {
    todos.value = []
    loading.value = false
    errorMessage.value = ''
  }

  return {
    todos,
    loading,
    errorMessage,
    completedCount,
    remainingCount,
    addTodo,
    removeTodo,
    reset,
  }
})
```

Setup Store 中需要把组件会使用的 State、Getter 和 Action 返回。未返回的响应式状态会影响开发工具、插件和 SSR 行为，因此不要把应属于 Store 的核心状态故意藏起来。

## 24. Option Store

Pinia 也支持 Options 风格：

```ts
export const useCounterStore = defineStore('counter', {
  state: () => ({
    count: 0,
  }),
  getters: {
    doubleCount: (state) => state.count * 2,
  },
  actions: {
    increment() {
      this.count++
    },
  },
})
```

Setup Store 与 Option Store 都是正式写法：

| 风格 | 特点 |
| --- | --- |
| Setup Store | 与 Composition API 一致，组合能力灵活 |
| Option Store | State、Getter、Action 分区直观，自带 `$reset()` |

项目中应保持主要风格一致。Setup Store 需要自行实现重置函数。

## 25. 在组件中使用 Store

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useTodoStore } from '@/stores/todos'

const todoStore = useTodoStore()
const { todos, loading, remainingCount } = storeToRefs(todoStore)
const { removeTodo } = todoStore
</script>

<template>
  <p>剩余 {{ remainingCount }} 项</p>
  <button
    v-for="todo in todos"
    :key="todo.id"
    type="button"
    :disabled="loading"
    @click="removeTodo(todo.id)"
  >
    删除 {{ todo.title }}
  </button>
</template>
```

直接解构 Store 的响应式属性会丢失响应式连接：

```ts
const { todos } = todoStore // 不推荐
```

State 和 Getter 使用 `storeToRefs`，Action 可以直接从 Store 解构，因为函数不需要转换为 ref。

## 26. State 可以直接修改，但仍需边界

Pinia 允许：

```ts
todoStore.loading = true
todoStore.todos.push(newTodo)
```

也支持批量修改：

```ts
todoStore.$patch({
  loading: false,
  errorMessage: '',
})
```

技术上可以直接修改不代表所有业务都应该散落修改。涉及接口、校验、多个字段一致性或可复用流程时，应通过 Action 表达：

```ts
await todoStore.createTodo(input)
```

简单 UI 赋值可以直接修改，重要业务变化集中到 Action，代码会更容易追踪。

## 27. 异步 Action

```ts
export const useTodoStore = defineStore('todos', () => {
  const todos = ref<Todo[]>([])
  const loading = ref(false)
  const errorMessage = ref('')

  async function fetchTodos() {
    loading.value = true
    errorMessage.value = ''

    try {
      const response = await fetch('/api/v1/todos')

      if (!response.ok) {
        throw new Error(`加载失败：${response.status}`)
      }

      todos.value = (await response.json()) as Todo[]
    } catch (error) {
      errorMessage.value = error instanceof Error
        ? error.message
        : '加载任务时出现未知错误'
      throw error
    } finally {
      loading.value = false
    }
  }

  return { todos, loading, errorMessage, fetchTodos }
})
```

关键原则：

- Store 维护共享请求状态；
- `finally` 恢复 loading；
- 是否重新抛出错误由调用约定决定；
- TypeScript 断言不会验证服务器 JSON；
- 重复请求、竞态和缓存策略需要按业务补充；
- API 调用变多后应提取独立 API 层，而不是全部堆在 Store 文件中。

## 28. API 层与 Store 的分工

推荐链路：

```text
View / Component
      ↓ 调用 Action
Pinia Store
      ↓ 编排业务状态
API Module
      ↓ HTTP
FastAPI
```

`src/api/todos.ts`：

```ts
import type { Todo } from '@/types/todo'

export async function getTodos(): Promise<Todo[]> {
  const response = await fetch('/api/v1/todos')

  if (!response.ok) {
    throw new Error(`加载失败：${response.status}`)
  }

  return response.json() as Promise<Todo[]>
}
```

Store：

```ts
import { getTodos } from '@/api/todos'

async function fetchTodos() {
  loading.value = true
  try {
    todos.value = await getTodos()
  } finally {
    loading.value = false
  }
}
```

API 层负责 HTTP 细节和数据适配，Store 负责跨组件业务状态，View 负责页面展示和用户交互。

## 29. 局部状态、URL 状态与 Store 状态

| 状态 | 推荐位置 | 原因 |
| --- | --- | --- |
| 输入框当前文字 | 组件 | 只服务当前交互 |
| 弹窗是否打开 | 组件或相关 composable | 生命周期局部 |
| Todo 搜索和页码 | Route Query | 可刷新恢复、可分享 |
| 当前 Todo ID | Route Params | 表示页面资源身份 |
| 当前用户 | Auth Store | 多页面共享 |
| Todo 缓存列表 | Todo Store | 多组件共享与统一更新 |
| 后端数据库数据 | 服务端 | 前端 Store 不是最终事实来源 |

判断顺序：

```text
只在一个组件使用       → 组件状态
需要随 URL 分享和恢复   → Route
多个远距离组件共同依赖  → Pinia
需要服务端永久保存      → API + 数据库
```

不要把所有状态都放进 Pinia。全局状态越多，生命周期和失效规则越难管理。

## 30. Getter 和带参数查询

普通 Getter：

```ts
const activeTodos = computed(
  () => todos.value.filter((todo) => !todo.completed),
)
```

需要参数时可以返回函数：

```ts
const findById = computed(
  () => (id: number) => todos.value.find((todo) => todo.id === id),
)
```

组件中：

```ts
const todo = computed(() => todoStore.findById(todoId.value))
```

返回函数的 Getter 不会自动为每个参数结果建立完整缓存。大量复杂查询应考虑规范化数据结构、索引 Map 或服务端查询，不要误以为 Getter 能自动解决所有性能问题。

## 31. Store 的订阅、持久化与清理

订阅状态变化：

```ts
const stop = todoStore.$subscribe((_mutation, state) => {
  localStorage.setItem('todo-preferences', JSON.stringify(state.preferences))
})

onBeforeUnmount(stop)
```

监听 Action：

```ts
todoStore.$onAction(({ name, after, onError }) => {
  const startedAt = performance.now()

  after(() => {
    console.log(name, performance.now() - startedAt)
  })

  onError((error) => {
    console.error(name, error)
  })
})
```

持久化不是 Pinia 默认自动完成的能力。可以手工保存少量偏好或选择合适插件，但需要明确：

- 保存哪些字段；
- 何时过期；
- 版本变化如何迁移；
- 退出登录时如何清理；
- 是否包含隐私或安全数据。

不要把密码、刷新令牌或完整敏感用户资料随意写入 `localStorage`。认证存储方案需要结合 XSS、CSRF、后端能力和部署环境设计。

## 32. 登录态 Store

```ts
// src/stores/auth.ts
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'

interface CurrentUser {
  id: number
  username: string
  roles: string[]
}

export const useAuthStore = defineStore('auth', () => {
  const user = ref<CurrentUser | null>(null)
  const initialized = ref(false)

  const isAuthenticated = computed(() => user.value !== null)

  async function restoreSession() {
    try {
      const response = await fetch('/api/v1/auth/me', {
        credentials: 'include',
      })

      user.value = response.ok
        ? ((await response.json()) as CurrentUser)
        : null
    } finally {
      initialized.value = true
    }
  }

  function clearSession() {
    user.value = null
    initialized.value = true
  }

  return {
    user,
    initialized,
    isAuthenticated,
    restoreSession,
    clearSession,
  }
})
```

应用启动时要区分：

```text
尚未确认登录态
已经确认未登录
已经确认已登录
```

如果只用一个布尔值，页面可能在会话恢复完成前错误跳转或闪烁。

## 33. 登录守卫的完整思路

```ts
router.beforeEach(async (to) => {
  const authStore = useAuthStore()

  if (!authStore.initialized) {
    await authStore.restoreSession()
  }

  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    return {
      name: 'login',
      query: { redirect: to.fullPath },
    }
  }

  const requiredRoles = to.meta.roles ?? []
  const hasRequiredRole = requiredRoles.every((role) =>
    authStore.user?.roles.includes(role),
  )

  if (requiredRoles.length > 0 && !hasRequiredRole) {
    return { name: 'forbidden' }
  }

  return true
})
```

登录成功后处理原目标：

```ts
const requestedRedirect = route.query.redirect
const redirect = typeof requestedRedirect === 'string'
  && requestedRedirect.startsWith('/')
  && !requestedRedirect.startsWith('//')
    ? requestedRedirect
    : '/'

await router.replace(redirect)
```

实际项目要防止开放重定向，只允许应用内部可信路径。后端仍必须对每个受保护 API 做认证和授权，不能信任前端守卫。

## 34. Todo 列表页面整合

```vue
<!-- src/views/TodoListView.vue -->
<script setup lang="ts">
import { computed, onMounted } from 'vue'
import { storeToRefs } from 'pinia'
import { useRoute, useRouter } from 'vue-router'
import { useTodoStore } from '@/stores/todos'

const route = useRoute()
const router = useRouter()
const todoStore = useTodoStore()

const { todos, loading, errorMessage } = storeToRefs(todoStore)

const status = computed(() => {
  const value = route.query.status
  return value === 'active' || value === 'completed' ? value : 'all'
})

const visibleTodos = computed(() => {
  if (status.value === 'active') {
    return todos.value.filter((todo) => !todo.completed)
  }

  if (status.value === 'completed') {
    return todos.value.filter((todo) => todo.completed)
  }

  return todos.value
})

async function setStatus(value: 'all' | 'active' | 'completed') {
  await router.push({
    name: 'todo-list',
    query: value === 'all' ? {} : { status: value },
  })
}

onMounted(async () => {
  if (todos.value.length === 0) {
    try {
      await todoStore.fetchTodos()
    } catch {
      // Store 已记录用于页面展示的错误状态。
    }
  }
})
</script>

<template>
  <main>
    <h1>任务列表</h1>

    <nav aria-label="任务筛选">
      <button type="button" @click="setStatus('all')">全部</button>
      <button type="button" @click="setStatus('active')">未完成</button>
      <button type="button" @click="setStatus('completed')">已完成</button>
    </nav>

    <p v-if="loading">加载中…</p>
    <p v-else-if="errorMessage" role="alert">{{ errorMessage }}</p>
    <ul v-else-if="visibleTodos.length">
      <li v-for="todo in visibleTodos" :key="todo.id">
        <RouterLink :to="{ name: 'todo-detail', params: { id: todo.id } }">
          {{ todo.title }}
        </RouterLink>
      </li>
    </ul>
    <p v-else>当前筛选条件下暂无任务</p>
  </main>
</template>
```

这里的状态归属很清楚：

```text
status          Route Query，可分享和刷新恢复
todos           Pinia Store，多页面共享
visibleTodos    页面 computed，由前两者派生
loading/error   Pinia Store，共享请求状态
```

## 35. 推荐项目结构

```text
src/
├─ api/
│  ├─ auth.ts
│  └─ todos.ts
├─ components/
├─ router/
│  ├─ index.ts
│  ├─ names.ts
│  └─ route-meta.d.ts
├─ stores/
│  ├─ auth.ts
│  └─ todos.ts
├─ types/
│  ├─ auth.ts
│  └─ todo.ts
├─ views/
│  ├─ HomeView.vue
│  ├─ LoginView.vue
│  ├─ TodoListView.vue
│  ├─ TodoDetailView.vue
│  ├─ ForbiddenView.vue
│  └─ NotFoundView.vue
├─ App.vue
└─ main.ts
```

依赖关系建议保持为：

```text
Router → Views
Views → Stores / Components
Stores → API / Types
API → HTTP
```

Store 不应直接操作页面 DOM，路由配置也不应堆叠具体业务请求实现。

## 36. 常见错误与准确理解

### 把所有筛选条件放入 Pinia

如果刷新后应该保留、可以分享或需要浏览器前进后退，优先放 Route Query。

### 把所有局部状态放入 Pinia

弹窗开关、单个输入框和一次性加载状态通常留在组件中。Store 不是默认状态容器。

### 直接解构 Store State

State 和 Getter 使用 `storeToRefs`；Action 可以直接解构。

### 只在 `onMounted` 加载动态参数页面

同一组件从一个参数导航到另一个参数时可能被复用，应监听具体参数或使用 `onBeforeRouteUpdate`。

### 把前端守卫当成权限系统

前端守卫只能控制界面导航，真正的数据权限必须由 FastAPI 等后端验证。

### 把访问令牌放进 Query

URL 会出现在历史、日志和分享链接中，不适合长期敏感凭据。

### History 模式部署后刷新 404

需要在静态服务器或网关配置 SPA 回退到 `index.html`，不是修改 Vue 组件能解决的问题。

### 每次进入页面都重复请求

需要明确缓存有效期、刷新条件和并发策略。不能只用 `todos.length > 0` 代表数据永远有效。

### Store 无限增长

跨用户、跨筛选或跨租户的数据要在身份变化时清理，并设计失效策略，避免展示上一个上下文的数据。

## 37. 指挥 AI 编写 Router 与 Pinia

### 创建路由指令

```text
请为现有 Vue 3 + TypeScript 项目配置 Vue Router 4。
创建首页、Todo 列表、Todo 详情、登录、403 和 404 路由。
Todo 详情使用 /todos/:id 和命名路由，路由页面使用动态 import 懒加载。
为需要登录的页面添加类型安全的 requiresAuth Meta。
使用项目现有目录和代码风格，不要创建重复 Router 实例。
说明生产 History 模式需要的 SPA 回退配置，并运行 type-check 和 build。
```

### 创建 Pinia Store 指令

```text
请在现有 Vue 3 + TypeScript + Pinia 项目中创建 Todo Setup Store。
先阅读已有 Todo 类型和 API 模块，不要重复定义接口契约。
Store 包含 todos、loading、errorMessage、remainingCount，以及 fetch、create、toggle、remove Action。
异步 Action 必须正确处理非 2xx、finally 和重复请求风险，不使用 any。
组件中用 storeToRefs 解构 State 和 Getter，Action 直接从 Store 获取。
不要把页面筛选条件放进 Store，它们由 Route Query 管理。
完成后运行现有类型检查、测试和构建。
```

### 登录守卫指令

```text
请基于现有 Auth Store 添加 Vue Router 登录守卫。
区分会话尚未初始化、未登录和已登录三种状态。
受保护页面使用 requiresAuth Meta，角色页面使用 roles Meta。
未登录时跳转登录页并保存应用内部 redirect，登录成功后安全返回原页面。
前端守卫只负责导航，不要把它描述成后端授权替代品。
避免无限重定向，并为 Meta 增加 TypeScript 类型扩展。
```

### 状态归属重构指令

```text
请审查当前 Vue 页面中的状态归属。
把可分享和刷新恢复的搜索、分页、排序放入 Route Query。
把多个页面共享的用户和业务实体状态放入 Pinia。
把输入框、弹窗和临时交互状态保留在组件。
保持现有行为和 URL 兼容，不要把所有状态机械迁入 Store。
输出重构前后的数据流，并执行相关测试。
```

### 排错指令

```text
请根据实际错误诊断 Vue Router 或 Pinia 问题。
检查 RouterView、路由匹配顺序、动态参数类型、History 回退、守卫重定向循环、Store 安装时机、直接解构导致的响应式丢失，以及旧用户状态未清理。
先给出根因证据，再实施最小修改。
不要顺带升级依赖或重写无关页面，修改后运行 type-check 和 build。
```

## 38. 审核 AI 生成代码的清单

```text
[ ] 应用只创建并安装预期的 Router 与 Pinia 实例
[ ] RouterView 位于正确布局层级
[ ] 路由名称唯一且导航不依赖散落的硬编码路径
[ ] 动态 Params 和 Query 做了运行时转换与验证
[ ] 页面级组件使用合理的懒加载
[ ] History 模式说明了生产 SPA 回退
[ ] 守卫没有无限重定向或混用 next 与返回值
[ ] 前端守卫没有被当作后端授权替代品
[ ] 局部状态、URL 状态和共享 Store 状态边界合理
[ ] State 与 Getter 使用 storeToRefs 解构
[ ] 异步 Action 覆盖 loading、error 和 finally
[ ] 登录态区分未初始化、未登录和已登录
[ ] 退出登录或身份变化时清理敏感业务状态
[ ] 没有把敏感令牌写入 URL 或随意持久化
[ ] type-check、test 和 build 按项目配置通过
```

## 39. 面试表达速记

### Vue Router 的工作方式

> Vue Router 监听浏览器地址变化，根据路由表匹配记录，并把对应组件渲染到 RouterView。RouterLink 和程序化导航通过 History API 切换 URL，普通 SPA 内部导航不需要重新加载整页。

### History 与 Hash 模式

> History 模式的 URL 更自然，但服务器必须把未知前端路径回退到 index.html，否则直接刷新会 404。Hash 模式把路由放在井号之后，对服务器配置要求较低，但 URL 会带井号。

### Params 与 Query

> Params 通常表示资源身份，例如 `/todos/:id`；Query 通常表示搜索、筛选、分页和排序。两者都来自 URL，进入业务逻辑前需要转换和验证，不能只依赖 TypeScript 类型断言。

### 路由懒加载

> 路由组件使用动态 import 后，构建工具可以按页面拆分代码，访问对应页面时再加载，从而降低首次加载体积。它解决前端代码加载，不会自动处理接口数据缓存。

### 导航守卫

> 守卫用于登录跳转、页面角色提示、未保存表单保护和导航统计。它只能控制前端导航，后端 API 仍必须执行真正的认证与授权。

### Pinia 的作用

> Pinia 用 Store 管理多个组件或页面共享的业务状态。State 保存原始状态，Getter 表达派生状态，Action 表达修改或异步流程。它的核心价值是明确状态归属和业务入口，而不是把所有变量全局化。

### `storeToRefs`

> 直接解构 Store 的响应式属性会失去与 Store 的响应式连接。State 和 Getter 使用 storeToRefs 转成 refs，Action 是普通函数，可以直接解构。

### Pinia 与 Composable

> Composable 主要复用组合式逻辑，每次调用通常产生独立状态；Pinia Store 具有明确身份，适合应用范围的共享业务状态。局部复用逻辑优先 composable，跨页面共享状态优先 Pinia。

### Pinia 与服务器数据

> Pinia 可以缓存和协调服务器数据，但不是数据库，也不是最终事实来源。项目仍要处理加载、失效、刷新、并发、身份切换和后端权限。

### 状态归属

> 局部交互状态留在组件，可分享和刷新恢复的页面状态放入 URL，多个远距离组件共享的业务状态放入 Pinia，需要永久保存的数据交给后端 API 和数据库。

## 40. 本课技术地图

```text
Vue Application
├─ Vue Router
│  ├─ Route Records
│  ├─ RouterLink / RouterView
│  ├─ Params / Query
│  ├─ Nested Routes
│  ├─ Lazy Loading
│  ├─ Meta / Guards
│  └─ History / SPA Fallback
├─ Pinia
│  ├─ State
│  ├─ Getter
│  ├─ Action
│  ├─ storeToRefs
│  ├─ Async State
│  └─ Subscription / Persistence
└─ 状态边界
   ├─ Component Local State
   ├─ URL State
   ├─ Shared Store State
   └─ Server Persistent State
```

## 41. 下一步

下一课学习 **Axios 与前后端接口层**：

- Axios 实例和基础配置；
- 请求、响应和错误类型；
- 拦截器的职责与边界；
- Token、Cookie 与刷新流程的准确理解；
- 请求取消、超时、重试与并发；
- API 模块和统一错误结构；
- Vue + Pinia 调用 FastAPI 的完整链路；
- 可直接交给 AI 的接口开发指令；
- 面试中的 HTTP 客户端与异常处理表达。

[进入下一课：Axios 与前后端接口层 →](./09-Axios与前后端接口层.md)
