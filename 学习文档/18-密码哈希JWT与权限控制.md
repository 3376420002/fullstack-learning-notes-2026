# 18｜密码哈希、JWT 与权限控制

> 所属阶段：后端安全基础
>
> 本课合并知识点：认证与授权、密码哈希、登录、JWT、Bearer、会话与 Cookie、RBAC、资源所有权、常见攻击边界
>
> 前置知识：HTTP、FastAPI、数据库、Vue Router 与 Axios
>
> 学习目标：能读懂常见登录链路，指挥 AI 在后端实现最小权限控制，并在面试中说清 JWT 能做什么、不能做什么。

[← uv 与 Uvicorn](./17-uv与Uvicorn后端运行.md) · [pytest、httpx、Apifox 与 OpenAPI →](./19-接口测试与OpenAPI协作.md)

## 阅读前说明

安全需求与系统规模、威胁模型和部署环境有关。本文代码是帮助理解边界的片段，不构成可直接上线的完整认证系统。接入真实业务时还需安全评审、测试、密钥管理、限流、审计与应急流程。

## 1. 认证、授权与审计

```text
认证 Authentication  → 你是谁？
授权 Authorization   → 你能做什么、能操作哪条数据？
审计 Audit           → 关键操作是谁在什么时候做的？
```

TodoLab 用户登录后，服务端认证其身份；访问 `GET /api/v1/todos` 时，服务端授权只返回这个用户可读的数据；修改 Todo 时，记录操作者与结果。前端隐藏按钮只是体验，不是授权。真正的权限必须在后端每个受保护入口执行。

## 2. 一条完整登录链路

```text
用户提交凭据（HTTPS）
  → 后端按规范化账号查用户
  → 验证密码哈希
  → 检查账户状态 / 风控
  → 签发短期访问凭证或建立服务端会话
  → 客户端携带凭证访问 API
  → 后端验证凭证、重新确认用户和资源权限
```

认证成功不等于所有接口可访问。令牌有效也不代表账户没被停用、角色没变化，重要操作要再查服务端当前状态。

## 3. 为什么不能保存明文密码

数据库泄漏时，明文密码会直接暴露，而且用户可能在其他网站复用。密码应使用专门的慢速、带盐密码哈希算法，如 Argon2id。普通 SHA-256 速度太快，不适合直接保存密码。加密可逆，密码验证通常不需要取回原密码，因此也不应把“可解密保存”当默认方案。

现代密码哈希库通常在结果中编码算法、盐和参数；业务代码不必另建一个“固定盐”字段。需要根据组织安全策略设置参数并考虑升级；大量用户并发登录时还要关注计算成本。

## 4. Python 中的密码哈希

FastAPI 当前官方教程使用 `pwdlib` 的推荐配置。示意：

```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()

stored_hash = password_hash.hash("user supplied password")
assert password_hash.verify("user supplied password", stored_hash)
assert not password_hash.verify("wrong password", stored_hash)
```

数据库只存 `stored_hash`，不存明文，也不把哈希返回给前端。注册、修改密码、重置密码都要经过同一套策略。已有遗留哈希迁移时不能简单地“换算法后旧密码全失效”，需要验证旧格式并在用户成功登录后逐步重哈希，或设计重置流程。

## 5. 注册与登录时的基础防护

- 对输入做长度、格式与编码约束，但不要静默截断密码；
- 登录失败响应不要泄露“账号存在/不存在”的过多差异；
- 对重复尝试做限流、延迟或风险控制，避免撞库；
- 全程 HTTPS，避免凭据在传输中暴露；
- 记录必要审计事件，但日志不记录明文密码、完整令牌或密码哈希；
- 高风险操作考虑重新认证或多因素认证。

仅用前端禁用按钮无法阻止攻击者直接调用 API。错误提示既要帮助合法用户，又不能变成账户枚举接口。

## 6. JWT 的结构和性质

常见 JWT 是三段：`header.payload.signature`。header/payload 通常是 Base64URL 编码，不是加密；任何拿到令牌的人都可能读到 payload。签名保证令牌未被未授权篡改，但不隐藏内容。

```json
{
  "sub": "user-123",
  "iss": "todolab-api",
  "aud": "todolab-web",
  "iat": 1760000000,
  "exp": 1760000900
}
```

示例时间戳只是说明结构，不是当前有效令牌。不要把密码、身份证号、完整权限清单或秘密放在 payload。JWT 是一种令牌格式，不等于 OAuth 2.0；“用了 JWT”也不自动意味着安全。

## 7. 常用 JWT 声明

| 声明 | 意义 | 注意 |
| --- | --- | --- |
| `sub` | 令牌主体，通常是稳定用户 ID | 不要用会随时变化的显示名 |
| `iss` | 签发者 | 验证时指定预期值 |
| `aud` | 预期接收者 | 防止跨服务误用 |
| `iat` | 签发时间 | 注意服务器时钟 |
| `nbf` | 不早于某时间生效 | 可选，按场景使用 |
| `exp` | 过期时间 | 必须设置合理有效期 |
| `jti` | 令牌 ID | 可辅助撤销/审计 |

