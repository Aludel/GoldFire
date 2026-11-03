# K04 能力接口（capability）

> **层级**：内核 L5 · **代码**：`server/kernel/capability` · **首次阶段**：1 · **主文档**：第 16.7、19.3 节

## 1. 职责

- **能力登记**：名称、输入 Schema、权限级别、费用规则、检查规则、处理器、能不能撤销
- **能力目录**：给 AI 和 MCP 看的说明书（只列出调用者有权使用的能力）
- **统一流水线**：所有改变世界的操作都在这里执行
- **预览**：只做检查、估算费用、返回"将要发生的改动"，不真正执行
- **撤销**：从 K17 取反向操作，再走一遍流水线执行
- **MCP 服务器**：把能力目录和调用以 MCP 的形式开放给外部 agent
- **速率限制**

## 2. 不负责

- 每个能力具体怎么执行（由各模块注册的处理器负责）
- AI 理解与规划（K16）

## 3. 依赖

K00、K01（令牌）、K02（权限、写入）、K07（计费）、K13（内容安全）、K17（记录）

## 4. 对外接口

```rust
pub struct CapabilityDef {
    pub name: &'static str,
    pub input_schema: Schema,
    pub level: PermissionLevel,          // 外观级 / 星球级 / 官方级
    pub cost: CostRule,                  // 从配置表计算
    pub undoable: bool,                  // 创作类 = true；官方结果类 = false
    pub doc: CapabilityDoc,              // 给 AI 看的说明和示例
    pub handler: Arc<dyn CapabilityHandler>,
}

pub trait CapabilityHandler: Send + Sync {
    fn validate(&self, ctx: &CallCtx, input: &Value) -> Result<Plan>;     // 规则检查，生成执行计划
    fn execute(&self, ctx: &mut ExecCtx, plan: Plan) -> Result<Outcome>;  // 在事务里执行
}

pub enum Caller { Player(UserId), PlanetMind(PlanetId), Agent(EntityId), Pack(PackId), External(TokenGrant) }

pub fn register(def: CapabilityDef) -> Result<()>;
pub async fn call(ctx: CallCtx, name: &str, input: Value, mode: Mode /* Preview | Execute */) -> Result<CallResult>;
pub fn catalog(caller: &Caller, planet: PlanetId) -> Vec<CapabilityDoc>;
```

- HTTP：`POST /capabilities/{name}`（预览或执行）；`GET /capabilities`（目录）
- MCP：同一套能力，以 MCP 工具的形式提供

## 5. 数据

| 表 | 关键字段 |
|---|---|
| capability_calls | id, client_request_id, caller, planet_id, name, input_hash, mode, result, cost, at |

## 6. 事件

- 发布：`CapabilityExecuted`（调用者、星球、能力名、改动摘要）——智能体运行时用它让人物"感知"世界变化

## 7. 配置

`capabilities.json`：每种调用者的速率限制；能力费用引用 `buildings.json` 等数值表

## 8. 关键规则

**流水线（顺序固定）**：

```
1 身份与令牌 → 2 权限（成员角色 × 能力级别 × 调用者类型）→ 3 Schema 校验
→ 4 处理器 validate（数值、边界）→ 5 内容安全（K13，检查输入里的文字和图片）
→ 6 计费（K07；预览时只估算）→ 7 执行（同一个事务）→ 8 记入 K17
→ 9 发出 CapabilityExecuted → 10 返回结果
```

- 任何一步失败，整个调用回滚，不留下半截改动
- 预览和执行使用同一个 `validate`，保证"预览看到的就是执行得到的"
- 调用者是玩法包或星球 AI 时，权限不能超过它被批准的范围
- 外部 agent 只能调用令牌权限范围内的能力
- `undoable = false` 的能力（战斗结果、资源变化）不能被撤销

- 同一个 `client_request_id` 重复提交时，返回第一次的结果，不重复执行

**内置能力**：

