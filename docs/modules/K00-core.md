# K00 核心基础（core）

> **层级**：内核 L0 · **代码**：`server/kernel/core`、`server/app` · **首次阶段**：1 · **主文档**：第 18、23、31 节

## 1. 职责

- **事件总线**：进程内的发布 / 订阅；每个事件带名称和版本号
- **配置加载**：启动时读取 `shared/config`，按 Schema 校验，不合法就拒绝启动
- **数据库访问**：连接池、迁移、事务
- **编号生成**：全局唯一、按时间可排序的编号（例如 ULID）
- **统一错误类型**：所有模块共用的错误分类（参数错误、权限不足、资源不足、配额超限、内部错误……）
- **时钟**：统一的服务器时间来源，测试时可以换成假时钟
- **日志与追踪**：每次请求一个追踪编号，贯穿所有模块
- **启动组装**（`server/app`）：创建各模块实例、注入接口实现（适配器）、注册官方玩法包、启动 HTTP / WebSocket 服务

## 2. 不负责

任何业务逻辑。K00 里不能出现"星球""金""战斗"这类概念。

例外：跨模块共用的编号类型（UserId、PlanetId、EntityId、RegionId、PackId）放在 K00。它们只是编号，不带任何逻辑；层级比 K01、K02 低的模块（例如 K07、K08、K13）要引用用户和星球，只能靠这些编号。

## 3. 依赖

无。

## 4. 对外接口

```rust
// 编号与时间
pub struct Timestamp(pub i64);                  // UTC 毫秒
pub struct UtcDay(pub i64);                     // 从 1970-01-01 起的天数；"每天"按 UTC 0 点划分
define_id!(UserId); define_id!(PlanetId); define_id!(EntityId); define_id!(RegionId);   // ULID 新类型
pub struct PackId(String);                      // 小写字母、数字、下划线，1–32 个字符

// 时钟（另有 SystemClock 和测试用的 FakeClock）
pub trait Clock: Send + Sync { fn now(&self) -> Timestamp; }

// 错误
pub enum ErrorKind { InvalidInput, PermissionDenied, InsufficientResources, QuotaExceeded, NotFound, Conflict, Internal }
pub struct Error { pub kind: ErrorKind, pub message: String }
pub type Result<T> = std::result::Result<T, Error>;

// 配置
pub fn load_config(dir: &Path) -> Result<ConfigSet, ConfigError>;
impl ConfigSet { pub fn get<T: DeserializeOwned + JsonSchema>(&self, name: &str) -> Result<T, ConfigError>; }

// 数据库与事务
impl Db { pub async fn begin(&self) -> Result<Tx<'_>>; }
impl Tx<'_> {
    pub async fn publish<E: Event>(&mut self, event: &E) -> Result<()>;   // 写进 outbox，提交后才送出
    pub async fn commit(self) -> Result<()>;                              // 没有提交就丢弃 = 回滚
}

// 事件
pub trait Event: Serialize + DeserializeOwned + Clone + Send + Sync + 'static {
    const NAME: &'static str;
    const VERSION: u32;
}
pub trait EventBus: Send + Sync {
    async fn publish<E: Event>(&self, event: E) -> Result<()>;
    fn subscribe<E: Event>(&self, handler: impl Fn(E) -> BoxFuture<'static, Result<()>> + Send + Sync + 'static);
}

// 种子随机数：影响游戏结果的随机只用这里
pub mod rng {
    pub fn seed(parts: &[&[u8]]) -> [u8; 32];                  // SHA-256（每部分前面加 4 字节小端长度）
    pub fn from_seed(seed: [u8; 32]) -> ChaCha8Rng;
    pub fn below(rng: &mut ChaCha8Rng, n: NonZeroU64) -> u64;  // [0, n)
    pub fn chance_bp(rng: &mut ChaCha8Rng, bp: u32) -> bool;   // 概率用基点表示
}

// 对象存储：快照、素材文件、上传的小说
#[async_trait]
pub struct CurrentUser { pub user: UserId, pub is_admin: bool }   // axum 提取器：读 K01 的认证中间件放进请求的身份；没有身份返回 PermissionDenied
pub struct AdminUser(pub UserId);                               // axum 提取器：is_admin 为假时返回 PermissionDenied
pub trait ObjectStore: Send + Sync {
    async fn put(&self, key: &str, bytes: Bytes) -> Result<()>;
    async fn get(&self, key: &str) -> Result<Bytes>;          // 不存在返回 NotFound
    async fn exists(&self, key: &str) -> Result<bool>;
    async fn delete(&self, key: &str) -> Result<()>;
}
```

