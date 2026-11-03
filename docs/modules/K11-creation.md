# K11 创作管线（creation）

> **层级**：内核 L6 · **代码**：`server/kernel/creation` · **首次阶段**：1（组装、地形）；2（人物、线性工程）；4（铸造）· **主文档**：第 9、28 节

## 1. 职责

- 把一段描述变成**造物方案**（配方 + 大致位置），本身不改变世界
- **五层生成**的路由：素材组装、参数化人物、程序化地形、线性工程、AI 铸造
- **功能标签分析**：创作时分析一次，结果固定
- **先检索，后挑选**：从 K12 检索约 20 个候选素材
- **先规划，后施工**：大请求先出规划清单，再逐项生成
- **布局算法**：AI 只给布局规则，由算法摆放
- **区域边界校验**：方案里的每一项都要在区域内
- **AI 铸造**（第 4 阶段）：概念图 → 3D 模型 → 自动处理 → 质检 → 入库
- **截图评审**（可选，只用于大型创作）
- 给 P03、P04 生成外观（灾厄 BOSS、剧本人物）

## 2. 不负责

- 真正放置到世界里 → 调用方拿到方案后通过 K04 执行
- 决定数值 → 放置时由能力处理器按配置表计算
- 理解玩家意图属于哪一类 → K16

## 3. 依赖

K00、K02（区域局部地图）、K04（注册"生成"相关能力）、K10、K12、K13；使用共享地形库（X01）检查地形可放置性

## 4. 对外接口

```rust
pub struct CreationRequest {
    pub planet: PlanetId,
    pub region: RegionId,
    pub prompt: String,
    pub style: StyleRef,             // 调色板、主题包
    pub initiator: Initiator,
    pub budget: Budget,
}
pub struct CreationPlan {
    pub items: Vec<PlannedItem>,     // 每项：层、配方、功能标签、位置、朝向、缩放
    pub rejected: Vec<Rejection>,    // 越界、不合格的条目及原因
    pub estimated_cost: Cost,
}
#[async_trait]
pub trait Creation: Send + Sync {
    async fn plan(&self, req: CreationRequest) -> Result<CreationPlan>;
    async fn appearance_for(&self, desc: &str, kind: AppearanceKind) -> Result<Recipe>;   // 给 BOSS、人物
    async fn forge(&self, req: ForgeRequest) -> Result<ForgeJobId>;                       // 第 4 阶段
}
// 纯函数：检查 AI 回答（Schema、越界、候选素材），X02 评测也用它
pub fn validate_generation_output(output: &Value, map: &LocalMap, candidates: &[AssetId]) -> Vec<Problem>;
```

## 5. 数据

| 表 | 关键字段 |
|---|---|
| generation_jobs | id, initiator, planet_id, region_id, prompt_hash, layer, status, model, attempts, result |
| forge_jobs | id, generation_job_id, stage（概念图 / 建模 / 处理 / 质检）, result_asset_id |

## 6. 事件

- 发布：`ForgeCompleted`

## 7. 配置

`creation.json`：候选数量、重试次数、规划阈值（多大的请求需要先规划）、布局规则库、侵蚀模拟参数（迭代次数、补丁网格的上限）

## 8. 关键规则

**一次组装层生成的步骤**：

```
1 K13 检查描述 → 2 K12 检索候选素材 → 3 K10 便宜档：在候选里挑选、填参数（Schema 约束）
→ 4 校验：Schema + 区域边界 + 地形可放置 → 5 失败的条目把原因发回 AI 重试 → 6 仍失败就用默认模板
```

- AI 输出里**不能有数值字段**
- 位置用"区域内的格子编号 + 相对描述"，由代码换算成坐标
- 颜色只能选调色板色号（0–15）；数量只能选 少量 / 若干 / 大量；地形参数只能选档位（例如 锋利度 平缓 / 中等 / 锋利）；对应的具体数值在 `creation.json`；功能标签只能从固定列表选
- **布局**：AI 输出规则（"主街沿河、广场居中、密度中等"），算法负责具体摆放
- 地形可放置性用共享地形库计算，保证和客户端看到的一致

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 组装层最小版 | 「樱花环绕的茶馆」能生成合理的方案；越界条目被退回重试 |
| 1 | 程序化地形入口 | 一句话生成一座山或一条河 |
| 1 | 功能标签 | 发电站被标为"能源"，名字不影响结果 |
| 2 | 参数化人物、线性工程 | 「红发女铁匠」「沿赤道的城墙」能生成 |
| 2 | 先规划后施工、布局算法、主题包 | 「魔法学院」能分解成清单并逐项生成 |
| 4 | AI 铸造、截图评审 | 铸造作品通过质检后入库 |

## 10. 测试要点

- X02 的 100 条造物提示词，合格率达标
- 恶意描述（试图让 AI 输出数值、越界坐标、不当内容）全部被拦下

## 11. 待定问题

- AI 3D 生成用哪个方案（自建开源模型或商业 API）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K11-x"（x 换成子任务编号）。

