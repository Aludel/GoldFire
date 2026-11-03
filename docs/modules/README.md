# GoldFire 模块开发文档

这个目录里，每个模块都有一份独立的开发文档。整体设计、玩法规则和架构原则见主文档 `GoldFire设计文档.md`。

## 怎么用这些文档

- **开始开发一个模块前**：先读它的文档，再读它依赖的模块的"对外接口"一节
- **改接口前**：先改文档，再改代码（主文档开发规范第 13 条）
- **遇到文档没写清楚的地方**：在文档末尾"待定问题"里补一条，不要自己猜

## 统一模板

每份文档都按同样的结构写：

| 小节 | 内容 |
|---|---|
| 头部信息 | 层级、代码位置、首次开发阶段、相关的主文档章节 |
| 1. 职责 | 这个模块负责什么 |
| 2. 不负责 | 边界：哪些事归别的模块 |
| 3. 依赖 | 依赖哪些模块（只能向下依赖） |
| 4. 对外接口 | 提供给其他模块的 Rust 接口；HTTP / WebSocket 接口 |
| 5. 数据 | 拥有的数据表或组件 |
| 6. 事件 | 发布和订阅的事件 |
| 7. 配置 | 读取哪些配置表 |
| 8. 关键规则 | 必须遵守的规则和算法要点 |
| 9. 开发步骤 | 按阶段拆分的任务和完成标准 |
| 10. 测试要点 | 必须覆盖的测试 |
| 11. 待定问题 | 还没决定的地方 |
| 12. 开发提示词 | 交给 AI 编程助手的任务说明：子任务、点名的测试、验收命令（用法见下文） |

客户端（C）和共享（X）模块的文档小节少一些，最后一节同样是"开发提示词"。

接口里的 Rust 代码就是开发时要实现的公开接口：类型名、trait 名、方法名、参数和返回值的含义都按文档来。只允许三种机械调整：访问数据库、网络或 AI 的方法加上 `async`；需要参与调用方事务的写方法加上 `tx: &mut Tx` 参数；全局注册函数（例如 `register(def)`）写成某个结构体的方法。做了这类调整，要在同一个 PR 里同步更新文档第 4 节。其他修改，先改文档（经确认），再改代码。

## 依赖层级（服务器）

只能依赖比自己层级低的模块；同层之间不直接依赖。下层需要上层的功能时，由下层定义 trait，上层或 `server/app` 里的适配器实现。

```
L8  星球层   K16 星球 AI · P01 供给与环境 · P02 战斗 · P03 BOSS · P04 剧本
L7           K06 玩法包管理
L6           K05 规则与脚本运行时 · K11 创作管线 · K14 演出 · K15 智能体运行时
L5           K04 能力接口
L4           K03 时间与调度 · K09 星系 · K17 版本历史
L3           K02 世界 · K18 额度与支付
L2           K01 账号 · K07 经济 · K10 AI 网关 · K12 素材
L1           K08 计量 · K13 内容安全
L0           K00 核心基础
```

客户端（C01–C10）只通过协议和 Schema 与服务器交互；共享层（X01、X02）不含业务逻辑。

层级写在仓库根目录的 `layers.toml` 里，由 `tools/check_layers.sh` 自动检查（开发依赖也算在内）。补充规定：

- 共享地形库 gf-terrain（X01）算作 L0，任何服务器模块都可以依赖它
- gf-app、gf-client-ext 和 `tools/` 里的工具在最顶层：可以依赖任何包，但任何包都不能依赖它们
- 同在 L8 的官方玩法包之间，只允许这几条依赖（同时写在玩法包的 manifest 里）：P02→P01；P03→P01、P02；P04→P02（订阅战斗事件）

## 文档列表

**内核**

