# C05 角色与居民（characters）

> **层级**：客户端 · **代码**：`client/characters` · **首次阶段**：2 · **主文档**：第 9.1、10.3 节

## 1. 职责

- 按参数组装人物：体型、脸、发型、服装、配饰、颜色、贴花
- 共享骨架与动作库：走路、干活、战斗、庆祝、待机
- 居民日程的表现：位置和动作由"星球时间 + 种子"计算，所有玩家看到的一样
- AI 人物的表现：播放服务器下发的动作和对话
- 远处人物使用简化替身

## 2. 不负责

- 人物的思考和决策（K15，在服务器）

## 3. 依赖

C01、C02、C03；`shared/schemas`（人物配方格式）

## 4. 对外接口

| 接口 | 作用 |
|---|---|
| `spawn_character(recipe)` | 按配方组装人物 |
| `play_action(character, action)` | 播放动作 |
| `say(character, line)` | 显示对话气泡 |

## 5. 关键规则

- 群众和配角的日常行动在客户端按"时间 + 种子"计算，不需要服务器逐帧推送
- 所有人物共用同一套骨架，任何新人物都能直接使用动作库

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 2 | 参数化组装 | 「红发女铁匠」显示正确 |
| 2 | 居民日程 | 两个客户端看到的居民位置一致 |
| 5 | AI 人物的动作与对话 | 老渔夫能讲故事 |

## 7. 测试要点

- 同一时刻两个客户端的居民位置比对
- 一百个人物同屏的帧率

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C05-x"（x 换成子任务编号）。

~~~text
【模块】C05 角色与居民（客户端目录 client/characters，测试 client/tests/characters）
【先读】docs/modules/C05-characters.md；K11-creation.md（参数化人物）；K15-agent.md 第 4 节（反应）；主文档第 9.1、10.3 节
【范围】client/characters/、client/tests/characters/、client/tests/fixtures/characters/
【允许的依赖】Godot 内置类（Skeleton3D、AnimationPlayer、AnimationTree、RandomNumberGenerator）；C01、C02、C03；GUT
【规则补充】
  - 人物配方只含零件编号和调色板色号（格式见 shared/schemas 的 recipe.schema.json）；所有人物共用一套骨架和动作库
  - 居民日程：位置和动作 = f(星球时间, 居民种子)，用 RandomNumberGenerator 并显式设置 seed 和 state；不用本机时间，不用全局随机
  - 距离超过 impostor_distance_m 的人物换成简化替身
【子任务】
C05-1 参数化组装（阶段 2）
  - 测试：test_red_haired_blacksmith_assembles_expected_parts（fixtures 里的"红发女铁匠"配方 → 女性体型、红色头发、铁匠服、锤子）、test_all_characters_share_skeleton、test_far_characters_use_impostor
C05-2 居民日程（阶段 2）
  - 测试：test_schedule_deterministic_for_time_and_seed、test_work_by_day_home_by_night
C05-3 AI 人物的动作与对话（阶段 5）
  - 测试：test_reaction_event_plays_action_and_bubble、test_long_line_bubble_wraps
  【人工验收】录屏：老渔夫讲故事（动作 + 气泡）
【验收（每个子任务）】tools/check_client.sh characters；tools/check_tests.sh characters <本子任务点名的测试>
【停止条件】需要服务器逐帧推送居民位置；需要第二套骨架
~~~
