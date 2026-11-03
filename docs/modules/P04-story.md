# P04 剧本（story）

> **层级**：星球层 L8 · **所属玩法包**：剧本星球（官方，可选安装）· **代码**：`server/packs/story` · **首次阶段**：6 · **主文档**：第 15、29 节

## 1. 职责

- **小说导入流水线**：切分 → 逐章提取 → 合并别名、统计、人物分级 → 提炼剧情骨架 → 检查 → 玩家确认
- **剧情骨架**：世界观、地点、人物、剧情节点
- **导演 AI**：每个节拍决定推进到哪个节点、是否偏离原著
- **神迹**：玩家对剧情的影响（改变环境、异象特效、放置造物）
- **祈祷**：人物的请求，推送给玩家
- **剧情事件日志**：剧情的"真相"
- **前情提要、结局、重演**

## 2. 不负责

- AI 人物的档案、记忆、感知、反应 → K15
- 搭建星球 → K16、K11
- 播放 → K14、C06、C09

## 3. 依赖

K00、K02、K03、K04、K07、K10、K11、K13、K14、K15；订阅 P02 的战斗事件（声明在 manifest）

## 4. 对外接口

**注册的能力**：

| 能力 | 级别 | 说明 |
|---|---|---|
| 导入小说 | 星球级 | 主人使用 |
| 施加神迹 | 星球级 | 按主人设置的权限；消耗金和 AI 额度 |
| 回应祈祷 | 星球级 | |
| 设置节奏 / 神迹权限 | 星球级 | 主人使用 |
| 重演 | 星球级 | 主人使用 |

HTTP：时间线和事件、人物档案、剧情骨架卡

## 5. 数据

| 表 | 关键字段 |
|---|---|
| stories | id, planet_id, title, source（原创 / 公有领域 / 授权）, worldview, pace, miracle_policy, status |
| story_beats | id, story_id, order, summary, preconditions, characters, location, status（未到 / 进行中 / 已发生 / 被改写） |
| story_characters | id, story_id, agent_id, tier, source_chapters |
| story_events | id, story_id, planet_time, type, actors, content, world_effects, caused_by |
| miracles | id, story_id, player_id, god_name, type, params, salience, status |
| prayers | id, agent_id, request, status（等待 / 已回应 / 过期） |

## 6. 事件

- 发布：`StoryBeatReached`、`StoryBranched`、`PrayerRaised`、`StoryEnded`
- 订阅：K04 的 `CapabilityExecuted`（环境变化）、P02 的 `BattleSettled`（变成"天外入侵"）

## 7. 配置

`story.json`：节拍间隔、无人观看多久暂停（默认 24 小时）、人物分级阈值、神迹价格和每日上限、显著程度规则

## 8. 关键规则

**导入**：
- 每一步都只处理一小块：逐章提取用便宜档，提炼骨架用强档且输入是结构化数据
- **人物分级**：按出场章节、台词量、关系数量、参与的剧情节点统计 → 主角 / 配角 / 次要人物（功能角色）/ 群众
- **有据可查**：人物档案的每一条都标明出处章节，没有出处就删掉
- 只接受原创、公有领域或获得授权的作品；公开星球不展示原文

**推进**：
- 剧情节点是锚点；节点的前提条件由代码判断，AI 只决定细节和措辞
- 人物想改变世界，通过 K15 以"人物"身份调用 K04，走正常规则
- **事件日志是真相**：回看和重演都读日志，不重新调用 AI
- 无人观看时按章节摘要推进，超过 24 小时暂停
- 剧情自动推进花主人的额度，标记为自主运转（不换算资源点、不计入排名）

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 6 | 导入流水线 | 《西游记》前几回：孙悟空被识别为主角，樵夫被识别为功能角色，每条档案都有出处 |
| 6 | 导演循环 | 剧情按节拍推进到"石猴称王" |
| 6 | 神迹与感知 | 流星雨让"出海寻仙"提前，并标出与原著的分歧点 |
| 6 | 祈祷、前情提要、结局、重演 | 完整跑通一个短篇 |