## 5. 数据

- `schema_migrations`：数据库迁移记录
- `outbox`：待发布的事件（见 8. 关键规则）

## 6. 事件

无。

## 7. 配置

- `server.toml`：端口、日志级别、开发模式、配置目录、管理员账号（admins）；数据库地址只从环境变量 DATABASE_URL 读
- 密钥（数据库密码、模型 API Key）**只从环境变量读取**，不写进任何文件

## 8. 关键规则

- **事务提交之后才发事件**：先把事件写进 `outbox` 表（和业务数据在同一个事务里），提交后再由后台发出。这样不会出现"事件发出去了，数据却没写进去"
- 事件处理失败不影响发布者，失败的事件会重试并记录
- 需要强一致的操作用数据库事务，不要靠事件
- 所有时间来自 `Clock`，代码里不直接读系统时间
- `server/app` 是唯一知道所有模块的地方；适配器（例如用 K10 实现 K13 的 `ContentClassifier`）写在这里

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | Cargo 工作区与所有内核 crate 的空骨架 | `cargo build` 通过；依赖方向符合层级表 |
| 1 | 配置加载与校验 | 配置写错时启动失败，并指出哪个文件哪一行 |
| 1 | 数据库连接、迁移、事务 | 一条命令建好本地数据库 |
| 1 | 事件总线 + outbox | 事务回滚时事件不会发出 |
| 1 | `server/app` 组装 | 客户端能连上本地服务器 |
| 1 | 依赖方向检查脚本 | CI 里自动检查有没有违反层级的依赖 |
| 1 | 时钟、编号、错误类型 | 假时钟能控制所有依赖时间的测试 |
| 1 | 种子随机数、对象存储 | 随机数金标准在三个操作系统上一致；本地目录和内存两种存储行为一致 |

## 10. 测试要点

- 事件总线：订阅者出错不影响其他订阅者；outbox 在回滚时不发事件
- 配置：缺字段、类型错、数值越界都能被发现
- 假时钟能让依赖时间的测试稳定复现
- 随机数：金标准文件；概率为 0 和 100% 的边界
- 对象存储：key 里带 `..` 被拒绝

## 11. 待定问题

- 以后是否需要跨进程的消息队列（拆分服务时再考虑）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K00-x"（x 换成子任务编号）。

~~~text
【模块】K00 核心基础（包 gf-core，路径 server/kernel/core，层级 L0）+ 启动组装（包 gf-app，路径 server/app，顶层）
【先读】docs/modules/K00-core.md；docs/modules/README.md 的全局约定、包名与路径、统一脚本；主文档第 18、23、24、31 节
【范围】根目录工程文件（Cargo.toml、rust-toolchain.toml、clippy.toml、rustfmt.toml、deny.toml、layers.toml、.gitignore、.github/）、根目录说明文件、server/kernel/core/、server/app/、server/migrations/（文件名带 k00）、tools/（K00-2 负责的脚本和 tools/check_layers/）、client/（只限 K00-1 的空项目）、fixtures/core/
【允许的依赖】
  gf-core：serde、serde_json、thiserror、tokio、futures、async-trait、sqlx（postgres、runtime-tokio、macros、migrate、json）、ulid、tracing、schemars、jsonschema、sha2、rand_core、rand_chacha、bytes、axum（只用于错误类型转换成 HTTP 响应）
  gf-app：axum、tokio、tower、tracing-subscriber、toml，以及工作区里所有 gf- 包
  gf-check-layers：cargo_metadata、toml、serde、anyhow
  测试：proptest、tempfile、tower（ServiceExt）
【规则补充】
  - 数据库查询用 sqlx 的 query / query_as（运行时检查），不用 query! 宏
  - 事件处理函数的返回值是 BoxFuture<'static, Result<()>>（文档第 4 节同步改成这样）
  - Internal 错误返回给客户端时只给"内部错误"四个字，详细信息只写日志
