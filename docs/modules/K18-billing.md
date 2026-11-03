# K18 额度与支付（billing）

> **层级**：内核 L3 · **代码**：`server/kernel/billing` · **首次阶段**：2（额度、成本闸门）；3（支付）· **主文档**：第 8.4 节、玩法总览

## 1. 职责

- **AI 额度账户**：每个玩家的每日免费额度、月卡加赠、购买余额。玩家看到的单位是**灵感**
- **预留与结算**：每次调用模型之前，按最坏情况预留额度；调用结束按实际成本结算，多的退回
- **谁付费**：按发起人决定扣谁的额度：玩家、星球主人，或全服系统预算
- **思考预算的执行**：每颗星球每天自主运转最多花主人多少灵感（数值由主人在 K16 里设置）
- **成本闸门**：按"今天官方替玩家承担的 AI 成本"分级降档、收紧、关闭免费额度；另有一条全服硬上限
- **防刷**：每个玩家同时进行中的调用数、每分钟的调用数
- **支付**：商品（额度包、月卡）、订单、支付渠道回调、到账、退款
- **收入与成本报表**：每天的收入、各类 AI 成本、毛利

## 2. 不负责

- 模型单价和一次调用的实际成本 → K10（单价写在 `models.json`）
- token 用量换算资源点 → K08、K07
- 金和火 → K07。**支付只买灵感，不买金和火**
- 注册、登录、令牌 → K01
- 星球 AI 的设置界面、"预算用完"通知主人 → K16、C10
- 商店界面 → C07

## 3. 依赖

K00、K01（HTTP 接口用 `CurrentUser` 拿到当前用户）、K10（实现 K10 定义的 `CostGate`，使用它的 `Initiator`、`TaskKind`、`Tier`、`CostMicros`）

gf-app 在启动时把本模块作为 `CostGate` 注入 K10（依赖倒置，见主文档第 31 节）。K10 不知道灵感和支付的存在。

## 4. 对外接口

```rust
// 实现 K10 定义的 CostGate：reserve / settle / release（trait 见 K10 第 4 节）

pub struct QuotaBalance {
    pub free_left: u64,              // 今日免费额度还剩多少（微元，下同）
    pub bonus_left: u64,             // 今日月卡加赠还剩多少
    pub purchased: u64,              // 购买余额，不过期
    pub debt: u64,                   // 退款欠额，下次购买先抵扣
    pub pass_until: Option<Date>,    // 月卡有效到哪天（含这一天，UTC）
    pub gate: GateLevel,
}
pub enum GateLevel { Normal, Downgrade, Restricted, FreeClosed, HardStop }   // 正常 / 降档 / 收紧 / 免费关闭 / 全服停止
pub struct Estimate { pub inspiration: u64, pub needs_confirm: bool }

#[async_trait]
pub trait Billing: Send + Sync {
    async fn balance(&self, user: UserId) -> Result<QuotaBalance>;
    async fn set_planet_budget(&self, planet: PlanetId, owner: UserId, daily_inspiration: u64) -> Result<()>;   // K16 保存思考预算时调用
    async fn planet_spent_today(&self, planet: PlanetId) -> Result<u64>;
    fn estimate(&self, task: TaskKind, units: u32) -> Estimate;      // 按配置估算，供预览和"花费较大，确认吗"
    async fn gate_level(&self) -> Result<GateLevel>;
}

#[async_trait]
pub trait PaymentProvider: Send + Sync {          // 支付渠道；本模块提供 FakeProvider，真实渠道等官方服地区定了再接
    async fn create_checkout(&self, order: &Order) -> Result<CheckoutUrl>;
    async fn verify_webhook(&self, headers: &HeaderMap, body: &[u8]) -> Result<ProviderEvent>;   // 验签失败返回 PermissionDenied
    async fn refund(&self, order: &Order) -> Result<()>;
}
```

HTTP（灵感一律按向下取整显示）：

| 方法 | 路径 | 作用 |
|---|---|---|
| GET | `/billing/quota` | 我的额度和当前闸门级别 |
| GET | `/billing/estimate?task=&units=` | 预计花费、是否需要确认 |
| GET | `/billing/products` | 商品列表 |
| POST | `/billing/orders` | 下单，返回订单号和支付页面地址 |
| GET | `/billing/orders/{id}` | 订单状态（客户端轮询） |
| POST | `/billing/webhooks/{provider}` | 支付渠道回调：不需要登录，靠验签 |
| GET | `/admin/billing/daily?date=` | 某一天的收入与成本报表 |
| POST | `/admin/billing/grants` | 运营补偿：给某个玩家加购买余额，必须写原因 |

