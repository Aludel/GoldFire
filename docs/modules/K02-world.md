# K02 世界（world）

> **层级**：内核 L3 · **代码**：`server/kernel/world` · **首次阶段**：1 · **主文档**：第 4、5、6、9.3、19.1、19.2、21 节

## 1. 职责

- **星球**：创建、属性（种子、调色板、建造模式）
- **实体与组件**：存取；组件类型登记（名称、所属玩法包、Schema、可见性）
- **组件所有权检查**：只有所属玩法包能写
- **空间索引**：用 h3o 把实体按位置放进格子，快速查询"某点半径内有什么"
- **区域**：任意形状的边界；重叠和面积检查；为 AI 提供"区域局部地图"
- **星球成员与权限**：主人 / 建造者 / 访客；公开建造或邀请制
- **纪念塔**：刻字的存取
- **变化同步**：在场玩家的 WebSocket 频道；按可见性过滤后广播变化
- **几何工具**：判断造物占地轮廓是否在区域内

## 2. 不负责

- 规则和数值 → 玩法包、K04
- 星球实例的加载、心跳、补算 → K03
- 历史与撤销 → K17
- AI → K10、K11、K16

## 3. 依赖

K00（HTTP 里的当前用户用 K00 的 `CurrentUser`）、K12（素材的占地轮廓）

## 4. 对外接口

```rust
pub trait WorldRead: Send + Sync {
    fn planet(&self, id: PlanetId) -> Result<Planet>;
    fn get<C: Component>(&self, e: EntityId) -> Option<C>;
    fn query_radius(&self, p: PlanetId, center: GeoPos, radius_m: f32) -> Vec<EntityId>;
    fn region(&self, id: RegionId) -> Result<Region>;
    fn region_local_map(&self, id: RegionId) -> Result<LocalMap>;   // 给 AI：区域内的格子、地形、已有造物
    fn role(&self, p: PlanetId, u: UserId) -> Role;
    fn visible_view(&self, p: PlanetId, viewer: Viewer, area: Area) -> Result<View>;  // 按可见性过滤
    fn footprint_inside(&self, region: RegionId, outline: &[GeoPos]) -> Result<bool>; // 占地轮廓是否完全在区域内
}

pub trait WorldWrite: Send + Sync {
    // WriteCtx 里带着"谁在写"（内核 / 某个玩法包），用来检查组件所有权
    fn spawn(&self, ctx: &WriteCtx, p: PlanetId, comps: ComponentSet) -> Result<EntityId>;
    fn set<C: Component>(&self, ctx: &WriteCtx, e: EntityId, c: C) -> Result<()>;
    fn despawn(&self, ctx: &WriteCtx, e: EntityId) -> Result<()>;
    // 下面四个只接受内核身份，由 K04 的内置能力调用
    fn create_planet(&self, ctx: &WriteCtx, owner: UserId, name: &str, kind: PlanetKind) -> Result<PlanetId>;  // 同时生成纪念塔
    fn claim_region(&self, ctx: &WriteCtx, p: PlanetId, owner: UserId, boundary: &[GeoPos]) -> Result<RegionId>;
    fn set_role(&self, ctx: &WriteCtx, p: PlanetId, u: UserId, role: Role) -> Result<()>;
    fn set_inscription(&self, ctx: &WriteCtx, p: PlanetId, u: UserId, text: Option<&str>) -> Result<()>;
}

pub struct ComponentDef {
    pub name: &'static str,
    pub owner: Owner,              // Kernel 或 Pack(PackId)
    pub schema: Schema,
    pub visibility: Visibility,    // Public / OwnerOnly / AfterScout / Custom(rule)
}
pub fn register_component(def: ComponentDef) -> Result<()>;
```

- `WorldWrite` 只给两类调用者用：K04 执行能力时、K03 运行系统时（系统只能写本包组件）
- 写方法的第一个参数是 `tx: &mut Tx`（和 K04 的流水线在同一个事务里，见 README 全局约定）
- HTTP 只读：`GET /planets/{id}`；`GET /planets/{id}/view`（某个范围内可见的实体和组件）；`GET /planets/{id}/changes?since=<序号>`（返回 changes、last_seq、reload）
- 创建星球、划定区域、设置成员、刻字 / 删除刻字都是 K04 的内置能力，经过能力流水线后调用上面的 `WorldWrite`
- WebSocket：星球频道——进入、离开、接收变化。**修改操作不走这里，一律走 K04**

## 5. 数据

| 表 | 关键字段 |
|---|---|
| planets | id, owner_id, name, seed, palette_id, build_mode, kind（normal / trial）, last_seq, created_at, archived |
| entities | id, planet_id, created_at |
| components | entity_id, type, data（JSONB）, owner, updated_at |
| component_types | name, owner, schema, visibility |
| regions | id, planet_id, owner_id, boundary（球面多边形）, area, name |
| entity_cells | cell_id, entity_id（空间索引，也可以只放内存） |
| planet_members | planet_id, user_id, role |
| inscriptions | planet_id, user_id, text, updated_at |
| planet_changes | planet_id, seq, change（JSONB）, at（变化记录，给客户端按序号补拉） |
| blueprints | user_id, recipe, created_at（蓝图库） |

