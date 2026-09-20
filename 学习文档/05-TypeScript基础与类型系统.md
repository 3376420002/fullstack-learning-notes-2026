# 05｜TypeScript 基础与类型系统

> 目标：能读懂 Vue 项目类型、正确设计 API 数据，并让 AI 在不滥用 any 的前提下开发。

[← HTML 与 CSS](./04-HTML与CSS页面基础.md) · [学习首页](../README.md) · [前端工程化 →](./06-Node.js、pnpm、Vite与create-vue工程化.md)

## 1. TypeScript 的定位

```text
TypeScript = JavaScript + 静态类型系统 + 编译工具
```

TypeScript 代码通常先移除类型并转换为 JavaScript，再由浏览器或 Node.js 执行。

它能提前发现一部分错误，但不能保证没有业务 Bug，也不会自动验证外部 JSON。

## 2. 基础类型

```ts
const title: string = '学习 TypeScript'
const count: number = 3
const completed: boolean = false
const tags: string[] = ['frontend', 'types']
```

能推断时不必重复注解：

```ts
const title = '学习 TypeScript'
```

函数边界、公共类型和空数组更值得明确标注。

类型推断适合局部且含义明确的值；以下边界建议显式声明：

- 导出的函数、组件 Props 和公共 API；
- 空数组、初始值为 null 的状态；
- 回调参数无法从上下文推断时；
- 希望编译器约束返回值而不只是推断时。

```ts
const todos = ref<Todo[]>([])

export function findTodo(id: number): Todo | undefined {
  return todos.value.find((todo) => todo.id === id)
}
```

## 3. Object 类型

```ts
interface Todo {
  id: number
  title: string
  completed: boolean
  description: string | null
}
```

```ts
const todo: Todo = {
  id: 1,
  title: '学习类型',
  completed: false,
  description: null,
}
```

不要使用宽泛的 `object` 或 `{}` 代替具体业务结构。

只读数据可以明确表达不能被重新赋值：

```ts
interface Todo {
  readonly id: number
  title: string
}

const statuses = ['active', 'completed'] as const
type TodoStatus = (typeof statuses)[number]
```

`readonly` 是编译期限制，不会在运行时冻结对象；`as const` 会保留字面量并把属性变为只读。

## 4. interface 与 type

```ts
interface Todo {
  id: number
  title: string
}

type TodoStatus = 'active' | 'completed'
type TodoWithStatus = Todo & { status: TodoStatus }
```

实用习惯：

- 对象公共结构常用 interface；
- 联合、交叉、映射和别名常用 type；
- 团队保持一致比争论绝对优劣更重要。

## 5. 联合类型与类型收窄

```ts
function formatId(id: number | string): string {
  if (typeof id === 'number') {
    return id.toString()
  }
  return id.trim()
}
```

常用收窄方式：

```text
typeof
instanceof
in
判别字段
自定义类型守卫
```

## 6. 判别联合

```ts
type LoadState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string }
```

```ts
function renderState(state: LoadState<Todo[]>) {
  switch (state.status) {
    case 'success':
      return state.data.length
    case 'error':
      return state.message
    default:
      return state.status
  }
}
```

它比多个可能互相矛盾的 boolean 更能表达有效状态。

## 7. null、undefined 与可选字段

```ts
interface TodoUpdate {
  title?: string
  dueAt?: string | null
}
```

```text
title 缺失         不修改标题
dueAt: null        明确清空截止时间
dueAt 缺失         不修改截止时间
```

开启 `strictNullChecks` 后，null 和 undefined 必须被明确处理。

## 8. 函数类型

```ts
function createTodo(title: string): TodoCreate {
  return { title: title.trim() }
}

type TodoPredicate = (todo: Todo) => boolean

const isActive: TodoPredicate = (todo) => !todo.completed
```

可选参数：

```ts
function listTodos(page = 1, status?: TodoStatus) {}
```

不要用多个位置 boolean 参数表达复杂含义，优先使用配置对象。

## 9. 泛型

```ts
interface Page<T> {
  items: T[]
  page: number
  pageSize: number
  total: number
}

function first<T>(items: T[]): T | undefined {
  return items[0]
}
```

泛型让同一结构保留具体类型信息。不要为了“高级”给不需要复用的函数增加泛型。

## 10. Utility Types

```ts
type TodoPatch = Partial<Pick<Todo, 'title' | 'completed'>>
type TodoSummary = Pick<Todo, 'id' | 'title' | 'completed'>
type TodoWithoutDates = Omit<Todo, 'createdAt'>
type TodoById = Record<number, Todo>
```

