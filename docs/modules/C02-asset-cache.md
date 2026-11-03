# C02 素材缓存（asset_cache）

> **层级**：客户端 · **代码**：`client/asset_cache` · **首次阶段**：1 · **主文档**：第 21 节

## 1. 职责

- 按素材指纹从 CDN 下载素材（模型、贴图、特效资源）
- 下载后校验指纹，对不上就重新下载
- 本地缓存与清理（按最近使用时间，有总容量上限）
- 提前下载：世界 BOSS 等官方素材、当前星球附近区域的素材
- 按远近需要加载不同精度的版本

## 2. 不负责

- 渲染（C03）

## 3. 依赖

C01；K12 的素材目录接口；CDN

## 4. 对外接口

| 接口 | 作用 |
|---|---|
| `request(asset_id, lod)` | 请求素材；已缓存就立即返回，否则下载完成后通知 |
| `prefetch(asset_ids)` | 后台预下载 |
| 信号 `asset_ready(asset_id)` | 素材可用 |

## 5. 关键规则

- 缓存的文件名就是指纹，相同内容只存一份
- 下载失败时显示占位模型，不阻塞画面

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 下载、校验、缓存 | 第二次打开同一颗星球不再下载 |
| 2 | 预下载与容量管理 | 缓存超过上限时自动清理最久没用的素材 |

## 7. 测试要点

- 篡改缓存文件后能被发现并重新下载

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C02-x"（x 换成子任务编号）。

~~~text
【模块】C02 素材缓存（客户端目录 client/asset_cache，测试 client/tests/asset_cache）
【先读】docs/modules/C02-asset-cache.md；C01-net.md 第 4 节；K12-asset.md 第 4、8 节（指纹、CDN 地址）；主文档第 21 节
【范围】client/asset_cache/、client/tests/asset_cache/、client/tests/fixtures/asset_cache/
【允许的依赖】Godot 内置类（HashingContext、FileAccess、DirAccess、ResourceLoader、GLTFDocument）；C01；GUT
【规则补充】
  - 缓存目录 user://asset_cache/，文件名 = 指纹的十六进制部分 + 原扩展名；相同内容只存一份
  - 下载后用 HashingContext（SHA-256）校验；读缓存时也校验，对不上就删掉重下
  - 下载失败显示占位模型，并在后台按退避间隔重试，不阻塞画面
  - 容量上限来自客户端设置（默认 2 GB）；超过时按最近使用时间删除最久没用的
【子任务】
C02-1 下载、校验、缓存（阶段 1）
  - 测试：test_cached_asset_not_downloaded_again、test_tampered_file_redownloaded、test_placeholder_shown_on_failure、test_asset_ready_signal_emitted
  【人工验收】第二次打开同一颗星球时，下载日志里没有素材下载记录
C02-2 预下载、容量管理、远近版本（阶段 2）
  - 测试：test_lru_eviction_over_capacity、test_prefetch_in_background、test_lod_file_selected
【验收（每个子任务）】tools/check_client.sh asset_cache；tools/check_tests.sh asset_cache <本子任务点名的测试>
【停止条件】需要绕过指纹校验；需要素材以外的服务器数据
~~~