**内核自带的组件**：

| 组件 | 内容 |
|---|---|
| Transform | 位置（经度、纬度、高度）、朝向、缩放 |
| Appearance | 造物配方（素材指纹、颜色色号、特效） |
| Footprint | 占地轮廓 |
| Creator | 创作者、创作时间 |
| FunctionTags | 功能标签（能源、粮食、居住、防御、武器、交通、信仰……） |
| Tags | 外观标签 |
| RegionRef | 所在区域 |
| TerrainEdit | 一次地形改动（类型、路径、参数，以及可选的侵蚀补丁）；地形改动就是带这个组件的实体，所以撤销和同步不需要特殊处理 |
| Monument | 纪念塔标记（星球创建时生成，不能删除） |

## 6. 事件

- 发布：`PlanetCreated`、`EntitySpawned`、`ComponentChanged`、`EntityDespawned`、`RegionClaimed`、`MemberChanged`、`InscriptionChanged`、`PlayerJoined`、`PlayerLeft`

## 7. 配置

`planet.json`：空间索引和区域的格子精度、区域面积上限、刻字长度、每层纪念塔的人数、变化记录保留条数、纪念塔默认位置、星球数量是否有软上限

## 8. 关键规则

- **组件所有权**：写入时检查 `WriteCtx.owner` 是否等于组件的 `owner`，不等就拒绝
- **可见性**：同步给客户端、返回给 HTTP 查询时，都按组件的可见性规则过滤；"需要侦察"类组件要问所属玩法包（通过 `Visibility::Custom` 注册的规则）
- **区域**：不能和已有的区域重叠（包括自己的）；面积不超过上限；区域之间可以相邻
- **空间查询**：先用格子粗筛，再用精确距离（球面距离）细筛；要处理极点附近和五边形格子
- 纪念塔不能被删除
- 造物配方删除时（主人主动删除）保留在蓝图库
- **变化序号**：每次写入在同一个事务里给这颗星球分配下一个序号（从 1 开始，连续不跳号）；事务回滚时序号也不消耗
- **区域的格子**：区域边界换成格子集合（格子中心在边界内的算），面积 = 格子面积之和；有共同格子才算重叠，只共边不算
- **区域主人**：划定区域的人；区域里的放置、移动、删除、改地形只能由区域主人或星球主人操作（由 K04 检查）

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 星球、实体、组件存取 | 重启后数据还在 |
| 1 | 组件类型登记与所有权检查 | 越权写入被拒绝 |
| 1 | 空间索引与半径查询 | 和暴力遍历的结果一致 |
| 1 | 区域与几何检查 | 能画任意形状区域；重叠、超面积被拒绝 |
| 1 | 变化记录与只读 HTTP | 客户端能按序号拉取变化；隐藏数据不外泄 |
| 2 | 成员与权限 | 主人 / 建造者 / 访客的权限矩阵测试通过 |
| 2 | WebSocket 频道与可见性过滤 | 两个客户端同时在线，看到一致的变化；隐藏数据不外泄 |
| 2 | 纪念塔 | 刻字、修改、主人删除 |

## 10. 测试要点

- 所有权：玩法包 A 写玩法包 B 的组件一定失败
- 可见性：没侦察的玩家拿不到"需侦察"组件，无论通过 HTTP 还是 WebSocket
- 空间：极点、经度 ±180° 交界、五边形格子附近的查询都正确

## 11. 待定问题

- 一个人拥有的星球数量是否需要软上限（防止滥用存储）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K02-x"（x 换成子任务编号）。