| 阶段 | 能力 | 级别 |
|---|---|---|
| 1 | 放置、移动（含旋转、缩放）、删除、改地形、播放特效、划定区域 | 外观级 |
| 1 | 创建星球（不针对已有星球，调用者成为主人） | — |
| 1 | 撤销（输入改动编号，执行 K17 给出的反向操作） | 和原改动相同 |
| 2 | 刻字 | 外观级 |
| 2 | 设置成员、删除刻字 | 星球级 |
| 5 | 生成实体（给规则和脚本用：素材、数量、随机位置） | 外观级 |
| 内部 | 恢复（只供撤销和时光机使用，不出现在目录里） | 和原改动相同 |

- 玩法包的能力名是 `<PackId>.<能力名>`，例如 `war.攻击`
- 区域里的放置、移动、删除、改地形只能由区域主人或星球主人操作

**权限矩阵**（第 2 步）：

| 级别 | 玩家 | 星球 AI | 玩法包 | 人物 | 外部令牌 |
|---|---|---|---|---|---|
| 外观级 | 建造者及以上 | 按发起人的角色 | 安装时批准了外观级 | 可以 | 令牌范围内，且令牌主人是建造者及以上 |
| 星球级 | 主人 | 主人发起或自主运转时可以；访客发起时按访客策略 | 安装时批准了星球级 | 不可以 | 令牌范围内，且令牌主人是主人 |
| 官方级 | 可以（由能力自己的规则检查） | 只有主人允许的（攻击要 allow_attack） | 只有官方玩法包 | 不可以 | 不可以 |

- 第 1 阶段不收费：`server/app` 注入一个估算为 0、扣费直接成功的经济实现，第 2 阶段换成 K07

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 能力登记与流水线 | 第 1 阶段的内置能力可用（见第 8 节） |
| 1 | 预览与撤销 | 预览结果和执行结果一致；能撤销最近的操作 |
| 2 | 权限与计费 | 权限矩阵测试通过；扣费写入流水 |
| 5 | 玩法包注册的能力 | 新玩法安装后，它的能力出现在目录里 |
| 5 | MCP 服务器、速率限制 | 外部 agent 能连接并调用；超速被限流 |

## 10. 测试要点

- 在第 1–9 步的每一步人为制造失败，确认都没有留下改动
- 权限矩阵：调用者类型 × 成员角色 × 能力级别
- 撤销：创作类能撤销，官方结果类不能

## 11. 待定问题

- 批量操作（例如一次放置 200 个物体）是一次调用还是多次（建议：一次调用，整体成功或整体失败）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K04-x"（x 换成子任务编号）。

