# C06 特效与演出（vfx）

> **层级**：客户端 · **代码**：`client/vfx` · **首次阶段**：1（特效组合、造物特效）；3（演出）· **主文档**：第 14.4–14.7 节

## 1. 职责

- **特效组合**：形态 × 元素 × 修饰（20 × 8 × 10 = 1600 种组合，只需 38 个零件）
- **造物特效**：按配方播放（例如神殿夜间的火光）
- **环境特效**：雨雪、极光、火山喷发、工业雾霾，由星球状态驱动
- **演出播放器**：按时间轴播放任何提供者生成的演出脚本
- **模板演出器**：默认的演出提供者，在客户端由战斗记录直接生成脚本
- **金缮裂痕**的显示

## 2. 不负责

- 战斗结果（P02）；AI 演出的生成（K14）

## 3. 依赖

C01、C02、C03、C05；`shared/config/effects.json`；演出脚本格式（X01）

## 4. 对外接口

| 接口 | 作用 |
|---|---|
| `play_effect(form, element, modifier, params, at)` | 播放一个组合特效 |
| `play_script(script, record)` | 播放演出脚本 |
| `template_script(record)` | 用模板演出器生成脚本 |

## 5. 关键规则

- 只读战斗记录，不重新计算结果
- 脚本里的位置和数字从记录里取（`target_ref`、`value_ref`）
- 遇到不认识的演出类型就跳过，保证旧客户端不出错
- 有通过校验的 AI 脚本时优先使用，否则用模板

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 特效零件与组合 | 「螺旋上升的冰龙卷」能播放 |
| 3 | 演出播放器与模板演出器 | 「冰霜巨龙」攻击农田平原的回放完整 |
| 3 | 金缮裂痕 | 破碎后出现，重建后保留 |
| 4 | 环境特效 | 工业值高时天空发灰 |

## 7. 测试要点

- 同一场战斗回放多次，画面一致
- 包含未知类型的脚本不会报错

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C06-x"（x 换成子任务编号）。

~~~text
【模块】C06 特效与演出（客户端目录 client/vfx，测试 client/tests/vfx）
【先读】docs/modules/C06-vfx.md；X01-shared.md 第 4 节（演出脚本示例）；K14-presentation.md 第 8 节和【规则补充】（校验规则）；P02-combat.md（战斗记录）；主文档第 14.4–14.7 节
【范围】client/vfx/、client/tests/vfx/、client/tests/fixtures/vfx/、shared/config/effects.json
【允许的依赖】Godot 内置类（GPUParticles3D、ShaderMaterial、Tween）；C01、C02、C03、C05；GUT
【规则补充】
  - 特效 = 形态 × 元素 × 修饰，零件定义在 effects.json（20 种形态、8 种元素、10 种修饰，共 38 个零件）；不认识的零件用 effects.json 里的 fallback
  - 演出播放器按 t 排序执行 cue；数字和位置只从记录里取（value_ref、target_ref），脚本里的数字只用于时间和缩放
  - 不认识的 cue 类型跳过并写日志，不报错
  - 有通过校验的 AI 脚本就用它，否则用 template_script(record) 生成的模板脚本
【子任务】
C06-1 特效零件与组合（阶段 1）
  - 测试：test_all_part_combinations_instantiate（1600 种组合都能实例化）、test_unknown_part_uses_fallback、test_ice_tornado_spiral_up（形态 龙卷、元素 冰、修饰 螺旋上升）
  【人工验收】录屏："螺旋上升的冰龙卷"
C06-2 演出播放器与模板演出器（阶段 3）
  - 测试：test_template_script_matches_x01_example（client/tests/fixtures/vfx/battle_b301.json 生成的模板脚本和 X01 文档里的示例一致）、test_unknown_cue_skipped、test_values_read_from_refs（伤害数字来自记录的 results[0].damage）、test_ai_script_preferred_when_valid、test_replay_identical_twice
  【人工验收】录屏："冰霜巨龙"攻击农田平原的完整回放
C06-3 金缮裂痕（阶段 3）
  - 测试：test_cracks_appear_after_shatter_and_persist_after_rebuild
C06-4 环境特效（阶段 4）
  - 测试：test_industry_value_maps_to_haze_intensity、test_weather_driven_by_planet_state
【验收（每个子任务）】tools/check_client.sh vfx；tools/check_tests.sh vfx <本子任务点名的测试>
【停止条件】需要在客户端重新计算战斗结果；需要从脚本里读结果数字
~~~
