# C04 地形与布局显示（terrain）

> **层级**：客户端 · **代码**：`client/terrain`（通过 godot-rust 调用 `shared/terrain`）· **首次阶段**：1 · **主文档**：第 9.1、21 节

## 1. 职责

- 调用共享 Rust 地形库，用"种子 + 改动"生成地形网格
- 程序化地貌（山脉、河流、火山、冰川……）；风化侵蚀由服务器在创作时算一次、存在地形改动里，客户端只读取，不模拟
- 按高度和坡度自动上材质（雪线以上铺雪、陡坡露岩石、缓坡长草和树）
- 显示线性工程（城墙、道路、桥）：沿路径重复摆放模块
- 按镜头远近分块生成，只生成看得到的部分

## 2. 不负责

- 地形规则和可放置性判断（服务器用同一个库计算）
- 布局算法（服务器 K11 计算好位置，客户端只显示）

## 3. 依赖

`shared/terrain`（Rust 库）、C01、C03

## 4. 对外接口

| 接口 | 作用 |
|---|---|
| `build_chunk(planet_seed, edits, area, lod)` | 生成一块地形 |
| `height_at(position)` | 查询某点高度（用于摆放物体） |

## 5. 关键规则

- **地形计算只有一份代码**：服务器和客户端调用同一个 Rust 库，保证结果完全一致
- 生成放在后台线程，不卡画面

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 接入共享地形库 | 同一个种子在服务器和客户端逐点比对一致 |
| 1 | 自动上材质 | 雪线、岩石、草地分布合理 |
| 2 | 线性工程显示 | 沿赤道的城墙连续、不穿插 |

## 7. 测试要点

- 不同操作系统上生成的地形完全一致

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C04-x"（x 换成子任务编号）。

~~~text
【模块】C04 地形与布局显示（客户端目录 client/terrain，测试 client/tests/terrain；Rust 扩展 client/ext）
【先读】docs/modules/C04-terrain.md；X01-shared.md 第 3 节和【开发提示词】（地形库接口）；K02-world.md（TerrainEdit 组件）；主文档第 9.1、21 节
【范围】client/terrain/、client/tests/terrain/、client/tests/fixtures/terrain/、client/ext/（只加 GfTerrain 类）、shared/config/terrain_materials.json
【允许的依赖】Godot 内置类（ArrayMesh、SurfaceTool、WorkerThreadPool）；C01、C03；gf-client-ext（gf-terrain、godot）；GUT
【规则补充】
  - 地形计算只调用 gf-terrain（通过 gf-client-ext 的 GfTerrain：height_at、biome_at、build_chunk）；GDScript 里不写任何地形公式
  - 分块生成放在 WorkerThreadPool，主线程只负责把生成好的网格挂上去
  - 上材质规则来自 shared/config/terrain_materials.json：高度 ≥ 雪线 → 雪；坡度 ≥ rock_slope_deg → 岩石；其余 → 草地（坡度 ≤ tree_slope_deg 时可以长树）
  - 线性工程：沿服务器给的路径按模块长度等距摆放模块，模块的高度取 height_at
【子任务】
C04-1 接入共享地形库（阶段 1）
  - 测试：test_golden_heights_match（读 shared/terrain/golden/seed_42.json 的 1000 个点，GfTerrain.height_at 的结果按 f32 位模式逐点相同）、test_chunk_built_off_main_thread、test_edits_change_chunk
  【验收】另加 cd client/ext && cargo test
C04-2 自动上材质（阶段 1）
  - 测试：test_material_rules_table（表驱动：雪线上下、陡坡缓坡的组合各一行）
  【人工验收】截图：雪线、岩石、草地分布合理
C04-3 线性工程显示（阶段 2）
  - 测试：test_wall_segments_continuous_along_path（相邻模块之间没有缝隙也不重叠，误差 ≤ 1 厘米）、test_modules_follow_terrain_height
【验收（每个子任务）】tools/check_client.sh terrain；tools/check_tests.sh terrain <本子任务点名的测试>
【停止条件】需要在 GDScript 里复制地形公式；金标准对不上（先报告，不要改金标准文件）
~~~
