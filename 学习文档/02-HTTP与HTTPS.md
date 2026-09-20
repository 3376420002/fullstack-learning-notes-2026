# 02｜HTTP 与 HTTPS

> 目标：看懂前后端请求、正确选择方法和状态码，并能让 AI 根据接口契约开发和排错。

[← MVC 与分层](./01-MVC分层架构.md) · [学习首页](../README.md) · [JSON →](./03-JSON数据格式.md)

## 1. 一次请求包含什么

```http
POST /api/v1/todos HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: Bearer <token>

{"title":"学习 HTTP"}
```

```text
Method   要执行的动作
Path     目标资源
Query    筛选、排序、分页
Header   内容类型、认证、缓存等元数据
Body     创建或更新的数据
```

响应：

```http
HTTP/1.1 201 Created
Content-Type: application/json

{"id":1,"title":"学习 HTTP","completed":false}
```

## 2. 请求位置

| 位置 | 适合内容 | 示例 |
| --- | --- | --- |
| Path | 资源身份 | `/todos/42` |
| Query | 搜索、筛选、分页 | `?status=active&page=2` |
| Header | 认证、格式、追踪 | `Authorization` |
| JSON Body | 结构化输入 | `{"title":"学习"}` |
| FormData | 文件与表单字段 | 头像上传 |

密码和长期 Token 不应放 URL，因为 URL 可能进入历史、日志和分享内容。

## 3. 常用 HTTP 方法

| 方法 | 用途 | Todo 示例 |
| --- | --- | --- |
| GET | 查询 | `GET /todos` |
| POST | 创建或执行动作 | `POST /todos` |
| PUT | 整体替换 | `PUT /todos/42` |
| PATCH | 部分更新 | `PATCH /todos/42` |
| DELETE | 删除 | `DELETE /todos/42` |

GET、PUT、DELETE 通常被设计为幂等操作；多次执行的最终状态与执行一次相同。POST 通常不是幂等的。

实际行为仍由接口实现决定，不能只看方法名称。

## 4. REST 风格路径

推荐使用名词：

```text
GET    /api/v1/todos
POST   /api/v1/todos
GET    /api/v1/todos/42
PATCH  /api/v1/todos/42
DELETE /api/v1/todos/42
```

业务动作可以明确表达：

```text
POST /api/v1/todos/42/complete
```

不要把接口全部设计成 `/getTodo`、`/deleteTodo`，也不要为了“纯 REST”牺牲清晰的业务语义。

## 5. 常用状态码

| 状态码 | 含义 | 常见场景 |
| ---: | --- | --- |
| 200 | 成功并返回内容 | 查询、更新 |
| 201 | 创建成功 | POST 创建资源 |
| 204 | 成功但无响应体 | 删除成功 |
| 400 | 通用错误请求 | 无法处理的输入 |
| 401 | 未认证 | Token 缺失或失效 |
| 403 | 已认证但无权限 | 操作他人资源 |
| 404 | 路由或资源不存在 | Todo 不存在 |
| 409 | 状态冲突 | 重复或版本冲突 |
| 415 | 不支持的媒体类型 | 要求 JSON 却发送其他格式 |
| 422 | 字段验证失败 | FastAPI 请求校验 |
| 429 | 请求过多 | 触发限流 |
| 500 | 未处理的服务端错误 | 程序异常 |
| 502/503/504 | 网关或服务不可用 | 上游故障、过载、超时 |

不要把所有业务失败都包装成 HTTP 200。状态码负责传输语义，响应体业务 code 提供更细分类。

## 6. Header 重点

```text
Content-Type       请求体格式
Accept             希望接收的格式
Authorization      认证凭据
Cache-Control      缓存策略
ETag               资源版本标识
If-None-Match      条件请求
Location           新资源地址
X-Request-ID       链路追踪标识
```

JSON 请求：

```http
Content-Type: application/json
```

FormData 的 boundary 通常由浏览器生成，不要手工写一个不完整的 Content-Type。

## 7. HTTP 是无状态协议

每个请求应带上处理它所需的信息。登录状态通常通过：

```text
Authorization Bearer Token
或 Cookie Session
```

无状态不表示服务器不能保存用户或会话数据，而是 HTTP 请求本身不自动记住上一次请求。

## 8. HTTPS

HTTPS 是 HTTP 运行在 TLS 保护之上，主要提供：

- 加密，降低窃听风险；
- 完整性，降低内容被篡改风险；
- 服务器身份验证。

HTTPS 不会自动修复：

- SQL 注入；
- XSS；
- 越权；
- 弱密码；
- 服务端日志泄密。

生产环境通常由反向代理、负载均衡或云平台终止 TLS。

## 9. CORS

当前端和 API 的协议、主机或端口不同，浏览器会执行跨源规则。

```text
http://localhost:5173
http://localhost:8000
```

它们端口不同，因此是不同 Origin。

CORS 是浏览器读取跨源响应的策略，不是认证和权限系统。curl 和服务端程序不受浏览器 CORS 限制。

带 Cookie 凭据时要配置明确可信 Origin，不能简单使用 `*`。

## 10. 缓存与条件请求

```http
ETag: "todo-list-v3"
```

客户端再次请求：

```http
If-None-Match: "todo-list-v3"
```

内容未变化时服务器可以返回：

```http
304 Not Modified
```

用户私有数据、敏感响应和公共静态资源需要不同缓存策略，不能统一长期缓存。

## 11. curl 联调

查询：

```powershell
curl.exe -i "http://127.0.0.1:8000/api/v1/todos?page=1"
```

创建：

```powershell
curl.exe -i `
  -X POST "http://127.0.0.1:8000/api/v1/todos" `
  -H "Content-Type: application/json" `
  -d '{"title":"学习 HTTP"}'
```

`-i` 查看响应头，`-v` 查看连接和请求细节。Windows PowerShell 使用 `curl.exe` 可避免旧别名混淆。

## 12. 排错顺序

```text
1. 浏览器 Network 中最终 URL 是否正确
2. Method、Query、Header、Body 是否符合契约
3. 请求是否到达 FastAPI
4. 状态码是 404、422、500 还是网络错误
5. Content-Type 和响应体是什么
6. Vite 代理与生产网关是否正确
7. 是否误把 CORS 当成所有网络问题
```

## 13. 给 AI 的开发指令

```text
请根据现有 FastAPI OpenAPI 设计 Vue 的 Todo 请求。
明确 Method、Path、Query、Header、JSON Body、成功状态码和错误状态码。
区分 401、403、404、409、422 与 5xx，不把所有失败处理成 200。
不要把 Token 放入 URL，不要手工设置 FormData boundary。
完成后给出 curl 验证命令和浏览器 Network 检查点。
```

排错指令：

```text
请根据实际请求、响应和日志诊断 HTTP 问题。
依次检查 URL、方法、Content-Type、认证、CORS、状态码和响应体。
先给出证据，再修改代码，不要用关闭 CORS 或忽略状态码掩盖问题。
```

## 14. 面试表达

> HTTP 请求由方法、URL、Header 和可选 Body 组成，响应由状态码、Header 和可选 Body 组成。

> 401 表示未通过认证，403 表示身份已知但没有权限，422 在 FastAPI 中常表示请求字段校验失败。

> PUT 通常表达整体替换，PATCH 表达部分更新；幂等表示重复执行的最终状态与执行一次相同。

> HTTPS 提供传输加密、完整性和服务器身份验证，但不能替代应用层安全。

> CORS 是浏览器跨源策略，不是后端权限控制。

[进入下一课：JSON 数据格式 →](./03-JSON数据格式.md)
