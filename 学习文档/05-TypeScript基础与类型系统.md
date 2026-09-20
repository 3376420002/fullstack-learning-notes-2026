# 05｜TypeScript 基础与类型系统

> 所属阶段：View 层程序设计基础  
> 本课合并知识点：TypeScript 类型系统、函数、对象建模、泛型、异步、DOM、表单与 API 契约  
> 前置知识：HTML、CSS、HTTP、JSON  
> 本课目标：能够使用严格 TypeScript 编写类型安全的前端逻辑，并正确描述 Todo API 的请求与响应。

[← HTML 与 CSS](./04-HTML与CSS页面基础.md) · [学习首页](../README.md) · [前端工程化 →](./06-Node.js、pnpm、Vite与create-vue工程化.md)

## 1. TypeScript 解决什么问题

JavaScript 是动态类型语言。下面的代码只有运行到对应位置时才可能暴露错误：

```js
function formatTodo(todo) {
  return todo.title.toUpperCase()
}

formatTodo({ title: null })
```

TypeScript 在 JavaScript 基础上增加静态类型检查：

```ts
interface Todo {
  title: string
}

function formatTodo(todo: Todo): string {
  return todo.title.toUpperCase()
}

formatTodo({ title: null }) // 类型检查失败
```

它的主要价值是：

- 在运行前发现大量类型错误；
- 让编辑器提供更准确的补全、跳转和重构；
- 把 JSON / API 结构表达成明确契约；
- 让函数输入、输出和状态变化更容易理解；
- 降低多人协作时的隐式约定成本。

TypeScript 不能替代测试、运行时校验和业务规则。类型正确的程序仍然可能有逻辑错误。

## 2. TypeScript 最终仍然运行 JavaScript

浏览器不会直接执行大多数 TypeScript 源码。TypeScript 需要经过检查和转换：

```text
源代码 .ts
   ↓ TypeScript 类型检查
发现类型错误
   ↓ 转换或由 Vite 等工具处理
JavaScript
   ↓
浏览器 / Node.js 执行
```

类型信息通常会在生成 JavaScript 时被移除：

```ts
function add(left: number, right: number): number {
  return left + right
}
```

生成的 JavaScript 近似为：

```js
function add(left, right) {
  return left + right
}
```

因此 TypeScript 类型不能直接验证网络传回的数据。API 数据仍需要后端 Pydantic 或前端运行时 Schema 进行校验。

## 3. 最小环境与严格模式

正式的 Node、pnpm 和 Vite 工程化会在下一课学习。本课可以先使用 TypeScript Playground 练习，也可以在已有 Node 环境中创建最小项目。

```powershell
pnpm init
pnpm add -D typescript
pnpm exec tsc --init
pnpm exec tsc --noEmit
```

推荐从严格模式开始：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noEmit": true,
    "skipLibCheck": true
  }
}
```

重要选项：

| 选项 | 作用 |
| --- | --- |
| `strict` | 开启一组严格类型检查 |
| `noUncheckedIndexedAccess` | 索引访问时考虑元素可能不存在 |
| `exactOptionalPropertyTypes` | 更严格地区分字段缺失和 undefined |
| `noEmit` | 只检查类型，不生成 JavaScript |
| `lib: ["DOM"]` | 提供浏览器 DOM 类型 |

不要为了让错误消失而关闭 strict。应理解错误揭示了哪个数据边界不明确。

## 4. 变量与类型推断

TypeScript 通常可以根据初始值推断类型：

```ts
const title = '学习 TypeScript' // string
let completed = false           // boolean
const priority = 3              // 3，字面量类型
let count = 0                   // number
```

不需要给每个明显变量重复写类型：

```ts
const title: string = '学习 TypeScript'
```

虽然正确，但类型注解没有提供额外信息。

适合显式标注类型的场景：

- 函数参数；
- 公共函数的返回值；
- 初始值不足以表达完整类型；
- API 契约和共享状态；
- 希望限制推断结果时。

```ts
let selectedTodo: Todo | null = null
```

## 5. 基础类型

```ts
const title: string = '学习 TypeScript'
const completed: boolean = false
const priority: number = 3
const emptyValue: null = null
const missingValue: undefined = undefined
```

类型名使用小写：

```ts
string
number
boolean
```

一般不要使用包装对象类型：

```ts
String
Number
Boolean
```

JavaScript 的普通 `number` 同时表示整数和浮点数。金额与超大整数仍需遵守上一课 JSON 中的边界设计。

## 6. const、let 与 var

默认使用 `const`，只有变量需要重新赋值时才使用 `let`：

```ts
const todos: Todo[] = []
let currentPage = 1
```

`const` 只保证变量绑定不能重新赋值，不代表对象内部完全不可变：

```ts
const todo = { title: '学习 TS', completed: false }
todo.completed = true // 允许
```

避免使用 `var`，因为它具有函数作用域和变量提升等容易产生误解的行为。

## 7. 数组与只读数组

```ts
const titles: string[] = ['HTML', 'CSS', 'TypeScript']
const priorities: Array<number> = [1, 2, 3]
```

两种数组写法等价，项目中保持一致即可。

对象数组：

```ts
const todos: Todo[] = [
  { id: 1, title: '学习 HTML', completed: true },
  { id: 2, title: '学习 TypeScript', completed: false },
]
```

只读数组表达调用方不应修改它：

```ts
function countCompleted(todos: readonly Todo[]): number {
  return todos.filter((todo) => todo.completed).length
}
```

`readonly` 是编译期约束，不会自动在运行时冻结数组。

在 `noUncheckedIndexedAccess` 下：

```ts
const first = todos[0] // Todo | undefined
```

因为数组可能为空，使用前需要检查。

## 8. 元组

元组表示长度和每个位置类型明确的数组：

```ts
type Coordinate = [x: number, y: number]