- 支付关闭时，商品、下单、订单、回调四个接口不注册（访问返回 404）
- `/admin/` 接口要求请求头 `X-GF-Admin-Token` 等于环境变量 `GF_ADMIN_TOKEN`；这个环境变量没设置时，`/admin/` 接口不注册

## 5. 数据

| 表 | 关键字段 |
|---|---|
| quota_accounts | user_id, purchased, debt, updated_at |
| quota_daily | user_id, date, free_used, bonus_used, paid_used |
| planet_budgets | planet_id, owner_id, daily_micros |
| planet_daily | planet_id, date, spent |
| reservations | id, payer, user_id, planet_id, pool, task, amount, from_free, from_bonus, from_purchased, max_tier, status（预留 / 已结算 / 已释放 / 已过期）, expires_at, created_at |
| quota_ledger | id, user_id, kind（购买 / 扣款 / 退回 / 退款 / 欠额抵扣 / 运营补偿）, amount, ref_id, reason, at |
| cost_daily | date, bucket（免费 / 月卡 / 购买 / 系统安全 / 系统后台 / 超出预留）, amount |
| passes | user_id, order_id, first_day, last_day |
| orders | id, user_id, product_id, price_minor, currency, provider, provider_ref, status（待支付 / 已支付 / 已退款 / 已过期）, created_at, paid_at, refunded_at |
| payment_events | provider, event_id（与 provider 一起唯一）, order_id, kind, received_at |

- 额度和成本一律用**微元**（结算货币的百万分之一），BIGINT；只有 `orders.price_minor` 用货币的最小单位（例如分）
- 不存支付卡号和支付账户信息，只存渠道的订单号
- 不存 AI 调用的内容

## 6. 事件

- 发布：`OrderPaid`、`OrderRefunded`、`GateLevelChanged`、`PlanetBudgetExhausted`（K16 订阅，用来通知主人）
- 订阅：无

## 7. 配置

`billing.json`：

| 字段 | 内容 |
|---|---|
| `micros_per_inspiration` | 1 灵感等于多少微元的成本 |
| `daily_free_inspiration` | 每天的免费额度 |
| `reservation_ttl_secs`、`max_inflight_per_user`、`max_calls_per_minute` | 预留多久没结算就退回；防刷 |
| `task_estimates`、`default_estimate`、`confirm_above_inspiration` | 每种任务"一次大约花多少灵感"；超过多少要玩家确认 |
| `gate` | 每日补贴预算、各级的触发比例、全服硬上限、安全预算和后台预算的每日上限 |
| `payments` | 是否开启（社区服默认关闭）、结算货币、渠道、手续费、最低毛利、订单有效期、单日和单月购买限额 |
| `products` | 商品：额度包（价格、灵感数）、月卡（价格、每天加赠多少、多少天） |

## 8. 关键规则

**灵感按成本定义**

- 1 灵感 = 固定的一份 AI 成本（例如 0.001 元）。账户里存的是微元，显示时换算成灵感、向下取整
- 这样一个商品包含的成本是确定的，定价可以保证不亏本（见下文"定价检查"）；换了更贵的模型，同样的灵感能做的次数变少，玩家看得见
- 灵感**不能转让、不能交易**；**玩法和玩法包都不能产生灵感**。来源只有每日免费、月卡、购买和运营补偿

**谁付费**

| 发起人（K10 的 `Initiator`） | 扣谁的额度 | 例子 |
|---|---|---|
| `Player(user)` | 这个玩家 | 自己对星球 AI 说话；在别人的星球上帮忙建造；和 AI 人物聊天；施加神迹；外部 agent 用令牌调用 |
| `PlanetMindAutonomous { planet, owner }` | 星球主人，同时受这颗星球的思考预算限制 | 每日反思、剧情自动推进、AI 人物按节拍行动 |
| `System(Safety)` | 全服系统预算：安全 | K13 的 AI 审核 |
| `System(Background)` | 全服系统预算：后台 | K12 素材打分和向量、K10 评审、野生灾厄的外观、AI 战报 |

**扣款顺序**：今日免费 → 今日月卡加赠 → 购买余额；结算时多预留的部分按相反顺序退回。免费额度和月卡加赠只在当天有效，UTC 0 点重置，不累积。

**预留与结算**

