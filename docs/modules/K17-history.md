# K17 版本历史（history）

> **层级**：内核 L4 · **代码**：`server/kernel/history` · **首次阶段**：1（记录、撤销）；5（时光机、试玩副本）· **主文档**：第 16.6、19.5 节

## 1. 职责

- **事件记录**：每次能力调用造成的改动都记下来
- **快照**：定期保存星球状态
- **反向操作**：为可撤销的改动生成反向操作（由 K04 的"撤销"能力执行）
- **时光机**：把星球的创作内容恢复到某一天
- **试玩副本**：从快照复制出一颗临时星球
- 历史的保留与清理

## 2. 不负责

- 执行撤销（交给 K04，走正常流水线）
- 回退官方结果（不允许）

## 3. 依赖

K00、K02

## 4. 对外接口

```rust
pub trait History: Send + Sync {
    fn record(&self, change: ChangeRecord) -> Result<ChangeId>;                       // 由 K04 调用
    fn inverse_ops(&self, change: ChangeId) -> Result<Vec<Op>>;                       // 只对可撤销的改动
    fn snapshot(&self, planet: PlanetId) -> Result<SnapshotId>;
    fn restore_plan(&self, planet: PlanetId, at: Timestamp) -> Result<Vec<Op>>;       // 时光机：生成恢复操作
    fn fork_trial(&self, planet: PlanetId) -> Result<TrialPlanetId>;                  // 试玩副本
    fn timeline(&self, planet: PlanetId, range: TimeRange) -> Result<Vec<ChangeSummary>>;
}
```

HTTP：查询星球的改动时间线（供 C10 的时光机界面使用）

## 5. 数据

| 表 | 关键字段 |
|---|---|
| changes | id, planet_id, caller, capability, ops（JSON）, undoable, at |
| snapshots | id, planet_id, storage_ref（对象存储）, at |
| trial_planets | id, source_planet_id, snapshot_id, expires_at |

## 6. 事件

- 发布：`SnapshotCreated`、`TrialCreated`、`TrialExpired`

## 7. 配置

`history.json`：快照频率、历史保留天数、试玩副本有效期

## 8. 关键规则

- **撤销只回退创作内容**：只有 `undoable = true` 的改动才能生成反向操作
- **撤销不退资源**：撤销一次放置，物体消失，但花掉的金和火不退；撤销本身也不收费
- **时光机同样只恢复创作类组件**：稳定度、废墟状态、资源、战斗记录都不受影响
- 撤销时如果目标已经被别人改过，提示冲突，由玩家选择
- 试玩副本不能被攻击、不计入任何排名、到期自动删除
- 快照存在对象存储，数据库只存引用

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 记录与撤销 | 撤销最近的放置、移动、删除 |
| 2 | 冲突处理 | 两人先后修改同一物体后撤销，有明确提示 |
| 5 | 快照、时光机 | 恢复到昨天，官方组件保持不变 |
| 5 | 试玩副本 | 副本里的改动不影响正式星球 |

## 10. 测试要点

- 战斗造成的废墟不能被撤销或时光机恢复
- 大量历史下的查询和恢复速度

## 11. 待定问题

- 历史保留多久（影响存储成本）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K17-x"（x 换成子任务编号）。

~~~text
【模块】K17 版本历史（包 gf-history，路径 server/kernel/history，层级 L4）
【先读】docs/modules/K17-history.md；K02-world.md 第 4、5 节；K04-capability.md 第 8 节（撤销、恢复）；主文档第 16.6、19.5 节
【范围】server/kernel/history/、server/migrations/（文件名带 k17）、shared/config/history.json、fixtures/history/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-world。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、schemars。测试：proptest
【规则补充】
  - 创作类组件：Transform、Appearance、Footprint、Creator、FunctionTags、Tags、RegionRef、TerrainEdit；其他组件（包括所有玩法包的组件）都不是创作类
  - ChangeRecord 由 K04 在事务里写入（record 的第一个参数是 tx）：调用者、能力名、undoable，以及每个受影响实体的 before / after（只记创作类组件）
  - 反向操作（Op = 能力名 + 输入）：after 为空（被删了）→ 恢复（用 before 重建，编号不变）；before 为空（新建的）→ 删除；都不为空 → 恢复（把创作类组件写回 before）
  - 冲突：实体当前的创作类组件和这次改动的 after 不同，说明别人改过 → 返回 Conflict，并列出冲突的实体
  - 测试里可以写一个只在测试中存在的小函数，把 Op 直接应用到 K02 来验证反向操作；正式代码一律交给 K04 执行
  - 快照存进 ObjectStore（key：snapshots/<星球>/<快照编号>.json），数据库只存引用
  - history.json：snapshot_interval_hours 24、keep_days 30、trial_ttl_hours 24
【子任务】
K17-1 记录与反向操作（阶段 1）
  - 测试：inverse_of_place_move_remove、non_undoable_change_has_no_inverse、apply_then_inverse_restores_state（proptest：随机改动序列，逐个反向执行后创作类组件和开始时完全相同）、official_components_not_recorded
  【验收】另加 tools/mutants.sh gf-history src/inverse.rs
K17-2 冲突处理（阶段 2）
  - 测试：undo_conflict_when_modified_by_someone_else、undo_without_conflict_succeeds
K17-3 快照与时光机（阶段 5）
  - restore_plan(planet, at)：找 at 之前最近的快照，重放到 at 得到目标状态，和当前的创作类组件比较，生成恢复操作（由 K04 执行）
  - 测试：restore_to_yesterday_keeps_official_components（废墟状态、稳定度这类组件不变）、restore_plan_goes_through_capabilities、timeline_marks_non_undoable
K17-4 试玩副本（阶段 5）
  - fork_trial：用 K02 创建 kind = trial 的星球，复制快照
  - 测试：trial_changes_do_not_leak、trial_planet_has_trial_kind、trial_expires（不能被攻击、不能设为主星分别由 P02、K09 检查）
【验收（每个子任务）】tools/check_server.sh gf-history；tools/check_tests.sh gf-history <本子任务点名的测试>
【停止条件】需要撤销或恢复官方结果（战斗、资源、废墟）；需要直接执行反向操作（必须交给 K04）
~~~