客户端自己解析 payload 只能用于 UI 展示；是否信任令牌必须由后端验证签名、允许算法、时间和预期声明。

## 8. 签名算法、密钥与算法混淆

HS256 使用共享秘密；RS256/ES256 等使用非对称密钥。算法选择取决于服务边界和密钥管理。服务端 `decode` 时要显式固定允许的算法，不能直接信任令牌 header 里自称的算法。HS256 的秘密应由安全随机源生成并存于秘密管理系统，而不是写死在代码、`.env.example` 或提交历史中。

密钥轮换需要考虑旧令牌有效期、`kid`、多把验证密钥和发布顺序。不要把“令牌过期”当作密钥泄漏后的全部应对措施。

## 9. 签发与验证的最小代码片段

下例假设已由部署环境提供强随机签名密钥；没有包含数据库查询和登录限流：

```python
from datetime import datetime, timedelta, timezone
import jwt

ISSUER = "todolab-api"
AUDIENCE = "todolab-web"
ALGORITHM = "HS256"


def create_access_token(user_id: str, secret: str) -> str:
    now = datetime.now(timezone.utc)
    claims = {
        "sub": user_id,
        "iss": ISSUER,
        "aud": AUDIENCE,
        "iat": now,
        "exp": now + timedelta(minutes=15),
    }
    return jwt.encode(claims, secret, algorithm=ALGORITHM)


def decode_access_token(token: str, secret: str) -> dict:
    return jwt.decode(
        token,
        secret,
        algorithms=[ALGORITHM],
        issuer=ISSUER,
        audience=AUDIENCE,
        options={"require": ["sub", "iss", "aud", "iat", "exp"]},
    )
```

PyJWT 会抛出签名、过期或声明校验相关异常；Web 层应统一转成 401 响应，同时不把内部验证细节暴露给客户端。代码必须使用真实、受管理的 `secret`，而不是向前端传密钥。

## 10. Bearer Token 如何随请求传输

```http
GET /api/v1/todos HTTP/1.1
Host: example.com
Authorization: Bearer <access-token>
```

Bearer 的意思是“持有者可使用”，拿到令牌的人在有效期内可能冒用身份，所以要用 HTTPS、防泄漏、短有效期并妥善保存。不要把令牌放 URL 查询参数；URL 常进入浏览器历史、代理日志和分析系统。

## 11. FastAPI 中抽取当前用户

`HTTPBearer` 可以让 OpenAPI 描述 Bearer 认证。下面展示关键控制点，实际数据库查询函数由项目实现：

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
import jwt

bearer = HTTPBearer(auto_error=False)


async def get_current_user(
    credentials: HTTPAuthorizationCredentials | None = Depends(bearer),
):
    if credentials is None:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Authentication required")
    try:
        claims = decode_access_token(credentials.credentials, secret=load_jwt_secret())
    except jwt.PyJWTError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid credentials") from None
    user = await find_user_by_id(claims["sub"])
    if user is None or not user.is_active:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid credentials")
    return user
```

`load_jwt_secret()`、`find_user_by_id()` 是项目需实现的函数，不能原样粘贴运行。实际项目还要决定数据库会话如何注入、用户查询失败如何处理、401 是否附 `WWW-Authenticate` 头，以及密钥是否支持轮换。若使用 `OAuth2PasswordBearer`，它主要定义 OAuth2 密码流程的文档与令牌提取；并不会自动验证 JWT。

## 12. 401、403 与 404

- 401：尚未提供有效身份凭证；
- 403：身份已确定，但无权执行此操作；
- 404：资源不存在；在某些资源隔离策略下，也可用于不暴露资源存在性。

不要因为前端“用户页面没显示编辑按钮”就跳过后端权限检查。接口返回什么状态码要保持一致，并避免通过错误差异泄露其他用户的数据是否存在。

## 13. RBAC：角色映射权限

RBAC（基于角色的访问控制）把“用户 → 角色 → 权限”分开。例如管理员可管理用户，普通成员只能管理自己的 Todo。角色名不应直接散落在每个接口里写字符串判断；权限规则可以集中定义，必要时落数据库。

```text
用户 Alice → member → todo:read / todo:create / todo:update-own
用户 Bob   → admin  → todo:read / todo:manage-all
```

但 RBAC 只能回答“有更新 Todo 的一般权限”，不能自动回答“能否更新这一条 Todo”。还需要资源所有权或租户归属检查。

## 14. 资源所有权与越权访问

错误实现：前端传 `owner_id`，后端照单写库。用户可以改请求参数，把 Todo 指向他人。正确方向：创建时从已认证身份确定 `owner_id`；查询/更新时在数据库条件中约束 `owner_id` 或 `tenant_id`。

```python
# 伪代码：数据库查询应同时限定资源 ID 与当前用户 ID
todo = await repo.get_one(todo_id=todo_id, owner_id=current_user.id)
if todo is None:
    raise HTTPException(status_code=404, detail="Todo not found")