- K10 每次调用模型前调用 `reserve(initiator, task, max_cost)`。max_cost 由 K10 算出：预算 token 数 × 调用链里最高的单价
- 可用额度不够 max_cost → 返回 `QuotaExceeded`，模型不会被调用。错误信息说明是哪一种不够：个人额度、思考预算、全服闸门、调用太频繁。客户端按这个提示玩家
- 调用结束（成功、失败、兜底都算）K10 调用 `settle(r, actual)`：按实际成本扣，多的退回。实际超过预留（不应该发生）时只扣预留的数，差额记进 cost_daily 的"超出预留"，写一条 warn 日志
- 请求一次都没发出去时，K10 调用 `release(r)`：全部退回
- 超过 `reservation_ttl_secs` 没结算的预留（例如服务器崩溃），由后台任务全部退回，状态记为"已过期"
- 余额和预留在同一个数据库事务里更新（锁住这个玩家的账户行），并发预留不会透支
- 购买余额永远等于 quota_ledger 之和

**思考预算**

- 主人在 K16 里设置每颗星球每天自主运转最多花多少灵感（`daily_ai_budget`）；K16 保存时调用 `set_planet_budget`
- 自主运转的预留要同时满足：主人的额度够；这颗星球今天已结算的花费 + 进行中的预留 + 这次预留 ≤ 思考预算
- 不满足就拒绝，并发布 `PlanetBudgetExhausted`（同一颗星球同一天只发一次）
- 主人自己发起的请求不受思考预算限制，只扣主人的个人额度

**成本闸门**

"官方承担的成本" = 今天从免费额度扣掉的 + 系统预算花掉的，按已结算的金额计算。月卡和购买余额是玩家付过钱的，不算在内。

| 级别 | 触发 | 效果 |
|---|---|---|
| 正常 | 官方承担的成本 < 每日补贴预算的 70% | 无限制 |
| 降档 | ≥ 70% | 用到免费额度的调用最高中档 |
| 收紧 | ≥ 90% | 用到免费额度的调用最高便宜档；系统后台预算暂停；自主运转跳过免费额度（主人有月卡加赠或购买余额时照常） |
| 免费关闭 | ≥ 100% | 所有调用都跳过免费额度：有月卡加赠或购买余额的照常，没有的拒绝 |
| 全服停止 | 今天的全部成本 ≥ 全服硬上限 | 除了系统安全预算，全部拒绝 |

- 比例都是基点，写在 `gate` 里
- 最高档位通过预留结果的 `max_tier` 告诉 K10，由 K10 换成更低档位的模型
- 安全预算（K13 的 AI 审核）不受前四级影响，只受自己的每日上限。超出时返回 `QuotaExceeded`，K13 按它的规则当作"检查失败"处理，也就是拒绝（宁可误拒，不能放行）
- 后台预算另有自己的每日上限
- 级别升高时写一条 warn 日志并发布 `GateLevelChanged`（同一天每个级别只发一次）；UTC 0 点回到正常

**防刷**

- 每个付费人同时进行中的预留最多 `max_inflight_per_user` 个；每分钟（按 `Clock` 的滑动窗口）最多 `max_calls_per_minute` 次。超过返回 `QuotaExceeded`（原因"太频繁"）
- 系统预算不受这两条限制

**支付**

- 支付关闭时（`payments.enabled = false`，社区服默认），只有每日免费额度和运营补偿
- 下单：检查商品存在、单日和单月购买限额 → 创建订单（待支付）→ 调用渠道的 `create_checkout` → 把支付页面地址交给客户端。客户端用系统浏览器打开，**游戏客户端不接触任何支付信息**
- **到账只认渠道的回调**，不认客户端说"我付过了"：验签 → 按 (provider, event_id) 去重 → 在一个事务里把订单改成已支付、写流水、先抵扣欠额、再加购买余额。同一个回调来几次，都只到账一次
- 订单超过 `order_ttl_secs` 没支付就过期。过期后又收到支付成功的回调：照常到账（钱已经收了），写一条 warn 日志
- 月卡：从支付当天（UTC）起连续 `days` 天，每天多 `daily_bonus_inspiration` 灵感，当天有效。有效期内再买，从原来的最后一天往后顺延
- 退款（渠道回调或运营操作）：从购买余额扣回这笔订单的灵感，不够的部分记为欠额，下次购买先抵扣，不产生负余额。月卡退款从退款当天起不再加赠
- **定价检查**：每个商品都要满足 售价 × `micros_per_minor` × (10000 − `fee_bp`) / 10000 ≥ 包含的灵感 × `micros_per_inspiration` × (10000 + `min_margin_bp`) / 10000；月卡包含的灵感 = `days` × `daily_bonus_inspiration`。不满足就拒绝启动（K00 配置校验）
- 单日和单月购买限额是防冲动消费的底线；实名、年龄和地区相关的规则，等官方服的地区定了再补（第 11 节）

**报表**

