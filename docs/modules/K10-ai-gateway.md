# K10 AI 网关（ai_gateway）

> **层级**：内核 L2 · **代码**：`server/kernel/ai_gateway` · **首次阶段**：1 · **主文档**：第 26、27 节

## 1. 职责

- **模型适配层**：所有模型通过统一接口调用（OpenAI 兼容接口优先，可扩展其他协议）
- **模型分档路由**：任务 → 档位（强 / 中 / 便宜）→ 具体模型，按 `models.json`
- **结构化输出**：调用方给出 Schema，网关负责解析和校验
- **重试与兜底链**：把错误发回模型重试 → 换备用模型 → 返回"兜底"信号，让调用方使用默认模板
- **示例库**：按任务类型取 3–5 个优质示例放进提示词
- **评审**：用便宜档模型给结果打分
- **缓存**：完全相同的请求复用结果
- **并发、超时、预算控制**
- **额度预留与结算**：每次调用模型前，通过 `CostGate`（K18 实现）按最坏情况预留额度；调用结束按实际成本结算
- **用量上报**：每次调用都报告给 K08（发起人、任务类型、是否自主）

## 2. 不负责

- 业务判断（调用方负责）
- 内容安全（调用方用 K13 检查）

## 3. 依赖

K00、K08、K13（示例进入示例库之前检查）。`CostGate` 由 K18 实现、启动时注入，本模块不依赖 K18

## 4. 对外接口

```rust
pub struct AiRequest {
    pub task: TaskKind,                 // 例如 CreationPick、ChapterExtract、AgentReaction
    pub tier: Option<Tier>,             // 不填就按 models.json 的默认档位
    pub template: PromptRef,            // shared/prompts 里的模板 + 版本
    pub vars: Value,                    // 模板变量
    pub output_schema: Option<Schema>,
    pub examples: ExampleQuery,         // 要几个示例、什么类型
    pub budget: Budget,                 // 最多花多少 token
    pub initiator: Initiator,           // 谁发起，也决定谁付费（见 K18 第 8 节）
}

pub enum Initiator {
    Player(UserId),
    PlanetMindAutonomous { planet: PlanetId, owner: UserId },   // 主人付费，受这颗星球的思考预算限制
    System(SystemPool),                                         // Safety：K13 的 AI 审核；Background：其他后台调用
}

pub struct CostMicros(pub u64);         // 成本：结算货币的百万分之一

#[async_trait]
pub trait CostGate: Send + Sync {       // 由 K18 实现，启动时注入
    async fn reserve(&self, initiator: &Initiator, task: TaskKind, max_cost: CostMicros) -> Result<Reservation>;   // 额度不够返回 QuotaExceeded
    async fn settle(&self, r: Reservation, actual: CostMicros) -> Result<()>;
    async fn release(&self, r: Reservation) -> Result<()>;
}
pub struct Reservation { pub id: ReservationId, pub max_tier: Tier }   // max_tier：这次调用最高能用哪一档

#[async_trait]
pub trait AiGateway: Send + Sync {
    async fn complete(&self, req: AiRequest) -> Result<AiResponse, AiError>;   // AiError::Fallback 表示该用兜底
    async fn review(&self, item: &Value, rubric: RubricRef) -> Result<Score>;
    async fn embed(&self, text: &str) -> Result<Vec<f32>>;                     // 给素材检索、玩法市场搜索
    async fn generate_image(&self, req: ImageRequest) -> Result<GeneratedFile, AiError>;   // 第 4 阶段：铸造的概念图
    async fn generate_3d(&self, req: Model3DRequest) -> Result<GeneratedFile, AiError>;   // 第 4 阶段：铸造的 3D 模型
}
```

- `FakeGateway`（本模块提供）：`scripted(回答列表)` 按顺序返回；`from_dir(目录)` 按"任务 + 变量的 SHA-256"读固定回答，找不到就报错。单元测试和端到端测试都用它

## 5. 数据

