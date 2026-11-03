# X02 AI 评测工具（eval）

> **层级**：共享（开发工具）· **代码**：`tools/eval` · **首次阶段**：1 · **主文档**：第 27 节

## 1. 职责

- 管理每个 AI 任务的**测试集**
- 对指定的模型 + 提示词版本跑测试集，**自动打分**
- 生成对比报告，决定 `models.json` 里每一档用哪个模型
- 接入 CI：改提示词或换模型时自动跑

## 2. 测试集

| 测试集 | 内容 | 评分方式 |
|---|---|---|
| 造物 100 条 | 各种风格的造物描述 | Schema 合格率、越界率、评审模型打分 |
| 功能标签 | 带正确答案的造物描述（包括故意误导的名字） | 准确率 |
| 素材检索 | 描述 + 应该检索到的素材 | 前 20 名命中率 |
| 意图分类 | 玩家的话 + 正确类别（操作 / 规则 / 居民） | 准确率 |
| 声明式规则生成 | 玩法描述 + 期望的规则 | 能否运行、是否符合描述 |
| 人物分级 | 30 章人工标注的小说片段 | 主角识别的准确率和召回率 |
| 人物反应 | 档案卡 + 事件 | 格式合格率、"像不像这个人物"评分 |
| 内容安全 | 正常内容和不当内容 | 拦截率、误伤率 |

## 3. 使用方式

```
eval run --task creation --model <模型名> --prompt-version 3
eval compare --task creation --models A,B,C
eval report --all
```

## 4. 关键规则

- **分数不降才能上线**：换模型或改提示词，必须先跑对应的测试集
- 测试集里的内容要经过 K13 检查；小说片段只用公有领域作品
- 开源社区可以通过 PR 贡献测试用例
- 带正确答案的测试集，每条都有 `reviewed` 字段：AI 起草的为 false，人工确认后改成 true；评测只用确认过的条目
- `--fake` 模式用固定回答跑一遍，不花钱，用来测试评测工具本身

## 5. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 评测框架 + 造物 100 条 | 能比较两个模型的合格率 |
| 1 | CI 接入 | 提示词改动的 PR 自动跑测试 |
| 1 | 功能标签、内容安全测试集 | K11、K13 有基准分数 |
| 2 | 素材检索测试集 | 语义检索上线前有基准分数 |
| 5 | 意图分类、规则生成、人物反应测试集 | 星球 AI 上线前有基准分数 |
| 6 | 人物分级测试集 | 剧本导入上线前有基准分数 |

## 6. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 X02-x"（x 换成子任务编号）。

~~~text
【模块】X02 AI 评测工具（包 gf-eval，路径 tools/eval，顶层）
【先读】docs/modules/X02-eval.md；K10-ai-gateway.md 第 4 节和【规则补充】（FakeGateway）；K11-creation.md【规则补充】；主文档第 26、27 节
【范围】tools/eval/、tools/eval.sh、.github/workflows/eval.yml、fixtures/ai/eval/
【允许的依赖】模块：gf-core、gf-ai-gateway、gf-creation（校验函数）、gf-moderation、gf-asset、gf-script、gf-agent、gf-pack-story（各评测任务用到的校验和执行函数）。库：clap、serde、serde_json、tokio、anyhow。测试：proptest、tempfile
【规则补充】
  - 命令：eval run --task <任务> --model <模型> --prompt-version <N> [--fake]；eval compare --task <任务> --models A,B,C；eval report --all
  - 测试集：tools/eval/datasets/<任务>.jsonl，一行一条；基准分：tools/eval/baselines/<任务>.json；报告写到 tools/eval/reports/（不进仓库）
  - --fake：AI 用 FakeGateway::from_dir(fixtures/ai/eval)，不花钱、结果固定，用来测试评测工具本身
  - 造物评测看模型第一次的原始回答（不重试）：用 K11 导出的校验函数检查 Schema 和越界，区域局部地图来自 tools/eval/datasets/region_map.json
  - 带正确答案的测试集（功能标签、意图、人物分级、内容安全等），每条都有 reviewed 字段：AI 起草的条目为 false，我人工确认后改成 true；评测只用 reviewed = true 的条目，数量少于 min_items 时报错
  - 分数低于基准分时，run 返回非零退出码
  - 任务名：creation、function_tags、moderation、retrieval、intent、rules、reactions、tiering
【子任务】
X02-1 评测框架与造物测试集（阶段 1）
  - tools/eval/datasets/creation.jsonl：100 条不同风格的造物描述
  - 测试：fake_run_produces_report、scores_computed_from_fixture（10 条里 9 条合格 → 合格率 0.9）、compare_lists_models_side_by_side、below_baseline_exits_nonzero
  【验收】tools/check_server.sh gf-eval；tools/eval.sh creation --fake
X02-2 接入 CI（阶段 1）
  - .github/workflows/eval.yml：所有 PR 跑全部任务的 --fake；改了 shared/prompts/** 或 shared/config/models.json 的 PR，用仓库密钥跑对应任务的真实评测（只在主仓库，不在 fork 里运行）
  【验收】actionlint .github/workflows/eval.yml
  【人工验收】开一个只改提示词的 PR，确认评测任务自动运行
X02-3 功能标签和内容安全测试集（阶段 1）
  - 测试：unreviewed_items_excluded、too_few_reviewed_items_is_error、function_tags_accuracy_computed、moderation_block_and_false_positive_rates_computed
X02-4 素材检索测试集（阶段 2）
  - 测试：retrieval_top20_hit_rate_computed
X02-5 意图、规则、人物反应测试集（阶段 5）
  - 测试：intent_accuracy_computed、rule_runs_and_matches_expectation_scored、reaction_format_rate_and_review_score_computed
X02-6 人物分级测试集（阶段 6）
  - 30 章人工标注的公有领域小说片段
  - 测试：tiering_precision_recall_computed
【验收（每个子任务）】tools/check_server.sh gf-eval；tools/check_tests.sh gf-eval <本子任务点名的测试>；tools/eval.sh <本子任务的任务名> --fake
【停止条件】需要把 API Key 写进仓库；带正确答案的条目没人确认就要用
~~~
