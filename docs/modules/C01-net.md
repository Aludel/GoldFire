# C01 网络与同步（net）

> **层级**：客户端 · **代码**：`client/net` · **首次阶段**：1 · **主文档**：第 21、22 节

## 1. 职责

- HTTP 请求的统一封装（登录状态、错误处理、重试）
- WebSocket 频道：星球频道、世界 BOSS 频道、通知频道
- 发送指令：所有修改都调用能力接口（`POST /capabilities/{name}`），带客户端编号便于去重
- 接收广播，把变化交给其他客户端模块
- 断线重连：重连后向服务器请求"断线期间错过的变化"
- 离线时缓存待提交的工坊作品（C08）

## 2. 不负责

- 任何规则判定；客户端看到的一切以服务器为准

## 3. 依赖

服务器的 HTTP / WebSocket 接口；`shared/schemas` 的数据格式

## 4. 对外接口（给其他客户端模块）

| 接口 | 作用 |
|---|---|
| `call_capability(name, input, preview)` | 预览或执行一个能力，返回结果 |
| `subscribe(channel, handler)` | 订阅频道里的变化 |
| `get(path)` | 一般的查询请求 |
| 信号 `connection_changed` | 连接状态变化 |

## 5. 关键规则

- 客户端永远只发**指令**，不发**结果**
- 收到的数据先按 Schema 校验，不合格的丢弃并记录
- 不在本地保存任何令牌明文以外的敏感信息；令牌存在系统安全存储里

## 6. 开发步骤

| 阶段 | 任务 | 完成标准 |
|---|---|---|
| 1 | HTTP 封装与能力调用 | 能放置一个物体 |
| 2 | WebSocket 星球频道 | 两个客户端看到一致的变化 |
| 2 | 断线重连与补发 | 断网 1 分钟后重连，状态一致 |

## 7. 测试要点

- 弱网、断网、服务器重启时的表现

## 8. 开发提示词

用法见 README 的[开发提示词怎么用](README.md#开发提示词怎么用)：新开一个 AI 会话，先发 README 里的"通用开头"，再发下面整段，最后说"本次只做子任务 C01-x"（x 换成子任务编号）。

~~~text
【模块】C01 网络与同步（客户端目录 client/net，测试 client/tests/net）
【先读】docs/modules/C01-net.md；K00-core.md 第 4 节（错误和 HTTP 状态码）；K02-world.md 第 4 节（changes 和 WebSocket）；K04-capability.md 第 4 节（POST /capabilities/{name}）；主文档第 21、22 节
【范围】client/net/、client/tests/net/、client/tests/fixtures/net/、client/ext/（C01-1 新建 gf-client-ext 并加进工作区和 layers.toml 的 top）、client/tests/e2e/ 和 tools/e2e.sh（C01-2）
【允许的依赖】Godot 内置类（HTTPClient 或 HTTPRequest、WebSocketPeer、JSON）；gf-client-ext（Rust：godot、jsonschema、serde_json；C01-4 另加 keyring）；GUT
【规则补充】
  - GDScript 一律写类型；对外的类用 class_name；状态变化用信号
  - 所有网络访问经过 Transport 接口：HttpTransport（真实）、FakeTransport（测试用：按脚本返回响应，并记录收到的请求）
  - call_capability(name, input, preview)：POST /capabilities/{name}，请求体 {input, mode: "preview" 或 "execute", client_request_id}；client_request_id 每次调用新生成，重试时不变
  - 错误：按 K00 的 kind 字段转成客户端的错误枚举（7 种 + network_error）
  - 收到的数据用 gf-client-ext 的 GfSchema.validate(schema 名, json) 按 shared/schemas 校验；不合格的丢弃并写日志，不弹窗、不崩溃
  - 第 1 阶段的同步：每次执行能力之后、以及每隔 poll_interval_ms，GET /planets/{id}/changes?since=<last_seq>；按序号顺序交给订阅者；重复的序号忽略；reload = true 时发出 reload_required 信号
【子任务】
C01-1 HTTP 封装与能力调用（阶段 1）
  - 测试：test_call_capability_builds_request、test_client_request_id_unique_per_call、test_retry_keeps_client_request_id、test_error_kind_mapping、test_invalid_response_dropped_and_logged
  【验收】另加 cd client/ext && cargo test
C01-2 变化拉取与端到端测试框架（阶段 1）
  - tools/e2e.sh <场景>：用 tools/dev_db.sh 建临时数据库 → 用 GF_AI_FAKE_DIR=fixtures/ai、dev_mode = true 启动 gf-app（空闲端口，等 /health 通过）→ godot --headless --path client -s res://tests/e2e/run.gd -- --scenario <场景> --server <地址> → 返回 Godot 的退出码；无论成功失败都关掉服务器
  - 场景脚本放 client/tests/e2e/<场景>.gd；先写 place_one_entity（放置一个物体，再从 changes 里看到它）
  - 测试：test_changes_dispatched_in_seq_order、test_duplicate_seq_ignored、test_reload_required_when_server_says_so
  【验收】另加 tools/e2e.sh place_one_entity
C01-3 WebSocket 星球频道（阶段 2）
  - 测试：test_ws_messages_dispatched_in_seq_order、test_ws_and_poll_do_not_duplicate
  【验收】另加 tools/e2e.sh two_clients_same_changes（两个无界面客户端，一个放置、另一个收到同样的变化）
C01-4 断线重连与令牌保存（阶段 2）
  - 重连后先拉取 last_seq 之后的变化，再恢复 WebSocket；令牌用 gf-client-ext 的 GfSecret（keyring）存进系统安全存储
  - 测试：test_reconnect_fetches_changes_since_last_seq、test_token_not_written_to_user_files（测试里用 keyring 的模拟后端）
  【人工验收】断网 1 分钟后恢复网络，客户端自动重连，画面和另一个客户端一致
【验收（每个子任务）】tools/check_client.sh net；tools/check_tests.sh net <本子任务点名的测试>
【停止条件】需要在客户端判定规则或计算结果；服务器接口和文档对不上
~~~