| 编号 | 文档 | 首次阶段 |
|---|---|---|
| K00 | [核心基础](K00-core.md) | 1 |
| K01 | [账号](K01-account.md) | 2 |
| K02 | [世界](K02-world.md) | 1 |
| K03 | [时间与调度](K03-scheduler.md) | 1 |
| K04 | [能力接口](K04-capability.md) | 1 |
| K05 | [规则与脚本运行时](K05-script.md) | 5 |
| K06 | [玩法包管理](K06-packs.md) | 5 |
| K07 | [经济](K07-economy.md) | 2 |
| K08 | [计量](K08-metering.md) | 1 |
| K09 | [星系](K09-galaxy.md) | 2 |
| K10 | [AI 网关](K10-ai-gateway.md) | 1 |
| K11 | [创作管线](K11-creation.md) | 1 |
| K12 | [素材](K12-asset.md) | 1 |
| K13 | [内容安全](K13-moderation.md) | 1 |
| K14 | [演出](K14-presentation.md) | 3 |
| K15 | [智能体运行时](K15-agent.md) | 5 |
| K16 | [星球 AI](K16-planet-mind.md) | 1 |
| K17 | [版本历史](K17-history.md) | 1 |
| K18 | [额度与支付](K18-billing.md) | 2 |

**官方玩法包**

| 编号 | 文档 | 首次阶段 |
|---|---|---|
| P01 | [供给与环境](P01-supply.md) | 2 |
| P02 | [战斗](P02-combat.md) | 3 |
| P03 | [BOSS](P03-boss.md) | 4 |
| P04 | [剧本](P04-story.md) | 6 |

**客户端**

| 编号 | 文档 | 首次阶段 |
|---|---|---|
| C01 | [网络与同步](C01-net.md) | 1 |
| C02 | [素材缓存](C02-asset-cache.md) | 1 |
| C03 | [星球渲染](C03-render.md) | 1 |
| C04 | [地形与布局显示](C04-terrain.md) | 1 |
| C05 | [角色与居民](C05-characters.md) | 2 |
| C06 | [特效与演出](C06-vfx.md) | 1 |
| C07 | [界面与交互](C07-ui.md) | 1 |
| C08 | [本地工坊](C08-workshop.md) | 2 |
| C09 | [剧情观看与神迹](C09-story-view.md) | 6 |
| C10 | [星球助手](C10-assistant.md) | 1 |

**共享**

| 编号 | 文档 | 首次阶段 |
|---|---|---|
| X01 | [共享层](X01-shared.md) | 1 |
| X02 | [AI 评测工具](X02-eval.md) | 1 |

## 开发提示词怎么用

每份模块文档的最后一节是"开发提示词"：写给 AI 编程助手（Claude Code、Codex 等）的任务说明。目标是**不管谁来做、用哪个 AI 做，结果都能被机器判定，并且收敛到同一个实现**。

### 使用步骤

1. 第一次使用前，把设计文档放进仓库：`docs/GoldFire设计文档.md` 和 `docs/modules/`
2. 选一个子任务（例如 K07-2），从 main 新建一个分支，例如 `k07-2-daily-settlement`
3. 开一个**新的** AI 会话，依次发送：
   - 下面的「通用开头」
   - 模块文档"开发提示词"一节里的整段提示词
   - 最后一句：`本次只做子任务 K07-2`
4. AI 先输出计划；你检查后回复"确认"，它才开始写代码
5. AI 做完会给出「完成报告」。你自己再跑一遍验收命令，全部通过再提 PR

规则：
- 一个会话只做一个子任务；同一模块的子任务按编号顺序做
- 子任务标了阶段，只做当前阶段的；跨模块的顺序见下面的"第 1 阶段推荐顺序"
- 依赖的模块还没做时，AI 只能在那个模块里补上它文档第 4 节写明的 trait 和数据结构（不写实现）

### 为什么能收敛到一个结果

| 机制 | 作用 |
|---|---|
| 接口冻结 | 模块文档第 4 节就是要实现的公开接口，AI 不能自己改名字和含义 |
| 全局约定 | ID、时间、数值、随机数、坐标、事务等全部统一（见下表），不同的人和 AI 写的代码能拼在一起 |
| 点名的测试和测试数值 | 每个子任务列出必须存在的测试名和具体数值；期望值来自文档，不来自代码的运行结果 |
| 固定范围和依赖 | 只能改列出的路径，只能用列出的库 |
| 统一验收命令 | 所有子任务用同一组脚本判定"做完没有"，结果只有通过和不通过 |
| 单元测试不调真模型 | 用 FakeGateway 和 `fixtures/ai/` 里固定的回答，每次结果一样；模型的真实效果另外用 X02 评测 |
| 变异测试 | 关键算法跑 `tools/mutants.sh`：故意把代码改坏，测试必须能发现，防止"测试写了但什么都没测" |
| 完成报告 | 固定格式，方便你核对，也方便下一个会话接着做 |

