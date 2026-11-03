# K09 星系（galaxy）

> **层级**：内核 L4 · **代码**：`server/kernel/galaxy` · **首次阶段**：2 · **主文档**：第 7 节

## 1. 职责

- 近 30 天点数排名与轨道环
- 主星、隐藏
- 星系视图数据：前 100 名主星 + 自己 + 好友 + 最近对手 + 星尘点
- 星球搜索
- 记录最近交过手的对手

## 2. 不负责

- PVP 匹配 → P02
- 点数计算 → K08

## 3. 依赖

K00、K01（好友）、K02（星球）；订阅 K08 的事件

## 4. 对外接口

```rust
pub trait Galaxy: Send + Sync {
    fn view(&self, viewer: UserId) -> Result<GalaxyView>;
    fn set_main_planet(&self, user: UserId, planet: PlanetId) -> Result<()>;
    fn set_hidden(&self, user: UserId, hidden: bool) -> Result<()>;
    fn search(&self, query: &str) -> Result<Vec<PlanetSummary>>;
    fn record_opponent(&self, user: UserId, opponent: UserId) -> Result<()>;   // 由 P02 调用
    fn position(&self, user: UserId) -> Result<GalaxyPos>;                     // 星系里的位置
    fn nearest(&self, user: UserId, n: usize) -> Result<Vec<UserId>>;          // 由 P03 用来找"邻近星球"
    fn active_users(&self, days: RangeInclusive<UtcDay>) -> Result<u64>;       // 这段时间有点数记录的玩家数（P03 算 BOSS 血量）
}
```

HTTP：星系视图、设置主星、设置隐藏、搜索

## 5. 数据

| 表 | 关键字段 |
|---|---|
| orbit_entries | user_id, main_planet_id, score_30d, rank, ring, hidden |
| daily_scores | user_id, date, points（用于滚动计算 30 天） |
| recent_opponents | user_id, opponent_id, at |

## 6. 事件

- 发布：`OrbitUpdated`（每天 0 点重排后）
- 订阅：`MeterRecorded`（K08）

## 7. 配置

`orbit.json`：轨道环名额（10 / 50 / 100）、统计天数（30）、星尘点上限

## 8. 关键规则

- 30 天分数不受每日封顶限制，但要乘模型系数
- **星球 AI 自主运转的用量不计入**
- 排名不产生任何数值加成
- 隐藏只影响展示，不影响 P02 的匹配
- 星尘点只发编号和位置，不发星球详情：每个点 24 字节（16 字节编号 + 两个小端 f32）
- 试玩副本的星球不能设为主星

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 2 | 30 天分数与每日重排 | 排名和手工计算一致 |
| 2 | 星系视图 | 一次请求返回视图所需的全部数据；十万个星尘点的数据量可接受 |
| 2 | 主星、隐藏、搜索 | 隐藏后顺位补上 |
| 3 | 最近对手 | P02 结算后对手出现在视图中 |

## 10. 测试要点

- 隐藏的玩家不出现在别人的视图里，但仍能被匹配
- 排名边界（第 10、11 名）的轨道环归属

## 11. 待定问题

- 隐藏是否需要冷却时间

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K09-x"（x 换成子任务编号）。

~~~text
【模块】K09 星系（包 gf-galaxy，路径 server/kernel/galaxy，层级 L4）
【先读】docs/modules/K09-galaxy.md；K01、K02、K08 文档的第 4 节；主文档第 7 节
【范围】server/kernel/galaxy/、server/migrations/（文件名带 k09）、shared/config/orbit.json、fixtures/galaxy/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-account、gf-world、gf-metering（事件类型）。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、schemars。测试：proptest
【规则补充】
  - orbit.json：rings = [10, 50, 100]（第 1 环前 10 名，第 2 环第 11–50 名，第 3 环第 51–100 名）、window_days 30、recent_opponents 5、dust_max 100000
  - 30 天分数 = 最近 window_days 个完整 UTC 日的点数之和（不含今天），不受每日封顶限制，自主运转的用量不算
  - 每天 UTC 0 点后重排一次；分数相同按 UserId 升序
  - 隐藏的玩家不参与排名展示，后面的人顺位补上；但仍然可以被 P02 匹配
  - 星系位置（新增 position、nearest 两个方法）：GoldFire 太阳在原点；环的半径 ring_radii = [100, 200, 300]，环上第 i 名（从 0 数）的角度 = i / 环的名额 × 360°；不在环上的玩家放在半径 400–1000 之间，半径和角度由 gf_core::rng 用 UserId 派生，固定不变；nearest(user, n) 按位置距离返回最近的 n 个玩家（不含自己）
  - 新增 active_users(日期范围) -> 人数：这段时间有点数记录的玩家数（给 P03 算世界 BOSS 血量）
  - 星尘点接口 GET /galaxy/dust 返回二进制：每个点 24 字节（16 字节的 ULID 编号 + 两个小端 f32 的 x、y），最多 dust_max 个
  - 试玩副本（kind = trial）的星球不能设为主星
【子任务】
K09-1 30 天分数与每日重排（阶段 2）
  - 测试：ring_boundaries（第 1、10 名 → 第 1 环；第 11、50 名 → 第 2 环；第 51、100 名 → 第 3 环；第 101 名 → 不在环上）、window_excludes_day_31、autonomous_points_excluded、hidden_users_skipped_and_next_promoted、ties_broken_by_user_id
K09-2 星系视图与位置（阶段 2）
  - 测试：active_users_counts_distinct_players、view_contains_top_own_friends_opponents、dust_binary_size（100000 个点 → 2400000 字节）、position_stable_across_calls、nearest_excludes_self
K09-3 主星、隐藏、搜索（阶段 2）
  - 测试：main_planet_must_be_owned、trial_planet_cannot_be_main、hidden_user_absent_from_others_view、search_by_name_prefix
K09-4 最近对手（阶段 3）
  - 测试：recent_opponents_capped_at_5（记录 7 个对手只保留最近 5 个）、opponent_appears_in_view
【验收（每个子任务）】tools/check_server.sh gf-galaxy；tools/check_tests.sh gf-galaxy <本子任务点名的测试>
【停止条件】排名需要产生数值加成；需要给星尘点发星球详情
~~~
