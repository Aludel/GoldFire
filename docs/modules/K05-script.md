# K05 规则与脚本运行时（script）

> **层级**：内核 L6 · **代码**：`server/kernel/script` · **首次阶段**：5 · **主文档**：第 16.5、19.4 节

## 1. 职责

- **声明式规则引擎**：执行"当……如果……就……"格式的规则
- **Luau 沙盒**：用 mlua 嵌入 Luau，执行复杂的玩法脚本
- **配额**：每次运行的指令数、内存、时间上限
- 把规则和脚本包装成 K03 的 `System`，按心跳运行
- 错误隔离与报告：一个玩法出错不影响其他玩法

## 2. 不负责

- 玩法包的安装、审批、静态检查 → K06
- 官方玩法包（用 Rust 原生代码，直接实现 `System`）

## 3. 依赖

K00、K02、K03、K04

## 4. 对外接口

```rust
pub fn load_rules(pack: PackId, rules: Vec<RuleDef>) -> Result<Vec<Arc<dyn System>>>;
pub fn load_script(pack: PackId, source: &str, limits: Limits) -> Result<Vec<Arc<dyn System>>>;
pub struct Limits { pub instructions: u64, pub memory_bytes: usize, pub wall_time: Duration }
```

**声明式规则格式**：

```json
{
  "当": "每天夜里",
  "如果": { "随机": 0.3 },
  "就": [
    { "能力": "播放特效", "参数": { "形态": "陨石", "元素": "火", "范围": "全星球" } },
    { "能力": "生成实体", "参数": { "素材": "流星碎片", "数量": 5, "位置": "随机" } }
  ]
}
```

- 触发器（当）：时间（每天夜里、每季开始……）、事件（有人进入区域、某种实体被拾取……）、玩家动作
- 条件（如果）：随机概率、组件值比较、时间范围、数量统计
- 动作（就）：调用能力；写本玩法包的组件

**Luau 中可用的 `world` 接口**：

| 函数 | 作用 |
|---|---|
| `world.time()` | 星球时间（是否夜晚、季节……） |
| `world.random()` | 内核提供的种子随机数 |
| `world.get(e, comp)` | 读组件（受可见性限制） |
| `world.set(e, comp, v)` | 写组件（只能写本玩法包的组件） |
| `world.query_radius(pos, r)` | 查询半径内的实体 |
| `world.call(cap, input)` | 以本玩法包的身份调用能力（走 K04 流水线） |
| `world.give(player, item)` | 给玩家本玩法包的物品 |
| `world.log(msg)` | 写日志（主人可见） |

## 5. 数据

无（规则和脚本的内容由 K06 保存）。

## 6. 事件

- 发布：`ScriptError`、`PackSuspended`（连续超配额被暂停）

## 7. 配置

`script.json`：默认配额、连续超额几次后暂停

## 8. 关键规则

- 沙盒里**不能访问网络、文件、其他星球**；移除 Luau 的 `os`、`io` 等库
- 随机数和时间只能来自内核，保证回放一致
- 超出配额 → 本次运行中止；连续超额 → 暂停这个玩法包并通知主人
- 脚本以"玩法包身份"调用能力，权限不超过安装时批准的范围
- 每条规则声明"没人看时"怎么办：暂停，或回来时补算

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 5 | 声明式规则引擎 | 流星雨示例能运行 |
| 5 | Luau 沙盒与 `world` 接口 | 钓鱼示例能运行 |
| 5 | 配额与暂停 | 死循环、超大内存的脚本被拦下，其他玩法不受影响 |
| 5 | 错误报告 | 主人能看到哪条规则、哪一行出错 |

## 10. 测试要点

- 恶意脚本：死循环、申请巨量内存、尝试加载 `os` 库、尝试写别的玩法包的组件
- 同一个种子回放，结果完全一致

## 11. 待定问题

- 以后是否支持 WebAssembly 作为高性能玩法的运行方式

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K05-x"（x 换成子任务编号）。

~~~text
【模块】K05 规则与脚本运行时（包 gf-script，路径 server/kernel/script，层级 L6）
【先读】docs/modules/K05-script.md；K03-scheduler.md 第 4 节（System、TickCtx）；K04-capability.md 第 4、8 节；主文档第 16.5、19.4 节
【范围】server/kernel/script/、shared/config/script.json、fixtures/script/
【允许的依赖】模块：gf-core、gf-world、gf-scheduler、gf-capability。库：serde、serde_json、tokio、async-trait、tracing、jsonschema、schemars、mlua（luau、async、serialize 功能）。测试：proptest
【规则补充】
  - 声明式规则只允许下面这些写法，其他一律拒绝（rule.schema.json 由 Rust 类型生成）：
      当：每天夜里、每天白天、每季开始、每隔N秒、实体被拾取、进入区域、玩家动作
      如果（可以省略，多个条件同时满足）：随机（0–1 的概率，换算成 bp 用 chance_bp）、组件比较、时间范围、数量
      就：能力（名字必须在调用者的能力目录里）、写组件（只能写本玩法包的组件）
  - 规则和脚本里的随机数只来自 TickCtx.rng；时间只来自 TickCtx.now
  - Luau 只开放 world 接口（第 4 节的表）；os、io、require、loadstring、debug、getfenv、setfenv 都不可用
  - script.json：instructions 1000000、memory_bytes 16777216、wall_time_ms 50、suspend_after 3
【子任务】
K05-1 声明式规则引擎（阶段 5）
  - fixtures/script/meteor_rule.json 就是第 4 节的流星雨规则
  - 测试：meteor_rule_example_runs（夜里、固定种子、随机值 < 0.3 时：调用"播放特效"1 次、"生成实体"1 次且数量为 5；随机值 ≥ 0.3 时都不调用）、invalid_rule_rejected_by_schema、unknown_capability_rejected、rule_actions_go_through_capability_pipeline
K05-2 Luau 沙盒与 world 接口（阶段 5）
  - fixtures/script/fishing.luau：抛竿时按种子随机决定钓到什么鱼，写进本玩法包的 fishing.图鉴 组件（玩法包的组件和能力都以 PackId 加点开头）
  - 测试：fishing_example_deterministic（同一种子跑两次，图鉴完全相同）、os_io_require_unavailable、cannot_write_other_pack_component、world_call_uses_pack_identity
K05-3 配额与暂停（阶段 5）
  - 测试：infinite_loop_aborted、memory_bomb_aborted、repeated_overrun_suspends_pack_others_continue（连续 suspend_after 次超额 → 发出 PackSuspended，另一个玩法包照常运行）
K05-4 错误报告（阶段 5）
  - 测试：script_error_reports_pack_rule_and_line（报告里有玩法包名、规则或脚本文件名、行号）、error_in_one_pack_does_not_stop_others
【验收（每个子任务）】tools/check_server.sh gf-script；tools/check_tests.sh gf-script <本子任务点名的测试>
【停止条件】规则需要的触发器、条件或动作不在上面的列表里；需要让脚本访问网络、文件或别的星球
~~~