## 10. 测试要点

- 导入：X02 的人物分级测试集（30 章人工标注）
- 一致性：死去的人物不会说话；人物不知道没见过的事
- 版权：知名版权作品被拒绝

## 11. 待定问题

- 是否加入"托梦"这类直接和人物对话的神迹
- 内容分级规则

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 P04-x"（x 换成子任务编号）。

~~~text
【模块】P04 剧本（包 gf-pack-story，路径 server/packs/story，层级 L8，官方玩法包 story）
【先读】docs/modules/P04-story.md；K02、K03、K04、K07、K10、K11、K13、K14、K15 文档的第 4 节；P02-combat.md 第 6 节；主文档第 15、29 节
【范围】server/packs/story/、server/migrations/（文件名带 p04）、shared/config/story.json、shared/prompts/story_*/、fixtures/story/、fixtures/xiyouji/、fixtures/ai/story_*/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-world、gf-scheduler、gf-capability、gf-economy、gf-ai-gateway、gf-creation、gf-moderation、gf-presentation、gf-agent、gf-pack-combat（只用它的事件类型）。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、jsonschema、schemars、regex。测试：proptest
【规则补充】
  - 能力名：story.导入小说、story.施加神迹、story.回应祈祷、story.设置节奏、story.重演（都是星球级）
  - fixtures/xiyouji/：《西游记》第一回和第二回原文（公有领域）；每章提取用的 AI 回答放 fixtures/ai/story_chapter_extract/
  - 分章：按"第…回"或"第…章"标题切分
  - 人物分级分数 = 出场章节数 × w_chapters + 台词数 × w_lines + 关系数 × w_relations + 参与的剧情节点数 × w_beats；按 story.json 的阈值分成主角、配角、功能角色、群众
  - 档案的每一条都要有出处章节；没有出处的条目删掉
  - 剧情节点的前提条件由代码判断；导演 AI 只选措辞和细节，不能决定节点是否达成
  - 回看和重演只读 story_events，不调用 AI
  - 剧情自动推进的 AI 用量记在主人名下，autonomous = true
【子任务】
P04-1 导入流水线（阶段 6）
  - 测试：split_by_chapter、aliases_merged_into_sun_wukong（美猴王、石猴、孙悟空合并成一个人物）、tiers_for_xiyouji_opening（孙悟空 → 主角，樵夫 → 功能角色，群猴 → 群众）、facts_without_source_removed、copyrighted_work_rejected、original_text_not_shown_on_public_planet
  【效果验收】tools/eval.sh tiering：主角识别的准确率和召回率都 ≥ 90%
P04-2 导演循环（阶段 6）
  - 测试：beat_advances_only_when_preconditions_met、director_chooses_wording_only、reaches_stone_monkey_king_beat（按节拍推进到"石猴称王"）、paused_after_24h_unwatched、replay_reads_event_log_without_ai
P04-3 神迹与感知（阶段 6）
  - 测试：meteor_miracle_advances_seek_immortality_and_marks_divergence（流星雨神迹让"出海寻仙"提前发生，并标记为和原著的分歧）、miracle_permissions_and_daily_limit、god_name_stele_built_via_capability（群猴立碑，碑文是玩家起的神名）、story_progress_usage_marked_autonomous
P04-4 祈祷、前情提要、结局、重演（阶段 6）
  - 测试：prayer_raised_then_answered、prayer_expires、recap_built_from_event_log、ending_recorded_and_replay_resets
【验收（每个子任务）】tools/check_server.sh gf-pack-story；tools/check_tests.sh gf-pack-story <本子任务点名的测试>
【停止条件】需要导入有版权的作品做测试；需要让 AI 决定剧情节点是否达成；需要在公开星球显示原文
~~~
