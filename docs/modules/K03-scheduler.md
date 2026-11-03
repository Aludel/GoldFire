# K03 时间与调度（scheduler）

> **层级**：内核 L4 · **代码**：`server/kernel/scheduler` · **首次阶段**：1 · **主文档**：第 10、19.4 节

## 1. 职责

- **星球时间**：由服务器时间和星球创建时间算出星球上的日、季节、年
- **星球实例**：有人进入时加载到内存，所有人离开一段时间后存回并卸载
- **心跳**：按固定间隔运行已登记的系统（规则）
- **补算**：实例加载时，按"上次更新到现在"的时间差，一次性算出离线期间的变化
- **种子随机数**：给每个系统、每次心跳派生确定的随机种子
- **心跳预算**：每颗星球每次心跳的计算时间有上限

## 2. 不负责

- 具体规则（由玩法包和 K05 以"系统"的形式提供）
- 数据存储（K02）

## 3. 依赖

K00、K02

## 4. 对外接口

```rust
pub trait System: Send + Sync {
    fn name(&self) -> &str;
    fn owner(&self) -> Owner;                 // 内核或某个玩法包
    fn interval(&self) -> Duration;           // 多久运行一次
    fn offline(&self) -> OfflinePolicy;       // 没人看时：Pause（暂停）/ CatchUp（回来补算）
    fn run(&self, ctx: &mut TickCtx) -> Result<()>;
    fn catch_up(&self, ctx: &mut TickCtx, elapsed: Duration) -> Result<()> { Ok(()) }
}

pub struct TickCtx<'a> {
    pub planet: PlanetId,
    pub now: PlanetTime,
    pub rng: SeededRng,                       // 只能用这个随机数
    pub world: &'a dyn WorldRead,
    pub writer: ScopedWriter<'a>,             // 只能写本系统所属玩法包的组件
}

pub fn register_system(s: Arc<dyn System>);
pub trait PlanetClock { fn planet_time(&self, p: PlanetId) -> PlanetTime; }
```

- HTTP：`GET /planets/{id}/time`（星球时间和服务器当前时间，供客户端校准昼夜）

## 5. 数据

| 表 | 关键字段 |
|---|---|
| planet_runtime | planet_id, last_tick_at, tick_count, loaded |

## 6. 事件

- 发布：`PlanetLoaded`、`PlanetUnloaded`、`DayStarted`、`NightStarted`、`SeasonChanged`、`SystemSuspended`（超预算被暂停）
- 订阅：`PlayerJoined` / `PlayerLeft`（K02）

## 7. 配置

- `time.json`：星球一天 = 现实 2 小时、一个季节 = 现实 1 周
- `scheduler.json`：心跳间隔、每颗星球每次心跳的预算、卸载前等待时间

## 8. 关键规则

- **补算按时间差计算，不逐秒模拟**。例如金矿每小时转换 1000 点，离线 10 小时就一次性算 1 万点
- 系统运行顺序固定（按登记顺序和声明的先后关系），保证结果可复现
- 随机种子 = 哈希（星球种子, 心跳编号, 系统名）
- 某个系统连续超出预算 → 暂停它并通知星球主人，不影响其他系统
- 星球卸载前保存状态

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 星球时间 | 昼夜、季节和配置一致 |
| 1 | 实例加载 / 卸载、简单心跳 | 有人在时运行，没人时卸载 |
| 2 | 补算 | 离线 10 小时后上线，结果与连续运行一致（对可累加的规则） |
| 5 | 预算与暂停 | 死循环的系统被暂停，其他系统照常 |

## 10. 测试要点

- 同一个种子、同样的输入，两次运行结果完全一样
- 补算与连续运行的结果对比
- 假时钟下的昼夜切换

## 11. 待定问题

- 心跳间隔是统一的，还是每个系统可以不同（目前设计为每个系统自己声明间隔）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K03-x"（x 换成子任务编号）。

~~~text
【模块】K03 时间与调度（包 gf-scheduler，路径 server/kernel/scheduler，层级 L4）
【先读】docs/modules/K03-scheduler.md；K02-world.md 第 4、6 节；K00-core.md 第 4 节（Clock、rng）；主文档第 10、19.4 节
【范围】server/kernel/scheduler/、server/migrations/（文件名带 k03）、shared/config/time.json、shared/config/scheduler.json、fixtures/scheduler/、server/app/（只限接入本模块）
【允许的依赖】模块：gf-core、gf-world。库：serde、serde_json、sqlx、tokio、async-trait、tracing、axum、rand_core、rand_chacha。测试：proptest
【规则补充】
  - time.json：day_seconds 7200、day_fraction_bp 5000（一天里白天占的比例）、season_seconds 604800、seasons_per_year 4
  - scheduler.json：tick_ms 1000、unload_delay_s 300、tick_budget_ms 50、suspend_after 3
  - 星球时间：elapsed = (now − 星球创建时间) 的整秒数；第几天 = elapsed / day_seconds + 1；一天里的秒数 = elapsed % day_seconds；白天 = 一天里的秒数 < day_seconds × day_fraction_bp / 10000；季节 = (elapsed / season_seconds) % seasons_per_year + 1；第几年 = elapsed / (season_seconds × seasons_per_year) + 1
  - 系统的随机数：gf_core::rng::from_seed(seed([星球种子 8 字节小端, 心跳编号 8 字节小端, 系统名 UTF-8]))
  - 心跳预算用 std::time::Instant 计时（这是性能计时，不是游戏时间，可以不经过 Clock）
【子任务】
K03-1 星球时间（阶段 1）
  - HTTP：GET /planets/{id}/time 返回星球时间和服务器当前时间（客户端用它校准昼夜）
  - 测试：planet_time_examples（elapsed = 0 → 第 1 天、白天、第 1 季、第 1 年；3600 → 夜晚；7200 → 第 2 天、白天；604800 → 第 2 季；2419200 → 第 2 年、第 1 季）、planet_time_endpoint
K03-2 实例加载、卸载与心跳（阶段 1）
  - 收到 PlayerJoined 加载星球；最后一个人离开 unload_delay_s 秒后保存并卸载；心跳按登记顺序运行到期的系统；ScopedWriter 只能写系统所属玩法包的组件
  - 测试：loads_on_join_unloads_after_delay、system_runs_at_its_interval、systems_run_in_registration_order、writer_limited_to_owner_components、day_and_night_events_while_loaded（FakeClock 前进时依次发出 NightStarted、DayStarted）
K03-3 种子随机数（阶段 1）
  - 测试：same_inputs_same_sequence、different_system_names_differ、different_ticks_differ、system_rng_matches_core_derivation
K03-4 补算（阶段 2）
  - 星球加载时，对 CatchUp 系统调用一次 catch_up(elapsed)，elapsed = 现在 − 上次心跳；Pause 系统不补算
  - 测试：catch_up_equals_continuous_for_additive_system（测试系统每小时加 1000：连续运行 10 小时和离线 10 小时后补算，结果都是 10000）、pause_policy_skips_offline_time
K03-5 预算与暂停（阶段 5）
  - 系统一次运行超过 tick_budget_ms 记一次超额；连续 suspend_after 次 → 暂停这个系统、发出 SystemSuspended；其他系统照常
  - 测试：overrunning_system_suspended_others_continue、suspended_system_not_run
【验收（每个子任务）】tools/check_server.sh gf-scheduler；tools/check_tests.sh gf-scheduler <本子任务点名的测试>
【停止条件】需要逐秒模拟离线时间；系统需要访问别的星球
~~~
