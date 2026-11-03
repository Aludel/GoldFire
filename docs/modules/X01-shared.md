# X01 共享层（shared）

> **层级**：共享 · **代码**：`shared/` · **首次阶段**：1 · **主文档**：第 20、23、26–28 节

## 1. 职责

| 目录 | 内容 |
|---|---|
| `shared/terrain/` | 地形生成 Rust 库：服务器原生调用，客户端通过 godot-rust 调用 |
| `shared/schemas/` | JSON Schema：由服务器的 Rust 类型自动生成（schemars） |
| `shared/config/` | 数值表和规则配置 |
| `shared/prompts/` | 提示词模板与示例（带版本号） |

## 2. 不负责

任何业务逻辑（地形库只做"种子 + 改动 → 高度和地貌"的纯计算）。

## 3. 地形库

- 输入：星球种子、改动列表、区域、精度；输出：高度场和地貌类型
- **确定性**：同样的输入在任何平台上得到完全相同的结果
  - 不使用平台自带的三角函数和指数函数，改用纯 Rust 实现的数学库
  - 不使用多线程的不确定合并顺序
- 有一套"金标准"测试：固定种子的输出存成文件，服务器和客户端都要逐点比对一致
- **侵蚀补丁**：地形改动可以带一张侵蚀补丁（覆盖改动范围的经纬度网格，每个格点一个高度差）。侵蚀模拟只在服务器跑一次（K11），结果存进补丁；地形库只负责把补丁按双线性插值加到高度上，只用加减乘除，所以服务器和客户端仍然逐点一致

## 4. 数据格式（Schema）

| 文件 | 描述 | 使用方 |
|---|---|---|
| `recipe.schema.json` | 造物配方 | K02、K11、C03–C05 |
| `generation_output.schema.json` | AI 造物输出（不含数值字段） | K11 |
| `command.schema.json` | 能力调用的输入 | K04、C01 |
| `battle.schema.json` | 战斗记录（含种子） | P02、C06 |
| `presentation.schema.json` | 演出脚本 | K14、C06 |
| `rule.schema.json` | 声明式规则 | K05、K06 |
| `pack_manifest.schema.json` | 玩法包 manifest | K06 |
| `ui.schema.json` | 声明式界面 | K06、C07 |
| `persona.schema.json` | 人物档案卡 | K15 |
| `reaction.schema.json` | 人物反应 | K15 |
| `story_skeleton.schema.json` | 剧情骨架 | P04 |
| `story_event.schema.json` | 剧情事件 | P04、C09 |

- 每个 Schema 带 `version` 字段；变更要升版本号
- 类型和 Schema 文件的对应关系登记在 `tools/schema_gen/src/registry.rs`
- CI 检查：Rust 类型改了但 Schema 没重新生成，构建失败

**演出脚本示例**：

```json
{
  "version": 1,
  "record_id": "b_301",
  "provider": "template",
  "duration_ms": 6000,
  "cues": [
    { "t": 0,    "type": "camera",          "target_ref": "impact", "move": "orbit_in" },
    { "t": 500,  "type": "effect_template", "form": "召唤兽冲锋", "element": "冰", "actor": "c_dragon", "scale": 2 },
    { "t": 2500, "type": "effect_template", "form": "区域受击", "element": "冰", "target_ref": "impact" },
    { "t": 2600, "type": "damage_number",   "target_ref": "impact", "value_ref": "results[0].damage" }
  ]
}
```

## 5. 配置表

| 文件 | 内容 | 使用方 |
|---|---|---|
| `buildings.json` | 建筑各等级的价格、转换速度、防御与覆盖范围、供给需求、服务半径、武器倍率 | K04、P01、P02 |
| `economy.json` | 递减档位、封顶、保底、能量池上限、默认转换速度、掠夺比例 | K07 |
| `metering.json` | token 类型权重、来源倍率、模型系数 | K08 |
| `billing.json` | 灵感与成本的换算、每日免费额度、任务估算、成本闸门、支付与商品 | K18 |
| `combat.json` | 伤害参数、攻击半径、克制表、侦察、次数、保护时长、赛季、每次攻击花火的范围 | P02 |
| `tags.json` | 功能标签和外观标签、联动规则 | K11、P01 |
| `supply.json` / `environment.json` | 供给规则 / 环境平衡与灾厄阈值 | P01 |
| `boss.json` | 灾厄与世界 BOSS 参数、奖励 | P03 |
| `time.json` | 昼夜、季节 | K03、C03 |
| `effects.json` | 特效零件 | K11、C06 |
| `palettes.json` | 调色板 | K11、C03 |
| `models.json` | 模型分档、任务默认档位、备用模型、模型单价 | K10、K18 |
| `presentation.json` | 演出提供者、预算、范围 | K14 |
| `story.json` | 剧本参数 | P04 |
| `agent.json` | 人物分级、记忆、预算 | K15 |
| `planet.json` / `planet_mind.json` | 星球与星球 AI 的默认值 | K02、K16 |
| `capabilities.json` / `script.json` / `packs.json` / `orbit.json` / `history.json` | 各模块参数 | K04、K05、K06、K09、K17 |