const point: Coordinate = [120, 30]
```

适合简短且位置语义稳定的数据。业务对象通常使用有字段名的 interface 更易读：

```ts
interface CoordinateObject {
  x: number
  y: number
}
```

不要把包含大量不同字段的数据硬塞进长元组。

## 9. 对象类型

```ts
const todo: {
  id: number
  title: string
  completed: boolean
} = {
  id: 1,
  title: '学习对象类型',
  completed: false,
}
```

类型需要复用时，应提取为 interface 或 type。

## 10. interface

```ts
interface Todo {
  id: number
  title: string
  description: string | null
  completed: boolean
  readonly createdAt: string
}
```

- `description: string | null`：字段必须存在，但允许为 null；
- `readonly createdAt`：不允许通过该类型重新赋值；
- readonly 只影响 TypeScript 检查，不等于运行时不可修改。

扩展接口：

```ts
interface Entity {
  id: number
  createdAt: string
}

interface Todo extends Entity {
  title: string
  completed: boolean
}
```

interface 适合描述对象结构，并允许声明合并。业务项目中应避免意外重复声明同名接口。

## 11. type

type 可以描述对象，也可以描述联合类型、元组、函数等：

```ts
type TodoStatus = 'todo' | 'doing' | 'done'

type Todo = {
  id: number
  title: string
  status: TodoStatus
}

type TodoPredicate = (todo: Todo) => boolean
```

interface 和 type 很多场景都能描述对象。实用选择原则：

- 对象和可扩展公共契约：interface 很自然；
- 联合、交叉、元组、映射类型：使用 type；
- 团队已有一致规范时，优先遵循团队规范；
- 不必为了选择它们中的一个进行无意义争论。

## 12. 可选字段

```ts
interface TodoCreate {
  title: string
  description?: string
}
```

`description?` 表示字段可以不存在。

它和下面的写法不同：

```ts
interface TodoCreateWithUndefined {
  title: string
  description: string | undefined
}
```

第二种写法要求字段存在，只是值可以为 undefined。

再对比 null：

```ts
interface TodoUpdate {
  title?: string
  assigneeId?: number | null
}
```

- `title` 可以不传；
- `assigneeId` 可以不传；
- 传 `assigneeId: null` 可以表示清空负责人。

## 13. 联合类型

联合类型表示值可以是多个类型之一：

```ts
let selectedId: number | null = null

selectedId = 42
selectedId = null
```

字面量联合能限制允许值：

```ts
type TodoStatus = 'todo' | 'doing' | 'done'

function updateStatus(status: TodoStatus): void {
  // ...
}

updateStatus('done')
updateStatus('finished') // 类型错误
```

对于来自 API 的固定字符串状态，字面量联合通常比普通 string 更精确。

## 14. 类型收窄

使用联合类型前，需要把范围缩小到当前分支确定的类型。

### 14.1 typeof

```ts
function formatId(id: string | number): string {
  if (typeof id === 'number') {
    return id.toFixed(0)
  }

  return id.trim()
}
```

### 14.2 等值判断

```ts
function printAssignee(name: string | null): string {
  if (name === null) {
    return '未分配'
  }

  return name
}
```

### 14.3 in

```ts
type ApiError = { error: string }
type TodoResponse = { todo: Todo }