### 三种验收

| 标记 | 谁来做 | 怎么判定 |
|---|---|---|
| 【验收】 | AI 自己跑，你再跑一遍 | 命令全部通过 |
| 【效果验收】 | 你来跑（要调用真实模型，需要 API Key，会花钱） | X02 评测分数达到给定的线；AI 只用 `--fake` 模式确认评测能跑通 |
| 【人工验收】 | 你来看 | 按给出的步骤操作，截图或录屏放进 PR |

### 全局约定

| 项目 | 约定 |
|---|---|
| 编号 | ULID；序列化成 26 位字符串。跨模块共用的编号（UserId、PlanetId、EntityId、RegionId、PackId）定义在 K00，其他编号定义在拥有它的模块 |
| 时间 | 只通过 K00 的 `Clock` 取当前时间，测试用 `FakeClock`；时间戳是 UTC 毫秒（`Timestamp`）；"每天"按 UTC 0 点划分 |
| 数值 | 资源、分数、token 数用整数（Rust 里 u64，数据库里 BIGINT，转换时检查范围）；比例用基点 bp（10000 = 100%）；中间计算用 u128；只在最后一步向下取整；数值都从 `shared/config` 读，代码里不写数值常量 |
| 随机数 | 影响游戏结果的随机只用 K00 的 `gf_core::rng`（ChaCha8，种子由 SHA-256 派生）；不用 `thread_rng`、标准库的哈希器、`rand` 的分布函数 |
| 坐标 | 经纬度（度，f64），北纬、东经为正；距离用大圆距离，星球半径取 h3o 的地球平均半径（约 6371 km）；客户端渲染用浮动原点（C03），不把这么大的坐标直接交给 Godot 的单精度浮点数 |
| 跨平台计算 | 服务器和客户端必须一致的计算（地形）只用 `libm`，不用平台数学函数，不用多线程合并结果 |
| 指纹 | `sha256:` 加 64 位小写十六进制 |
| 错误 | K00 的 `ErrorKind`；HTTP 状态码按 K00 的映射表 |
| 异步 | 会访问数据库、网络或 AI 的 trait 方法是 `async`（dyn trait 用 `async-trait`）；纯计算保持同步 |
| 事务 | 能力流水线的计费、执行、记录三步在同一个数据库事务里。K02 的 `WorldWrite`、K07 的 `spend` / `grant` / `loot`、K17 的 `record` 第一个参数是 `tx: &mut Tx`（K00 提供）；事务里发的事件先写进 outbox，提交后才送出 |
| HTTP | 每个 crate 导出 `pub fn router(state) -> axum::Router`，由 gf-app 合并；JSON 字段名用 snake_case |
| 数据库 | PostgreSQL；迁移文件放 `server/migrations/`，文件名 `<时间戳>_<模块>_<内容>.sql`（例如 `20261102000000_k02_planets.sql`）；每个模块只读写自己的表；查询用 sqlx 的 `query` / `query_as`，不用 `query!` 宏（那样编译时就要连数据库） |
| 数据库测试 | 用 `#[sqlx::test]`，每个测试一个全新的数据库；先运行 `tools/dev_db.sh` 拿到 `DATABASE_URL` |
| 测试数据 | 服务器的放仓库根目录 `fixtures/<模块或主题>/`；客户端的放 `client/tests/fixtures/<目录>/`（读 client/ 之外的文件用 `ProjectSettings.globalize_path("res://")` 拼路径）；AI 的固定回答放 `fixtures/ai/<任务>/`，手写，或者录下真实模型的回答再人工检查 |
| 金标准文件 | 只用于"确定性"测试（同样的输入永远得到同样的输出）：可以用代码生成一次，在 PR 里声明并由人确认。"正确性"测试的期望值不能这样生成，必须来自文档 |
| 命名 | Rust 测试函数用英文 snake_case；GDScript 测试函数以 `test_` 开头；同一个模块里测试名不重复。能力名：内核内置能力用中文名（例如 `放置`）；玩法包的能力和组件以 PackId 加点开头（例如 `war.攻击`、`fishing.图鉴`） |
| GDScript | 一律写类型（`var hp: int`）；对外的类用 `class_name`；状态变化用信号 |
| 日志 | `tracing`；不记录密钥和玩家的私密内容 |
| 密钥 | 只从环境变量读取；不进代码、测试、fixtures 和提交记录 |