~~~text
【模块】K02 世界（包 gf-world，路径 server/kernel/world，层级 L3）
【先读】docs/modules/K02-world.md；K12-asset.md 第 4 节（占地轮廓）；K04-capability.md 第 8 节（写操作都走能力接口）；主文档第 4、5、6、9.3、19.1、19.2、21 节
【范围】server/kernel/world/、server/migrations/（文件名带 k02）、shared/config/planet.json、fixtures/world/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-asset（占地轮廓）。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、h3o（含 geo 功能）、geo、jsonschema、schemars。测试：proptest、tokio-tungstenite（K02-7）
【规则补充】
  - planet.json：index_resolution 10、region_resolution 12、region_area_max_m2 4000000、inscription_max_chars 50、change_log_keep 10000、monument_lat 0、monument_lon 0
  - 空间索引：实体按 Transform 的位置放进 index_resolution 的格子；query_radius 先取覆盖半径的格子圈，再按大圆距离精确过滤（距离 ≤ 半径算在内）；结果按 EntityId 排序
  - 区域：边界是首尾闭合、不自交的经纬度多边形；换成 region_resolution 的格子集合（格子中心在多边形内的算）；面积 = 这些格子的面积之和；和任何已有区域（包括自己的）有共同格子就拒绝；只共边不算重叠
  - 占地轮廓在区域内：轮廓按实体的位置、朝向、缩放变换后，所有和它相交的格子都必须属于这个区域（新增 WorldRead::footprint_inside(region, outline) -> bool）
  - 变化记录：每次写入在同一个事务里给这颗星球分配下一个序号（从 1 开始，连续不跳号），写进 planet_changes；只保留最近 change_log_keep 条
  - HTTP 只读：GET /planets/{id}、GET /planets/{id}/view（某个范围内可见的实体和组件）、GET /planets/{id}/changes?since=<序号>（返回 changes、last_seq、reload；since 比保留的最早一条还早时 reload = true、changes 为空）。创建星球、划区域、设置成员、刻字这些写操作都是 K04 的内置能力，由 K04 调用本模块的 WorldWrite
  - WorldWrite 新增：create_planet（kind 为 normal 或 trial；同时生成纪念塔实体）、claim_region、set_role、set_inscription；只接受内核身份的 WriteCtx
  - 内核组件在文档的表之外再加两个：TerrainEdit（一次地形改动：类型、路径、参数、可选的侵蚀补丁 erosion_patch，格式见 X01-1）、Monument（纪念塔标记）
  - planets 表加 kind、last_seq；新增表 planet_changes（planet_id、seq、change JSONB、at）、blueprints（user_id、recipe、created_at）
【子任务】
K02-1 星球、实体、组件存取（阶段 1）
  - 实现组件登记机制，启动时登记全部内核组件
  - 测试：spawn_get_roundtrip_survives_restart（用同一个数据库新建一个 World 实例再读）、despawn_removes_all_components、planet_seed_and_palette_persisted、trial_planet_kind_persisted、monument_created_with_planet
K02-2 组件登记与所有权（阶段 1）
  - 玩法包登记自己的组件；写入时检查 WriteCtx 的身份和组件的所有者，并按 Schema 校验数据
  - 测试：pack_cannot_write_other_pack_component、kernel_component_requires_kernel_ctx、unregistered_component_rejected、component_data_validated_against_schema
  【验收】另加 tools/mutants.sh gf-world src/ownership.rs
K02-3 空间索引与半径查询（阶段 1）
  - 测试：query_radius_matches_bruteforce（proptest：随机 500 个实体、随机圆心和半径，结果和逐个算距离完全相同）、query_near_poles（圆心纬度 89.9°）、query_across_antimeridian（圆心经度 179.95°）、query_near_pentagon_cell（圆心放在一个五边形格子的中心）
K02-4 区域与几何检查（阶段 1）
  - 测试：overlap_with_other_owner_rejected、overlap_with_own_region_rejected、adjacent_regions_allowed（两个共一条边的正方形）、area_over_limit_rejected、concave_region_supported（L 形：缺口里的点不属于区域）、self_intersecting_boundary_rejected、region_across_antimeridian、footprint_inside_and_outside（同一个轮廓放在区域中间通过，压在边界上不通过）
K02-5 变化记录与只读 HTTP（阶段 1）
  - 可见性：Public 所有人可见；OwnerOnly 只有星球主人可见；AfterScout 和 Custom 问登记组件时提供的规则
  - 测试：changes_have_consecutive_seq、changes_since_returns_only_newer、too_old_since_requests_reload、rolled_back_write_leaves_no_change、owner_only_component_hidden_from_others_http
K02-6 成员与权限（阶段 2）
  - role(p, u)：主人 → Owner；planet_members 里的建造者 → Builder；星球是公开建造时任何人 → Builder；其余 → Visitor
  - 测试：role_matrix（主人 / 建造者 / 陌生人 × 公开建造 / 邀请制，6 种组合）、set_role_persists_and_publishes_member_changed
K02-7 WebSocket 星球频道（阶段 2）
  - GET /planets/{id}/ws：进入时发布 PlayerJoined、断开时发布 PlayerLeft；推送的每条变化带序号，按每个观看者的可见性过滤
  - 测试：hidden_component_never_sent_ws、changes_delivered_in_seq_order、player_joined_and_left_events、two_viewers_receive_same_public_changes
K02-8 纪念塔与蓝图（阶段 2）
  - 刻字：只有在这颗星球上创作过（有 Creator 是自己的实体）的人可以刻；长度不超过 inscription_max_chars；Monument 实体不能被删除；删除带 Appearance 和 Creator 的实体时，把配方存进创作者的蓝图库
  - 测试：inscription_requires_prior_creation、inscription_length_limit、monument_cannot_be_despawned、deleted_creation_kept_in_blueprints
【验收（每个子任务）】tools/check_server.sh gf-world；tools/check_tests.sh gf-world <本子任务点名的测试>
【停止条件】需要 HTTP 写接口；需要读写别的模块的数据表；K12 的占地轮廓格式没定义
~~~