function handleResponse(result: ApiError | TodoResponse): void {
  if ('error' in result) {
    console.error(result.error)
    return
  }

  console.log(result.todo.title)
}
```

### 14.4 自定义类型守卫

```ts
function isTodo(value: unknown): value is Todo {
  if (typeof value !== 'object' || value === null) {
    return false
  }

  const candidate = value as Record<string, unknown>

  return (
    typeof candidate.id === 'number' &&
    typeof candidate.title === 'string' &&
    typeof candidate.completed === 'boolean'
  )
}
```

简单守卫适合学习和小边界；复杂 API 契约通常使用专门的运行时 Schema 库或由 OpenAPI 生成客户端。

## 15. 判别联合

为联合类型提供共同的判别字段：

```ts
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string }
```

处理状态：

```ts
function renderState(state: RequestState<Todo[]>): string {
  switch (state.status) {
    case 'idle':
      return '尚未加载'
    case 'loading':
      return '加载中'
    case 'success':
      return `共 ${state.data.length} 条任务`
    case 'error':
      return state.message
  }
}
```

与多个互相矛盾的 boolean 相比，判别联合能避免不可能状态：

```ts
// 可能同时出现 isLoading=true 和 hasError=true
interface WeakState {
  isLoading: boolean
  hasError: boolean
  data?: Todo[]
}
```

## 16. 交叉类型

交叉类型组合多个类型：

```ts
type Timestamped = {
  createdAt: string
  updatedAt: string
}

type TodoWithTimestamps = Todo & Timestamped
```

如果两个类型包含无法兼容的同名字段，结果可能变成无法使用的 `never`。组合前要确认字段语义一致。

## 17. 函数参数与返回值

```ts
function createTitle(prefix: string, index: number): string {
  return `${prefix} ${index}`
}
```

可选参数：

```ts
function greet(name?: string): string {
  return name ? `你好，${name}` : '你好'
}
```

默认参数：

```ts
function listTodos(page = 1, pageSize = 20): void {
  console.log(page, pageSize)
}
```

对象参数通常比多个位置参数更清晰：

```ts
interface ListTodoOptions {
  page?: number
  pageSize?: number
  completed?: boolean
}

function listTodos(options: ListTodoOptions = {}): void {
  const { page = 1, pageSize = 20, completed } = options
  console.log(page, pageSize, completed)
}
```

## 18. void 与 never

`void` 表示调用方不使用有意义的返回值：

```ts
function logTodo(todo: Todo): void {
  console.log(todo.title)
}
```

`never` 表示函数不会正常结束，或某个分支理论上不可能出现：

```ts
function fail(message: string): never {
  throw new Error(message)
}
```

穷尽检查：

```ts
type Priority = 'low' | 'medium' | 'high'

function priorityLabel(priority: Priority): string {
  switch (priority) {
    case 'low':
      return '低'
    case 'medium':
      return '中'
    case 'high':
      return '高'
    default: {
      const unreachable: never = priority
      return unreachable
    }
  }
}
```

将来新增状态但忘记处理时，never 检查会提示错误。

## 19. 函数类型

```ts
type TodoFilter = (todo: Todo) => boolean

const isCompleted: TodoFilter = (todo) => todo.completed

function filterTodos(
  todos: readonly Todo[],
  predicate: TodoFilter,
): Todo[] {
  return todos.filter(predicate)
}
```

回调参数可以从函数类型中自动推断，无需重复标注。

## 20. 泛型

泛型让类型之间保持关系，而不是退化成 any。

```ts
function first<T>(items: readonly T[]): T | undefined {
  return items[0]
}

const firstTodo = first<Todo>(todos)
const firstTitle = first<string>(['HTML', 'CSS'])
```

编译器通常能够推断 T：

```ts
const firstTitle = first(['HTML', 'CSS'])
```

API 响应模型：

```ts
interface ApiResult<T> {
  data: T
  traceId: string
}

type TodoResult = ApiResult<Todo>
type TodoListResult = ApiResult<Todo[]>
```

泛型的目标是表达关系。不要为了“代码高级”给没有类型关系的函数强行增加泛型。

## 21. 泛型约束

```ts
interface Entity {
  id: number
}

