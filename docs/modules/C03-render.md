# C03 星球渲染（render）

> **层级**：客户端 · **代码**：`client/render` · **首次阶段**：1 · **主文档**：第 4、14.1–14.3 节

## 1. 职责

- **两种尺度**：轨道视图（整颗星球）和地表视图（当前区域，细节拉满），两者之间平滑过渡
- **统一出口**：所有模型使用同一套风格化着色器；颜色来自星球调色板（换调色板即整体换风格）
- **后期**：描边、柔和阴影与环境光遮蔽、调色、近景移轴
- **光照**：唯一光源是 GoldFire 太阳；昼夜由星球时间决定；夜景灯光（越热门越亮）；大气层光晕
- **LOD**：远处自动简化；画质档位（低 / 中 / 高）
- 星系视图的渲染（轨道环、主星、星尘点）

## 2. 不负责

- 地形生成（C04）、人物（C05）、特效（C06）

## 3. 依赖

C01、C02、C04；`shared/config/palettes.json`、`time.json`

## 4. 对外接口

| 接口 | 作用 |
|---|---|
| `show_planet(planet_id)` / `show_galaxy()` | 切换场景 |
| `fly_to(position, zoom_level)` | 镜头移动到某个位置和层级 |
| `set_palette(palette_id)` | 换调色板 |
| `set_quality(level)` | 画质档位 |

## 5. 关键规则

- **浮动原点**：服务器的坐标是经纬度（f64），星球半径约 6371 km，直接交给 Godot 的单精度浮点数会抖动。客户端以当前区域附近的一点为渲染原点，物体位置先用 f64 算出相对原点的偏移，再转成 f32；镜头离原点超过 render.json 的 origin_shift_m 时，把原点移到镜头附近，所有物体一起平移。轨道视图的原点就是星球中心（这时 0.5 米的误差看不出来）
- 镜头层级：星系 → 轨道 → 大陆（聚合标记）→ 区域（45° 俯视）→ 近景（移轴）
- 十万个星尘点要一次绘制完成（用点精灵或实例化渲染）
- 不在客户端做任何规则判定

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 球面、旋转、两种视图 | 从轨道平滑降落到地表再飞回 |
| 1 | 统一着色器、调色板、后期 | 切换调色板，整颗星球一起变色 |
| 1 | 昼夜与光照 | 随星球时间变化 |
| 2 | 星系视图、夜景灯光、聚合标记 | 十万个星尘点流畅显示 |

## 7. 测试要点

- 中低配电脑上的帧率（定一个目标机型）

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C03-x"（x 换成子任务编号）。

~~~text
【模块】C03 星球渲染（客户端目录 client/render，测试 client/tests/render）
【先读】docs/modules/C03-render.md；C01、C02、C04 文档的第 4 节；K03-scheduler.md 第 8 节和【规则补充】（星球时间）；K09-galaxy.md（星尘点格式）；主文档第 4、14.1–14.3 节
【范围】client/render/、client/tests/render/、client/tests/fixtures/render/、shared/config/palettes.json（只加调色板）
【允许的依赖】Godot 内置类（着色器、MultiMeshInstance3D、WorldEnvironment、Camera3D、Tween）；C01、C02、C04；GUT
【规则补充】
  - 浮动原点：位置换算统一走一个函数（经纬度 + 高度 → 相对当前原点的 f32 坐标），C04、C05、C06 都用它；原点移动时发出信号，所有节点一起平移
  - 所有模型用同一个风格化着色器；颜色只来自当前调色板（16 个色号），换调色板只改一个全局着色器参数
  - 镜头层级固定为：星系 → 轨道 → 大陆 → 区域 → 近景；fly_to 用 Tween 平滑过渡，过渡中不能跳过中间层级
  - 太阳方向角 = 一天里的秒数 / day_seconds × 360°（day_seconds 来自 time.json）；星球时间从服务器拿，不用本机时间
  - 星尘点用一个 MultiMesh 一次绘制；GET /galaxy/dust 的每个点 24 字节（16 字节编号 + 两个小端 f32）
  - 不在客户端做任何规则判定
【子任务】
C03-1 球面、旋转、两种视图（阶段 1）
  - 测试：test_fly_orbit_to_surface_and_back、test_zoom_levels_in_order、test_floating_origin_precision（在离星球中心 6371 km 的地表放两个相距 0.1 米的物体，渲染后两者的距离和 0.1 米相差不到 1 毫米）、test_origin_shift_keeps_screen_positions（镜头离原点超过 origin_shift_m 时原点移动，移动前后物体在屏幕上的位置不变）
  【人工验收】录屏：从轨道平滑降落到地表，再飞回轨道
C03-2 统一着色器、调色板、后期（阶段 1）
  - 测试：test_all_models_use_unified_shader、test_palette_swap_updates_all_materials
  【人工验收】切换调色板前后各一张截图，整颗星球一起变色
C03-3 昼夜与光照（阶段 1）
  - 测试：test_sun_direction_from_planet_time（一天里的秒数 0 → 0°；1800 → 90°；3600 → 180°；5400 → 270°）、test_night_lights_on_at_night
C03-4 星系视图、夜景灯光、聚合标记（阶段 2）
  - 测试：test_dust_binary_parsed、test_dust_points_single_multimesh、test_cluster_markers_at_continent_level
  【人工验收】十万个星尘点在目标机型上 ≥ 60 帧（在 PR 里写明机型）
【验收（每个子任务）】tools/check_client.sh render；tools/check_tests.sh render <本子任务点名的测试>
【停止条件】需要第二套着色器或调色板以外的颜色；没有定下目标机型（C03-4 前先问我）
~~~
