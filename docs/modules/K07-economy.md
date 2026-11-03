# K07 经济（economy）

> **层级**：内核 L2 · **代码**：`server/kernel/economy` · **首次阶段**：2 · **主文档**：第 8、12 节

## 1. 职责

- **每日资源点结算**：收益递减、每日封顶、每日保底
- **个人能量池**
- **转换**：按转换速度把能量池里的点数变成金和火（访问时补算）
- **个人账户**：金、火的扣款与入账
- **流水**：每一笔变动都记录（包括花在哪颗星球上）
- **掠夺执行**：按调用方给出的上限，从一个玩家转给另一个玩家
- **全服修正**：例如世界 BOSS 失败后全服转换速度 −10%（只接受官方调用）

## 2. 不负责

- 统计 token → K08
- 转换速度怎么算 → P01（通过 `ConversionRateProvider` 提供）
- 掠夺上限怎么算 → P02

## 3. 依赖

K00；订阅 K08 的事件

## 4. 对外接口

```rust
pub trait ConversionRateProvider: Send + Sync {          // 由 P01 实现
    fn rate(&self, user: UserId) -> ConversionRate;      // 每小时转换多少点成金、多少点成火
}

pub trait Economy: Send + Sync {
    fn wallet(&self, user: UserId) -> Result<Wallet>;    // 读之前先补算转换
    fn spend(&self, user: UserId, res: Resource, amount: u64, reason: Reason, planet: Option<PlanetId>) -> Result<()>;
    fn grant(&self, caller: OfficialCaller, user: UserId, res: Resource, amount: u64, reason: Reason) -> Result<()>;
    fn loot(&self, caller: OfficialCaller, from: UserId, to: UserId, cap: u64) -> Result<LootResult>;
    fn spent_on_planet(&self, user: UserId, planet: PlanetId) -> u64;   // 用于计算掠夺上限
    fn set_global_modifier(&self, caller: OfficialCaller, m: Modifier, until: Timestamp) -> Result<()>;
}
```

- `grant`、`loot`、`set_global_modifier` 只接受官方调用（`OfficialCaller` 只能由官方玩法包获得）
- HTTP：查询我的账户和流水

## 5. 数据

| 表 | 关键字段 |
|---|---|
| wallets | user_id, gold, fire, energy_pool, last_converted_at, gold_carry, fire_carry（转换的余数，保证分多次补算和一次补算结果相同） |
| daily_points | user_id, date, raw_points, granted_points |
| ledger | id, user_id, type（入账 / 扣款 / 转换 / 掠夺）, resource, amount, reason, planet_id, ref_id, at |
| global_modifiers | id, kind, value, until |

## 6. 事件

- 发布：`ResourcesChanged`
- 订阅：`MeterRecorded`（K08）→ 累加当日点数

## 7. 配置

`economy.json`：递减档位、每日封顶、保底、能量池上限、无产出建筑时的默认转换速度、掠夺比例、是否计入星球 AI 自主运转的用量

## 8. 关键规则

- **结算公式**（见主文档 8.2）：0–5 万点 100%，5–20 万 50%，20–50 万 10%，50 万以上 0；每日最多 15.5 万；保底 1 万
- **星球 AI 自主运转的用量不计入资源点**：`MeterRecorded` 里带"是否自主"的标记，按配置决定是否忽略
- **每日结算对谁**：每天 UTC 0 点后，对前一天有用量记录（不算自主运转的）的玩家各结算一次；同一玩家同一天只结算一次
- **补算转换**：查询或扣款前，先按（现在 − 上次转换时间）× 转换速度算出转换量，不超过能量池余额；金和火分别带余数计算；全服修正的开始和到期时间把这段时间切成几段，每段用当时的速度
- **转换速度变化前先补算**：P01 在产出建筑新增、删除、停工、恢复之前，先调用 `wallet(user)`，把旧速度下的转换算完
- **掠夺上限 = 主人自己在这颗星球上花的金 × 掠夺比例**：`spent_on_planet` 只统计主人本人的流水
- **账户不够扣**：实际转走 = min(上限, 当前的金)，扣到 0 为止，不产生负债
- **玩法包不能调用会产生或转移金火的接口**
- 每一笔变动都写流水，流水只增不改

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 2 | 账户、流水、扣款 | 余额永远等于流水之和 |
| 2 | 每日结算 | 主文档 8.5 的小明和大户例子算出 5.2 万和 15.5 万 |
| 2 | 能量池与补算转换 | 小明的例子：约 10.4 小时转完，得到 3.12 万金、2.08 万火 |
| 3 | 掠夺 | 上限正确；账户不够时扣到 0 为止 |
| 4 | 全服修正 | 世界 BOSS 失败后转换速度下降，到期恢复 |