### 包名与路径

| 模块 | 包名 | 路径 | 层级 |
|---|---|---|---|
| K00 | gf-core | `server/kernel/core` | L0 |
| X01 地形库 | gf-terrain | `shared/terrain` | L0 |
| K08 | gf-metering | `server/kernel/metering` | L1 |
| K13 | gf-moderation | `server/kernel/moderation` | L1 |
| K01 | gf-account | `server/kernel/account` | L2 |
| K07 | gf-economy | `server/kernel/economy` | L2 |
| K10 | gf-ai-gateway | `server/kernel/ai_gateway` | L2 |
| K12 | gf-asset | `server/kernel/asset` | L2 |
| K02 | gf-world | `server/kernel/world` | L3 |
| K18 | gf-billing | `server/kernel/billing` | L3 |
| K03 | gf-scheduler | `server/kernel/scheduler` | L4 |
| K09 | gf-galaxy | `server/kernel/galaxy` | L4 |
| K17 | gf-history | `server/kernel/history` | L4 |
| K04 | gf-capability | `server/kernel/capability` | L5 |
| K05 | gf-script | `server/kernel/script` | L6 |
| K11 | gf-creation | `server/kernel/creation` | L6 |
| K14 | gf-presentation | `server/kernel/presentation` | L6 |
| K15 | gf-agent | `server/kernel/agent` | L6 |
| K06 | gf-packs | `server/kernel/packs` | L7 |
| K16 | gf-planet-mind | `server/kernel/planet_mind` | L8 |
| P01 | gf-pack-supply | `server/packs/war/supply` | L8 |
| P02 | gf-pack-combat | `server/packs/war/combat` | L8 |
| P03 | gf-pack-boss | `server/packs/war/boss` | L8 |
| P04 | gf-pack-story | `server/packs/story` | L8 |
| 启动组装 | gf-app | `server/app` | 顶层 |
| 客户端扩展 | gf-client-ext | `client/ext`（godot-rust：地形、Schema 校验） | 顶层 |
| 工具 | gf-check-layers、gf-schema-gen、gf-eval | `tools/check_layers`、`tools/schema_gen`、`tools/eval` | 顶层 |

- Cargo 工作区在仓库根目录（`Cargo.toml`），共用的库版本写在 `[workspace.dependencies]`
- 客户端模块的代码在 `client/<目录>/`，测试在 `client/tests/<目录>/`（GUT）

### 统一脚本

| 脚本 | 作用 | 由谁创建 |
|---|---|---|
| `tools/dev_db.sh` | 用 Docker 启动本地 PostgreSQL，输出 `DATABASE_URL` | K00-2 |
| `tools/check_server.sh <包名>` | 格式检查 + clippy（警告算错误）+ 这个包的全部测试 | K00-2 |
| `tools/check_client.sh <目录>` | gdformat 检查 + gdlint + 这个目录的 GUT 测试（无界面） | K00-2 |
| `tools/check_tests.sh <包名或客户端目录> <测试名…>` | 检查点名的测试都存在 | K00-2 |
| `tools/check_layers.sh` | 依赖层级检查 | K00-2 |
| `tools/mutants.sh <包名> <文件>` | 对一个文件做变异测试，不允许有"存活"的变异 | K00-2 |
| `tools/check_all.sh` | 依次运行已有的全部检查（CI 用） | K00-2 |
| `tools/check_schemas.sh` | 重新生成 Schema 并和 `shared/schemas/` 比对；校验全部配置表和提示词 | X01-2 |
| `tools/e2e.sh <场景>` | 用假 AI 启动本地服务器，再用无界面客户端跑一个完整场景 | C01-2 |
| `tools/eval.sh <任务> [--fake]` | 跑 X02 评测 | X02-1 |

### 第 1 阶段推荐顺序

同一行里的子任务可以并行（不同分支），行与行之间按顺序：

