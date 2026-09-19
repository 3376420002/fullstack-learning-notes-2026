# 03｜JSON 数据格式

> 所属阶段：前后端通信基础  
> 前置知识：MVC、HTTP / HTTPS  
> 本课目标：能够正确读写 JSON，理解它与 TypeScript、Python、数据库类型的差异，并为前后端设计稳定的数据契约。

[← HTTP 与 HTTPS](./02-HTTP与HTTPS.md) · [学习首页](../README.md) · [HTML 与 CSS →](./04-HTML与CSS页面基础.md)

## 1. JSON 在全栈系统中的位置

上一课学习了 HTTP 请求与响应，但 HTTP 只负责传输，并不规定业务数据一定长什么样。

在本课程技术栈中，一次典型的数据流是：

```text
Vue / TypeScript 对象
        ↓ JSON.stringify 或 Axios 自动序列化
HTTP Request Body
        ↓ FastAPI 解析与 Pydantic 校验
Python 对象
        ↓ Service 与 SQLAlchemy
数据库记录
        ↓ 反向转换
HTTP Response JSON
        ↓ Axios 解析
Vue / TypeScript 对象
```

JSON 是这条链路中的**数据交换格式**。它不是数据库，也不是编程语言中的类，更不负责业务校验。

## 2. JSON 是什么

JSON 全称 JavaScript Object Notation，是一种文本数据格式。它适合表示对象、列表和嵌套关系，也能被几乎所有主流编程语言解析。

一个合法的 Todo JSON：

```json
{
  "id": 42,
  "title": "学习 JSON",
  "completed": false,
  "tags": ["frontend", "backend"],
  "assignee": null
}
```

JSON 文本有几个重要特点：

- 对象字段名必须使用双引号；
- 字符串必须使用双引号；
- 不能写注释；
- 最后一项后面不能多写逗号；
- JSON 只描述数据，不包含函数或类行为；
- JSON 本身没有日期、BigInt、Decimal、undefined 等专用类型。

## 3. JSON 的六种数据类型

JSON 只有六种数据类型。

| 类型 | 示例 | 说明 |
| --- | --- | --- |
| object | `{"name": "Alice"}` | 无序的键值集合，键必须是字符串 |
| array | `[1, 2, 3]` | 有序值列表 |
| string | `"hello"` | 双引号字符串 |
| number | `42`、`3.14`、`-1` | JSON 不区分整数与浮点数 |
| boolean | `true`、`false` | 必须小写 |
| null | `null` | 明确表示空值 |

下面这些不是合法 JSON 值：

```text
undefined
NaN
Infinity
new Date()
10n
function () {}
```

它们必须先转换为 JSON 支持的类型。

## 4. JSON 与 JavaScript 对象不是一回事

JavaScript 对象：

```js
const todo = {
  id: 42,
  title: '学习 JSON',
  completed: false,
}
```

JSON 文本：

```json
{
  "id": 42,
  "title": "学习 JSON",
  "completed": false
}
```

区别包括：

- JavaScript 对象存在于程序内存中，JSON 是字符串；
- JavaScript 可以使用单引号、方法、Symbol、undefined 等，JSON 不可以；
- JavaScript 对象的字段名在符合规则时可以省略引号，JSON 不可以；
- JSON 需要经过解析才能变成程序中的对象。

因此下面的值不是 JSON 文本：

```js
{ title: '学习 JSON' }
```

它只是 JavaScript 对象字面量。

## 5. JSON 的嵌套结构

对象和数组可以组合成复杂结构：

```json
{
  "id": 42,
  "title": "完成全栈课程",
  "owner": {
    "id": 7,
    "name": "Alice"
  },
  "tags": [
    {
      "id": 1,
      "name": "学习"
    },
    {
      "id": 2,
      "name": "后端"
    }
  ]
}
```

嵌套层级并非越深越好。层级过深会带来：

- 前端读取和更新困难；
- 类型定义复杂；
- 响应体积变大；
- ORM 关系加载容易产生额外查询；
- 相同实体可能在多个位置重复出现。

接口应围绕页面真实需要返回数据，不要无边界地把整个数据库关系树序列化出去。

## 6. 序列化与反序列化

### 6.1 序列化

把程序中的对象转换成 JSON 文本：

