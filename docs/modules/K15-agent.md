# K15 智能体运行时（agent）

> **层级**：内核 L6 · **代码**：`server/kernel/agent` · **首次阶段**：5 · **主文档**：第 15.3、15.5、16.1、29 节

## 1. 职责

- **AI 人物**：创建、分级、存放档案
- **人物档案卡**：性格（3 个关键词）、目标、恐惧、说话风格 + 3 句示范台词、关系
- **记忆**：短期记忆（最近的事件）+ 长期记忆（定期压缩成摘要）
- **感知**：按"显著程度"把世界变化放进人物的感知队列
- **反应**：生成人物的情绪、解释、行动、台词（选择题格式）
- **行动**：人物想做的事，以"人物"身份通过 K04 执行，走正常规则
- **调用节奏**：只在节拍和重大感知时调用 AI；控制每个人物的预算

## 2. 不负责

- 剧情推进和导演决策 → P04
- 决定星球上要有哪些人物 → K16、P04

## 3. 依赖

K00、K02、K04、K10、K13；订阅 K04 的 `CapabilityExecuted`

## 4. 对外接口

```rust
pub enum AgentTier { Main, Supporting, Functional, Crowd }   // 主角 / 配角 / 功能角色 / 群众

pub struct PersonaCard {
    pub traits: [String; 3],
    pub goal: String,
    pub fear: String,
    pub voice: String,
    pub sample_lines: [String; 3],
    pub worldview: WorldviewRef,        // 用来解释异象：修仙世界、科幻世界……
}

#[async_trait]
pub trait Agents: Send + Sync {
    fn create(&self, planet: PlanetId, entity: EntityId, tier: AgentTier, card: PersonaCard) -> Result<AgentId>;
    fn perceive(&self, event: &PerceivableEvent) -> Result<()>;            // 按显著程度分发给附近人物
    async fn react(&self, agent: AgentId, ctx: ReactionContext) -> Result<Reaction>;   // 由 P04 或 K16 在节拍时调用
    fn act(&self, agent: AgentId, action: Action) -> Result<CallResult>;   // 通过 K04 执行
}

pub struct Reaction {
    pub emotion: Emotion,              // 从固定列表选
    pub interpretation: String,
    pub action: Option<Action>,        // 从"当前可执行的行动"列表选
    pub line: Option<String>,          // 唯一的自由文本，有长度限制
}
```

## 5. 数据

| 表 | 关键字段 |
|---|---|
| agents | id, planet_id, entity_id, tier, persona_card, beliefs |
| agent_memory | agent_id, kind（短期 / 长期）, content, source_event, at |
| perception_queue | agent_id, event_ref, salience, at |

## 6. 事件

- 发布：`AgentReacted`、`AgentActed`
- 订阅：`CapabilityExecuted`（K04）→ 计算显著程度，放进感知队列

## 7. 配置

`agent.json`：各级别人物的数量上限、记忆长度、压缩频率、每个人物每天的调用预算、显著程度规则

## 8. 关键规则

- **分级**：主角由 AI 驱动；配角按日程和规则行动，重要时刻才调用 AI；功能角色用完即走；群众由"时间 + 种子"计算，不调用 AI
- **每次调用只给相关材料**：档案卡 + 最近记忆摘要 + 刚感知到的事件 + 当前可执行的行动列表
- **一致性检查**：代码检查硬规则（死去的人物不能说话、人物不能知道自己没见过的事）；便宜档模型给台词打分，低分重做
- 人物的行动和玩家一样走 K04，不能绕过规则
- 记忆定期压缩，控制 token 用量

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 5 | 档案卡与分级 | 能创建「讲故事的老渔夫」 |
| 5 | 感知与反应 | 流星雨后，附近的人物做出符合性格的反应 |
| 5 | 记忆与压缩 | 一周后人物仍记得关键事件，token 用量稳定 |
| 6 | 供 P04 使用 | 剧本人物通过同一套接口运行 |

## 10. 测试要点

- 人物"声音"一致性：用 X02 的评审测试集打分
- 感知范围：远处的小事件不会被感知

## 11. 待定问题

- 是否允许玩家直接和人物对话（"托梦"）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K15-x"（x 换成子任务编号）。

~~~text
【模块】K15 智能体运行时（包 gf-agent，路径 server/kernel/agent，层级 L6）
【先读】docs/modules/K15-agent.md；K02、K04、K10、K13 文档的第 4 节；主文档第 15.3、15.5、16.1、29 节
【范围】server/kernel/agent/、server/migrations/（文件名带 k15）、shared/config/agent.json、shared/prompts/agent_*/、fixtures/agent/、fixtures/ai/agent_*/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-world、gf-capability、gf-ai-gateway、gf-moderation。库：serde、serde_json、sqlx、tokio、async-trait、tracing、jsonschema、schemars。测试：proptest
【规则补充】
  - 情绪只能从这 7 个里选：好奇、恐惧、崇拜、愤怒、喜悦、悲伤、无视
  - 行动只能从"当前可执行的行动"列表里选（列表由代码生成，来自这个人物有权调用的能力）
  - 台词是唯一的自由文本：长度 ≤ agent.json 的 line_max_chars，并经过 K13 检查
  - 显著程度 = 基础值 × max(0, 1 − 距离 / 感知半径)；基础值和感知半径按事件类型写在 agent.json；低于 perceive_threshold 的不进感知队列
  - 群众和功能角色不调用 AI；配角只在显著值超过 supporting_ai_threshold 时调用；主角在节拍和重大感知时调用
  - 一致性硬规则（代码检查）：死去的人物不能说话、不能行动；提示词里只放这个人物感知过的事件
  - 人物的行动以 Caller::Agent 身份通过 K04 执行
【子任务】
K15-1 档案卡与分级（阶段 5）
  - 测试：create_old_fisherman_from_fixture（fixtures/agent/old_fisherman.json：性格 3 个关键词、目标、恐惧、说话风格、3 句示范台词）、persona_card_requires_three_traits_and_lines、crowd_and_functional_tiers_never_call_ai
K15-2 感知与反应（阶段 5）
  - 测试：meteor_shower_perceived_only_within_range（测试配置：流星雨基础值 100、感知半径 2000 米、perceive_threshold 10 → 500 米处显著值 75，感知到；5 公里外为 0，感知不到）、reaction_emotion_must_be_in_enum、action_must_be_from_available_list、reaction_line_checked_and_length_limited、reaction_action_executed_via_capability
K15-3 记忆与压缩（阶段 5）
  - 测试：short_memory_compressed_after_threshold、key_event_kept_after_compression、dead_agent_never_speaks、context_contains_only_perceived_events、daily_call_budget_enforced
K15-4 供 P04 使用（阶段 6）
  - 测试：story_character_uses_same_interface
【效果验收】tools/eval.sh reactions：格式合格率 ≥ 98%，评审平均分 ≥ 4（满分 5）
【验收（每个子任务）】tools/check_server.sh gf-agent；tools/check_tests.sh gf-agent <本子任务点名的测试>
【停止条件】需要让人物绕过 K04 直接改世界；需要玩家和人物直接对话（文档里还是待定）
~~~
