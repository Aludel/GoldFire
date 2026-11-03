# K08 计量（metering）

> **层级**：内核 L1 · **代码**：`server/kernel/metering` · **首次阶段**：2（游戏内）；7（网关、本地日志）· **主文档**：第 8.2 节

## 1. 职责

- 记录每一次 AI 用量：来源、模型、输入 / 输出 / 缓存读取的 token 数、发起人、是否为星球 AI 自主运转
- 计算有效 token 和点数
- 三种来源接入：游戏内（K10 上报）、GoldFire 网关、本地日志上传

## 2. 不负责

- 递减和封顶 → K07
- 排名 → K09

## 3. 依赖

K00

## 4. 对外接口

```rust
pub struct UsageReport {
    pub user: UserId,              // 谁付费、谁得点数
    pub source: Source,            // Game / Gateway / LocalLog
    pub model: ModelId,
    pub input: u64, pub output: u64, pub cache_read: u64,
    pub autonomous: bool,          // 是否为星球 AI 自主运转
    pub task: TaskKind,
}
pub trait Metering: Send + Sync {
    fn record(&self, r: UsageReport) -> Result<MeterRecordId>;
}
```

- 网关：OpenAI / Anthropic 兼容的代理接口，转发请求并计数
- HTTP：上传本地用量日志

## 5. 数据

| 表 | 关键字段 |
|---|---|
| meter_records | id, user_id, source, model, input, output, cache_read, effective, points, autonomous, task, at |

## 6. 事件

- 发布：`MeterRecorded`（包含点数和 autonomous 标记）

## 7. 配置

`metering.json`：token 类型权重（输出 ×1、输入 ×0.2、缓存读取 ×0.01）、来源倍率（游戏内 ×1.0、网关 ×0.4、本地日志 ×0.1）、模型系数表、本地日志每日上限

## 8. 关键规则

- **有效 token = 输出 × 1 + 输入 × 0.2 + 缓存读取 × 0.01**
- **点数 = 有效 token × 来源倍率 × 模型系数**
- **只存用量，不存对话内容**；网关转发的内容不写入磁盘
- 本地日志来源单独设每日上限
- 不让玩家把 API Key 交给游戏

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 2 | 游戏内计量 | K10 的每次调用都有记录；点数计算和主文档例子一致 |
| 7 | 网关 | Claude Code、Codex 等通过网关使用时被正确计数，内容不落盘 |
| 7 | 本地日志上传 | 上限生效 |

## 10. 测试要点

- 主文档 8.5 的例子作为自动化测试
- 网关压力测试：转发不能明显增加延迟

## 11. 待定问题

- 各外部工具的服务条款是否允许接入网关（上线前逐一确认）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K08-x"（x 换成子任务编号）。

~~~text
【模块】K08 计量（包 gf-metering，路径 server/kernel/metering，层级 L1）
【先读】docs/modules/K08-metering.md；主文档第 8.2、8.5 节
【范围】server/kernel/metering/、server/migrations/（文件名带 k08）、shared/config/metering.json、fixtures/metering/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、schemars；网关（K08-2）另加 reqwest、bytes、futures。测试：proptest、wiremock
【规则补充】
  - metering.json：output_bp 10000、input_bp 2000、cache_read_bp 100；source_bp：game 10000、gateway 4000、local_log 1000；model_bp：每个模型的系数（没列出的模型用 default_model_bp 10000）；local_log_daily_cap 20000
  - 有效 token = (输出 × output_bp + 输入 × input_bp + 缓存读取 × cache_read_bp) / 10000（向下取整）
  - 点数 = 有效 token × 来源 bp × 模型 bp / 100000000（u128 计算，向下取整）
  - 记录里只有数字和标记，没有任何对话内容字段
【子任务】
K08-1 游戏内计量（阶段 2）
  - 测试：points_examples（游戏内：输出 10000、输入 20000 → 有效 14000、点数 14000；网关：输出 30000、输入 100000、缓存读取 5000000 → 有效 100000、点数 40000；网关：输出 1000000、输入 5000000、缓存读取 100000000 → 有效 3000000、点数 1200000；游戏内：输出 200000、输入 500000 → 点数 300000）、model_coefficient_applied（模型系数 3000 bp 时点数乘 0.3）、autonomous_flag_kept_in_event、record_has_no_content_fields（MeterRecord 和 meter_records 表都没有文本字段）
  【验收】另加 tools/mutants.sh gf-metering src/points.rs
K08-2 GoldFire 网关（阶段 7）
  - OpenAI 兼容和 Anthropic 兼容的转发接口，从响应（含流式响应的最后一块）里读出用量；请求和响应正文只在内存里转发，不写磁盘、不写日志
  - 测试（wiremock 模拟上游）：openai_usage_parsed、anthropic_usage_parsed、streaming_usage_parsed、bodies_never_persisted、upstream_error_passed_through
K08-3 本地日志上传（阶段 7）
  - 先向我要真实的 Claude Code 用量日志样本（去掉内容后放进 fixtures/metering/local_logs/），没有样本不开始
  - 测试：local_log_points_use_local_multiplier、local_log_daily_cap_applied、duplicate_upload_ignored
【验收（每个子任务）】tools/check_server.sh gf-metering；tools/check_tests.sh gf-metering <本子任务点名的测试>
【停止条件】需要保存对话内容；外部工具的服务条款不清楚（K08-2、K08-3 开始前先问我）
~~~