```ts
const todo = {
  id: 42,
  title: '学习 JSON',
  completed: false,
}

const text = JSON.stringify(todo)
console.log(text)
```

输出：

```json
{"id":42,"title":"学习 JSON","completed":false}
```

Axios 在发送 JSON 请求时通常会自动完成序列化，因此一般不需要手动调用 `JSON.stringify`。

### 6.2 反序列化

把 JSON 文本转换成 JavaScript 值：

```ts
const text = '{"id":42,"title":"学习 JSON","completed":false}'
const todo = JSON.parse(text)
```

Axios 也通常会根据响应的 Content-Type 自动解析 JSON。

### 6.3 Python 中的转换

```python
import json

todo = {
    "id": 42,
    "title": "学习 JSON",
    "completed": False,
}

text = json.dumps(todo, ensure_ascii=False)
restored = json.loads(text)
```

- `json.dumps()`：Python 对象转换为 JSON 字符串；
- `json.loads()`：JSON 字符串转换为 Python 对象；
- `ensure_ascii=False`：输出时保留可读中文，而不是全部转成 Unicode 转义。

FastAPI 会结合 Pydantic 自动完成大量转换工作，但你仍然需要理解转换边界。

## 7. JSON、TypeScript 与 Python 类型映射

| JSON | TypeScript | Python | 说明 |
| --- | --- | --- | --- |
| object | `object`、接口或类型别名 | `dict`、Pydantic Model | 实际项目应定义明确结构 |
| array | `T[]`、`Array<T>` | `list[T]` | 元素最好保持一致类型 |
| string | `string` | `str` | 日期、UUID 常以字符串传输 |
| number | `number` | `int` 或 `float` | JS 只有一种普通 number 类型 |
| boolean | `boolean` | `bool` | JSON 使用小写 true/false |
| null | `null` | `None` | 不等同于字段不存在 |

TypeScript 契约示例：

```ts
export interface Todo {
  id: number
  title: string
  completed: boolean
  assignee: UserSummary | null
  tags: Tag[]
}
```

Pydantic 契约示例：

```python
from pydantic import BaseModel

class TodoOut(BaseModel):
    id: int
    title: str
    completed: bool
    assignee: UserSummary | None
    tags: list[Tag]
```

两端类型定义的作用是描述和校验 JSON 契约，而不是改变 JSON 本身的六种类型。

## 8. 字段缺失、null 与空值

以下三份数据含义不同。

字段缺失：

```json
{}
```

字段存在但明确为空：

```json
{
  "assignee": null
}
```

字段存在且为空字符串：

```json
{
  "assignee": ""
}
```

在更新接口中，这个区别非常重要：

- 字段缺失：用户没有要求修改；
- 字段为 `null`：用户要求清空该值；
- 字段为空字符串：用户提交了字符串，只是内容为空。

PATCH 请求示例：

```json
{
  "assignee_id": null
}
```

它可以表示“取消任务负责人”。如果该字段完全不出现，则表示“不修改负责人”。

TypeScript 中也要区分：

```ts
interface TodoUpdate {
  title?: string
  assigneeId?: number | null
}
```

- `?` 表示字段可以缺失；
- `| null` 表示字段出现时允许明确为空。

## 9. undefined 的序列化行为

JavaScript 的 `undefined` 不属于 JSON。

```ts
JSON.stringify({ name: undefined, age: 18 })
```

结果是：

```json
{"age":18}
```

对象中值为 undefined 的字段会被忽略。

但在数组中：

```ts
JSON.stringify([1, undefined, 3])
```

结果是：

```json
[1,null,3]
```

所以不要用 undefined 和 null 混乱地表达同一个业务状态。接口契约应明确规定字段是否可缺失、是否可为 null。

## 10. 日期和时间

JSON 没有日期类型，通常使用符合 ISO 8601 的字符串：

```json
{
  "created_at": "2026-09-17T10:30:00Z"
}
```

其中 `Z` 表示 UTC。也可以携带明确偏移量：

```json
{
  "created_at": "2026-09-17T19:30:00+09:00"
}
```

前端收到后仍然是字符串：

```ts
const date = new Date(todo.createdAt)
```

设计建议：

