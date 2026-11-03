# K01 账号（account）

> **层级**：内核 L2 · **代码**：`server/kernel/account` · **首次阶段**：1（本地开发账号）；2（注册登录）· **主文档**：第 5、16.7、22 节

## 1. 职责

- 注册与登录：邮箱、GitHub 登录
- 会话管理
- 个人资料：名字、头像
- 好友关系
- **API 令牌**：给外部 agent（MCP）使用，带权限范围和有效期，可以随时撤销
- 第 1 阶段：提供一个本地开发账号

## 2. 不负责

- 星球上的权限（主人 / 建造者 / 访客）→ K02
- 资源 → K07

## 3. 依赖

K00、K13（检查名字）

## 4. 对外接口

```rust
pub trait Accounts: Send + Sync {
    fn user(&self, id: UserId) -> Result<User>;
    fn authenticate(&self, credential: Credential) -> Result<Session>;
    fn friends(&self, id: UserId) -> Result<Vec<UserId>>;
    fn verify_token(&self, token: &str) -> Result<TokenGrant>;   // 返回用户 + 权限范围
}

pub struct TokenGrant {
    pub user: UserId,
    pub scopes: Vec<Scope>,      // 例如：只能操作某几颗星球、只能调用外观级能力
    pub expires_at: Timestamp,
}
```

HTTP：注册、登录、退出；查询和修改资料；好友申请 / 接受 / 删除；创建 / 列出 / 撤销 API 令牌

- 导出认证中间件 `auth_layer`：识别会话或 API 令牌（开发模式下没有凭据时就是开发账号），把 K00 定义的 `CurrentUser { user, is_admin }` 放进请求；`is_admin` = 账号编号在 server.toml 的 `admins` 里。gf-app 把它装在合并后的 router 上，其他模块（包括 L1 的 K08、K13）用 K00 的 `CurrentUser`、`AdminUser` 提取器拿身份

## 5. 数据

| 表 | 关键字段 |
|---|---|
| users | id, name, avatar, created_at |
| identities | user_id, provider（email / github）, external_id |
| sessions | id, user_id, expires_at |
| friendships | user_id, friend_id, created_at |
| friend_requests | from_id, to_id, created_at |
| api_tokens | id, user_id, hash, scopes, expires_at, revoked_at |

## 6. 事件

- 发布：`UserCreated`、`FriendAdded`、`TokenRevoked`

## 7. 配置

`account.json`：名字长度、令牌最长有效期

## 8. 关键规则

- 令牌只存哈希，不存明文；明文只在创建时显示一次
- 外部 agent 的令牌默认权限最小：只能操作指定的星球、只能调用外观级和星球级能力
- 名字要经过 K13 检查

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 本地开发账号 | 无需登录即可使用一个固定账号 |
| 2 | 邮箱和 GitHub 登录、会话 | 能注册、登录、退出 |
| 2 | 资料和好友 | 能加好友，好友出现在星系视图里 |
| 5 | API 令牌 | 外部 agent 用令牌连接 MCP，超出权限范围的调用被拒绝 |

## 10. 测试要点

- 撤销后的令牌立即失效
- 权限范围检查：令牌不能访问范围外的星球和能力

## 11. 待定问题

- 是否支持手机号登录

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K01-x"（x 换成子任务编号）。

~~~text
【模块】K01 账号（包 gf-account，路径 server/kernel/account，层级 L2）
【先读】docs/modules/K01-account.md；K13-moderation.md 第 4 节；主文档第 5、16.7、22 节
【范围】server/kernel/account/、server/migrations/（文件名带 k01）、shared/config/account.json、fixtures/account/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-moderation。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、argon2、sha2、rand_core、base64；GitHub 登录（K01-2）另加 oauth2、reqwest。测试：wiremock、proptest
【规则补充】
  - 导出认证中间件 auth_layer：读 Authorization: Bearer <会话或令牌>；dev_mode 下没有这个头就是开发账号；把 K00 的 CurrentUser { user, is_admin } 放进请求的扩展里。is_admin = 账号编号在 server.toml 的 admins 里（dev_mode 下开发账号也算）。gf-app 把它装在合并后的 router 上；其他模块的 HTTP 处理函数用 K00 的 CurrentUser、AdminUser 拿身份（文档第 4 节补上）
  - 会话和 API 令牌都是 32 字节随机数，用 base64url 编码给客户端；数据库只存 SHA-256；API 令牌明文以 gf_ 开头
  - 好友要先申请、对方接受后才成立（新增表 friend_requests：from_id、to_id、created_at）
  - account.json：name_min_chars、name_max_chars、session_ttl_hours、token_max_days
【子任务】
K01-1 本地开发账号（阶段 1）
  - server.toml 的 dev_mode = true 时存在固定账号：id 01J00000000000000000000000，名字"开发者"；dev_mode = false 时这个账号不存在、无法使用
  - 测试：dev_account_authenticates_in_dev_mode、dev_account_disabled_outside_dev_mode、current_user_extractor_uses_dev_account、admins_from_server_toml（server.toml 的 admins 里的账号 is_admin 为真，其他账号为假；dev_mode 下开发账号也为真）
K01-2 邮箱和 GitHub 登录、会话（阶段 2）
  - 邮箱 + 密码注册（argon2 哈希）、登录、退出；会话按 session_ttl_hours 过期（用 FakeClock 测）；GitHub OAuth 用 wiremock 模拟
  - 测试：register_login_logout、wrong_password_rejected、password_stored_as_hash_only、session_expires、github_login_creates_identity_once（同一个 GitHub 账号登录两次只建一个用户）
K01-3 资料与好友（阶段 2）
  - 改名经过 K13 检查，长度按 account.json；好友申请、接受、删除
  - 测试：rename_checked_by_moderation、rename_length_limits、friend_request_accept_is_mutual、remove_friend_is_mutual、friend_request_to_self_rejected
K01-4 API 令牌（阶段 5）
  - 创建（返回一次明文）、列出（不含明文）、撤销；默认范围：指定的星球 + 外观级和星球级能力；有效期不超过 token_max_days
  - verify_token 对已撤销或已过期的令牌返回 PermissionDenied
  - 测试：token_plaintext_returned_once_hash_stored、revoked_token_rejected_immediately、expired_token_rejected、scope_limits_planets_and_levels、token_lifetime_capped
  【验收】另加 tools/mutants.sh gf-account src/tokens.rs
【验收（每个子任务）】tools/check_server.sh gf-account；tools/check_tests.sh gf-account <本子任务点名的测试>
【停止条件】需要真实的 GitHub OAuth 应用或邮件发送服务；需要手机号登录
~~~