| 表 | 关键字段 |
|---|---|
| ai_calls | id, task, model, template_version, tokens, status, latency, initiator, at |
| examples | id, task, input, output, quality_score, source |
| ai_cache | request_hash, response, expires_at |

## 6. 事件

无（用量通过 K08 的接口上报）。

## 7. 配置

- `models.json`：每个档位有哪些模型、每个任务默认档位、备用模型、超时、重试次数、每个模型的单价（输入、输出、缓存读取，单位是微元 / 百万 token）
- `shared/prompts/`：提示词模板（带版本号）

## 8. 关键规则

- **按测试分数分档，不按模型来源**；`models.json` 的修改要先跑 X02 测试集
- 调用记录只存模板版本、token 数、结果状态，**不存玩家的完整内容**
- 重试时把校验错误原文发回模型（"字段 color 必须是 0–15 的整数"）
- 好结果（评审高分、玩家点赞）可以进入示例库，进入前经过 K13 检查
- 调用方传入的预算用完就停止
- **先预留，后调用**：预留额 = 预算 token 数 × 调用链（这一档的模型和备用模型）里最高的单价；预留被拒就不调用模型，把 `QuotaExceeded` 原样返回给调用方
- 任务的档位高于预留结果的 `max_tier` 时，改用 `max_tier` 那一档的模型（成本闸门的降档）
- 实际成本 = 所有尝试的（输入 × 输入单价 + 输出 × 输出单价 + 缓存读取 × 缓存单价）之和 / 1000000，u128 计算，最后向下取整。调用结束（成功、失败、兜底）都要 `settle`；请求一次都没发出时 `release`
- 缓存命中的请求不调用模型：不预留、不扣额度、不上报用量
- `System` 发起的调用不上报 K08（没有玩家得点数）；成本照常记在 K18 的系统预算里

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 适配层 + 一个模型 | 能完成一次结构化输出 |
| 1 | 分档路由、重试与兜底链 | 故意给错格式时能自动修正或兜底 |
| 1 | 用量上报 | 每次调用都有计量记录 |
| 1 | 示例库 | 造物任务自动带示例 |
| 2 | 评审、缓存、embedding | 素材检索可用 |
| 2 | 接入成本闸门 | 额度不够时不调用模型；按实际成本结算 |
| 4 | 图像与 3D 生成接口 | 铸造流程能通过网关调用生成服务 |

## 10. 测试要点

- 模型返回乱码、超时、超预算时的行为
- 换模型后跑 X02 测试集，分数不降

## 11. 待定问题

- 第一批接入哪些模型（由 X02 测试结果决定）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K10-x"（x 换成子任务编号）。

~~~text
【模块】K10 AI 网关（包 gf-ai-gateway，路径 server/kernel/ai_gateway，层级 L2）
【先读】docs/modules/K10-ai-gateway.md；K08-metering.md 第 4 节；K13-moderation.md 第 4 节；K18-billing.md 第 8 节（K10-6）；X02-eval.md；主文档第 26、27 节
【范围】server/kernel/ai_gateway/、server/migrations/（文件名带 k10）、shared/config/models.json、shared/prompts/、fixtures/ai/、server/app/（只限接入本模块：环境变量 GF_AI_FAKE_DIR 存在时注入 FakeGateway::from_dir，否则注入真实网关）
【允许的依赖】模块：gf-core、gf-metering、gf-moderation。库：serde、serde_json、sqlx、tokio、async-trait、tracing、reqwest、jsonschema、schemars、sha2。测试：wiremock、proptest
【规则补充】
  - 模型 API Key 只从环境变量读（变量名写在 models.json 里，例如 "api_key_env": "GF_KEY_DEEPSEEK"）
  - 提示词模板：shared/prompts/<任务>/v<N>.md，变量写成 {{变量名}}；同目录的 meta.toml 声明变量列表和对应的 X02 测试集名
  - FakeGateway（在本包里，测试和 e2e 都用）：FakeGateway::scripted(回答列表) 按顺序返回；FakeGateway::from_dir(目录) 按"任务名 + 变量的 SHA-256"查找 <目录>/<任务>/<哈希>.json，找不到就返回错误（不能临时编造）
  - 重试：Schema 校验失败时，把校验错误原文附在下一次请求里；同一模型重试 models.json 的 retries 次 → 换备用模型 → 全部失败返回 AiError::Fallback
  - 示例库：按任务取 quality_score 最高的 examples_per_call 个
  - 用量上报：每次调用（包括失败和重试）都向 K08 报告，带发起人、任务和是否自主运转；System 发起的调用不报告
  - models.json 的单价：每个模型写 price_micros_per_mtok = {input, output, cache_read}（结算货币的微元 / 百万 token）
  - ai_calls 只存模板版本、token 数、状态、耗时，不存变量和回答正文
