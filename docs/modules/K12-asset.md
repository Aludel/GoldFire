# K12 素材（asset）

> **层级**：内核 L2 · **代码**：`server/kernel/asset` · **首次阶段**：1 · **主文档**：第 14.8、21、28 节

## 1. 职责

- 素材目录与元数据：类别、标签、描述、许可证、来源、占地轮廓、远近版本文件
- **检索**：按标签和语义（向量）检索候选素材
- **质检状态**：待质检 / 通过 / 拒绝
- **主题包**：一组素材 + 调色板 + 布局规则 + 示例作品
- 发布到 CDN
- 记录每个素材的许可证

## 2. 不负责

- 生成素材 → K11
- 客户端缓存 → C02

## 3. 依赖

K00、K13（检查图片和文字）。AI 打分和语义检索用的向量通过 `AssetScorer`、`Embedder` 两个接口在启动时注入（见第 4 节），本模块不直接依赖 AI 网关

## 4. 对外接口

```rust
pub trait AssetScorer: Send + Sync {                 // 由 app 注入
    fn score(&self, asset: &AssetFiles) -> Result<QualityReport>;
}
pub trait Embedder: Send + Sync {                    // 由 app 注入（调用 K10 的 embed）
    fn embed(&self, text: &str) -> Result<Vec<f32>>;
}
pub trait Assets: Send + Sync {
    fn get(&self, id: AssetId) -> Result<AssetMeta>;
    fn search(&self, query: &str, category: Option<Category>, limit: usize) -> Result<Vec<AssetMeta>>;
    fn footprint(&self, id: AssetId) -> Result<Footprint>;
    fn submit(&self, files: AssetFiles, meta: NewAssetMeta) -> Result<AssetId>;   // 进入待质检
    fn theme(&self, id: ThemeId) -> Result<ThemePack>;
}
```

HTTP：素材目录（客户端用）；主题包列表。CDN 地址：`/assets/{指纹}`

## 5. 数据

| 表 | 关键字段 |
|---|---|
| assets | id（内容指纹）, category, tags, description, license, source（官方 / 社区 / 铸造）, creator_id, footprint, files, status, embedding |
| themes | id, name, palette_id, asset_ids, layout_rules, examples |

## 6. 事件

- 发布：`AssetApproved`、`AssetRejected`

## 7. 配置

`asset.json`：质检阈值、面数上限、远近版本数量

## 8. 关键规则

- **编号 = 内容指纹**；素材发布后不可修改，改了就是新素材、新编号
- 每个素材都必须带许可证；官方素材用 CC BY-SA 4.0
- 检索时先按类别和标签过滤，再按语义相似度排序
- 版权过滤：铸造作品通过 K13 检查是否像知名角色或作品

## 9. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 目录与元数据，导入 50 个占位素材 | 客户端能列出并下载 |
| 1 | 标签检索 | 「茶馆」能检索到相关素材 |
| 2 | 语义检索、主题包 | 「东方风格的小店」也能检索到茶馆 |
| 4 | 质检流程 | 铸造作品自动打分，不合格被拒绝 |

## 10. 测试要点

- 指纹校验：文件被改动后编号对不上
- 检索质量：用 X02 的检索测试集评估

## 11. 待定问题

- 第一批素材和主题包的清单；美术规范

## 12. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 K12-x"（x 换成子任务编号）。

~~~text
【模块】K12 素材（包 gf-asset，路径 server/kernel/asset，层级 L2）
【先读】docs/modules/K12-asset.md；K13-moderation.md 第 4 节；K00-core.md 第 4 节（ObjectStore）；主文档第 14.8、21、28 节
【范围】server/kernel/asset/、server/migrations/（文件名带 k12）、shared/config/asset.json、assets/placeholder/、fixtures/asset/、server/app/（只限接入本模块，以及 AssetScorer、Embedder 两个适配器）
【允许的依赖】模块：gf-core、gf-moderation。库：serde、serde_json、sqlx（含 pgvector 支持）、tokio、async-trait、tracing、axum、sha2、schemars、clap（导入命令）。测试：proptest
【规则补充】
  - 素材编号 = 指纹：对素材目录里的全部文件按路径排序，依次拼接"路径 + 0 字节 + 文件内容的 SHA-256"，再整体做 SHA-256
  - 每个素材目录有 meta.json：类别、标签、描述、许可证、来源、占地轮廓（以模型原点为中心、单位米的多边形）、远近版本文件；没有许可证或占地轮廓的拒绝导入
  - 文件存进 ObjectStore，key 是 assets/<指纹>/<文件名>；CDN 地址 /assets/{指纹}/{文件名}
  - 语义检索需要向量：新增 Embedder trait（由 app 用 K10 的 embed 实现并注入），本模块不直接依赖 K10
  - 导入命令：cargo run -p gf-asset --bin import_assets -- assets/placeholder
【子任务】
K12-1 目录、元数据、导入（阶段 1）
  - 准备 assets/placeholder/ 下至少 50 个占位素材（简单几何体即可，CC BY-SA 4.0），其中必须有 茶馆_01、樱花树_01
  - 测试：asset_id_is_sha256_of_content、changed_file_gets_new_id、license_and_footprint_required、import_is_idempotent、catalog_http_lists_assets
  【人工验收】导入后用浏览器打开 /assets 目录接口，能看到 50 个以上素材并下载其中一个
K12-2 标签检索（阶段 1）
  - 先按类别和标签过滤，再按标签命中数排序，命中数相同按编号排序
  - 测试：search_by_tag_and_category（"茶馆"→ 茶馆_01 排第一）、search_respects_limit、unknown_category_rejected
K12-3 语义检索与主题包（阶段 2）
  - 开始前 K10-5 要已经合并：Embedder 适配器要调用 K10 的 embed
  - 测试：semantic_search_ranks_by_similarity（FakeEmbedder 返回固定向量）、theme_pack_loads_assets_palette_rules
  【效果验收】tools/eval.sh retrieval 的前 20 名命中率 ≥ 90%
K12-4 质检流程（阶段 4）
  - 测试：low_quality_score_rejected、copyright_flag_rejected、approved_asset_published_event、submitted_asset_not_searchable_until_approved
【验收（每个子任务）】tools/check_server.sh gf-asset；tools/check_tests.sh gf-asset <本子任务点名的测试>
【停止条件】需要修改已发布的素材；素材缺许可证信息
~~~
