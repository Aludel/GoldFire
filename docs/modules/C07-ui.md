# C07 界面与交互（ui）

> **层级**：客户端 · **代码**：`client/ui` · **首次阶段**：1 · **主文档**：第 5、6、7、8.4、11、16.5 节

## 1. 职责

- 星系视图的交互：点击星球、搜索、设置主星和隐藏
- 划区域：笔刷 / 套索画出任意形状的边界
- 建造操作：选中、拖动、旋转、缩放（每个操作都是一次能力调用）
- 纪念塔：查看和刻字；点击刻字高亮对应作者的作品
- 战斗界面：匹配、侦察、选武器、选落点、攻击、战斗记录
- 世界 BOSS 界面：血量、出手、排行
- 通知：被攻击、灾厄警告、祈祷
- **商店**：额度包和月卡；用系统浏览器打开支付页面，轮询订单状态；显示剩余灵感（K18）
- **声明式界面渲染器**：把玩法包里的界面描述（"这里有一个按钮，点了调用某个能力"）显示出来

## 2. 不负责

- 和星球 AI 对话（C10）；剧情界面（C09）

## 3. 依赖

C01、C03

## 4. 声明式界面

玩法包的界面只能用这种描述，不能带代码：

```json
{
  "面板": "钓鱼",
  "组件": [
    { "类型": "按钮", "文字": "抛竿", "调用能力": "fishing.抛竿" },
    { "类型": "列表", "数据": "fishing.图鉴", "显示": ["名称", "稀有度"] }
  ]
}
```

## 5. 关键规则

- 界面上的所有修改都通过 C01 调用能力，不直接改本地状态
- 声明式界面只支持白名单里的组件类型
- 被隐藏的信息（例如未侦察的重要度）服务器不会发来，界面不需要也无法显示

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 建造操作、划区域 | 能自由放置、拖动、旋转、缩放；能画任意形状区域 |
| 2 | 星系视图、纪念塔、通知 | 能找到好友的星球并刻字 |
| 3 | 战斗界面 | 完整走完侦察到攻击 |
| 4 | 世界 BOSS 界面 | 多人同时攻击时血量实时更新 |
| 5 | 声明式界面渲染器 | 钓鱼玩法的面板正常显示和操作 |

## 7. 测试要点

- 声明式界面里出现未知组件类型时安全忽略

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C07-x"（x 换成子任务编号）。

~~~text
【模块】C07 界面与交互（客户端目录 client/ui，测试 client/tests/ui）
【先读】docs/modules/C07-ui.md；C01、C03 文档的第 4 节；K18-billing.md 第 4 节和第 8 节的「支付」（C07-4）；K04-capability.md【规则补充】（内置能力的名字）；K02-world.md【规则补充】（区域）；主文档第 5、6、7、11、16.5 节
【范围】client/ui/、client/tests/ui/、client/tests/fixtures/ui/
【允许的依赖】Godot 内置类（Control 系列、Camera3D 射线检测）；C01、C03；GUT
【规则补充】
  - 每个修改操作都是一次 C01 的 call_capability；界面不直接改本地状态，等服务器的变化到达后才更新画面（拖动时可以显示半透明的预览）
  - 划区域：笔刷或套索画出的边界转成经纬度多边形，首尾闭合，发给"划定区域"能力
  - 声明式界面只支持白名单里的组件类型（按钮、列表、文字、进度条、开关）；不认识的类型跳过并写日志；按钮只能调用描述里写明的能力
  - 服务器没发来的数据（例如没侦察的重要度）界面不显示占位值
  - 商店：下单后用 OS.shell_open 在系统浏览器里打开支付页面，客户端不显示也不保存任何支付信息；是否到账只看 /billing/orders/{id} 的状态；支付关闭（/billing/products 返回 404）时不显示商店入口
【子任务】
C07-1 建造操作与划区域（阶段 1）
  - 测试：test_drag_sends_move_capability、test_rotate_scale_send_transform、test_lasso_sends_closed_lonlat_polygon、test_brush_region_outline_closed、test_local_state_unchanged_until_server_change
  【人工验收】录屏：放置、拖动、旋转、缩放一个物体，再画一个不规则区域
C07-2 星系视图、纪念塔、通知（阶段 2）
  - 测试：test_click_planet_opens_info、test_set_main_and_hidden_call_api、test_inscribe_calls_capability、test_click_inscription_highlights_author_works、test_notifications_listed
C07-3 战斗界面（阶段 3）
  - 测试：test_scout_then_attack_sequence、test_hidden_importance_not_displayed、test_weapon_picker_lists_own_functioning_weapons
C07-4 商店（阶段 3）
  - 测试：test_shop_lists_products、test_buy_opens_checkout_in_browser（打开的是服务器返回的地址）、test_paid_only_after_server_status（服务器返回已支付之前，额度显示不变）、test_shop_hidden_when_payments_disabled
  【人工验收】录屏：用假渠道买一个额度包，看到灵感增加
C07-5 世界 BOSS 界面（阶段 4）
  - 测试：test_hp_bar_follows_channel_updates、test_attack_button_disabled_after_daily_limit
C07-6 声明式界面渲染器（阶段 5）
  - 测试：test_fishing_panel_from_fixture（fixtures 里的钓鱼面板：一个按钮调用 fishing.抛竿，一个列表显示 fishing.图鉴 的名称和稀有度）、test_unknown_component_ignored、test_button_calls_declared_capability、test_only_whitelisted_types_rendered
【验收（每个子任务）】tools/check_client.sh ui；tools/check_tests.sh ui <本子任务点名的测试>
【停止条件】需要在客户端判定规则；声明式界面需要执行代码；需要在客户端里输入、显示或保存支付信息
~~~