function findById<T extends Entity>(
  items: readonly T[],
  id: number,
): T | undefined {
  return items.find((item) => item.id === id)
}
```

`T extends Entity` 表示 T 至少具有 id 字段，同时保留具体类型的其他字段。

## 22. keyof 与索引访问类型

```ts
function getProperty<T, K extends keyof T>(object: T, key: K): T[K] {
  return object[key]
}

const title = getProperty(todo, 'title')
```

- `keyof T` 得到 T 的字段名联合；
- `T[K]` 得到对应字段的值类型；
- 不存在的字段名会在编译时被拒绝。

这类能力常用于表格列、表单字段和通用组件。

## 23. 常用工具类型

### Partial

```ts
type TodoPatch = Partial<Todo>
```

把所有字段变成可选。注意：直接对完整 Todo 使用 Partial 也会允许修改 id 等不该修改的字段。

更稳妥：

```ts
type TodoUpdate = Partial<Pick<Todo, 'title' | 'description' | 'completed'>>
```

### Pick

```ts
type TodoSummary = Pick<Todo, 'id' | 'title' | 'completed'>
```

### Omit

```ts
type TodoCreate = Omit<Todo, 'id' | 'createdAt' | 'updatedAt'>
```

真实项目中仍可能需要显式定义 Create 类型，因为服务端输出模型和客户端输入模型往往并不是简单删字段关系。

### Record

```ts
type StatusLabels = Record<TodoStatus, string>

const statusLabels: StatusLabels = {
  todo: '待办',
  doing: '进行中',
  done: '已完成',
}
```

联合类型增加成员时，Record 会提醒补充映射。

## 24. as const 与 satisfies

`as const` 保留更精确的字面量，并使属性只读：

```ts
const statuses = ['todo', 'doing', 'done'] as const
type TodoStatus = (typeof statuses)[number]
```

`satisfies` 检查值符合某个类型，同时尽量保留值的具体推断：

```ts
const statusLabels = {
  todo: '待办',
  doing: '进行中',
  done: '已完成',
} satisfies Record<TodoStatus, string>
```

不要把 `as const` 当作深度运行时冻结；它仍然主要是类型层能力。

## 25. any、unknown 与类型断言

### any

```ts
let value: any
value.notExisting.deep.callAnything()
```

any 会绕过类型检查并向外传播，应尽量避免。

### unknown

```ts
let value: unknown

if (typeof value === 'string') {
  console.log(value.toUpperCase())
}
```

unknown 表示“类型尚不确定”，使用前必须收窄，适合不可信边界。

### 类型断言

```ts
const todo = value as Todo
```

类型断言不会转换或验证 value，只是告诉编译器“相信我”。过度断言会掩盖真实问题。

双重断言尤其危险：

```ts
const todo = value as unknown as Todo
```

它通常说明类型设计或数据校验被绕过了。

## 26. null、undefined 与非空断言

DOM 查询可能返回 null：

```ts
const form = document.querySelector<HTMLFormElement>('#todo-form')

if (!form) {
  throw new Error('未找到 Todo 表单')
}
```

非空断言：

```ts
const form = document.querySelector<HTMLFormElement>('#todo-form')!
```

`!` 不会在运行时检查元素存在。DOM 结构变化后可能出现空引用错误，因此边界处更推荐显式检查。

可选链：

```ts
const title = selectedTodo?.title
```

空值合并：

```ts
const displayName = user.nickname ?? user.name
```

`??` 只在左侧为 null 或 undefined 时使用右侧值；与 `||` 不同，它不会把 `0`、`false`、空字符串都视为缺失。

## 27. enum 还是字符串联合

可以使用 enum：

```ts
enum TodoStatusEnum {
  Todo = 'todo',
  Doing = 'doing',
  Done = 'done',
}
```

也可以使用字符串联合：

```ts
type TodoStatus = 'todo' | 'doing' | 'done'
```

面向 JSON API 时，字符串联合通常更轻量，和实际 JSON 值对应直观。enum 在需要运行时对象或团队已有规范时仍然可以使用。

## 28. 模块化

导出类型和函数：

```ts
// types/todo.ts
export interface Todo {
  id: number
  title: string
  completed: boolean
}

export type TodoCreate = Pick<Todo, 'title'>
```

```ts
// api/todos.ts
import type { Todo, TodoCreate } from '../types/todo'

