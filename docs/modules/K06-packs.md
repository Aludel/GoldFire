# K06 玩法包管理（packs）

> **层级**：内核 L7 · **代码**：`server/kernel/packs` · **首次阶段**：5· **主文档**：第 16.4、16.5 节

## 1. 职责

- **玩法包格式**：定义 manifest 和目录结构
- **静态检查**：权限、组件所有权、不涉及金火、接口白名单、文字内容
- **安装、卸载、升级**：按指纹锁定版本
- **权限审批**：主人确认玩法包申请的权限
- **试玩**：请 K17 创建试玩副本，在副本里安装运行
- **玩法市场**：发布、搜索、评分、复刻修改
- **官方玩法包登记**：星球攻防默认安装、不可卸载（第 5 阶段之前，官方玩法包由 `server/app` 在启动时直接注册，不经过 K06）

## 2. 不负责

- 执行规则和脚本 → K05
- 生成玩法包的内容 → K16（借助 K10、K11）

## 3. 依赖

K00、K02、K04、K05、K10（玩法市场的向量检索）、K12、K13、K17

## 4. 对外接口

**玩法包结构**：

```
某玩法包/
├── manifest.json   # 名称、版本、作者、需要的权限、依赖的其他玩法包
├── components/     # 新组件的 Schema 和可见性
├── rules/          # 声明式规则
├── scripts/        # Luau 脚本（可选）
├── capabilities/   # 注册给 AI 的新能力
├── ui/             # 声明式界面描述
├── prompts/        # 让 AI 理解这个玩法的说明
└── assets.json     # 用到的素材指纹
```

```rust
pub trait Packs: Send + Sync {
    fn check(&self, pack: &PackBundle) -> CheckReport;                 // 静态检查
    fn install(&self, planet: PlanetId, pack: PackId, approved: Permissions) -> Result<()>;
    fn uninstall(&self, planet: PlanetId, pack: PackId) -> Result<()>;
    fn trial(&self, planet: PlanetId, pack: &PackBundle) -> Result<TrialPlanetId>;
    fn search_market(&self, query: &str) -> Result<Vec<Listing>>;
    fn publish(&self, pack: PackId, author: UserId) -> Result<ListingId>;
}
```

HTTP：上传或生成后检查；试玩；安装 / 卸载；玩法市场搜索、发布、评分

## 5. 数据

| 表 | 关键字段 |
|---|---|
| packs | id（内容指纹）, name, version, author, manifest, status |
| pack_installs | planet_id, pack_id, approved_permissions, installed_at, suspended |
| market_listings | id, pack_id, author, description, rating, installs |

## 6. 事件

- 发布：`PackInstalled`、`PackUninstalled`、`PackPublished`

## 7. 配置

`packs.json`：每颗星球最多安装多少个玩法包、玩法市场审核规则

## 8. 关键规则

**静态检查清单**（任何一项不通过就拒绝）：
- 申请的权限级别没有超过"星球级"（官方级只给官方玩法包）
- 只写自己声明的组件
- 不调用任何会产生、收取或转移金火的能力
- 脚本只使用 `world` 白名单接口
- 所有文字（名字、描述、对话模板）通过 K13 检查
- 依赖的其他玩法包存在且没有循环

**其他**：
- 卸载时，这个玩法包的组件保留为"休眠数据"，重新安装可以恢复
- 升级时，新增的权限需要主人重新批准
- 玩法市场搜索用向量检索（pgvector），让星球 AI 能"先复用，再生成"

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 5 | 接管官方玩法包登记 | 星球攻防随星球自动安装，不能卸载 |
| 5 | manifest 格式与静态检查 | 违规的玩法包被拒绝并说明原因 |
| 5 | 试玩与安装 | 在副本里试玩后安装到正式星球 |
| 5 | 玩法市场 | 能发布、搜索、安装别人的玩法包 |

## 10. 测试要点

- 构造违反每一条检查规则的玩法包，确认都被拦下
- 卸载再安装后数据恢复

## 11. 待定问题

- 玩法市场的审核流程（自动检查 + 人工抽查？）
- 玩法包作者的署名与授权方式（写进用户协议）

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K06-x"（x 换成子任务编号）。

~~~text
【模块】K06 玩法包管理（包 gf-packs，路径 server/kernel/packs，层级 L7）
【先读】docs/modules/K06-packs.md；K02、K04、K05、K10、K12、K13、K17 文档的第 4 节；主文档第 16.4、16.5 节
【范围】server/kernel/packs/、server/migrations/（文件名带 k06）、shared/config/packs.json、fixtures/packs/、server/app/（只限接入本模块，以及把官方玩法包的登记从 app 移到本模块）
【允许的依赖】模块：gf-core、gf-world、gf-capability、gf-script、gf-ai-gateway（玩法市场的向量检索）、gf-asset、gf-moderation、gf-history。库：serde、serde_json、sqlx（含 pgvector 支持）、tokio、async-trait、tracing、axum、jsonschema、schemars、sha2。测试：proptest
【规则补充】
  - 玩法包编号 PackId = manifest 里的 name；版本由内容指纹锁定（packs 表的 id 是整个目录的 sha256 指纹）
  - fixtures/packs/：meteor_shards/（合规的完整示例）；每条检查规则各一个违规包：bad_official_level/、bad_undeclared_component/、bad_gold_fire/、bad_api/、bad_text/、bad_cycle_a/ 和 bad_cycle_b/
  - 金火相关能力 = K07 相关的能力和所有官方级能力；玩法包申请或调用任何一个都不通过
  - 试玩：调用 K17 的 fork_trial，在副本星球上安装运行
【子任务】
K06-1 接管官方玩法包登记（阶段 5）
  - 测试：war_pack_auto_installed_and_not_uninstallable、story_pack_optional
K06-2 manifest 格式与静态检查（阶段 5）
  - check 返回 CheckReport：每条不通过的规则都列出来（不在第一条就停），带文件和原因
  - 测试：accepts_meteor_shards_pack、rejects_official_level_permission、rejects_undeclared_component_write、rejects_gold_fire_capabilities、rejects_non_whitelisted_api、rejects_text_failing_moderation、rejects_dependency_cycle、report_lists_all_failures
  【验收】另加 tools/mutants.sh gf-packs src/check.rs
K06-3 试玩、安装、卸载、升级（阶段 5）
  - 测试：trial_does_not_affect_real_planet、uninstall_keeps_dormant_data_and_reinstall_restores、upgrade_requires_reapproval_for_new_permissions、install_limit_per_planet
K06-4 玩法市场（阶段 5）
  - 搜索 = 先按文字过滤，再按向量相似度排序（向量来自 K10 的 embed；测试里用 FakeGateway 返回固定向量）
  - 测试：publish_then_search_finds_pack、search_ranks_by_similarity、install_from_market、unpublished_pack_not_listed
【验收（每个子任务）】tools/check_server.sh gf-packs；tools/check_tests.sh gf-packs <本子任务点名的测试>
【停止条件】需要执行玩法包里的代码来做静态检查；市场需要人工审核流程（文档里还是待定）
~~~
