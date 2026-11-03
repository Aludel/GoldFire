# P01 供给与环境（supply）

> **层级**：星球层 L8 · **所属玩法包**：星球攻防（官方，默认安装，不可卸载）· **代码**：`server/packs/war/supply` · **首次阶段**：2（转换速度）；3（重要度）；4（环境与灾厄）· **主文档**：第 8.3、10.2、10.4、11.1 节

## 1. 职责

- **供给网络**：按功能标签和距离，自动建立建筑之间的依赖关系（发电站给 300 米内的建筑供电……）
- **运转状态**：正常 / 停工（缺少供给）/ 废墟（被攻击打坏，由 P02 调用 `mark_ruined` 设置）
- **重要度**：按供给网络计算每个造物的重要度
- **转换速度**：实现 K07 的 `ConversionRateProvider`，按玩家名下所有星球的产出建筑计算
- **环境平衡**：火、水、生命、工业四项平衡值，由建筑标签推动；累积灾厄值
- 越过阈值时发出事件，交给 P03 生成野生灾厄

## 2. 不负责

- 战斗结算 → P02
- 生成 BOSS → P03
- 资源记账 → K07

## 3. 依赖

K00、K02、K03（注册系统）、K04、K07

## 4. 对外接口

```rust
impl ConversionRateProvider for Supply { fn rate(&self, user: UserId) -> ConversionRate { /* … */ } }

pub trait SupplyQuery: Send + Sync {                 // 给 P02、P03 使用
    fn importance(&self, e: EntityId) -> u32;                                          // 基点，全星球之和为 10000
    fn importance_in_radius(&self, p: PlanetId, center: GeoPos, r_m: u32) -> Vec<(EntityId, u32)>;
    fn environment(&self, p: PlanetId) -> Environment;
    fn mark_ruined(&self, entities: &[EntityId]) -> Result<()>;    // P02 结算后调用，随后重算供给网络
    fn mark_rebuilt(&self, entities: &[EntityId]) -> Result<()>;
}
pub trait ScoutOracle: Send + Sync {                // 由 P02 实现、启动时注入：Importance 组件的可见性规则用它
    fn has_scouted(&self, viewer: UserId, planet: PlanetId, pos: GeoPos) -> bool;
}
```

**登记的系统**：

| 系统 | 触发 | 没人看时 |
|---|---|---|
| 供给网络重算 | 造物增删改、运转状态变化 | 不需要（事件触发） |
| 环境累积 | 每小时 | 补算 |

## 5. 数据（本玩法包拥有的组件）

| 组件 | 内容 | 可见性 |
|---|---|---|
| Supply | 供给来源、服务半径 | 公开 |
| Operational | 运转状态：正常 / 停工 / 废墟 | 公开 |
| Importance | 重要度 | 主人和侦察过的人可见 |
| Production | 产出类型（金 / 火）、转换速度 | 主人可见 |
| Environment（星球级） | 火、水、生命、工业、灾厄值 | 公开 |

## 6. 事件

- 发布：`SupplyRecomputed`、`EnvironmentThresholdReached`
- 订阅：K02 的 `EntitySpawned` / `ComponentChanged` / `EntityDespawned`
- **不订阅 P02 和 P03 的事件**（它们依赖 P01，反过来订阅会形成循环）；P02 通过 `mark_ruined` 直接调用 P01

## 7. 配置

- `buildings.json`：每种建筑的服务半径、供给需求、转换速度
- `supply.json`：功能标签之间的供给规则
- `environment.json`：标签对平衡值的影响、灾厄阈值

## 8. 关键规则

- **功能决定产出，产出决定重要度**：只看功能标签，不看名字和描述
- **重要度 = 产出占比 + 被依赖程度**（简化的图中心性），所有造物重要度之和归一化
- 交通类造物（道路、桥）扩大供给范围
- 废墟状态的建筑不供给、不产出
- 转换速度 = 玩家名下所有星球上正常运转的产出建筑之和；一座都没有时用 K07 的默认值
- 转换速度要变化之前，先调用 K07 的 `wallet(user)` 把旧速度补算完
- 重要度用整数基点计算，全星球之和正好是 10000（算法见开发提示词）

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 2 | 产出建筑与转换速度 | 小明的例子：3 座金矿 + 2 座熔炉 = 每小时 5000 点 |
| 3 | 供给网络与重要度 | 主发电站的重要度明显高于民居 |
| 4 | 环境平衡与阈值事件 | 堆满火山和熔炉的星球触发灾厄 |

## 10. 测试要点

- 改名不改变重要度
- 打掉供能节点后，依赖它的建筑停工