- 数据库存储具有时区语义的时间；
- API 返回带时区或 UTC 的明确时间字符串；
- 前端在显示层转换为用户所在时区；
- 不发送含糊的 `2026-09-17 10:30:00`，因为它没有说明时区；
- 日期字段的格式应在接口契约中固定。

## 11. 数字、大整数与金额

JSON 只有 number，但不同语言处理数字的方式不完全相同。

JavaScript 普通 number 无法精确表示所有超大整数。超过安全整数范围的数据库 ID 可能在前端丢失精度。

不安全示例：

```json
{
  "id": 9223372036854775807
}
```

更稳妥的跨语言表示：

```json
{
  "id": "9223372036854775807"
}
```

金额也不应随意使用二进制浮点数进行精确计算。常见方案有：

使用最小货币单位整数：

```json
{
  "amount_minor": 1999,
  "currency": "CNY"
}
```

或者使用十进制字符串：

```json
{
  "amount": "19.99",
  "currency": "CNY"
}
```

具体方案要由团队统一，并在前后端使用相应的精确计算类型。

JSON 不支持 `NaN` 和 `Infinity`。出现非有限数字时，应在业务层处理，而不是把它们强行写入响应。

## 12. 循环引用

普通对象可以形成循环引用，但 JSON 是树状结构，无法表达循环：

```ts
const user: Record<string, unknown> = { name: 'Alice' }
user.self = user

JSON.stringify(user) // 抛出错误
```

数据库实体之间经常存在双向关系，例如 User 拥有 Todos，而 Todo 又引用 User。接口响应不能直接无限展开这些关系。

正确思路是设计专门的输出 Schema：

```json
{
  "id": 42,
  "title": "学习 JSON",
  "owner": {
    "id": 7,
    "name": "Alice"
  }
}
```

这里返回的是 `UserSummary`，而不是把 User 的所有 Todo 再嵌套回来。

## 13. 字段命名规范

JavaScript / TypeScript 常使用 camelCase：

```json
{
  "createdAt": "2026-09-17T10:30:00Z"
}
```

Python 和数据库常使用 snake_case：

```json
{
  "created_at": "2026-09-17T10:30:00Z"
}
```

两种风格都可以，关键是项目统一。不要让同一接口出现：

```json
{
  "created_at": "...",
  "updatedAt": "...",
  "ownerID": 7
}
```

课程初期可以采用后端一致的 snake_case，减少映射复杂度。项目如果决定对外使用 camelCase，则应在统一的 Schema 或序列化层做别名转换，不要在每个页面手工改字段名。

## 14. 请求 Schema 与响应 Schema 分离

创建请求：

```json
{
  "title": "学习 JSON"
}
```

创建响应：

```json
{
  "id": 42,
  "title": "学习 JSON",
  "completed": false,
  "created_at": "2026-09-17T10:30:00Z"
}
```

请求和响应不应该共用一个“万能模型”，因为：

- `id` 由服务器生成，客户端创建时不应传入；
- `created_at` 由服务器维护；
- 密码确认字段只在请求中存在；
- 内部字段不能暴露给客户端；
- 更新请求通常允许字段缺失。

推荐的 Schema 命名：

```text
TodoCreate  创建请求
TodoUpdate  更新请求
TodoOut     单个资源响应
TodoListOut 列表响应
```

## 15. 一个完整的 Todo 数据契约

### 15.1 创建请求

```json
{
  "title": "学习 JSON",
  "description": "完成第 3 课",
  "tag_ids": [1, 2]
}
```

### 15.2 更新请求

```json
{
  "completed": true
}
```

### 15.3 单个资源响应

```json
{
  "id": 42,
  "title": "学习 JSON",
  "description": "完成第 3 课",
  "completed": true,
  "owner": {
    "id": 7,
    "name": "Alice"
  },
  "tags": [
    {
      "id": 1,
      "name": "全栈"
    },
    {
      "id": 2,
      "name": "基础"
    }
  ],
  "created_at": "2026-09-17T10:30:00Z",
  "updated_at": "2026-09-17T11:00:00Z"
}
```

### 15.4 分页列表响应

```json
{
  "items": [
    {
      "id": 42,
      "title": "学习 JSON",
      "completed": true
    }
  ],
  "page": 1,
  "page_size": 20,
  "total": 1
}
```