~~~text
【模块】K04 能力接口（包 gf-capability，路径 server/kernel/capability，层级 L5）
【先读】docs/modules/K04-capability.md；K01、K02、K07、K13、K17 文档的第 4 节；主文档第 16.5、16.7、19.3 节
【范围】server/kernel/capability/、server/migrations/（文件名带 k04）、shared/config/capabilities.json、fixtures/capability/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-account、gf-world、gf-economy、gf-moderation、gf-history。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、jsonschema、schemars；MCP（K04-5）另加 rmcp。测试：proptest
【规则补充】
  - 内置能力的名字（输入的 Schema 写在代码里并生成到 shared/schemas/）：
      阶段 1：放置、移动（含旋转和缩放）、删除、改地形、播放特效、创建星球、划定区域、撤销
      阶段 2：设置成员、刻字、删除刻字
      阶段 5：生成实体（给规则和脚本用：素材、数量、位置"随机"或坐标、范围）
      内部：恢复（只供撤销和时光机使用，不出现在目录里）
    玩法包的能力名是"<PackId>.<能力名>"，例如 war.攻击
  - 内置能力的级别：放置、移动、删除、改地形、播放特效、划定区域、刻字 = 外观级；设置成员、删除刻字 = 星球级；创建星球不针对已有星球，任何玩家都可以调用，调用者成为主人；撤销和恢复 = 原改动的级别
  - 区域：有建造权限的人给自己划区域（区域主人 = 调用者）；放置、移动、删除、改地形只能由目标所在区域的主人或星球主人操作
  - 放置的输入带 region_id；实体的占地轮廓必须完全在这个区域里（K02 的 footprint_inside）
  - 改地形 = 生成一个带 TerrainEdit 组件的实体；播放特效只广播给在场的人，不写入世界
  - 同一个 client_request_id 重复提交时，返回第一次的结果，不重复执行
  - 撤销：输入 change_id；只有这次改动的发起人或星球主人可以撤销；从 K17 取反向操作，在同一次调用、同一个事务里执行
  - 权限矩阵（第 2 步）：
      外观级：玩家要有建造者及以上角色；星球 AI 按发起人的角色；玩法包要在安装时批准了外观级；人物可以；外部令牌要在令牌范围内且令牌主人是建造者及以上
      星球级：玩家要是主人；星球 AI 只在主人发起或自主运转时可以，访客发起时按访客策略；玩法包要在安装时批准了星球级；人物不可以；外部令牌要在令牌范围内且令牌主人是主人
      官方级：玩家可以（由能力自己的 validate 检查）；星球 AI 只能调用主人允许的（攻击要 allow_attack）；只有官方玩法包可以；人物和外部令牌不可以
  - 第 1 阶段不收费：gf-app 注入 FreeEconomy（估算费用为 0，扣费直接成功），第 2 阶段换成 K07
  - capabilities.json：每种调用者每分钟的调用上限；各内置能力的费用（放置按 buildings.json 的类别价格）
【子任务】
K04-1 能力登记与流水线（阶段 1）
  - 第 8 节的 10 步按固定顺序执行；第 6–9 步在同一个事务里；预览只走第 1–6 步（第 6 步只估算）并返回"将要发生的改动"
  - 测试：pipeline_steps_run_in_fixed_order、failure_at_any_step_leaves_no_changes（在第 1 到第 9 步分别制造失败，每次都确认：没有新实体、没有流水、没有历史记录、outbox 里没有新事件）、preview_writes_nothing、input_validated_against_schema、catalog_lists_only_permitted_capabilities、duplicate_client_request_id_not_executed_twice
  【验收】另加 tools/mutants.sh gf-capability src/pipeline.rs
K04-2 第 1 阶段内置能力（阶段 1）
  - 测试：create_planet_registers_owner、claim_region_rejects_overlap、place_inside_region_ok、place_outside_region_rejected、place_in_others_region_rejected、move_rotate_scale_updates_transform、remove_deletes_entity、terraform_spawns_terrain_edit_entity、play_effect_broadcast_not_persisted
K04-3 预览一致与撤销（阶段 1）
  - 测试：preview_equals_execution（proptest：随机的放置、移动、删除序列，预览列出的改动和执行后的改动完全相同）、undo_last_place、undo_last_move、undo_last_remove、undo_rejected_for_non_undoable、undo_by_other_builder_rejected
K04-4 权限、计费与第 2 阶段能力（阶段 2）
  - 测试：permission_matrix（把上面的矩阵逐格写成表驱动测试）、insufficient_gold_rejected_without_changes、spend_recorded_with_planet_id、inscription_goes_through_moderation、set_member_requires_owner
K04-5 玩法包能力、MCP、限流（阶段 5）
  - 测试：pack_capability_appears_in_catalog_after_install、pack_capability_limited_to_approved_level、spawn_entities_uses_system_rng、mcp_tools_match_catalog、mcp_call_respects_token_scope、rate_limit_returns_quota_exceeded
【验收（每个子任务）】tools/check_server.sh gf-capability；tools/check_tests.sh gf-capability <本子任务点名的测试>
【停止条件】某个能力需要绕过流水线；一次调用需要跨两颗星球；权限矩阵里出现上面没写的情况
~~~
