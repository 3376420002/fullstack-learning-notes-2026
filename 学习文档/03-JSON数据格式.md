# 03｜JSON 数据格式与接口契约

> 目标：能正确读写 JSON、理解前后端类型映射，并让 AI 按稳定契约开发接口。

[← HTTP 与 HTTPS](./02-HTTP与HTTPS.md) · [学习首页](../README.md) · [HTML 与 CSS →](./04-HTML与CSS页面基础.md)

## 1. JSON 在项目中的位置

```text
TypeScript 对象
  ↓ JSON.stringify / Axios
HTTP Body
  ↓
FastAPI + Pydantic
  ↓ Python 对象
```

JSON 是文本数据格式，不是编程语言，也不是数据库。

## 2. JSON 支持的类型

```json
{
  "id": 42,
  "title": "学习 JSON",
  "completed": false,
  "priority": null,
  "tags": ["api", "data"],
  "owner": {
    "id": 7,
    "username": "alice"
  }
}
```

JSON 只有：

```text
object、array、string、number、boolean、null
```

JSON 没有原生 Date、BigInt、Decimal、undefined、函数、Map 或 Set。

## 3. 基本语法

- 对象键必须使用双引号；
- 字符串使用双引号；
- 不允许尾随逗号；
- 不支持普通注释；
- `true`、`false`、`null` 使用小写；
- 数字不能包含 `NaN` 和 `Infinity`。

合法：

```json
{"title":"学习","completed":false}
```

不是合法 JSON：

```js
{ title: '学习', completed: false }
```

后者是 JavaScript 对象字面量写法。

## 4. 序列化与反序列化

TypeScript：

```ts
const text = JSON.stringify({ title: '学习 JSON' })
const value: unknown = JSON.parse(text)
```

Python：

```python
import json

text = json.dumps({"title": "学习 JSON"}, ensure_ascii=False)
value = json.loads(text)
```

`JSON.parse` 返回的数据来自运行时，不能因为 TypeScript interface 存在就自动可信。

## 5. 前后端类型映射

| JSON | TypeScript | Python |
| --- | --- | --- |
| string | `string` | `str` |
| number | `number` | `int` / `float` |
| boolean | `boolean` | `bool` |
| null | `null` | `None` |
| array | `T[]` | `list[T]` |
| object | interface/type | Pydantic Model / dict |

JSON number 不区分整数与浮点，后端 Schema 应明确约束。

## 6. 日期和时间

JSON 没有 Date，API 通常发送 ISO 8601 字符串：

```json
{
  "created_at": "2026-09-20T03:00:00Z"
}
```

TypeScript：

```ts
const createdAt = new Date(dto.created_at)
```

Python/Pydantic 可以解析为带时区 datetime。系统应统一使用 UTC 保存和传输，显示时再转换用户时区。

## 7. 大整数与金额

JavaScript number 不能安全表达所有数据库 BIGINT。超出安全范围的 ID 可以在 API 中使用字符串：

```json
{"id":"9223372036854775807"}
```

金额不要用二进制浮点直接传递重要计算结果。常见策略：

```json
{"amount":"19.99","currency":"CNY"}
```

或使用最小货币单位整数：

```json
{"amount_cents":1999,"currency":"CNY"}
```

具体方案要前后端统一。

## 8. 缺失、null 和空值

```json
{}
```

表示字段未发送。

```json
{"description":null}
```

表示明确发送空值。

```json
{"description":""}
```

表示发送空字符串。

PATCH 接口尤其要区分未设置字段与显式 null。Pydantic 可以使用 `model_dump(exclude_unset=True)` 保留这种差异。

## 9. 请求和响应模型分离

```ts
interface TodoCreate {
  title: string
}

interface TodoUpdate {
  title?: string
  completed?: boolean
}

interface TodoDto {
  id: number
  title: string
  completed: boolean
  created_at: string
}
```

创建请求不应包含服务器生成的 id、created_at 和内部字段。

后端同样使用 `TodoCreate`、`TodoUpdate`、`TodoOut` 分离契约。

## 10. TypeScript 类型不做运行时校验

```ts
const todo = (await response.json()) as TodoDto
```

`as TodoDto` 只是告诉编译器相信该数据，不会检查 JSON。

可靠方案：

```text
后端 Pydantic 校验输入和响应
OpenAPI 描述契约
前端生成或同步类型
重要外部数据在前端使用运行时 Schema 校验
```

## 11. 字段命名

Python 常用 snake_case：

```json
{"created_at":"2026-09-20T03:00:00Z"}
```

TypeScript 常用 camelCase：

```ts
interface Todo {
  createdAt: Date
}
```

可以统一使用一种命名，也可以在 API 模块集中转换。不要在每个 Vue 组件中零散转换。

## 12. 嵌套深度与重复数据

过深响应：

```text
todo.owner.team.organization...
```

会带来体积、循环引用和更新困难。接口应只返回当前页面需要的结构。

列表通常返回摘要，详情接口返回更完整内容：

```text
GET /todos       TodoSummary[]
GET /todos/42    TodoDetail
```

## 13. 错误响应

推荐稳定结构：

```json
{
  "code": "TODO_NOT_FOUND",
  "message": "Todo does not exist",
  "request_id": "req-123"
}
```

字段校验错误可以增加：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Input is invalid",
  "field_errors": {
    "title": "Title must not be blank"
  }
}
```

不要把堆栈、SQL、文件路径或内部异常直接返回给客户端。

## 14. 常见错误

- 使用单引号、尾随逗号或注释，导致 JSON 无法解析；
- 把 `undefined` 当成 JSON 类型；
- 把日期字符串当成已经是 Date；
- 用 number 接收超大整数；
- 创建请求发送 id、角色或其他只读字段；
- 用 TypeScript 类型断言代替运行时验证；
- 前后端字段命名不统一；
- 把敏感数据库字段直接序列化。

## 15. 给 AI 的开发指令

```text
请根据现有 OpenAPI 设计 Todo 的 TypeScript DTO 和 Pydantic Schema。
分别定义创建、更新、列表和详情结构，区分字段缺失、null 和空字符串。
日期通过 ISO 8601 字符串传输，超大整数和金额明确编码策略。
不要把 TypeScript interface 描述为运行时验证，不要暴露数据库内部字段。
所有 snake_case 与 camelCase 转换集中在 API 层。
```

审查指令：

```text
请检查这组 JSON 契约的类型、可空性、日期、金额、超大整数和敏感字段。
核对前端 DTO、Pydantic Schema 和 OpenAPI 是否一致。
优先报告运行时类型失真与向后兼容风险，不要只调整格式。
```

## 16. 面试表达

> JSON 是文本数据交换格式，只支持对象、数组、字符串、数字、布尔和 null，不直接支持 Date、BigInt、Decimal 或 undefined。

> 序列化把对象转换为 JSON 文本，反序列化把 JSON 文本解析为运行时数据。

> TypeScript interface 在编译后消失，不能验证后端响应；重要外部数据需要运行时校验。

> 字段缺失、null 和空字符串语义不同，PATCH 更新时尤其要区分。

> API Schema 与数据库模型不是同一概念，响应模型应过滤内部和敏感字段。

[进入下一课：HTML 与 CSS 页面基础 →](./04-HTML与CSS页面基础.md)