- 收入 = 当天已支付的金额 − 当天退款的金额 − 当天已支付的金额 × `fee_bp`
- 成本按 cost_daily 的各类分别列出；毛利 = 收入 − 全部成本；另列当天有 AI 用量的玩家数，以及每个这样的玩家平均花掉多少免费额度
- 报表只有数字，没有玩家的对话内容

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 2 | 额度账户、每日免费、扣款顺序 | 购买余额永远等于流水之和；UTC 0 点免费额度重置 |
| 2 | 预留与结算、思考预算、防刷 | 并发预留不透支；没结算的预留到期自动退回 |
| 2 | 成本闸门 | 各级按比例触发；付费的灵感只受全服停止限制 |
| 3 | 商品、订单、支付回调（假渠道） | 同一个回调重复到达只到账一次；亏本的商品不能上架 |
| 3 | 月卡、退款、报表 | 退款不产生负余额；报表和流水一致 |

## 10. 测试要点

- 并发预留、重复回调、回调先于下单结果到达
- 主文档 8.4 的例子作为自动化测试
- 闸门各级的边界值

## 11. 待定问题

- 官方服的地区和支付渠道：决定接哪家支付、要不要实名和防沉迷、能不能用境外模型
- 未成年人的消费限制：现在只有通用的单日和单月限额
- 每日免费额度、灵感和商品的价格：要按真实的 AI 成本和留存数据调整
- 观众能不能给剧本星球"打赏"灵感，替主人承担剧情推进的费用
- AI 战报（K14）的成本：现在记在系统后台预算里
- 恶意退款怎么处理：现在只记欠额

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K18-x"（x 换成子任务编号）。

~~~text
【模块】K18 额度与支付（包 gf-billing，路径 server/kernel/billing，层级 L3）
【先读】docs/modules/K18-billing.md；K10-ai-gateway.md 第 4、8 节和【规则补充】；K01-account.md 第 4 节（CurrentUser）；K13-moderation.md 第 8 节；主文档第 8.4 节和第 36 节
【范围】server/kernel/billing/、server/migrations/（文件名带 k18）、shared/config/billing.json、fixtures/billing/、server/app/（只限接入本模块：把本模块作为 CostGate 注入 K10；注入 FakeProvider）
【允许的依赖】模块：gf-core、gf-account（只用 CurrentUser）、gf-ai-gateway（只用 CostGate、Reservation、Initiator、SystemPool、TaskKind、Tier、CostMicros）。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、schemars、hmac、sha2、subtle。测试：proptest
【规则补充】
  - billing.json：
      micros_per_inspiration 1000；daily_free_inspiration 200；reservation_ttl_secs 600；max_inflight_per_user 3；max_calls_per_minute 30
      task_estimates：CreationPick 10、ChapterExtract 15、AgentReaction 3（灵感 / 次）；default_estimate 20；confirm_above_inspiration 100
      gate：daily_subsidy_budget_micros 100000000；downgrade_at_bp 7000；restrict_at_bp 9000；free_closed_at_bp 10000；hard_cap_micros 300000000；safety_daily_cap_micros 20000000；background_daily_cap_micros 10000000
      payments：enabled false；currency "CNY"；micros_per_minor 10000；provider "fake"；fee_bp 100；min_margin_bp 3000；order_ttl_secs 1800；daily_purchase_limit_minor 20000；monthly_purchase_limit_minor 100000
      products：pack_small（600 分，3000 灵感）；pack_large（3000 分，16000 灵感）；pass_30（1500 分，每天 300 灵感，30 天）
  - 额度和成本一律是微元（u64，中间计算用 u128）；只在显示成灵感时向下取整
  - 扣款顺序：免费 → 月卡加赠 → 购买余额；退回顺序相反
  - Initiator → 付费人：Player(user) 扣 user；PlanetMindAutonomous { planet, owner } 扣 owner，并检查这颗星球的思考预算；System(pool) 扣对应的系统预算
  - estimate(task, units) = task_estimates[task]（没列出的用 default_estimate）× units；needs_confirm = 结果 > confirm_above_inspiration
  - 闸门按"官方承担的成本"（今天已结算的、从免费额度扣掉的 + 系统预算花掉的）计算；级别升高时发布 GateLevelChanged，同一天每个级别只发一次
  - FakeProvider（在本包里，测试和 e2e 都用）：create_checkout 返回 fake://checkout/<订单号>；回调体是 JSON {event_id, order_id, kind}，签名是回调体的 HMAC-SHA256（密钥来自环境变量 GF_FAKE_PAY_SECRET，测试里自己设置），十六进制放在请求头 X-GF-Signature
  - /admin/ 接口：请求头 X-GF-Admin-Token 等于环境变量 GF_ADMIN_TOKEN（用 subtle 做常量时间比较）；环境变量没设置时不注册这些路由
  - 报表：收入 = 当天已支付 − 当天退款 − 当天已支付 × fee_bp / 10000；毛利 = 收入 − 全部成本