export async function createTodo(input: TodoCreate): Promise<Todo> {
  // ...
}
```

`import type` 明确表示只导入类型，有助于构建工具移除纯类型依赖。

推荐按职责组织：

```text
src/
├─ api/          HTTP 请求封装
├─ components/   UI 组件
├─ types/        共享数据类型
├─ utils/        通用纯函数
└─ main.ts       应用入口
```

避免把所有类型和函数放进一个巨大文件。

## 29. Promise 与 async / await

异步函数返回 Promise：

```ts
async function loadTodos(): Promise<Todo[]> {
  const response = await fetch('/api/v1/todos')

  if (!response.ok) {
    throw new Error(`请求失败：${response.status}`)
  }

  return response.json() as Promise<Todo[]>
}
```

这里的 `as Promise<Todo[]>` 仍然只是类型假设，没有验证服务器数据。

调用：

```ts
async function initialize(): Promise<void> {
  try {
    const todos = await loadTodos()
    console.log(todos)
  } catch (error: unknown) {
    console.error(getErrorMessage(error))
  }
}
```

错误提取：

```ts
function getErrorMessage(error: unknown): string {
  if (error instanceof Error) {
    return error.message
  }

  return '发生未知错误'
}
```

catch 中使用 unknown 比 any 更安全。

## 30. Promise 并发

互不依赖的请求可以并发：

```ts
const [todos, profile] = await Promise.all([
  loadTodos(),
  loadProfile(),
])
```

不要在可以并发时无意串行等待：

```ts
const todos = await loadTodos()
const profile = await loadProfile()
```

但只有当两个操作相互独立，并且同时失败的处理符合需求时才使用 Promise.all。

## 31. Map 与 Set

Set 适合唯一值集合：

```ts
const selectedIds = new Set<number>()
selectedIds.add(42)
selectedIds.has(42)
selectedIds.delete(42)
```

Map 适合按键查找：

```ts
const todosById = new Map<number, Todo>()

for (const todo of todos) {
  todosById.set(todo.id, todo)
}

const selected = todosById.get(42) // Todo | undefined
```

Set 和 Map 不能直接按预期序列化为 JSON。发送到 API 前应转换为数组或普通对象。

## 32. DOM 元素类型

```ts
const titleInput = document.querySelector<HTMLInputElement>('#todo-title')
const form = document.querySelector<HTMLFormElement>('#todo-form')
const list = document.querySelector<HTMLUListElement>('#todo-list')
```

泛型参数告诉 TypeScript 期望的具体元素类型，但不会保证选择器在运行时一定匹配，因此仍要检查 null。

```ts
if (!titleInput || !form || !list) {
  throw new Error('页面结构不完整')
}
```

## 33. DOM 事件

```ts
const button = document.querySelector<HTMLButtonElement>('#complete-button')

button?.addEventListener('click', (event: MouseEvent) => {
  console.log(event.currentTarget)
})
```

表单提交：

```ts
form.addEventListener('submit', (event: SubmitEvent) => {
  event.preventDefault()
  console.log('准备提交')
})
```

- `target` 是事件最初发生的节点；
- `currentTarget` 是当前监听器绑定的节点；
- 事件冒泡时二者可能不同。

回调中的事件类型通常能根据 addEventListener 自动推断，不一定需要手写注解。

## 34. 从表单读取数据

```html
<form id="todo-form">
  <label for="todo-title">任务标题</label>
  <input id="todo-title" name="title" required />

  <button type="submit">创建</button>
</form>
```

```ts
interface TodoCreate {
  title: string
}

form.addEventListener('submit', (event) => {
  event.preventDefault()

  const formData = new FormData(form)
  const rawTitle = formData.get('title')

  if (typeof rawTitle !== 'string') {
    throw new Error('标题字段不存在')
  }

  const title = rawTitle.trim()

  if (title.length === 0 || title.length > 100) {
    throw new Error('标题长度必须为 1～100 个字符')
  }

  const input: TodoCreate = { title }
  console.log(input)
})
```

`FormData.get()` 返回 `FormDataEntryValue | null`，其中值还可能是 File，所以必须收窄。

## 35. 安全地创建 DOM

不可信文本应该赋给 textContent：

```ts
function createTodoElement(todo: Todo): HTMLLIElement {
  const item = document.createElement('li')
  const title = document.createElement('h3')

  title.textContent = todo.title
  item.append(title)

  return item
}
```

不要直接把用户输入拼进 innerHTML：

```ts
list.innerHTML = `<li>${todo.title}</li>`
```

如果 todo.title 包含恶意 HTML，可能产生 XSS。Vue 的普通文本插值会默认转义，但使用 `v-html` 时同样需要谨慎。

## 36. API 类型建模

请求模型：

```ts
export interface TodoCreate {
  title: string
  description?: string
  priority: 1 | 2 | 3
}