```

对多租户系统，`tenant_id` 往往是每次查询都必须携带的边界。不要先查出对象后只在前端判断；批量操作、导出、搜索和统计接口也要统一应用权限条件。

## 15. JWT 与服务端 Session 的取舍

服务端 Session 通常让浏览器持有不透明会话 ID，服务器保存会话状态；JWT 可以让多个服务在不每次查询会话存储的情况下验证签名。但 JWT 默认难以立即撤销，角色更新可能在令牌有效期内滞后。没有“JWT 一定比 Session 更先进”的结论。

可根据架构采用短期 access token、refresh token 轮换、撤销列表或服务端会话。越复杂的方案，越需要明确失效、泄漏、跨设备登出和密钥轮换策略。

## 16. 浏览器里的存储、XSS 与 CSRF

浏览器 `localStorage` 可被同源运行的 JavaScript 读取，发生 XSS 时令牌可能被窃取。敏感会话凭据更适合使用经过设计的 `HttpOnly`、`Secure`、`SameSite` Cookie 方案，但 Cookie 会随请求自动发送，因此还要根据跨站请求场景配置 CSRF 防护。

`HttpOnly` 不能替代 XSS 防护，`SameSite` 也不一定覆盖所有跨站场景。选择“Authorization 头”还是“Cookie”时，要一起考虑前后端同源、CORS、CSRF、XSS、刷新机制和部署域名，而不是只改 Axios 一行配置。

## 17. 退出登录、刷新与撤销

删除客户端令牌只能让当前客户端停止发送它，已泄漏的 JWT 在过期前可能仍有效。想要即时撤销，可结合服务端状态：会话表、令牌版本、撤销列表、用户停用检查等。刷新令牌通常比访问令牌寿命更长，更需安全存储、轮换与复用检测。

重置密码、修改重要权限、账户停用后是否让现有会话立即失效，应由产品安全策略明确，并通过测试覆盖。

## 18. 前端路由守卫的职责

Vue Router 守卫可在没有登录状态时跳转登录页，Pinia 可保存用户资料与界面状态。它们改善用户体验，但不提供安全边界；用户可以直接调 API、修改浏览器状态或绕过前端页面。因此每个后端接口都要独立校验身份和授权。

如果 API 返回 401，Axios 可以统一清理本地状态或尝试受控刷新；不要对每个 401 无限重试，也不要把 403 当作“重新登录就好”。

## 19. 一次具体排错：A 能改 B 的 Todo

依次检查：

1. 更新路由是否依赖 `get_current_user`；
2. 查询 Todo 时是否把当前用户/租户加入过滤条件；
3. Service/Repository 是否接受前端提交的 `owner_id` 并信任它；
4. 批量更新或嵌套资源接口是否绕过同一规则；
5. 是否有回归测试验证 A 的令牌修改 B 的资源被拒绝。

这是服务端越权漏洞，不应通过“隐藏前端按钮”修复。

## 20. 指挥 AI 实现认证与权限

```text
请基于现有 FastAPI、PostgreSQL、Vue 项目设计登录与 Todo 资源授权。
先列出当前认证方式、用户表、已有依赖和接口契约；
密码用合适的慢速带盐哈希，密钥由部署环境提供；
JWT 验证固定算法、签发者、接收者和有效期；
在后端查询中约束 owner_id/tenant_id，不信任前端传来的所有者；
覆盖未登录、过期令牌、停用用户、跨用户访问、角色不足等测试。
请解释 Cookie/Authorization 头的取舍、CSRF/XSS 风险和登出失效策略。
不要把密钥写入代码、示例文件或日志；不要直接部署。
```

让 AI 交付的是“威胁边界 + 实现 + 测试 + 未解决风险”，不只是一个登录页面。

## 21. 面试表达速记

> 认证确认身份，授权判断操作权限；前端路由守卫不是安全边界，后端必须对每个资源按当前用户和租户做权限校验。

> 密码不应明文、可逆加密或直接 SHA-256 保存，应使用 Argon2id 等慢速、带盐的专用密码哈希算法，并考虑参数升级、限流与凭据泄漏应对。

> JWT 的 payload 可读，签名保证完整性而非保密。验证时固定算法并检查过期、签发者、接收者和主体；令牌泄漏后，在过期或服务端撤销前可能被冒用。

> RBAC 解决角色到权限的映射，资源所有权解决“这条数据是否属于你”；两者通常都需要。401 表示未有效认证，403 表示已认证但无权。

## 22. 技术地图与官方资料

```text
凭据 → 密码哈希验证 → 身份确认 → 签发凭证
                                  ↓
请求 → 令牌/会话验证 → 当前用户 → 角色权限 + 资源归属 → API
```

- [FastAPI：OAuth2、密码哈希与 JWT](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)
- [OWASP：密码存储](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OWASP：认证](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP：Session 管理](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP：JWT 注意事项](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html)

## 23. 下一步

下一课用 pytest、httpx、Apifox 和 OpenAPI，把“接口应该是什么”与“实际是否按契约运行”连接起来。

[进入下一课：接口测试与 OpenAPI 协作 →](./19-接口测试与OpenAPI协作.md)