列表中的资源可以使用精简结构，避免每一项都携带页面不需要的详细关系。

### 15.5 错误响应

```json
{
  "error": {
    "code": "TODO_NOT_FOUND",
    "message": "任务不存在",
    "details": null,
    "trace_id": "fdb22e08"
  }
}
```

稳定的机器可读错误码比只返回中文句子更适合前端判断；`trace_id` 可以帮助关联后端日志。

## 16. JSON 与 HTTP 配合

客户端发送 JSON 时，应使用：

```http
Content-Type: application/json
```

客户端希望得到 JSON 时，可以声明：

```http
Accept: application/json
```

完整示例：

```http
POST /api/v1/todos HTTP/1.1
Host: localhost:8000
Content-Type: application/json
Accept: application/json

{
  "title": "学习 JSON"
}
```

如果请求头声称是 JSON，但 Body 实际使用了单引号、注释或多余逗号，服务端解析会失败。

## 17. JSON 与 Form Data 的选择

| 场景 | 推荐格式 |
| --- | --- |
| 普通创建和更新接口 | `application/json` |
| 嵌套对象或数组 | `application/json` |
| 上传文件 | `multipart/form-data` |
| 文件与少量字段一起上传 | `multipart/form-data` |
| 传统浏览器表单 | `application/x-www-form-urlencoded` |

不要把文件内容直接塞入普通 JSON，除非接口明确要求 Base64。Base64 会增加体积和内存开销，普通文件上传应优先使用 multipart。

## 18. JSON Schema、Pydantic 与 OpenAPI

只有 JSON 示例还不够，因为示例无法完整表达：

- 哪些字段必填；
- 字符串长度；
- 数值范围；
- 枚举值；
- 嵌套结构；
- 字段是否可为 null。

Pydantic 可以把 Python 类型与约束转换成 JSON Schema，FastAPI 再把它们写入 OpenAPI 文档。

```python
from pydantic import BaseModel, Field

class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    priority: int = Field(default=1, ge=1, le=5)
```

它表达的契约包括：

```text
title    必填字符串，长度 1～100
priority 可选整数，默认 1，范围 1～5
```

这套契约可以被 Swagger UI、Apifox、测试工具和前端类型生成工具使用。

## 19. TypeScript 类型不等于运行时校验

```ts
interface Todo {
  id: number
  title: string
}

const response = await fetch('/api/v1/todos/42')
const todo = (await response.json()) as Todo
```

`as Todo` 只是在编译阶段告诉 TypeScript“相信它是 Todo”，并没有在运行时检查数据。

如果服务端实际返回：

```json
{
  "id": "wrong-type",
  "title": null
}
```

类型断言不会自动报错。因此：

- 后端必须根据 Pydantic 契约校验输出；
- 前后端应共享 OpenAPI 契约；
- 对外部或不可信数据，前端可使用运行时 Schema 校验；
- 不要把 TypeScript 类型断言误认为数据验证。

## 20. JSON 的安全边界

### 20.1 不要泄露内部字段

错误响应：

```json
{
  "id": 7,
  "email": "alice@example.com",
  "password_hash": "...",
  "is_admin": false
}
```

数据库有某个字段，不代表 API 必须返回它。响应 Schema 应采用白名单思维，只声明允许输出的字段。

### 20.2 不要在日志中记录完整敏感 JSON

密码、Token、身份证号等字段应删除或脱敏。调试方便不能成为泄露凭证的理由。

### 20.3 限制请求大小和嵌套深度

超大 JSON 或极深嵌套会消耗内存和 CPU。生产系统应在网关和应用层设置合理限制。

### 20.4 解析数据不等于信任数据

`JSON.parse()` 成功只说明语法合法，不说明字段正确、用户有权限或业务规则成立。

## 21. 如何检查 JSON

### 浏览器控制台

```js
JSON.parse('{"title":"学习 JSON"}')
JSON.stringify({ title: '学习 JSON' }, null, 2)
```

第二个例子使用两个空格进行格式化，适合阅读和调试。

### Python

```python
import json

json.loads('{"title":"学习 JSON"}')
```

### 编辑器

VS Code 可以格式化 `.json` 文件，并提示常见语法错误。保存配置文件前，应检查是否存在注释、单引号或尾随逗号。