export interface TodoUpdate {
  title?: string
  description?: string | null
  completed?: boolean
}
```

响应模型：

```ts
export interface Todo {
  id: number
  title: string
  description: string | null
  priority: 1 | 2 | 3
  completed: boolean
  createdAt: string
  updatedAt: string
}
```

分页模型：

```ts
export interface Page<T> {
  items: T[]
  page: number
  pageSize: number
  total: number
}
```

错误模型：

```ts
export interface ApiErrorBody {
  error: {
    code: string
    message: string
    details: unknown
    traceId: string
  }
}
```

创建、更新与响应类型分离，避免客户端发送服务器维护字段。

## 37. 类型安全的请求函数

下面的泛型函数能够描述“调用方期望的响应类型”，但仍不提供运行时验证：

```ts
class HttpError extends Error {
  constructor(
    public readonly status: number,
    public readonly body: unknown,
  ) {
    super(`HTTP ${status}`)
  }
}

async function request<T>(
  url: string,
  options?: RequestInit,
): Promise<T> {
  const response = await fetch(url, {
    ...options,
    headers: {
      Accept: 'application/json',
      ...options?.headers,
    },
  })

  const contentType = response.headers.get('content-type') ?? ''
  const body: unknown = contentType.includes('application/json')
    ? await response.json()
    : await response.text()

  if (!response.ok) {
    throw new HttpError(response.status, body)
  }

  return body as T
}
```

创建 Todo：

```ts
async function createTodo(input: TodoCreate): Promise<Todo> {
  return request<Todo>('/api/v1/todos', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(input),
  })
}
```

实际工程中可由 Axios 统一封装请求，并通过 OpenAPI 生成或同步类型。无论使用哪种客户端，泛型都不会神奇地验证服务器响应。

## 38. 运行时边界

下面的代码编译通过：

```ts
const todo = await request<Todo>('/api/v1/todos/42')
```

但如果服务端实际返回：

```json
{
  "id": "not-a-number",
  "title": null
}
```

TypeScript 无法在运行时阻止它。

可靠边界通常包括：

```text
FastAPI / Pydantic 校验响应
        +
OpenAPI 作为接口契约
        +
前端生成或同步类型
        +
对不可信外部接口进行运行时校验
```

不能仅靠复制粘贴一份 interface 认为前后端永远一致。

## 39. 完整的 Todo 交互示例

HTML：

```html
<form id="todo-form">
  <label for="todo-title">任务标题</label>
  <input id="todo-title" name="title" maxlength="100" required />
  <button type="submit">创建</button>
</form>

<p id="todo-message" role="status"></p>
<ul id="todo-list"></ul>
```

TypeScript：

```ts
interface Todo {
  id: number
  title: string
  completed: boolean
}

interface TodoCreate {
  title: string
}

type TodoState =
  | { status: 'idle'; todos: Todo[] }
  | { status: 'submitting'; todos: Todo[] }
  | { status: 'error'; todos: Todo[]; message: string }

const form = document.querySelector<HTMLFormElement>('#todo-form')
const list = document.querySelector<HTMLUListElement>('#todo-list')
const message = document.querySelector<HTMLParagraphElement>('#todo-message')

if (!form || !list || !message) {
  throw new Error('页面缺少 Todo 应用所需元素')
}

let state: TodoState = {
  status: 'idle',
  todos: [],
}

function renderTodo(todo: Todo): HTMLLIElement {
  const item = document.createElement('li')
  const title = document.createElement('span')
  const toggle = document.createElement('button')

  title.textContent = todo.title
  toggle.type = 'button'
  toggle.textContent = todo.completed ? '恢复' : '完成'
  toggle.setAttribute('aria-label', `${toggle.textContent}：${todo.title}`)

  if (todo.completed) {
    title.style.textDecoration = 'line-through'
  }

  item.append(title, toggle)
  return item
}

function render(): void {
  list.replaceChildren(...state.todos.map(renderTodo))

  if (state.status === 'submitting') {
    message.textContent = '正在创建任务……'
    return
  }

  if (state.status === 'error') {
    message.textContent = state.message
    return
  }

  message.textContent = `共 ${state.todos.length} 个任务`
}