【子任务】
K00-1 项目骨架（阶段 1）
  - 根目录 Cargo.toml：工作区成员是 server/app、server/kernel/ 下 19 个内核包、server/packs/ 下 4 个玩法包、shared/terrain；包名按 README"包名与路径"表；[workspace.dependencies] 统一版本；[workspace.lints] 把 clippy 的 unwrap_used、expect_used、panic、todo、unimplemented 设为 deny；clippy.toml 设 allow-unwrap-in-tests = true、allow-expect-in-tests = true
  - rust-toolchain.toml 固定一个具体的稳定版本号（当前最新的稳定版）
  - 每个包只有空的 lib.rs；gf-app 的 main.rs 只返回成功（K00-6 再实现）。这是唯一允许空实现的子任务
  - client/：Godot 4 空项目（README 里写明固定的 Godot 版本号），GUT 装在 client/addons/gut，建好 client/tests/
  - 根目录文件：.gitignore（含 .env、target/、client/.godot/）、LICENSE（AGPL-3.0 全文）、LICENSE-ASSETS（CC BY-SA 4.0 全文）、README.md（简介、编译、测试）、AGENTS.md（已经存在，不要重写：保留现有内容，只更新「当前状态」一节，写明哪些工程和脚本已经建好）、CLAUDE.md（已经存在，不要改）、CONTRIBUTING.md（分支、PR、AI 使用声明）、SECURITY.md（漏洞报告方式）、.github/pull_request_template.md（含"是否使用了 AI、用在哪里"一栏）
  【验收】cargo build --workspace；godot --headless --path client --quit；git status --ignored 显示 target/ 和 client/.godot/ 被忽略