## 10. 测试要点

- 并发扣款不会扣成负数
- 主文档里的所有数字例子作为自动化测试

## 11. 待定问题

- 无（"掠夺时账户不够扣"已决定：扣到 0 为止，见主文档第 36 节）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K07-x"（x 换成子任务编号）。

~~~text
【模块】K07 经济（包 gf-economy，路径 server/kernel/economy，层级 L2）
【先读】docs/modules/K07-economy.md；K08-metering.md 第 4、6 节；主文档第 8、12 节和第 36 节
【范围】server/kernel/economy/、server/migrations/（文件名带 k07）、shared/config/economy.json、fixtures/economy/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-metering（只用它的事件类型）。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、schemars。测试：proptest
【规则补充】
  - economy.json：tiers = [{up_to: 50000, rate_bp: 10000}, {up_to: 200000, rate_bp: 5000}, {up_to: 500000, rate_bp: 1000}]（50 万以上为 0）；daily_floor 10000；energy_pool_cap 500000；default_rate_per_hour 1000；default_gold_share_bp 5000；loot_ratio_bp 1000；count_autonomous_usage false
  - 每日结算：每天 UTC 0 点后，对前一天有用量记录（不算自主运转的）的玩家各结算一次；同一玩家同一天只结算一次（重复执行不重复发放）；结算结果进能量池，超过 energy_pool_cap 的部分丢弃
  - 补算转换（读余额或扣款之前）：
      ms = 现在 − 上次转换时间；金 = (金余数 + 金速度 × ms) / 3600000，金余数 = 取余（火同样计算）
      如果 金 + 火 > 能量池：金 = 能量池 × 金速度 / (金速度 + 火速度)（向下取整），火 = 能量池 − 金，两个余数清零
      能量池 −= 金 + 火；上次转换时间 = 现在
      全服修正的开始和到期时间把这段时间切成几段，每段用当时的速度
  - 没有提供者或提供者返回 0 时用默认速度：每小时 default_rate_per_hour，金占 default_gold_share_bp
  - 掠夺：实际转走 = min(上限, 被掠夺者当前的金)；不产生负债
  - wallets 表加 gold_carry、fire_carry 两列
【子任务】
K07-1 账户、流水、扣款（阶段 2）
  - 测试：balance_equals_ledger_sum（proptest：随机的入账和扣款序列）、concurrent_spend_never_negative（20 个并发扣款）、spend_records_planet_id、insufficient_balance_rejected_without_changes
K07-2 每日结算（阶段 2）
  - 纯函数 settle_points(raw) 加上每日任务
  - 测试：daily_settlement_examples（54000 → 52000；1500000 → 155000；200000 → 125000；500000 → 155000；50000 → 50000；12000 → 12000；5000 → 10000）、settlement_properties（proptest：结果 ≤ 155000、≥ 10000、随 raw 不减）、autonomous_usage_ignored_when_configured、settlement_idempotent_per_day、inactive_user_not_settled、pool_cap_enforced
  【验收】另加 tools/mutants.sh gf-economy src/settle.rs
K07-3 能量池与补算转换（阶段 2）
  - 测试：xiaoming_conversion_example（能量池 52000，金每小时 3000、火每小时 2000，过 37440 秒 → 金 31200、火 20800、能量池 0）、conversion_stops_when_pool_empty（同样的条件过 72000 秒，结果相同）、split_access_equals_single_access（proptest：能量池不耗尽时，把一段时间拆成任意多次访问，结果和一次访问完全相同）、default_rate_without_provider（每小时金 500、火 500）
  【验收】另加 tools/mutants.sh gf-economy src/convert.rs
K07-4 掠夺（阶段 3）
  - spent_on_planet 只统计这颗星球主人本人的扣款流水
  - 测试：loot_cap_counts_owner_spending_only（主人花 20000、帮忙的人花 100000 → 上限 2000）、loot_limited_by_balance（上限 2000、余额 500 → 转走 500）、loot_requires_official_caller、loot_written_to_both_ledgers
K07-5 全服修正（阶段 4）
  - 测试：global_modifier_reduces_rate_then_expires（−10% 修正：每小时 5000 → 4500，到期后回到 5000）、modifier_split_at_expiry（一次补算跨过到期时间，前后两段分别按 4500 和 5000 计算）
【验收（每个子任务）】tools/check_server.sh gf-economy；tools/check_tests.sh gf-economy <本子任务点名的测试>
【停止条件】需要知道具体建筑（那是 P01 的事，通过 ConversionRateProvider 提供）；需要让玩法包产生或转移金火
~~~
