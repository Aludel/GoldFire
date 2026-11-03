# C09 剧情观看与神迹（story_view）

> **层级**：客户端 · **代码**：`client/story_view` · **首次阶段**：6 · **主文档**：第 15 节

## 1. 职责

- **小说导入界面**：上传、声明来源、查看并修改"剧情骨架卡"（主角名单、剧情节点）
- **剧情时间线**：按时间回看剧情事件；标出与原著的分歧点
- **字幕与对话气泡**；镜头自动跟随主角
- **人物档案**：性格、记忆、关系、对"神"的信仰
- **前情提要**
- **神迹面板**：选择神迹类型、设置神名、查看今日剩余次数
- **祈祷通知**：人物的请求，可以选择回应

## 2. 不负责

- 剧情推进（P04）；播放演出（复用 C06 的演出播放器）

## 3. 依赖

C01、C03、C05、C06

## 4. 对外接口

| 接口 | 作用 |
|---|---|
| `open_story(planet_id)` | 打开剧情界面 |
| `import_novel(file, source)` | 上传小说 |
| `cast_miracle(type, params, god_name)` | 施加神迹（调用能力） |
| `answer_prayer(prayer_id, response)` | 回应祈祷 |

## 5. 关键规则

- 公开星球不显示原文，只显示剧情骨架和生成的内容
- 剧情回看读取事件日志，不触发任何 AI 调用

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 6 | 导入界面与剧情骨架卡 | 能修改主角名单后开始剧情 |
| 6 | 时间线、字幕、人物档案、前情提要 | 回来时能看懂错过的剧情 |
| 6 | 神迹与祈祷 | 流星雨后看到石猴的反应和分歧点标记 |

## 7. 测试要点

- 长时间线的加载速度

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C09-x"（x 换成子任务编号）。

~~~text
【模块】C09 剧情观看与神迹（客户端目录 client/story_view，测试 client/tests/story_view）
【先读】docs/modules/C09-story-view.md；P04-story.md 第 4、5 节和【规则补充】（能力名）；C06-vfx.md 第 4 节；主文档第 15 节
【范围】client/story_view/、client/tests/story_view/、client/tests/fixtures/story_view/
【允许的依赖】Godot 内置类；C01、C03、C05、C06；GUT
【规则补充】
  - 导入时必须选择来源（原创 / 公有领域 / 已获授权），不选不能提交
  - 回看只读服务器的剧情事件（GET 请求），不调用任何能力或 AI
  - 公开星球上不显示原文，只显示剧情骨架和生成的内容
【子任务】
C09-1 导入界面与剧情骨架卡（阶段 6）
  - 测试：test_import_requires_source_declaration、test_skeleton_card_edit_main_characters
C09-2 时间线、字幕、人物档案、前情提要（阶段 6）
  - 测试：test_timeline_from_event_fixture、test_divergence_marker_shown、test_replay_makes_no_capability_calls、test_original_text_hidden_on_public_planet
  【人工验收】离开一段时间后回来，看前情提要能看懂错过的剧情
C09-3 神迹与祈祷（阶段 6）
  - 测试：test_cast_miracle_sends_god_name、test_remaining_miracles_shown、test_answer_prayer
【验收（每个子任务）】tools/check_client.sh story_view；tools/check_tests.sh story_view <本子任务点名的测试>
【停止条件】需要在客户端推进剧情；需要显示受版权保护的原文
~~~