【子任务】
K10-1 适配层、模板、FakeGateway（阶段 1）
  - OpenAI 兼容接口；结构化输出按 output_schema 解析并校验
  - 测试：openai_compatible_request_matches_fixture（请求体和 fixtures/ai/requests/creation_pick.json 一致）、structured_output_parsed_and_validated、template_variables_substituted、missing_template_variable_rejected、fake_gateway_scripted_and_from_dir、fake_gateway_miss_is_error
K10-2 分档路由、重试与兜底（阶段 1）
  - 测试：tier_routing_follows_models_json、invalid_output_retried_with_validation_error_in_prompt、fallback_model_used_after_retries、fallback_signal_after_all_models_fail、timeout_counts_as_failure、budget_exhausted_stops_calls
  【验收】另加 tools/mutants.sh gf-ai-gateway src/retry.rs
K10-3 用量上报（阶段 1）
  - 测试：every_call_reports_usage_with_initiator_and_autonomous_flag、failed_and_retried_calls_also_reported、system_calls_not_reported、ai_calls_row_has_no_content
K10-4 示例库（阶段 1）
  - 测试：top_examples_injected_by_quality、example_rejected_by_moderation_not_stored
K10-5 评审、缓存、embedding（阶段 2）
  - 测试：review_score_parsed、identical_request_served_from_cache、cache_respects_expiry、embedding_dimension_matches_config
K10-6 接入成本闸门（阶段 2）
  - CostGate trait、Initiator 的新写法、CostMicros、Reservation 按文档第 4 节；测试用一个写在 tests/ 里的替身记录每次 reserve / settle / release，也可以设成拒绝
  - 测试：reserve_uses_budget_times_highest_chain_price（预算 10000 token；调用链：模型 A 输入 1000000、输出 2000000，模型 B 输入 4000000、输出 16000000（微元 / 百万 token）→ 预留 160000）、quota_exceeded_means_no_model_call（替身拒绝 → wiremock 收到 0 个请求，返回 QuotaExceeded）、settle_actual_cost_across_retries（模型 A 第一次输入 4000、输出 500，校验失败；重试输入 4600、输出 500 → 结算 10600）、max_tier_downgrades_routing（任务默认强档、max_tier 为中档 → 请求发给中档模型）、release_when_nothing_sent（模板变量缺失）、cache_hit_not_charged
  【验收】另加 tools/mutants.sh gf-ai-gateway src/cost.rs
K10-7 图像与 3D 生成接口（阶段 4）
  - 新增 generate_image、generate_3d 两个方法（文档第 4 节补上），返回生成的文件；具体服务商由 models.json 配置
  - 测试：generation_goes_through_fake_provider、generation_usage_reported、generation_timeout_returns_fallback
【效果验收】K10-2 完成后：tools/eval.sh creation 的合格率不低于 models.json 修改前的分数
【验收（每个子任务）】tools/check_server.sh gf-ai-gateway；tools/check_tests.sh gf-ai-gateway <本子任务点名的测试>
【停止条件】需要真实的 API Key 才能跑单元测试；需要在数据库里存回答正文；还没决定接入哪个模型（由 X02 的测试结果决定）；需要在本模块里知道灵感、余额或支付（那是 K18 的事）
~~~