【子任务】
K18-1 额度账户、每日免费、扣款顺序（阶段 2）
  - 测试：free_quota_resets_at_utc_midnight（用掉 150 灵感 → 剩 50；FakeClock 跨过 UTC 0 点 → 200）、purchased_equals_ledger_sum（proptest：随机的购买、扣款、退回序列）、display_rounds_down（1999 微元显示为 1 灵感）、admin_grant_requires_reason_and_token、estimate_and_confirm_threshold（ChapterExtract × 20 → 300 灵感，需要确认；CreationPick × 1 → 10 灵感，不需要；没列出的任务 × 1 → 20）
K18-2 预留与结算、思考预算、防刷（阶段 2）
  - 测试：reserve_draws_free_then_bonus_then_purchased（免费剩 50000、月卡加赠 0、购买 1000000；预留 80000 → 免费 0、购买 970000；结算实际 60000 → 购买 990000、免费仍是 0）、reserve_rejected_without_change（免费 10000、购买 0，预留 160000 → QuotaExceeded，余额不变）、concurrent_reserve_never_overdraws（可用 200000，20 个并发预留各 20000 → 正好 10 个成功）、settle_above_reservation_charges_reserved_only（预留 160000、实际 200000 → 扣 160000，"超出预留"记 40000）、release_returns_everything、expired_reservation_released（过 601 秒后全部退回，状态为已过期）、planet_budget_limits_autonomous_only（思考预算 30 灵感：自主预留 20000 成功，再自主预留 20000 被拒；主人自己预留 20000 成功）、planet_budget_exhausted_published_once_per_day、inflight_and_rate_limits（第 4 个同时进行的预留被拒；一分钟内第 31 次被拒）
  【验收】另加 tools/mutants.sh gf-billing src/reserve.rs
K18-3 成本闸门（阶段 2）
  - 测试：gate_levels_by_subsidized_cost（官方承担的成本 69999999 → 正常；70000000 → 降档，用到免费额度的预留 max_tier 为中档；90000000 → 收紧，最高便宜档，System(Background) 被拒；100000000 → 免费关闭：只有免费额度的玩家被拒，有购买余额的玩家照常、从购买余额扣）、paid_funds_ignore_subsidy_levels、hard_cap_stops_all_but_safety（今天的全部成本 300000000 → 只有 System(Safety) 能预留）、safety_pool_own_cap（安全预算用到 20000000 后被拒）、gate_resets_at_utc_midnight、gate_level_changed_published_once_per_level
  【验收】另加 tools/mutants.sh gf-billing src/gate.rs
K18-4 商品、订单、支付回调（阶段 3）
  - 测试：pricing_check_rejects_loss_making_product（售价 300 分、3000 灵感 → 配置校验失败；pack_small、pack_large、pass_30 通过）、order_paid_credited_once（同一个回调发 3 次 → 购买余额只加 3000000）、bad_signature_rejected（PermissionDenied，余额不变）、unknown_order_rejected、order_expires_after_ttl、late_payment_after_expiry_still_credited、purchase_limits_enforced（今天已付 19800 分，再买 600 分的商品被拒）、payments_disabled_routes_404
K18-5 月卡、退款、报表（阶段 3）
  - 测试：pass_bonus_for_30_days（第 1 天到第 30 天每天加赠 300 灵感，第 31 天没有；有效期内再买一张，顺延到第 60 天）、refund_deducts_then_records_debt（买 pack_small、从购买余额用掉 1000 灵感后退款 → 购买余额 0、欠额 1000000；再买 pack_small → 欠额 0、购买余额 2000000）、refund_idempotent、pass_refund_stops_bonus_from_today、daily_report_numbers（两笔 pack_small 已支付，其中一笔当天退款；成本：免费 3000000、购买 1000000、系统安全 500000、系统后台 200000；15 个玩家有用量 → 收入 5880000、全部成本 4700000、毛利 1180000、平均每人免费成本 200000）
【验收（每个子任务）】tools/check_server.sh gf-billing；tools/check_tests.sh gf-billing <本子任务点名的测试>
【停止条件】要接真实的支付渠道（先问我选哪家、在哪个地区）；需要实名、年龄验证或地区相关的消费规则；需要保存支付卡号或支付账户信息；需要让支付产生金或火；需要让灵感可以转让，或让玩法产生灵感
~~~