## 11. 待定问题

- 供给规则的完整列表（随标签体系一起定）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 P01-x"（x 换成子任务编号）。

~~~text
【模块】P01 供给与环境（包 gf-pack-supply，路径 server/packs/war/supply，层级 L8，属于官方玩法包 war）
【先读】docs/modules/P01-supply.md；K02、K03、K04、K07 文档的第 4 节；主文档第 8.3、10.2、10.4、11.1 节
【范围】server/packs/war/supply/、shared/config/buildings.json、shared/config/supply.json、shared/config/environment.json、shared/config/tags.json（只加产出和供给相关的标签）、fixtures/supply/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-world、gf-scheduler、gf-capability、gf-economy。库：serde、serde_json、tokio、async-trait、tracing、schemars。测试：proptest
【规则补充】
  - 组件名带包名前缀：war.Supply、war.Operational、war.Importance、war.Production、war.Environment
  - 只看功能标签，不看名字和描述。产出建筑：带"产金"或"产火"标签的造物，速度来自 buildings.json
  - 转换速度 = 玩家名下所有星球上 Operational 为正常的产出建筑速度之和；一座都没有时返回 0（K07 会改用默认速度）
  - 转换速度要变化之前（产出建筑新增、删除、停工、恢复），先调用 K07 的 wallet(user) 把旧速度补算完
  - 供给：供给方的服务半径内、并且 supply.json 规定它能供给的标签的造物，依赖这个供给方；交通类造物（道路、桥）彼此相距 ≤ transport_link_m 就算连通，供给可以沿连通的交通造物传到离它们 ≤ transport_service_m 的造物
  - 缺少任何一种必需供给 → 停工；废墟（由 P02 调用 mark_ruined）不供给、不产出
  - 重要度（整数 bp，全星球之和正好是 10000）：
      out(e) = e 的产出速度 / 全星球产出速度之和（没有产出时为 0）
      dep(e) = 直接和间接依赖 e 的造物数 / (造物总数 − 1)（只有一个造物时为 0）
      raw(e) = out(e) × w_out_bp + dep(e) × w_dep_bp（默认各 5000，用 u128 的分数计算）
      归一化：每个造物先取 floor(raw / Σraw × 10000)，剩下的余数按 raw 从大到小每个加 1（raw 相同按 EntityId）；Σraw 为 0 时平均分配
  - 文档第 4 节的 importance 返回值改成 u32（bp）；新增 trait ScoutOracle（由 P02 实现、启动时注入），Importance 组件的可见性规则用它判断"侦察过"
  - 环境：每小时依次 ① 每个正常运转的造物按标签给火、水、生命、工业四项加上 environment.json 的增量（结果限制在 0–10000）② 失衡 = 四项最大值 − 最小值 ③ 失衡 > imbalance_threshold 时，灾厄值 += 失衡 × calamity_rate_bp / 10000（向下取整）④ 灾厄值 ≥ calamity_threshold 时发出 EnvironmentThresholdReached，灾厄值清零。补算按整小时逐小时计算
【子任务】
P01-1 产出建筑与转换速度（阶段 2）
  - 测试：xiaoming_rate_example（3 座金矿 + 2 座熔炉，各每小时 1000 → 金 3000、火 2000）、no_buildings_returns_zero_rate、rate_sums_across_all_owned_planets、stopped_buildings_excluded、rate_change_settles_conversion_first
P01-2 供给网络与重要度（阶段 3）
  - 测试：power_plant_more_important_than_houses、importance_sums_to_10000_bp（proptest）、renaming_does_not_change_importance、road_extends_supply_radius、mark_ruined_stops_dependents、ruined_buildings_excluded_from_rate、importance_hidden_without_scout
  【验收】另加 tools/mutants.sh gf-pack-supply src/importance.rs
P01-3 环境平衡与阈值事件（阶段 4）
  - 测试用配置 fixtures/supply/environment_test.json：火山 火 +100/小时；熔炉 火 +50、工业 +50/小时；imbalance_threshold 1000、calamity_rate_bp 1000、calamity_threshold 10000
  - 测试：volcano_furnace_planet_reaches_threshold（2 座火山 + 2 座熔炉：第 25 小时灾厄值 9570，第 26 小时发出事件）、threshold_event_emitted_once、environment_catch_up_equals_continuous（离线 26 小时补算和连续运行结果相同）
【验收（每个子任务）】tools/check_server.sh gf-pack-supply；tools/check_tests.sh gf-pack-supply <本子任务点名的测试>
【停止条件】需要订阅 P02 或 P03 的事件（会形成循环）；需要按名字或描述判断建筑类型
~~~