Utility Types 适合从现有类型派生，减少重复；但 API 请求类型涉及不同语义时，单独定义往往更清楚。

当你既想检查对象是否符合某个类型，又想保留它自身更精确的推断时，可使用 `satisfies`：

```ts
const statusLabels = {
  active: '进行中',
  completed: '已完成',
} satisfies Record<TodoStatus, string>
```

它与 `as` 不同：`satisfies` 会检查结构，不是强行告诉编译器“相信我”。

## 11. unknown、any 与 never

```ts
function parseJson(text: string): unknown {
  return JSON.parse(text)
}
```

`unknown` 要求先验证再使用，比 `any` 安全。

```ts
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${String(value)}`)
}
```

`never` 可用于检查判别联合是否处理完整。

`any` 会关闭类型检查，只应出现在明确隔离的兼容边界，并说明原因。

## 12. 类型断言

```ts
const todo = value as Todo
```

断言不会改变运行时数据，也不会执行验证。

DOM 中可在已知上下文使用：

```ts
const input = event.target as HTMLInputElement
```

外部 JSON 不应只靠断言：

```ts
const data: unknown = await response.json()
const todo = todoSchema.parse(data)
```

TypeScript 类型在编译后通常会被删除，所以它无法自动阻止以下运行时问题：接口返回错误结构、字符串无法转为日期、网络失败、用户输入非法、对象被第三方脚本修改。来自 HTTP、localStorage 和表单的数据仍要验证或解析。

## 13. API 类型

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

export interface Todo {
  id: number
  title: string
  completed: boolean
  createdAt: Date
}
```

DTO 转换应集中在 API 模块，不要散落在组件中。

## 14. Vue 常见类型

```ts
const count = ref(0)
const todos = ref<Todo[]>([])
const selectedTodo = ref<Todo | null>(null)
```

Props：

```ts
const props = defineProps<{
  todo: Todo
  disabled?: boolean
}>()
```

Emits：

```ts
const emit = defineEmits<{
  toggle: [id: number]
  remove: [id: number]
}>()
```

## 15. tsconfig 重点

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true
  }
}
```

严格模式能提前暴露可空性和隐式 any 等问题。不要为消除报错而全局关闭 strict。

Vue 项目通常使用 `vue-tsc` 检查 `.vue` 文件：

```powershell
pnpm type-check
```

## 16. 模块与类型导入

每个含 `import` 或 `export` 的文件都是模块。只导入类型时使用 `import type`：

```ts
import type { Todo } from '@/types/todo'
import { listTodos } from '@/api/todos'

export type { Todo }
```

这样可以清楚区分“只给编译器使用的类型”和“运行时必须存在的值”，也避免某些构建配置下产生多余导入。

注意：interface 和 type 不能当运行时值使用。需要遍历状态时，应另外定义常量数组或枚举式对象。

## 17. 常见错误

- 大量使用 any；
- 用类型断言掩盖后端响应错误；
- 把可选字段、undefined 和 null 混为一谈；
- 创建、更新、响应共用一个巨大类型；
- 为所有简单函数添加无意义泛型；
- 页面能运行就跳过 `vue-tsc`；
- 前后端复制类型后长期不同步。

## 18. 给 AI 的开发指令

```text
请为现有 Vue 3 + TypeScript 项目实现 Todo 类型和组件。
分别定义 TodoCreate、TodoUpdate、TodoDto 和前端 Todo。
使用 strict 模式，不使用 any，不用类型断言掩盖外部 JSON。
Props 和 Emits 完整类型化，正确区分可选字段、null 和 undefined。
复用项目现有 API 转换层，完成后运行 type-check、lint 和 build。
```

审查指令：

```text
请检查类型是否准确表达运行时状态。
重点查找 any、过度断言、错误可空性、失真的 DTO、未穷尽联合和无必要泛型。
先修复数据契约或控制流根因，不要通过关闭 strict 消除报错。
```

## 19. 面试表达

> TypeScript 在 JavaScript 上增加静态类型，类型通常在编译后消失，运行时仍是 JavaScript。

> interface 常用于对象结构，type 能表达联合、交叉和映射；两者都能描述对象，选择应结合能力和团队规范。

> unknown 比 any 安全，因为使用前必须缩小类型；any 会传播并关闭类型检查。

> 泛型用于保留输入与输出之间的类型关系，不是为了让代码看起来复杂。

> 类型断言只影响编译器，不验证后端 JSON。外部数据需要运行时 Schema 或明确解析。

[进入下一课：前端工程化 →](./06-Node.js、pnpm、Vite与create-vue工程化.md)