- 每个配置表都有 Schema，服务器启动时校验

## 6. 提示词

- 一个任务一个目录：模板、变量说明、示例、对应的测试集名称
- 改提示词要升版本号，并跑 X02 测试集

## 7. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 地形库与金标准测试 | 服务器和客户端逐点一致 |
| 1 | Schema 自动生成与 CI 检查 | 改类型不改 Schema 会构建失败 |
| 1 | 第一批配置表和提示词 | 服务器启动时全部校验通过 |

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 X01-x"（x 换成子任务编号）。

~~~text
【模块】X01 共享层：地形库（包 gf-terrain，路径 shared/terrain，算作 L0）、Schema 生成（包 gf-schema-gen，路径 tools/schema_gen，顶层）、配置表和提示词（shared/config、shared/prompts）
【先读】docs/modules/X01-shared.md；docs/modules/README.md 的全局约定；K00-core.md 第 4 节（配置加载）；主文档第 20、23、26–28 节
【范围】shared/terrain/、tools/schema_gen/、tools/check_schemas.sh、shared/schemas/、shared/config/（只限 X01-3 的整体校验）、shared/prompts/（只限 meta.toml 的格式和校验）、.github/workflows/terrain.yml
【允许的依赖】gf-terrain：只有 libm 和 serde（不依赖工作区里的任何包）。gf-schema-gen：schemars、serde_json、toml、clap，以及工作区里定义 Schema 类型的 gf- 包。测试：proptest
【规则补充】
  - 地形库接口：height(seed: u64, edits: &[TerrainEdit], lat_deg: f64, lon_deg: f64) -> f32（米）；biome(同样的参数) -> Biome；build_chunk(seed, edits, 区域, 精度) -> Chunk（顶点高度和地貌）
  - 侵蚀补丁：TerrainEdit 可以带 erosion_patch: Option<HeightPatch>；HeightPatch { lat_min, lat_max, lon_min, lon_max: f64, cols, rows: u16, deltas_dm: Vec<i16> }（高度差，单位分米，按行存）；height() 在补丁范围内加上四个相邻格点的双线性插值，只用加减乘除
  - 噪声在库里自己实现（用整数哈希），不用第三方噪声库；三角、指数、对数、幂函数只用 libm；在 shared/terrain/clippy.toml 里用 disallowed-methods 禁止 f32/f64 的 sin、cos、tan、asin、acos、atan、atan2、exp、ln、log10、log2、powf、powi、sinh、cosh、tanh、hypot、cbrt（同时保留根目录 clippy.toml 的设置）
  - 不用多线程合并结果；同样的输入在任何平台上逐位相同
  - 金标准 shared/terrain/golden/seed_42.json：种子 42、空改动列表，1000 个点（纬度从 −89.9° 到 89.9°、经度从 −180° 到 179.64° 均匀取样），每个点存经纬度和高度的 f32 位模式（u32）。这是确定性金标准：用代码生成一次，在 PR 里声明
  - Schema：每个文件带 version 字段；类型和文件的对应关系登记在 tools/schema_gen/src/registry.rs（所有模块新增数据格式或配置表时都在这里登记）
  - 提示词目录 shared/prompts/<任务>/：v<N>.md 模板 + meta.toml（variables = [...]、eval_dataset = "..."）
【子任务】
X01-1 地形库与金标准（阶段 1）
  - 测试：golden_heights_match、height_continuous（proptest：两点相距 1 米以内，高度差不超过 max_slope_per_m）、edit_affects_only_its_radius（改动半径外的高度和没有改动时逐位相同）、mountain_edit_raises_height、height_patch_applied（补丁格点上的高度差正好是 deltas_dm / 10 米，补丁范围外不变；金标准里包含一个带补丁的改动）
  - .github/workflows/terrain.yml：在 Linux、macOS、Windows 上跑 golden_heights_match
  【验收】tools/check_server.sh gf-terrain；cargo clippy -p gf-terrain -- -D warnings（禁用的方法会报错）
X01-2 Schema 自动生成与检查（阶段 1）
  - gf-schema-gen 把 registry.rs 里登记的类型生成到 shared/schemas/；tools/check_schemas.sh 生成到临时目录再和 shared/schemas/ 比较，不一致就失败
  - 测试：generated_schemas_up_to_date、every_schema_has_version、registry_names_unique
  【验收】tools/check_server.sh gf-schema-gen；tools/check_schemas.sh
X01-3 配置表与提示词的整体校验（阶段 1，K10-1 之后）
  - tools/check_schemas.sh 增加两项：shared/config/ 里每个文件都能按登记的类型通过校验，没有登记的文件报错；每个提示词模板里的 {{变量}} 和 meta.toml 的 variables 完全一致
  - 测试：all_configs_valid、unregistered_config_rejected、prompt_variables_declared
  【验收】tools/check_server.sh gf-schema-gen；tools/check_schemas.sh
【验收（每个子任务）】另加 tools/check_tests.sh <包> <本子任务点名的测试>
【停止条件】地形库需要依赖平台数学函数或第三方噪声库；金标准对不上（先报告，不要重新生成金标准）
~~~
