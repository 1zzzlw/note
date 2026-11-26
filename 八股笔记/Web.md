## 1. Cookie、Session 和 Token 分别是什么？

| 概念      | 一句话理解                                  | 在登录中的作用                                                                                                |
| ------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Cookie  | 浏览器保存少量数据，并在满足条件的请求中自动携带这些数据的 HTTP 机制。 | 服务端可以用 `Set-Cookie` 写入 `sessionId` 或 Token。浏览器根据域名、路径、`Secure`、`SameSite` 等条件，决定是否在请求的 `Cookie` 头中带上它。 |
| Session | 服务端维护的一份会话状态。                          | 登录成功后，服务端生成 `sessionId`，用它关联用户 ID 等信息。会话数据可放在应用内存或 Redis 中，客户端一般通过 Cookie 携带 `sessionId`。              |
| Token   | 客户端向服务端出示的凭据。                          | 客户端可以通过 Cookie 或 `Authorization` 请求头发送 Token。服务端可查询 Token 对应的记录，也可以验证 JWT 的签名和有效期，具体取决于 Token 的设计。     |

[MDN · HTTP Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies) · [RFC 7519 · JWT](https://www.rfc-editor.org/rfc/rfc7519.html)

**用“登录后查询个人资料”来理解：**

- **Session 登录**：登录成功后，服务端存一条 `sessionId → userId` 的记录，并通过 `Set-Cookie` 返回 `sessionId`。查询资料时，浏览器带上这个 Cookie，服务端拿 `sessionId` 查会话，知道当前是谁。
- **Token 登录**：登录成功后，服务端签发 Token。查询资料时，客户端可以显式设置 `Authorization: Bearer <token>`。如果使用签名 JWT，服务端验证签名和过期时间后，可以从其中读取用户标识。需要主动注销时，也可以结合 Redis 检查凭据是否仍然有效。

**总结：** Cookie 解决浏览器怎么保存和携带数据，Session 解决服务端怎么保存会话状态，Token 解决客户端拿什么凭据来访问接口。它们可以组合使用，例如用 Cookie 带 Session ID，或者用 Cookie 带 Token。

### 常见追问

- **Token 相比 Session 有什么优势？** 如果采用可独立验证的签名 JWT，服务端可以根据签名和有效期校验凭据，并从 JWT 读取用户 ID，省去逐个保存和查询登录会话。传统 Session 登录通常由客户端携带 `sessionId`，服务端保存 `sessionId → 会话数据` 的映射。活跃会话越多，需要保存的会话记录越多。多实例部署时，还要考虑如何共享或定位这份状态。使用同一验证配置的服务实例可以各自校验 JWT，因此更容易横向扩容。这个优势取决于鉴权是否能独立完成。
- **Token 和 JWT 有什么关系？** Token 是凭据，JWT 是一种可用于承载凭据的格式。Token 也可以是一个随机字符串，由服务端查表解释。常见的签名 JWT 可以校验内容是否被篡改，里面的字段通常可以直接解码读到，因此别放密码等敏感信息。 [RFC 7519 · JWT](https://www.rfc-editor.org/rfc/rfc7519.html)
- **用 JWT 就一定是无状态的吗？** 取决于每次请求验证时需要什么。如果只靠签名和 `exp` 判断，服务端可以直接验证，通常要等凭据到期才会自然失效。如果还要查询 Redis 判断凭据有没有被撤销，鉴权仍依赖服务端状态。
- **Token 放 Cookie 和 LocalStorage 有什么区别？** Cookie 符合条件时由浏览器自动发送。`HttpOnly` 可阻止页面脚本直接读取该 Cookie，`SameSite` 可限制它在跨站请求中的发送。LocalStorage 里的 Token 需要前端代码读取后主动加到请求头，且页面脚本能读到它。前者要考虑自动携带凭据带来的 CSRF 风险，后者要考虑 XSS 时凭据被读取的风险。 [MDN · HTTP Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies)

## 2. 单 Token 和双 Token 有什么区别？

这里的**单 Token**指客户端只持有一种登录凭据。延续登录要靠这份凭据本身或重新认证，例如在它仍有效时滑动续期，或者到期后重新登录。**双 Token**把“访问业务”和“续期”分给两种凭据。 [RFC 6749 · Access Token 和 Refresh Token](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.4)

| 维度      | 单 Token                       | 双 Token                                                      |
| ------- | ----------------------------- | ------------------------------------------------------------ |
| 访问业务接口  | 使用同一份 Token。                  | 使用 Access Token。                                             |
| 如何续期    | 可以在凭据有效时实现滑动续期或换发。失效后通常要重新认证。 | Access Token 到期后，用仍有效的 Refresh Token 向认证服务申请新的 Access Token。 |
| 有效期怎么设计 | 这一份凭据同时影响登录体验和泄露后的可访问时间。      | Access Token 可以设得较短，Refresh Token 通常活得更久，并只用于认证服务的指定流程。      |
| 实现成本    | 凭据少，客户端和服务端的流程更简单。            | 多了刷新接口、凭据存储、过期后的重试，以及并发刷新和撤销等处理。                             |

**为什么要分成两个？** 假设业务 Token 只有 1 小时有效期。单 Token 如果到期就得输密码，体验不好。直接延长业务 Token 的有效期，又会延长它泄露后可能被利用的时间。双 Token 让短期的 Access Token 随业务请求发送，长期的 Refresh Token 只在认证流程中使用。这样能缩短 Access Token 泄露后的可用窗口，同时让正常用户在凭据到期后继续使用。Refresh Token 更能延续会话，因此也要保护好它。 [RFC 9700 · Refresh Token 保护](https://www.rfc-editor.org/rfc/rfc9700.html#section-4.14)

**总结：** 单 Token 用同一份凭据访问业务，并围绕它管理续期。双 Token 将访问和续期拆开，短期 Access Token 走业务接口，较长期的 Refresh Token 走刷新接口。Access Token 过期时，客户端用 Refresh Token 换新凭据，再重试原请求。这个方案能兼顾较短的访问凭据有效期和连续登录体验，代价是多了刷新流程和凭据管理。

### 扩展：
* **单点登录（Single Sign-On，SSO）：** 用户在统一认证中心完成一次登录后，在认证会话仍然有效、满足各应用登录策略的情况下，可以进入多个信任该认证中心的应用，省去重复输入密码。
* **双 Token 在单点登录场景下的作用：** 单点登录让多个应用复用认证中心的登录会话。双 Token 则让认证中心把「当前能访问多久」和「以后能不能续期」分开控制。业务接口使用短有效期的 Access Token，客户端需要继续访问时，才拿 Refresh Token 到认证中心换发新的 Access Token。
  例如，管理员封禁某个账号后，认证中心可以立即撤销该账号的 Refresh Token，并在刷新和重新登录时检查账号状态，拒绝再签发凭据。如果业务接口只校验 Access Token 的签名和有效期，已经发出去的 Access Token 仍可能用到过期。把 Access Token 的有效期设短，就能缩短这段时间。需要让封禁在后续请求中立即生效时，可以让网关额外检查账号状态、Token 撤销记录或服务端会话。