async function fakeCreateTodo(input: TodoCreate): Promise<Todo> {
  await new Promise((resolve) => window.setTimeout(resolve, 300))

  return {
    id: Date.now(),
    title: input.title,
    completed: false,
  }
}

form.addEventListener('submit', async (event) => {
  event.preventDefault()

  const formData = new FormData(form)
  const rawTitle = formData.get('title')

  if (typeof rawTitle !== 'string') {
    return
  }

  const title = rawTitle.trim()

  if (title.length === 0 || title.length > 100) {
    state = {
      status: 'error',
      todos: state.todos,
      message: '标题长度必须为 1～100 个字符',
    }
    render()
    return
  }

  state = {
    status: 'submitting',
    todos: state.todos,
  }
  render()

  try {
    const created = await fakeCreateTodo({ title })
    state = {
      status: 'idle',
      todos: [...state.todos, created],
    }
    form.reset()
  } catch (error: unknown) {
    state = {
      status: 'error',
      todos: state.todos,
      message: getErrorMessage(error),
    }
  }

  render()
})

render()
```

这个示例还没有使用 Vue，但已经包含未来 Vue 组件的重要思想：

- 明确的数据类型；
- 单一状态；
- 根据状态渲染界面；
- 表单输入转换为请求模型；
- 异步操作包含 loading 和 error 状态；
- 不直接把用户输入拼接进 innerHTML。

## 40. 结构类型系统

TypeScript 主要根据对象的结构判断兼容性，而不是类名：

```ts
interface HasTitle {
  title: string
}

const article = {
  title: 'TypeScript 入门',
  content: '...',
}

function printTitle(item: HasTitle): void {
  console.log(item.title)
}

printTitle(article)
```

article 即使没有显式声明为 HasTitle，只要结构满足要求就可以传入。

但对象字面量会进行额外属性检查：

```ts
function createTodo(input: TodoCreate): void {
  console.log(input)
}

createTodo({
  title: '学习 TS',
  unexpected: true, // 类型错误
})
```

它有助于发现字段拼写错误，但不能替代运行时边界校验。

## 41. 类型和值的命名空间

interface 和 type 主要存在于类型空间，不能直接作为运行时值使用：

```ts
interface Todo {
  id: number
}

// 不能写：if (value instanceof Todo)
```

因为 interface 编译后不存在。

运行时检查需要：

- `typeof`；
- `instanceof` 某个真实 class；
- `in`；
- 自定义类型守卫；
- 运行时 Schema。

## 42. 注释与文档

好的类型和命名能减少注释：

```ts
function calculateRemainingCount(todos: readonly Todo[]): number {
  return todos.filter((todo) => !todo.completed).length
}
```

注释应解释“为什么”，而不是翻译代码：

```ts
// 服务端使用字符串 ID，避免 64 位整数在 JavaScript 中丢失精度。
type TodoId = string
```

公共函数可使用 TSDoc：

```ts
/**
 * 按 ID 查找任务。
 * @returns 找不到时返回 undefined。
 */
