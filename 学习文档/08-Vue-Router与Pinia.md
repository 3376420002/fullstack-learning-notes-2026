# 08｜Vue Router 与 Pinia

> 目标：能实现页面导航和共享状态，让 AI 正确区分 URL、组件和 Store 状态。

[← Vue 3](./07-Vue3核心基础与组件化.md) · [学习首页](../README.md) · [Axios →](./09-Axios与前后端接口层.md)

## 1. 两项技术的职责

```text
Vue Router   URL 对应哪个页面，怎样导航
Pinia        多个组件或页面怎样共享业务状态
```

```text
URL → Router → View → Component → Store → API
```

Router 不是状态仓库，Pinia 也不应替代 URL。

## 2. 最小路由配置

```ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      name: 'home',
      component: () => import('@/views/HomeView.vue'),
    },
    {
      path: '/todos',
      name: 'todo-list',
      component: () => import('@/views/TodoListView.vue'),
    },
    {
      path: '/todos/:id',
      name: 'todo-detail',
      component: () => import('@/views/TodoDetailView.vue'),
      props: (route) => ({ id: Number(route.params.id) }),
    },
    {
      path: '/:pathMatch(.*)*',
      name: 'not-found',
      component: () => import('@/views/NotFoundView.vue'),
    },
  ],
})
```

路由页面使用动态 import 可以按页面拆分代码。

## 3. RouterLink 与 RouterView

```vue
<nav>
  <RouterLink :to="{ name: 'home' }">首页</RouterLink>
  <RouterLink :to="{ name: 'todo-list' }">任务</RouterLink>
</nav>

<RouterView />
```

RouterView 是匹配页面的渲染出口。命名路由比散落的硬编码路径更容易维护。

## 4. Params 与 Query

```text
/todos/42                  Params 表示资源身份
/todos?status=active&page=2 Query 表示筛选和分页
```

```ts
const route = useRoute()
const router = useRouter()

const todoId = computed(() => Number(route.params.id))

await router.push({
  name: 'todo-list',
  query: { status: 'active', page: '1' },
})
```

Params 和 Query 来自 URL，使用前要转换和验证，不能直接相信 TypeScript。

## 5. History 模式

`createWebHistory` 生成自然 URL：

```text
https://example.com/todos/42
```

生产服务器必须把未知前端路径回退到 `index.html`，否则直接刷新会 404。

Hash 模式不依赖此回退，但 URL 会包含 `#`。

## 6. 导航守卫

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

  return true
})
```

前端守卫只控制页面导航，后端仍必须对 API 做认证和授权。

防止登录页和受保护页面互相无限重定向。

## 7. Pinia Store

```ts
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'

export const useTodoStore = defineStore('todos', () => {
  const todos = ref<Todo[]>([])
  const loading = ref(false)
  const errorMessage = ref('')

  const remainingCount = computed(
    () => todos.value.filter((todo) => !todo.completed).length,
  )

  async function fetchTodos() {
    loading.value = true
    errorMessage.value = ''
    try {
      todos.value = await todoApi.list()
    } catch (error) {
      errorMessage.value = normalizeApiError(error).message
      throw error
    } finally {
      loading.value = false
    }
  }

  return { todos, loading, errorMessage, remainingCount, fetchTodos }
})
```

```text
ref/reactive   State
computed       Getter
function       Action
```

## 8. storeToRefs

```ts
const todoStore = useTodoStore()
const { todos, loading, remainingCount } = storeToRefs(todoStore)
const { fetchTodos } = todoStore
```

State 和 Getter 使用 `storeToRefs` 保持响应式，Action 可以直接解构。

直接写 `const { todos } = todoStore` 会丢失响应式连接。

## 9. 状态放在哪里

| 状态 | 位置 |
| --- | --- |
| 输入框文字、弹窗开关 | 组件 |
| 可分享的搜索、筛选、页码 | Route Query |
| 当前资源 ID | Route Params |
| 当前用户、跨页面 Todo 缓存 | Pinia |
| 永久数据 | 后端数据库 |

不要把所有局部状态都放进 Pinia。

## 10. 登录态

```ts
const user = ref<CurrentUser | null>(null)
const initialized = ref(false)
const isAuthenticated = computed(() => user.value !== null)
```

必须区分：

```text
尚未检查会话
已经确认未登录
已经确认已登录
```

否则应用启动时可能错误跳转或闪烁。

退出登录时要清理与用户有关的 Store，避免下一个用户看到旧数据。

## 11. 常见错误

- RouterView 放错布局层级或缺失；
- History 模式部署后没有 SPA 回退；
- Params 和 Query 不做转换；
- 只在 onMounted 加载动态参数页面；
- 把前端守卫当作权限系统；
- 直接解构 Store State；
- 所有筛选条件都放 Pinia；
- 登录态未初始化就执行守卫；
- 用户退出后未清理业务 Store。

## 12. 给 AI 的开发指令

```text
请为现有 Vue 3 项目实现 Todo 路由和 Pinia Store。
列表、详情、登录和 404 使用命名路由，页面动态 import。
Todo ID 放 Params，筛选和分页放 Query，共享 Todo 数据放 Pinia。
State 和 Getter 用 storeToRefs，Action 直接调用。
登录守卫区分未初始化、未登录、已登录，不能替代后端权限。
说明 History 模式的生产回退，完成后运行 type-check、test 和 build。
```

## 13. 面试表达

> Vue Router 监听 URL，根据路由表匹配组件并渲染到 RouterView，普通 SPA 导航不重新加载整页。

> Params 表示资源身份，Query 表示筛选、分页和排序；二者都来自 URL，需要运行时转换和验证。

> Pinia 用 State、Getter、Action 管理共享业务状态，核心是明确状态所有权，不是把所有变量全局化。

> storeToRefs 用于保持解构后的 State 和 Getter 响应式，Action 可以直接解构。

> 局部交互放组件，可分享状态放 URL，跨页面业务状态放 Pinia，永久数据放后端。

[进入下一课：Axios 与前后端接口层 →](./09-Axios与前后端接口层.md)
