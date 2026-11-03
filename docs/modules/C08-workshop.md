# C08 本地工坊（workshop）

> **层级**：客户端 · **代码**：`client/workshop` · **首次阶段**：2 · **主文档**：第 21 节

## 1. 职责

- 在本地设计和预览造物草稿（不影响任何星球）
- 断网时继续设计；联网后发布
- 发布 = 把配方上传，由服务器重新校验后放到星球上
- 管理本地草稿和蓝图

## 2. 不负责

- 校验（服务器负责）

## 3. 依赖

C01、C02、C03、C04

## 4. 对外接口

| 接口 | 作用 |
|---|---|
| `open_workshop()` | 打开工坊场景 |
| `save_draft(recipe)` / `list_drafts()` | 草稿管理 |
| `publish(draft, planet, region, transform)` | 发布到星球（调用"放置"能力） |

## 5. 关键规则

- 草稿只存在本地；发布前服务器会重新检查，本地通过不代表服务器通过
- 发布失败时保留草稿，并显示原因

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 2 | 草稿、预览、发布 | 断网设计、联网发布成功 |

## 7. 测试要点

- 发布被服务器拒绝时，草稿不丢失

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C08-x"（x 换成子任务编号）。

~~~text
【模块】C08 本地工坊（客户端目录 client/workshop，测试 client/tests/workshop）
【先读】docs/modules/C08-workshop.md；C01–C04 文档的第 4 节；K04-capability.md【规则补充】（"放置"能力）；主文档第 21 节
【范围】client/workshop/、client/tests/workshop/、client/tests/fixtures/workshop/
【允许的依赖】Godot 内置类；C01、C02、C03、C04；GUT
【规则补充】
  - 草稿只存在本机 user://workshop/drafts/，一个草稿一个 JSON 文件（格式按 recipe.schema.json）；不上传，除非发布
  - 发布 = 调用"放置"能力；服务器拒绝时保留草稿，并显示服务器给的原因
  - 断网时可以继续编辑和保存；联网后由玩家点"发布"，不自动发布
【子任务】
C08-1 草稿、预览、发布（阶段 2）
  - 测试：test_draft_saved_offline_and_published_when_online、test_rejected_publish_keeps_draft_and_shows_reason、test_drafts_stored_locally_only、test_publish_calls_place_capability
  【人工验收】断网时设计一个造物并保存；联网后发布成功
【验收（每个子任务）】tools/check_client.sh workshop；tools/check_tests.sh workshop <本子任务点名的测试>
【停止条件】需要在客户端做服务器那样的校验并当作最终结果
~~~
