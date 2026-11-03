# K13 内容安全（moderation）

> **层级**：内核 L1 · **代码**：`server/kernel/moderation` · **首次阶段**：1 · **主文档**：第 15.8、34 节

## 1. 职责

- 检查文本和图片：提示词、名字、刻字、对话、导入的小说、玩法包里的文字、铸造结果
- 版权过滤：拒绝生成或导入知名的版权作品和角色
- 处理玩家举报
- 内容分级（例如剧本星球的分级）

## 2. 不负责

- 判断造物是否越界 → K11、K04
- 判断脚本是否安全 → K06、K05

## 3. 依赖

K00。AI 审核通过 `ContentClassifier` 接口在启动时注入（见第 4 节），本模块不直接依赖 AI 网关

## 4. 对外接口

```rust
pub trait ContentClassifier: Send + Sync {          // 由 app 注入（调用 AI 审核模型）
    fn classify(&self, item: &ContentItem) -> Result<Classification>;
}
pub trait Moderation: Send + Sync {
    fn check(&self, item: ContentItem, context: CheckContext) -> Result<Verdict>;   // 通过 / 拒绝 + 原因
    fn report(&self, reporter: UserId, target: ReportTarget, reason: String) -> Result<ReportId>;
}
```

HTTP：提交举报

## 5. 数据

| 表 | 关键字段 |
|---|---|
| moderation_logs | id, target_type, target_ref, verdict, reason, at |
| reports | id, reporter_id, target_type, target_ref, reason, status |

## 6. 事件

- 发布：`ContentRejected`、`ReportFiled`

## 7. 配置

- `moderation.json`（公开部分）：检查级别、分级规则
- 私有词库：**不进仓库**，从私有配置加载

## 8. 关键规则

- **多层防护**：词库（快）→ AI 审核模型（准）→ 玩家举报（兜底）
- 版权检测：对照知名作品和角色列表，同时让 AI 判断"是否明显在模仿某个已知作品"
- 拒绝时给出原因，但不暴露词库内容
- 检查失败（审核服务不可用）时，默认拒绝，而不是放行
- 没有接入 AI 审核时，只有开发模式可以只用词库；正式环境没有 AI 审核就拒绝启动

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 词库 + AI 审核 | 测试用的不当内容全部被拦下 |
| 2 | 举报 | 举报后主人和管理员能看到 |
| 4 | 版权检测 | 知名角色的名字和描述被拒绝 |
| 6 | 小说导入检查与内容分级 | 导入时给出分级建议 |

## 10. 测试要点

- 绕过尝试：谐音、拆字、夹在长文本中间
- 误伤率：正常内容不应被大量拒绝（用 X02 的测试集评估）

## 11. 待定问题

- 人工复核流程由谁来做

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K13-x"（x 换成子任务编号）。

~~~text
【模块】K13 内容安全（包 gf-moderation，路径 server/kernel/moderation，层级 L1）
【先读】docs/modules/K13-moderation.md；主文档第 15.8、34 节
【范围】server/kernel/moderation/、server/migrations/（文件名带 k13）、shared/config/moderation.json、fixtures/moderation/、server/app/（只限接入本模块；ContentClassifier 适配器要调用 K10，由 K10-1 写）
【允许的依赖】模块：gf-core。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、schemars、unicode-normalization、pinyin。测试：proptest
【规则补充】
  - 私有词库的路径从环境变量 GF_PRIVATE_WORDLIST 读；测试只用 fixtures/moderation/wordlist_test.txt（只放无害的占位词，例如"违禁样例"）
  - 文本先规范化再查词库：NFKC、全角转半角、去掉空白、标点、符号和零宽字符；再把整段转成不带声调的拼音，和词库词的拼音比对（找谐音）
  - 检查顺序：词库 → AI 审核（ContentClassifier）；任何一层拒绝就拒绝；AI 审核出错或超时就拒绝
  - 拒绝原因只给类别（例如"包含不当内容"），不回显命中的词
  - 测试用 FakeClassifier（固定返回值），不调用 AI
  - 没有注入分类器时：server.toml 的 dev_mode = true 只用词库，每次检查写一条 warn 日志；dev_mode = false 拒绝启动。K13-1 合并后、K10-1 接好适配器之前，本地开发就是这种情况。"没有分类器"和"分类器出错"是两回事：分类器出错时，开发模式下也拒绝
【子任务】
K13-1 词库 + AI 审核（阶段 1）
  - 测试：wordlist_blocks、classifier_blocks、classifier_unavailable_rejects_by_default、rejection_reason_does_not_reveal_wordlist、blocked_text_hidden_in_long_text_found（2000 字正常文本中间夹一个词库词）、spaced_and_symbol_variants_found（"违 禁 样 例""违*禁*样*例"、全角字符）、homophone_variant_found（同音不同字）、split_character_variant_goes_to_classifier（拆字变体词库查不到，由 AI 审核拦下）、normal_text_passes、dev_mode_without_classifier_uses_wordlist_only（词库词照样拦下，正常文本通过）、classifier_required_outside_dev_mode
  【验收】另加 tools/mutants.sh gf-moderation src/check.rs
  【效果验收】tools/eval.sh moderation：拦截率 ≥ 95%，误伤率 ≤ 5%
K13-2 举报（阶段 2）
  - 测试：report_visible_to_owner_and_admin、report_not_visible_to_others
K13-3 版权检测（阶段 4）
  - 知名作品和角色的名单放私有配置（环境变量 GF_PRIVATE_IP_LIST），测试用 fixtures/moderation/ip_list_test.txt
  - 测试：known_work_names_rejected、classifier_imitation_flag_rejected
K13-4 小说检查与分级（阶段 6）
  - 测试：long_novel_checked_in_chunks（分块检查，任何一块不通过就整体不通过并指出章节）、rating_suggested_from_classifier
【验收（每个子任务）】tools/check_server.sh gf-moderation；tools/check_tests.sh gf-moderation <本子任务点名的测试>
【停止条件】需要把真实的违禁词写进仓库；需要人工复核流程（文档里还是待定）
~~~