K00-2 统一脚本与层级检查（阶段 1）
  - 写好 README"统一脚本"表里由 K00-2 创建的 7 个脚本：bash，set -euo pipefail，从任何目录运行都以仓库根目录为准；参数缺失时打印用法并返回 2
  - tools/check_client.sh：client/ext 存在时先 cargo build -p gf-client-ext
  - tools/check_tests.sh：服务器包用 cargo test -p <包> -- --list 找测试名；客户端目录在 client/tests/<目录>/ 里找 func <测试名>(；缺一个就失败并列出缺的名字
  - tools/mutants.sh：cargo mutants --package <包> --file <文件>；有存活的变异就失败。确实等价的变异用 #[mutants::skip] 标出，并在完成报告里说明理由
  - tools/check_layers/（包 gf-check-layers，加进工作区）：读 layers.toml 和 cargo metadata（含开发依赖）。layers.toml 格式：
      [levels]：每个包的层级（按 README 的表，gf-terrain = 0）
      [top]：packages = ["gf-app", "gf-client-ext", "gf-check-layers", "gf-schema-gen", "gf-eval"]
      [pack_deps]："gf-pack-combat" = ["gf-pack-supply"]、"gf-pack-boss" = ["gf-pack-supply", "gf-pack-combat"]、"gf-pack-story" = ["gf-pack-combat"]
    规则：依赖只能指向更低层级；同层只允许 pack_deps 里声明的；任何包都不能依赖 top 里的包；工作区里每个包都必须出现在 layers.toml。核心检查写成纯函数，测试直接构造依赖关系
  - 测试：detects_upward_dependency、detects_same_level_dependency、allows_declared_pack_dependency、rejects_dependency_on_top_package、all_packages_listed_in_layers_toml（这一个检查真实工作区）
  【验收】tools/check_server.sh gf-check-layers；tools/check_layers.sh；tools/check_all.sh
K00-3 配置加载（阶段 1）
  - load_config(dir) 读取目录里所有 .json 文件（不递归），文件名去掉 .json 就是配置名
  - ConfigSet::get::<T>(name)：先用 schemars 生成 T 的 Schema 并用 jsonschema 校验，再反序列化
  - ConfigSet::ensure_all_used(names)：目录里有没被任何模块读取的文件就报错（防止拼错文件名）
  - load_server_toml(path)：server.toml 只含 port、log_level、dev_mode、config_dir、admins（管理员的账号编号列表，默认空）；数据库地址只从环境变量 DATABASE_URL 读
  - ConfigError 带文件路径；语法错误带行号和列号；Schema 错误带 JSON Pointer（例如 /tiers/1/rate_bp）
  - 测试（测试用的配置类型和文件放 fixtures/core/config/）：valid_config_loads、invalid_json_reports_line、missing_field_rejected_with_path、wrong_type_rejected_with_path、out_of_range_rejected_with_path（用 schemars 的 range 约束）、unknown_config_file_rejected、database_url_only_from_env
  【验收】tools/check_server.sh gf-core
K00-4 数据库（阶段 1）
  - Db::connect(url)、Db::migrate()（sqlx::migrate! 指向 server/migrations）、Db::begin() → Tx；Tx::commit()；Tx 被丢弃而没有提交就回滚
  - 迁移 20261102000000_k00_outbox.sql 建 outbox 表（id BIGSERIAL、name、version、payload JSONB、created_at、delivered_at）
  - 测试（#[sqlx::test]）：migrations_apply_on_empty_db、commit_persists_writes、transaction_rollback_discards_writes
  【验收】tools/dev_db.sh；tools/check_server.sh gf-core
K00-5 事件总线与 outbox（阶段 1）
  - Tx::publish(&event)：把事件写进同一事务里的 outbox；EventBus::publish(event)：自己开一个事务写入
  - 后台分发：按 outbox id 顺序取出已提交、未送出的事件，交给订阅了这个事件名的处理函数，全部处理完标记 delivered_at
  - 处理函数必须幂等（同一事件可能收到不止一次）；失败的处理函数单独重试，最多 server.toml 的 events.retry_max 次、间隔逐次加倍；重试用尽写一条错误日志；其他处理函数不受影响
  - 测试：rollback_publishes_nothing、published_only_after_commit、subscriber_failure_isolated、failed_delivery_retried（失败 2 次后成功，共调用 3 次）、order_preserved_per_publisher
  【验收】tools/check_server.sh gf-core；tools/mutants.sh gf-core src/events.rs
K00-6 时钟、编号、错误、启动组装（阶段 1）
  - Clock / SystemClock / FakeClock（可设置、可前进）；Timestamp（UTC 毫秒 i64）；Timestamp::utc_day() 返回从 1970-01-01 起的天数（UtcDay），UtcDay::start() 返回当天 0 点
  - define_id! 宏：生成 ULID 新类型（serde 用 26 位字符串，实现 Display、FromStr）；用它定义 UserId、PlanetId、EntityId、RegionId；PackId 是字符串新类型（小写字母、数字、下划线，1–32 个字符）
  - CurrentUser、AdminUser 两个 axum 提取器（文档第 4 节）：从请求的扩展里读身份（由 K01 的认证中间件放进去）；没有身份、或者 AdminUser 遇到不是管理员时，返回 PermissionDenied
  - 测试（提取器）：current_user_missing_is_permission_denied、admin_user_rejects_non_admin
  - Error { kind: ErrorKind, message }、Result<T>；HTTP 映射：InvalidInput 400、PermissionDenied 403、NotFound 404、Conflict 409、InsufficientResources 409、QuotaExceeded 429、Internal 500；响应体 {"error": {"kind": "invalid_input", "message": "..."}}
  - gf-app：读 server.toml → 加载配置 → 连接数据库并迁移 → 合并各包的 router → 启动 HTTP；GET /health 返回 200 和 {"status": "ok"}
  - 测试：fake_clock_advances、utc_day_boundaries（86_399_999 → 第 0 天，86_400_000 → 第 1 天）、id_string_roundtrip、pack_id_rejects_invalid_chars、error_kind_http_mapping（7 种全覆盖）、internal_error_message_hidden、health_endpoint_ok（用 tower 的 oneshot，不开端口）
  【验收】tools/check_server.sh gf-core；tools/check_server.sh gf-app；cargo run -p gf-app 之后 curl localhost:<port>/health 返回 ok
K00-7 种子随机数与对象存储（阶段 1）
  - gf_core::rng：seed(parts: &[&[u8]]) -> [u8; 32] = SHA-256(对每一部分依次拼接：4 字节小端长度 + 内容)；from_seed(seed) -> ChaCha8Rng；below(rng, n: NonZeroU64) -> u64（拒绝采样，结果在 [0, n)）；chance_bp(rng, bp: u32) -> bool（below(10000) < bp）
  - fixtures/core/rng_golden.json：parts = ["goldfire", 42 的 8 字节小端] 时的种子十六进制、前 5 个 next_u64、10 次 below(6)、10 次 chance_bp(3000)。这是确定性金标准：用代码生成一次，在 PR 里声明
  - ObjectStore（put / get / exists / delete，按 key 存取字节）；LocalDirStore（开发用，存到本地目录）；MemoryStore（测试用）；key 只允许小写字母、数字和 _ . / : -，不能以 / 开头，不能含 ..，否则 InvalidInput；get 不存在的 key 返回 NotFound
  - 测试：rng_golden_values、below_always_in_range（proptest）、chance_bp_edges（bp = 0 永远 false，bp = 10000 永远 true）、seed_parts_are_length_prefixed（["ab","c"] 和 ["a","bc"] 的种子不同）、local_store_roundtrip、missing_key_is_not_found、key_with_parent_dir_rejected、memory_and_local_stores_behave_the_same
  【验收】tools/check_server.sh gf-core；tools/mutants.sh gf-core src/rng.rs
【验收（每个子任务）】除了子任务里写的命令，还要：tools/check_tests.sh <包> <本子任务点名的测试>
【停止条件】需要的工具（Docker、Godot、cargo-mutants）没装好；任何会让 gf-core 出现业务概念（星球的规则、金、战斗）的需求
~~~