~~~text
【模块】K11 创作管线（包 gf-creation，路径 server/kernel/creation，层级 L6）
【先读】docs/modules/K11-creation.md；K02、K04、K10、K12、K13 文档的第 4 节；X01-shared.md；主文档第 9、26、28 节
【范围】server/kernel/creation/、server/migrations/（文件名带 k11）、shared/config/creation.json、shared/config/tags.json、shared/prompts/creation_*/、fixtures/creation/、fixtures/ai/creation_*/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-world、gf-capability、gf-ai-gateway、gf-asset、gf-moderation、gf-terrain。库：serde、serde_json、sqlx、tokio、async-trait、tracing、jsonschema、schemars、h3o。测试：proptest
【规则补充】
  - AI 的输出 Schema（generation_output.schema.json）里没有表示数量、尺寸、坐标、价格的数值字段：素材只能填候选列表里的编号；颜色只能填调色板色号（0–15 的整数）；数量只能选 少量 / 若干 / 大量（对应的个数在 creation.json：3、6、12）；地形参数只能选档位（例如 锋利度 平缓 / 中等 / 锋利，对照表在 creation.json）；功能标签只能从 tags.json 的列表里选；位置填"区域局部地图里的格子编号 + 相对描述（中心 / 边缘 / 环绕某项）"
  - 导出纯函数 validate_generation_output(回答, 区域局部地图, 候选素材) -> 问题列表，给 X02 评测复用
  - 数量、坐标、朝向、缩放都由代码换算（用 gf_core::rng，种子 = seed([星球种子, 请求编号])）
  - 校验失败的条目把原因（例如"第 3 项超出了区域边界"）发回 AI 重试，最多 creation.json 的 retries 次；仍然失败的条目放进 rejected，全部失败时用 creation.json 里的默认模板
  - 地形可放置性用 gf-terrain 计算（和客户端同一份代码）
  - 本模块只出方案，不改世界；方案由调用方通过 K04 执行
  - 单元测试的 AI 回答放 fixtures/ai/creation_pick/ 等目录，用 FakeGateway::from_dir
【子任务】
K11-1 组装层最小版（阶段 1）
  - 测试：teahouse_example_plan（"樱花环绕的茶馆"，固定回答是 茶馆_01 一个、樱花树_01 若干、环绕茶馆、颜色 3 → 方案里有 1 个茶馆_01 和 6 个环绕它的樱花树_01，全部在区域内，颜色都是 3）、out_of_region_item_retried_then_fixed（第一次回答有一项越界，重试请求里含"超出了区域边界"，第二次回答修正）、numeric_fields_rejected_and_retried、fallback_template_when_ai_fails、cell_refs_converted_to_positions、description_checked_by_moderation_first
  【效果验收】tools/eval.sh creation ≥ 90%
K11-2 程序化地形入口（阶段 1）
  - AI 给出地形参数后，服务器对改动范围跑一次风化侵蚀模拟（只在服务器跑；随机数用 gf_core::rng，同样的输入得到同样的补丁），结果存成这个 TerrainEdit 的 erosion_patch（格式见 X01-1）；客户端只读补丁，不跑模拟
  - 测试：dragon_ridge_mountain_to_terrain_edit（"一座龙脊一样的山"，固定回答是 类型 山脉、锋利度 锋利、雪线 高 → 一个 TerrainEdit：类型山脉、sharpness_bp 8000、snowline_bp 6000）、river_to_terrain_edit、terrain_edit_raises_heights_along_path（用 gf-terrain 算改动前后路径上的高度）、erosion_patch_deterministic（同样的种子和参数跑两次，补丁逐格相同）、erosion_patch_within_grid_limit（补丁格数不超过 creation.json 的上限）
K11-3 功能标签（阶段 1）
  - 测试：function_tags_only_from_enum、tags_saved_in_plan、name_does_not_change_tags（同一个造物改个名字，标签不变；用 fixtures/ai 里两份不同名字的固定回答）
  【效果验收】tools/eval.sh function_tags ≥ 90%
K11-4 参数化人物与线性工程（阶段 2）
  - 测试：red_haired_blacksmith_params（发色红、职业铁匠、性别女）、equator_wall_path_and_modules（路径沿纬度 0°，模块按固定间距重复）
K11-5 先规划后施工、布局算法、主题包（阶段 2）
  - 测试：magic_academy_plan_items（"魔法学院"→ 规划清单里有主楼、图书馆、宿舍、广场）、layout_deterministic_for_seed、layout_no_overlap_and_inside_region、street_follows_river
  【验收】另加 tools/mutants.sh gf-creation src/layout.rs
K11-6 AI 铸造（阶段 4）
  - 概念图 → 3D 模型（K10 的 generate_image、generate_3d）→ 自动处理 → 交给 K12 质检
  - 测试：forge_stages_progress_in_order、forged_asset_submitted_for_qc、rejected_forge_falls_back_to_assembly
【验收（每个子任务）】tools/check_server.sh gf-creation；tools/check_tests.sh gf-creation <本子任务点名的测试>
【停止条件】AI 的输出需要数值字段；需要直接写世界；需要真实模型才能让测试通过
~~~
