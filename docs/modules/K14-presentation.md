# K14 演出（presentation）

> **层级**：内核 L6 · **代码**：`server/kernel/presentation` · **首次阶段**：3 · **主文档**：第 14.5、14.6 节

## 1. 职责

- 管理**演出提供者**：模板演出器（在客户端）、AI 战报、剧情演出器、预留的 AI 编排 / 新素材 / 视频
- 为结果记录（战斗、剧情）生成、校验、保存**演出脚本**
- **演出校验器**：确保演出不歪曲结果
- AI 战报文字

## 2. 不负责

- 结果本身 → P02、P04
- 播放 → C06

## 3. 依赖

K00、K10、K12、K13

## 4. 对外接口

```rust
pub trait PresentableRecord: Send + Sync {          // 由 P02、P04 实现
    fn kind(&self) -> RecordKind;                    // Battle / Story / ...
    fn refs(&self) -> RecordRefs;                    // 脚本可以引用的东西：落点、伤害、人物……
    fn summary(&self) -> Value;
}
pub trait PresentationProvider: Send + Sync {
    fn name(&self) -> &str;
    fn supports(&self, kind: RecordKind) -> bool;
    async fn generate(&self, record: &dyn PresentableRecord, client_caps: &ClientCaps) -> Result<Script>;
}
pub trait Presentation: Send + Sync {
    fn submit(&self, record: Box<dyn PresentableRecord>) -> Result<RecordId>;   // 结果模块调用
    fn script_for(&self, record: RecordId) -> Option<Script>;                    // 没有就返回空，客户端用模板
    fn register_provider(&self, p: Arc<dyn PresentationProvider>);
}
```

HTTP：获取某条记录可用的演出脚本

## 5. 数据

| 表 | 关键字段 |
|---|---|
| presentations | id, record_id, provider, version, script（JSON）, status（生成中 / 可用 / 未通过 / 超时）, created_at |

## 6. 事件

- 发布：`PresentationReady`

## 7. 配置

`presentation.json`：启用哪些提供者、超时、每日预算、适用范围（例如 AI 演出只用于世界 BOSS 决战）

## 8. 关键规则

- **演出只读结果，绝不修改结果**
- **演出校验器**：脚本格式正确；所有数字和位置都用 `value_ref` / `target_ref` 引用结果记录；引用的东西真实存在；时长在限制内。不通过就丢弃
- 任何失败或超时，客户端都回退到模板演出
- 同一条记录的脚本生成后缓存，所有观看者看到同一个版本
- 客户端声明支持哪些演出类型，提供者只生成客户端能播放的内容
- AI 战报 5 秒内没返回，就用模板文字

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 3 | 提供者接口、脚本格式、校验器 | 构造一个"打出 999 伤害"的假脚本，被校验器拒绝 |
| 3 | AI 战报 | 超时自动用模板文字 |
| 6 | 剧情演出器 | 剧情事件能转成脚本播放 |

## 10. 测试要点

- 校验器：数字写死、引用不存在的实体、超长时长，都被拒绝

## 11. 待定问题

- AI 实时演出什么时候开启、先开放哪一级

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K14-x"（x 换成子任务编号）。

~~~text
【模块】K14 演出（包 gf-presentation，路径 server/kernel/presentation，层级 L6）
【先读】docs/modules/K14-presentation.md；X01-shared.md 第 4 节（演出脚本示例）；K10、K13 文档的第 4 节；主文档第 14.5、14.6 节
【范围】server/kernel/presentation/、server/migrations/（文件名带 k14）、shared/config/presentation.json、shared/prompts/battle_report/、fixtures/presentation/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-ai-gateway、gf-asset、gf-moderation。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、jsonschema、schemars。测试：proptest
【规则补充】
  - 校验器规则：
      1. 脚本符合 presentation.schema.json
      2. 数字只能出现在 t、duration_ms、scale 这几个字段；伤害、血量等结果数值只能用 value_ref 引用（例如 "results[0].damage"）
      3. value_ref、target_ref、actor 引用的东西必须在记录的 refs() 里
      4. duration_ms ≤ presentation.json 的 max_duration_ms；所有 t ≤ duration_ms
      5. 不认识的 cue 类型允许存在，但标记为 unknown（客户端会跳过）
  - presentation.json：providers（启用哪些）、max_duration_ms 15000、report_timeout_ms 5000、daily_budget、ai_scope
  - 同一条记录只保存一份可用的脚本；之后的观看者都拿这一份
【子任务】
K14-1 提供者接口、脚本格式、校验器（阶段 3）
  - 测试：accepts_x01_example_script（X01 文档里的示例原样通过）、rejects_literal_damage_999（把 value_ref 换成写死的 999）、rejects_unknown_ref、rejects_overlong_duration、unknown_cue_type_allowed_but_flagged
  【验收】另加 tools/mutants.sh gf-presentation src/validate.rs
K14-2 AI 战报（阶段 3）
  - 测试：battle_report_timeout_uses_template_text（FakeGateway 延迟 6 秒，用 tokio 的暂停时间测试，不真的等）、battle_report_checked_by_moderation、same_record_same_script_cached、failed_provider_leaves_template
K14-3 剧情演出器（阶段 6）
  - 测试：story_events_to_script_fixture（fixtures/presentation/story_events.json 转成的脚本通过校验器，并和期望的 cue 列表一致）
【验收（每个子任务）】tools/check_server.sh gf-presentation；tools/check_tests.sh gf-presentation <本子任务点名的测试>
【停止条件】演出需要修改结果；需要在服务器上渲染画面或视频
~~~
