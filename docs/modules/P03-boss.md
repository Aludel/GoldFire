# P03 BOSS（boss）

> **层级**：星球层 L8 · **所属玩法包**：星球攻防（官方）· **代码**：`server/packs/war/boss` · **首次阶段**：4 · **主文档**：第 13 节

## 1. 职责

- **野生灾厄**：环境失衡时生成；72 小时计时；跃迁到邻近星球，或被世界 BOSS 吸收
- **世界 BOSS**：每周一只 + 赛季末大 BOSS；全服共享血量；弱点标签；每人每天 3 次；三层奖励；失败惩罚
- 调用 P02 的 `set_truce` 开关停战期

## 2. 不负责

- 伤害计算（复用 P02 的结算函数）
- 外观生成（调用 K11）
- 资源发放和扣除（调用 K07）

## 3. 依赖

K00、K02、K03、K04、K07、K09（邻近星球、上周活跃人数）、K11、P01、P02（声明在 manifest）

## 4. 对外接口

**注册的能力**：攻击 BOSS（官方级，消耗火）

HTTP：当前世界 BOSS 信息、贡献排行。WebSocket：世界 BOSS 频道（血量广播）

## 5. 数据

| 表 | 关键字段 |
|---|---|
| wild_bosses | id, planet_id, recipe, hp, spawned_at, deadline, status（存活 / 击败 / 跃迁 / 被吸收） |
| world_bosses | id, week, hp_total, hp_left, weakness_tags, phase, starts_at, ends_at, absorbed_count |
| boss_damage | boss_id, user_id, planet_id, damage, attacks_today |

## 6. 事件

- 发布：`WildBossSpawned`、`WildBossEscaped`、`WorldBossStarted`、`WorldBossEnded`
- 订阅：P01 的 `EnvironmentThresholdReached`

## 7. 配置

`boss.json`：灾厄存活时长、跃迁规则、世界 BOSS 血量公式（按上周活跃人数）、每日次数、奖励表、失败惩罚

## 8. 关键规则

- 灾厄外观只在诞生时生成一次，之后固定为数据；战斗照常由代码结算
- 世界 BOSS 血量 = 每个活跃玩家的血量 × 上周活跃人数（K09 的 `active_users`）
- 灾厄到期没被击败：世界 BOSS 在场就被吸收，否则跃迁到 K09 `nearest` 找到的第一个不在保护期的玩家的主星
- 弱点标签：带对应标签的武器伤害 ×2
- 超时失败 → 通过官方权限让 K07 全服转换速度 −10%，持续一周
- 没被讨伐的野生灾厄飞向 GoldFire 太阳，让世界 BOSS 变强

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 4 | 野生灾厄 | 「熔铸巨像」从火山星球诞生，72 小时后跃迁 |
| 4 | 世界 BOSS | 多个客户端同时攻击，血量一致 |
| 4 | 奖励与惩罚 | 参与奖、排行、失败惩罚都正确 |

## 10. 测试要点

- 并发攻击时血量不会算错
- 停战期开关与 P02 同步

## 11. 待定问题

- GoldFire 太阳的世界观与剧情

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 P03-x"（x 换成子任务编号）。

~~~text
【模块】P03 BOSS（包 gf-pack-boss，路径 server/packs/war/boss，层级 L8，属于官方玩法包 war）
【先读】docs/modules/P03-boss.md；P01、P02 文档的第 4、6 节；K07、K09、K11 文档的第 4 节；主文档第 13 节
【范围】server/packs/war/boss/、server/migrations/（文件名带 p03）、shared/config/boss.json、fixtures/boss/、fixtures/ai/boss_*/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-world、gf-scheduler、gf-capability、gf-economy、gf-galaxy、gf-creation、gf-pack-supply、gf-pack-combat。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、schemars。测试：proptest
【规则补充】
  - 能力名：war.攻击BOSS（官方级，消耗火）
  - 野生灾厄：收到 EnvironmentThresholdReached 就在那颗星球生成；外观只在诞生时调用一次 K11 的 appearance_for，之后作为数据保存；期限 boss.json 的 wild_lifetime_hours（72）
  - 到期没被击败：如果世界 BOSS 在场 → 被吸收（世界 BOSS 的 hp_total 和 hp_left 各加 absorb_hp，absorbed_count + 1）；否则跃迁到 K09 nearest 返回的第一个不在保护期的玩家的主星，重新计时
  - 世界 BOSS 血量 = hp_per_active_player × 上周活跃人数（K09 新增 active_users(日期范围)：这段时间有点数记录的玩家数）
  - 共享血量用数据库的原子更新（hp_left = GREATEST(hp_left − 伤害, 0) 并返回新值），不在内存里加减
  - 伤害复用 P02 的 CombatControl::damage；带弱点标签的攻击 counter_bp 为 20000；每人每天 daily_attacks（3）次
  - 世界 BOSS 在场期间调用 P02 的 set_truce(true)，结束时 set_truce(false)
  - 失败（到期血量仍大于 0）：调用 K07 的 set_global_modifier（转换速度 −1000 bp，持续 7 天）
【子任务】
P03-1 野生灾厄（阶段 4）
  - 测试：calamity_spawned_on_threshold_with_appearance_generated_once、escapes_after_72h_to_nearest_unprotected、absorbed_when_world_boss_active、wild_boss_damage_settled_by_code
P03-2 世界 BOSS（阶段 4）
  - 测试：shared_hp_consistent_under_100_concurrent_attacks（100 个并发攻击后 hp_left = hp_total − 伤害之和，不小于 0）、weakness_tag_doubles_damage、daily_attack_limit_3、truce_set_via_combat_control、hp_scaled_by_last_week_active_players
P03-3 奖励与惩罚（阶段 4）
  - 测试：participation_reward_after_one_attack、ranking_rewards_by_damage（按 boss.json 的奖励表）、rewards_granted_once、failure_sets_minus_10pct_modifier_for_one_week
【验收（每个子任务）】tools/check_server.sh gf-pack-boss；tools/check_tests.sh gf-pack-boss <本子任务点名的测试>
【停止条件】需要每次战斗都调用 AI；需要让 P01 或 P02 订阅本模块的事件
~~~
