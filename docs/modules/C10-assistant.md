# C10 星球助手（assistant）

> **层级**：客户端 · **代码**：`client/assistant` · **首次阶段**：1 · **主文档**：第 8.4、16 节

## 1. 职责

- 和星球 AI 对话：输入自然语言，显示星球 AI 的回复和提问
- **预览改动**：在星球上高亮将要新增、修改、删除的东西，并列出清单（"将放置 12 个物体、安装 1 个玩法"）和预计花费；有花费时提示"撤销不退还花掉的资源"
- 确认 / 取消
- **撤销**与**时光机**：查看改动时间线，撤销某次改动，恢复到某一天（只回退创作内容）
- 试玩副本的进入和退出
- 星球设置（主人可见）：星球设定、自主程度、思考预算、访客策略、星球 AI 的花费权限（金和火的每日上限、是否允许发起攻击）
- **额度**：显示剩余灵感；预计花费较大的请求先让玩家确认；额度不够时说明原因（个人额度、思考预算、全服闸门、太频繁），并给出不用 AI 的做法

## 2. 不负责

- 理解和规划（K16，在服务器）

## 3. 依赖

C01、C03

## 4. 对外接口

| 接口 | 作用 |
|---|---|
| `send(text)` | 发送一句话给星球 AI |
| `show_preview(proposal)` | 显示预览 |
| `confirm(proposal_id)` / `cancel(proposal_id)` | 确认或取消 |
| `open_timeline()` | 打开改动时间线 |

## 5. 关键规则

- 没有预览就不执行（"只听指令"模式下也一样）
- 撤销不能回退的内容（战斗结果、资源）在时间线上标为不可撤销

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | 对话、预览、确认、撤销 | 「在湖边种一片樱花林」完整走通 |
| 2 | 星球设置 | 主人能写星球设定、设置访客策略 |
| 5 | 时光机、试玩副本、建议与自主模式 | 恢复到昨天；在副本里试玩新玩法 |

## 7. 测试要点

- 预览和实际执行的结果一致

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C10-x"（x 换成子任务编号）。

~~~text
【模块】C10 星球助手（客户端目录 client/assistant，测试 client/tests/assistant）
【先读】docs/modules/C10-assistant.md；K16-planet-mind.md 第 4 节和【规则补充】；K17-history.md 第 4 节；K18-billing.md 第 4 节（HTTP）；C01、C03 文档的第 4 节；主文档第 16 节
【范围】client/assistant/、client/tests/assistant/、client/tests/fixtures/assistant/、client/tests/e2e/sakura_forest.gd
【允许的依赖】Godot 内置类；C01、C03；GUT
【规则补充】
  - 发送时带上当前选中的区域和镜头看着的位置（K16 的 RequestContext）
  - 没有预览就不能确认；预览时在星球上高亮将要新增（绿）、修改（黄）、删除（红）的东西，并列出数量和预计花费；有花费时同时显示"撤销不退还花掉的资源"
  - 时间线上不可撤销的改动（战斗结果、资源）标成灰色、没有撤销按钮
  - 星球设置只对主人显示
  - 额度以灵感显示，数字直接用服务器给的（K18 的 /billing/quota），客户端不自己换算；估算（/billing/estimate）的 needs_confirm 为真时，先弹确认再发送
【子任务】
C10-1 对话、预览、确认、撤销（阶段 1）
  - 测试：test_preview_highlights_listed_changes、test_no_confirm_without_preview、test_confirm_then_undo_calls、test_request_includes_region_and_focus、test_preview_warns_undo_not_refunded（预览有花费时显示提示，没有花费时不显示）
  【验收】另加 tools/e2e.sh sakura_forest（"在湖边种一片樱花林"→ 预览 → 确认 → 看到樱花树 → 撤销 → 樱花树消失）
C10-2 星球设置与额度（阶段 2）
  - 测试：test_settings_send_spending_permissions（金、火每日上限和是否允许攻击）、test_settings_visible_only_to_owner、test_quota_shown_from_server、test_confirm_when_estimate_needs_confirm、test_quota_exceeded_reason_shown（四种原因各显示对应的提示）
C10-3 时光机、试玩副本、建议与自主模式（阶段 5）
  - 测试：test_timeline_marks_non_undoable、test_restore_to_day_calls_capability、test_enter_and_exit_trial
【验收（每个子任务）】tools/check_client.sh assistant；tools/check_tests.sh assistant <本子任务点名的测试>
【停止条件】需要在客户端理解或规划玩家的话；需要跳过预览直接执行
~~~