| 顺序 | 子任务 | 对应主文档第 33 节 |
|---|---|---|
| 1 | K00-1 → K00-2 → K00-3 → K00-4 → K00-5 → K00-6 → K00-7 | 1.1 |
| 2 | X01-1、X01-2、K13-1、K01-1、K08-1 | 1.1、1.6、1.10 |
| 3 | K12-1 → K12-2；K02-1 → K02-2 → K02-3 → K02-4 → K02-5 | 1.2、1.8、1.9 |
| 4 | K03-1 → K03-2 → K03-3；K17-1 | 1.4 |
| 5 | K04-1 → K04-2 → K04-3 | 1.3、1.4 |
| 6 | K10-1 → K10-2 → K10-3 → K10-4；X01-3 | 1.10 |
| 7 | K11-1 → K11-2 → K11-3 | 1.10 |
| 8 | K16-1；X02-1 → X02-2 → X02-3 | 1.11、1.12 |
| 9（客户端，可以和 3–8 并行） | C01-1 → C01-2；C02-1；C03-1 → C03-2 → C03-3；C04-1 → C04-2；C06-1；C07-1；C10-1 | 1.5–1.9、1.11 |

### 通用开头

每次开新会话，先发这一段：

~~~text
你是 GoldFire 项目的开发者。GoldFire 是一款开源（AGPL-3.0）的多人 3D 星球建造游戏：服务器用 Rust（Cargo 工作区在仓库根目录），客户端用 Godot 4 + GDScript。

【工作方式】
1. 先读：AGENTS.md（如果已存在）、docs/modules/README.md、本模块文档，以及【先读】列出的文档。文档里没有定义或者前后矛盾的地方，停下来问我，不要自己猜。
2. 先输出计划：要新建或修改的文件；每个测试的名字和它验证什么；实现思路。等我回复"确认"再写代码。
3. 先写测试，再写实现。测试里的期望值只能来自文档或本提示词，不能先运行代码再把结果抄进测试。
4. 只做我指定的一个子任务。做完就停下，输出【完成报告】。

【硬性约束】
- 只修改【范围】里列出的路径。另外总是允许：在根目录 Cargo.toml 的 [workspace.dependencies] 里加入【允许的依赖】里的库；新增数据格式或配置表时在 tools/schema_gen/src/registry.rs 里登记；按下一条的规定更新本模块文档第 4 节。需要改别的地方，先问我。
- 公开接口（类型、trait、方法、HTTP 路由）按模块文档第 4 节实现。只允许三种机械调整：访问数据库、网络或 AI 的方法加 async；需要参与调用方事务的写方法加 tx: &mut Tx 参数；全局注册函数写成结构体的方法。做了这类调整，同时更新本模块文档第 4 节。其他修改只能在完成报告里提建议。
- 只使用【允许的依赖】里的模块和库。需要新的依赖，先问我。
- 依赖的模块还没实现时，只能在那个模块里补上它文档第 4 节写明的 trait 和数据结构（不写实现），并在完成报告里说明。
- 不得为了让测试通过而修改已有的测试或测试数据。
- 正式代码里不用 unwrap、expect、panic!、todo!、unimplemented!；错误用 K00 的 ErrorKind。测试代码可以用 unwrap 和 expect。
- 遵守 docs/modules/README.md 的【全局约定】。
- 单元测试不调用真实的 AI 模型，也不访问网络：AI 用 FakeGateway 和 fixtures/ai/ 里的固定回答。
- 不留 TODO、空实现和假数据；注释只写"为什么"，不写"做了什么"。
- 密钥只从环境变量读取，不写进代码、测试、fixtures 和提交记录。

【验收】
- 子任务列出的【验收】命令全部通过；另外 tools/check_layers.sh 必须通过；tools/check_schemas.sh 存在后（X01-2 完成后）也必须通过。
- 子任务里点名的测试都存在，名字完全一致（用 tools/check_tests.sh 检查）。
- 【效果验收】要调用真实模型，由我来跑；你只用 --fake 模式确认评测能跑通。
- 【人工验收】由我来做；你在完成报告里写清楚操作步骤和应该看到什么。
- 需要数据库的测试，先运行 tools/dev_db.sh，用它输出的 DATABASE_URL。

【完成报告】
1. 修改的文件（新增 / 修改 / 删除）
2. 新增的测试：名字 → 验证什么
3. 每条验收命令的结果（通过 / 失败；失败时贴出关键输出）
4. 对接口做的机械调整（如果有）
5. 发现的文档问题：不一致、遗漏、写得不清楚的地方
6. 需要我决定的问题
~~~