### Apifox

使用 JSON Body 时，可以结合接口 Schema 检查字段类型、必填项和响应结构。不要只验证“状态码是 200”。

## 22. 常见错误

### 错误一：使用单引号

```text
{'title': '学习 JSON'}
```

这是 Python 或 JavaScript 风格，不是合法 JSON。

正确写法：

```json
{"title": "学习 JSON"}
```

### 错误二：最后一项多逗号

```text
{
  "title": "学习 JSON",
}
```

标准 JSON 不允许尾随逗号。

### 错误三：把布尔值写成字符串

```json
{
  "completed": "false"
}
```

字符串 `"false"` 不是布尔值 `false`。

### 错误四：把日期当作自动生成的 Date

JSON 解析后得到字符串，需要前端明确转换。

### 错误五：数据库模型直接输出

这样可能泄露内部字段、触发循环引用或导致意外的关系查询。应通过响应 Schema 控制输出。

### 错误六：字段缺失和 null 混为一谈

在 PATCH 等接口中，两者通常具有不同业务含义。

### 错误七：假设 TypeScript 会验证服务端数据

TypeScript 类型在运行时会被擦除，不能替代真实数据校验。

## 23. 本课实操

### 练习 A：判断是否合法

判断下列内容是不是合法 JSON，并说明原因。

```text
1. {"name": "Alice", "age": 18}
2. {'name': 'Alice'}
3. {"completed": True}
4. {"tags": ["a", "b",]}
5. {"assignee": null}
6. {"value": undefined}
```

### 练习 B：设计契约

为 Todo 设计四份 JSON：

1. `TodoCreate`；
2. `TodoUpdate`；
3. `TodoOut`；
4. 分页列表响应。

要求体现：必填字段、可选字段、可清空字段、时间字符串和嵌套的用户摘要。

### 练习 C：找出数据边界问题

```json
{
  "id": 9223372036854775807,
  "price": 0.1,
  "created_at": "2026-09-17 10:30:00",
  "password_hash": "$2b$...",
  "completed": "false"
}
```

至少找出四个可能的问题，并给出更稳妥的设计。

### 练习 D：完成类型映射

为下面的 JSON 分别编写 TypeScript interface 和 Pydantic Model：

```json
{
  "id": 42,
  "title": "学习 JSON",
  "completed": false,
  "assignee": null,
  "tags": ["基础", "全栈"]
}
```

## 24. 练习 A 参考答案

1. 合法；
2. 不合法，JSON 字符串和字段名必须使用双引号；
3. 不合法，JSON 布尔值必须写成小写 `true` 或 `false`；
4. 不合法，JSON 不允许尾随逗号；
5. 合法；
6. 不合法，JSON 没有 undefined 类型。

## 25. 自测问题

进入下一课前，确保你能回答：

- JSON 有哪六种数据类型？
- JSON 文本与 JavaScript 对象有什么区别？
- 序列化和反序列化分别是什么？
- 为什么 `undefined`、Date 和 BigInt 不能直接作为普通 JSON 类型？
- 字段缺失、null 和空字符串有什么区别？
- 为什么超大整数和金额需要特别设计？
- 为什么请求 Schema 和响应 Schema 应分开？
- TypeScript interface 为什么不能替代运行时校验？
- Pydantic、JSON Schema 与 OpenAPI 有什么关系？
- JSON 解析成功后，为什么还需要权限和业务校验？

## 26. 验收标准

完成本课后，你应当能够：

- 独立编写和检查合法 JSON；
- 在 JSON、TypeScript 与 Python 类型之间正确映射；
- 解释序列化和反序列化；
- 正确处理 null、字段缺失、日期、大整数和金额；
- 为创建、更新、详情、列表和错误响应设计不同契约；
- 正确选择 JSON 与 multipart/form-data；
- 理解 Pydantic 如何通过 JSON Schema 与 OpenAPI 描述契约；
- 识别敏感字段泄露、类型假设和循环引用等风险。

## 27. 下一步

下一课建议学习 **HTML**。MVC、HTTP 与 JSON 已经建立了系统地图和通信边界；HTML 将正式进入 View 层，学习如何用语义化元素描述页面结构，为后续 CSS、TypeScript 与 Vue 3 打基础。