function findTodoById(
  todos: readonly Todo[],
  id: number,
): Todo | undefined {
  return todos.find((todo) => todo.id === id)
}
```

## 43. 常见错误

### 错误一：到处使用 any

any 会让类型系统失效并向调用链扩散。未知数据优先使用 unknown，再进行收窄。

### 错误二：用 as 强行消除错误

断言只压制编译器，不修复运行时数据。

### 错误三：所有字段都加问号

类型虽然不再报错，但契约失去意义。必填、可选、可空应按业务真实含义建模。

### 错误四：把 null 和 undefined 混用

接口必须统一字段缺失、显式清空和默认值的语义。

### 错误五：API 请求只写 Promise<any>

调用方失去补全和约束，也容易把错误数据传入页面。

### 错误六：相信 interface 已经验证 JSON

类型在运行时被移除。外部数据必须在可信边界上校验。

### 错误七：滥用非空断言

`!` 隐藏 DOM 不存在或异步状态为空的问题。

### 错误八：用一个 Todo 类型覆盖所有阶段

TodoCreate、TodoUpdate、Todo、TodoSummary 的字段权限和完整度不同，应分别建模。

### 错误九：用多个 boolean 表示互斥状态

可能产生 loading、success、error 同时为 true 的无效组合。使用判别联合。

### 错误十：把 TypeScript 当成另一门完全不同的语言

TypeScript 是 JavaScript 的类型化超集。仍然需要理解 JavaScript 的运行时、对象、数组、Promise 和 DOM。

## 44. 本课实操

### 练习 A：类型建模

根据下面的 JSON 定义 TypeScript 类型：

```json
{
  "id": 42,
  "title": "学习 TypeScript",
  "description": null,
  "priority": 2,
  "completed": false,
  "tags": ["frontend", "typescript"],
  "createdAt": "2026-09-18T03:00:00Z"
}
```

要求：priority 只能为 1、2、3；createdAt 只读；description 可为 null 但不可缺失。

### 练习 B：请求模型

定义：

- TodoCreate；
- TodoUpdate；
- TodoSummary；
- Page<T>；
- ApiResult<T>。

注意区分服务器生成字段和客户端可修改字段。

### 练习 C：类型收窄

编写函数处理 `string | number | null`：

- null 返回“无 ID”；
- number 转为十进制字符串；
- string 去除首尾空格。

### 练习 D：判别联合

为请求状态定义四种状态：idle、loading、success、error。success 携带 Todo[]，error 携带 message。使用 switch 输出对应提示，并加入 never 穷尽检查。

### 练习 E：表单

从 FormData 中读取：

- title：非空字符串；
- priority：转换为 1、2、3；
- completed：复选框布尔值。

验证后组成 TodoCreate，不允许使用 any 或双重断言。

### 练习 F：API 边界

解释为什么下面代码不安全，并提出改进方案：

```ts
const todo = (await response.json()) as Todo
```

## 45. 部分练习参考答案

### 练习 A

```ts
type Priority = 1 | 2 | 3

interface Todo {
  id: number
  title: string
  description: string | null
  priority: Priority
  completed: boolean
  tags: string[]
  readonly createdAt: string
}
```

### 练习 C

```ts
function normalizeId(value: string | number | null): string {
  if (value === null) {
    return '无 ID'
  }

  if (typeof value === 'number') {
    return value.toString(10)
  }

  return value.trim()
}
```

### 练习 D

```ts
type LoadState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: Todo[] }
  | { status: 'error'; message: string }

function stateMessage(state: LoadState): string {
  switch (state.status) {
    case 'idle':
      return '尚未加载'
    case 'loading':
      return '正在加载'
    case 'success':
      return `共 ${state.data.length} 条任务`
    case 'error':
      return state.message
    default: {
      const unreachable: never = state
      return unreachable
    }
  }
}
```

## 46. 自测问题

进入下一课前，确保你能回答：

- TypeScript 类型在运行时是否仍然存在？
- 类型推断和类型注解分别适合什么场景？
- interface 与 type 有什么常见使用区别？
- 可选字段、undefined 和 null 有何区别？
- 联合类型为什么需要收窄？
- 判别联合如何避免无效状态？
- any 和 unknown 有什么区别？
- 类型断言为什么不是运行时转换或验证？
- 泛型解决了什么问题？
- `keyof`、`Pick`、`Omit`、`Record` 分别适合什么场景？
- `async` 函数的返回类型是什么？
- querySelector 为什么需要处理 null？
- 为什么不应把用户输入直接拼入 innerHTML？
- 为什么 `request<T>()` 仍然不能验证服务器响应？

## 47. 验收标准

完成本课后，你应当能够：

- 在 strict 模式下编写基础 TypeScript；
- 使用类型推断、显式注解、数组、元组和对象类型；
- 使用 interface 与 type 建模业务数据；
- 正确区分必填、可选、undefined 和 null；
- 使用联合类型、类型收窄和判别联合；
- 编写带明确参数和返回值的函数；
- 使用泛型和常用工具类型表达类型关系；
- 使用 unknown 处理不可信数据和异常；
- 使用 Promise、async / await 处理异步任务；
- 安全读取 DOM、事件和 FormData；
- 为 Todo API 分别设计请求、响应、分页和错误类型；
- 清楚说明编译期类型与运行时校验的边界。

## 48. 下一步

下一课建议把 **Node.js、pnpm、Vite 与工程配置** 合并学习。它们共同解决：

- TypeScript 源码在哪里执行和构建；
- 如何安装与锁定依赖；
- package.json 和 scripts 如何组织命令；
- Vite 如何提供开发服务器、热更新和生产构建；
- 环境变量、代理、代码检查与目录结构如何配置。

完成工程底座后，就可以进入 Vue 3 组件和响应式系统。
