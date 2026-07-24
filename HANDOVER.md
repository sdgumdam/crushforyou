# CrushForYou 交接文档

> 本文档供接手工作的 Agent 阅读。crushforyou 仓库当前状态：从 crush（FSL-1.1-MIT）clone，已改 LICENSE 为 MIT，但代码仍是 crush 原始代码（FSL 约束）。目标：重写为纯 MIT 项目。

---

## 1. 协议变更

| 项目 | 原 crush | crushforyou 目标 |
|---|---|---|
| LICENSE | FSL-1.1-MIT（2年后转MIT，限制竞品SaaS） | **纯 MIT** |
| CLA | 需要 CLA bot 签署 | **不需要** |
| 代码来源 | Charm 原创 | **全部重写，不复制 crush 的 FSL 代码** |
| 可引用的依赖 | — | charm.land 系列库（bubbletea/lipgloss/glamour/bubbles 等，纯 MIT） |
| 风险 | — | **FSL 有传染性**：只要仓库里有 crush 的 derivative work，整个仓库就必须挂 FSL。当前仓库代码全是 crush 的，LICENSE 改 MIT 是无效的，必须把 FSL 代码全部替换掉 |

## 2. crush 已有代码的处置

### 2.1 需要全部删除的（FSL derivative work）

| 目录 | 文件数 | 说明 |
|---|---|---|
| `internal/ui/` | ~50 | TUI 层，crush 原创 |
| `internal/agent/` | ~30 | agent 核心（Run 函数 764 行、coordinator、tools） |
| `internal/config/` | ~35 | 配置加载（多 scope merge） |
| `internal/session/` | ~2 | 会话管理 |
| `internal/message/` | ~5 | 消息模型 |
| `internal/permission/` | ~3 | 权限系统 |
| `internal/app/` | ~5 | App 层 |
| `internal/backend/` | ~10 | server 后端 |
| `internal/client/` | ~5 | client 连接 |
| `internal/cmd/` | ~15 | CLI 命令 |
| `internal/db/` | ~10 | 数据库 |
| `internal/csync/` | ~5 | 并发原语 |
| `internal/` 其他 | ~40 | 其他包（hooks/lsp/event/format 等） |
| `*.go` 根目录 | ~5 | main.go 等 |
| **合计** | **~220 个 .go 文件** | **全部是 FSL derivative work，必须删除重写** |

### 2.2 可保留的（MIT 库引用，不是 derivative work）

| 依赖 | 用途 | 许可证 |
|---|---|---|
| `charm.land/bubbletea/v2` | TUI 框架 | MIT |
| `charm.land/lipgloss/v2` | 样式 | MIT |
| `charm.land/glamour/v2` | markdown 渲染 | MIT |
| `charm.land/bubbles/v2` | TUI 组件（textarea/spinner/help） | MIT |
| `charm.land/log/v2` | 日志 | MIT |
| `charm.land/catwalk` | provider 元数据 | MIT |
| `charm.land/fantasy` | agent 循环框架 | MIT |
| `charm.land/x/ansi` | ANSI 处理 | MIT |

### 2.3 可复用的设计模式（不是代码，是思路）

| 设计模式 | crush 来源 | 复用方式 |
|---|---|---|
| Workspace 作为 UI/agent 边界 | `workspace.go:79` | 拆成 4 个小接口（Sessions/Messages/Agent/Permissions） |
| pubsub 事件流 | `ui.go:673-917` | agent→UI 通知（11 种泛型事件） |
| 工具渲染器注册表 | 替代 `tools.go:211` 的 30+ case | map + interface 自注册 |
| Common struct 持有共享配置 | `common/common.go:24` | Workspace + Styles |
| Dialog overlay 栈 | `dialog/dialog.go:65` | 栈式管理 + grace period 防误触 |
| list 版本化缓存 | `list/list.go` | 指针 key + 宽度失效 |
| E2E 真二进制测试范式 | `clientserverrace/race_test.go` | buildBinary + exec.Command |

---

## 3. crush 业务→架构映射（供重写参考）

### 3.1 前后端边界

crush 的 UI 层和 agent 层通过 `Workspace` 接口解耦，UI 不直接 import agent 核心包：

```
┌─ TUI 层（internal/ui/）─┐     ┌─ Agent 层（internal/agent/）─┐
│                         │     │                              │
│  model/ui.go (主Model)  │     │  coordinator.go              │
│  chat/tools.go (渲染)   │←──→│  agent.go (Run 764行)        │
│  dialog/* (对话框)      │ W   │  agent_tool.go               │
│  list/* (列表)          │ o   │  tools/*                     │
│  diffview/* (diff)      │ r   │  config/*                    │
│  styles/* (样式)        │ k   │  session/*                   │
│                         │ s   │  message/*                   │
│  只引用:                │ p   │  permission/*                │
│  - agent/tools (渲染)   │ a   │                              │
│  - agent/notify (通知)  │ c   │  不引用 UI                   │
│  - agent/hyper (检测)   │ e   │                              │
└─────────────────────────┘     └──────────────────────────────┘
```

**Workspace 接口（30+ 方法，需拆小）**：
- Sessions: Create/Get/List/Save/Delete + AgentToolSessionID
- Messages: List/ListUser/ListAll
- Agent: Run/Cancel/IsBusy/Model/Summarize/UpdateModels/Init
- Permissions: Grant/GrantPersistent/Deny/SkipRequests
- Questions: Answer/Cancel
- FileTracker: RecordRead/LastReadTime/ListReadFiles
- History: ListSessionHistory
- LSP: Start/StopAll/GetStates

### 3.2 UI→Agent 数据流

**UI 触发 Agent**：
- 发消息：`Workspace.AgentRun(ctx, sessionID, prompt)` fire-and-forget
- 取消：`Workspace.AgentCancel(sessionID)`
- 摘要：`Workspace.AgentSummarize(ctx, sessionID)`

**Agent→UI 通知（pubsub 事件流）**：
| 事件 | 用途 |
|---|---|
| `Event[notify.Notification]` | agent 完成/权限请求 |
| `Event[session.Session]` | 会话删除/更新 |
| `Event[message.Message]` | 消息创建/更新/删除/子会话 |
| `Event[permission.PermissionRequest]` | 权限弹窗 |
| `Event[question.Request]` | 提问表单 |
| `Event[history.File]` | 文件变更 |
| `Event[app.LSPEvent]` | LSP 状态 |
| `Event[skills.Event]` | 技能状态 |
| `Event[mcp.Event]` | MCP 状态 |

### 3.3 TUI 业务功能清单

| 功能 | 快捷键 | 关键组件 |
|---|---|---|
| 发消息 | enter | textarea + sendMessage |
| 选模型 | ctrl+m/l | dialog/models |
| 管理会话 | ctrl+s | dialog/sessions |
| 命令面板 | ctrl+p | dialog/commands |
| 权限批准 | a/s/d | dialog/permissions |
| 看 diff | t/f | diffview |
| 取消 | esc | cancelAgent |
| 图片附件 | ctrl+f/v | dialog/filepicker + attachments |
| @提及 | @ | completions |
| bang模式 | ! | ui.go:219 |
| 回答提问 | — | dialog/question_form |

### 3.4 工具渲染器（30+ 工具）

bash/job_output/job_kill/view/write/edit/multiedit/glob/grep/ls/download/fetch/sourcegraph/diagnostics/agent/agentic_fetch/web_fetch/web_search/todos/question/references/definition/rename/replace_symbol/call_hierarchy/symbols/lsp_restart + default(docker_mcp/mcp_/generic)

crush 用 switch-case 硬编码路由。重写时改为注册表模式（map + interface 自注册）。

### 3.5 关键自定义组件

| 组件 | 说明 |
|---|---|
| 动画 spinner | 渐变色循环 + label + ellipsis |
| Markdown 渲染器 | glamour + 缓存 + 增量渲染 |
| Diff 视图 | builder 模式 unified/split + chroma 高亮 |
| 通用列表 | 懒加载 + 版本化缓存 |
| 补全弹窗 | @ 触发模糊匹配 |
| 附件组件 | 文件/图片 chip |
| 对话框 Overlay | 栈式管理 + grace period 防误触 |

---

## 4. TDD 迁移进度

### 4.1 可直接复用的测试（18 个，通用范式与业务无关）

| 测试文件 | 测的功能 | 复用方式 |
|---|---|---|
| `diffview/diffview_test.go` | DiffView 全行为矩阵 + golden | builder + golden 快照，纯渲染组件 |
| `diffview/udiff_test.go` | udiff 库 unified diff | 第三方库测试 |
| `list/list_test.go` | list 缓存指针 key + 宽度失效 | 通用列表缓存 |
| `model/filter_test.go` | 滚轮事件合并/节流 | 通用滚轮 coalesce |
| `common/ansi16_test.go` | ANSI16→truecolor 重映射 | 纯函数 |
| `dialog/overlay_test.go` | grace period 吞键/防误触 | 通用对话框 |
| `dialog/permissions_test.go` | 按键→动作映射 + tab 导航 | 按键映射范式 |
| `model/session_busy_test.go` | 渲染不阻塞探针 + stub 计数 | 热路径契约 |
| `chat/version_bump_test.go` | version bump/dedupe 契约 | 缓存失效契约 |
| `dialog/version_bump_test.go` | 同上 | 同上 |
| `completions/version_bump_test.go` | 同上 | 同上 |
| `chat/incremental_glamour_test.go` | 增量 markdown 视觉等价 | 流式渲染 |
| `attachments/attachments_test.go` | parseCells 逐格背景色断言 | TUI 像素级测试 |
| `completions/completions_test.go` | 补全排序优先级 | 通用补全 |
| `dialog/question_choice_base_test.go` | hover/键盘一致性 | 通用交互 |
| `chat/applyhighlight_callback_test.go` | 回调驱动 highlight 冻结/解冻 | 缓存契约 |
| `common/elements_test.go` | token/cost 估算显示 | 格式化测试 |
| `chat/tool_result_content_test.go` | 工具结果内容路由 | 内容路由 |

### 4.2 思路可借鉴需改写的测试（13 个）

| 测试文件 | 需改写原因 |
|---|---|
| `notification/notification_test.go` | 绑 crush Notification struct |
| `chat/shell_test.go` | 绑 crush shell 子系统 |
| `model/layout_test.go` | 需适配新 model 结构 |
| `model/session_test.go` | 需改写渲染 |
| `model/permission_test.go` | 需适配新权限模型 |
| `chat/prefix_cache_test.go` | 需适配新缓存实现 |
| `chat/assistant_test.go` | 需适配新消息项 |
| `chat/assistant_thinking_window_test.go` | 需适配 |
| `chat/assistant_section_cache_test.go` | 需适配 |
| `model/chat_expand_test.go` | 需适配 dispatch |
| `model/pill_border_test.go` | 需适配 pill 系统 |
| `model/chat_draw_cache_test.go` | 需适配 uv 库 |
| `model/attachments_mouse_test.go` | 需适配 |

### 4.3 TDD 迁移状态

- [ ] **第 0 步**：定义渲染层接口（ToolRenderer/Versioned/Dialog/DiffView/list.Item）
- [ ] **第 1 步**：从 18 个可复用测试提取断言点，写成新测试文件
- [ ] **第 2 步**：按测试驱动实现渲染组件（用 bubbletea + lipgloss）
- [ ] **第 3 步**：定义业务层接口（拆小的 Workspace）
- [ ] **第 4 步**：写 P0 需求的黑盒测试
- [ ] **第 5 步**：按测试驱动实现 agent 核心

---

## 5. pi 的高级特性（参考架构）

pi（earendil-works/pi，75K stars，MIT，TypeScript）已实现 crush 用户想要但 crush 没给的高级特性：

### 5.1 三层上下文压缩（crush 缺失 P0）

pi 的 `packages/coding-agent/src/core/compaction/`：
- `compaction.ts`：主压缩逻辑，跟踪文件操作（readFiles/modifiedFiles），压缩时注入到摘要
- `branch-summarization.ts`：**会话分叉压缩**——切换到不同分支时自动生成当前分支摘要，上下文不丢
- `utils.ts`：文件操作提取 + 序列化对话

对比 crush：摘要后全丢，无会话分叉，无文件跟踪。

### 5.2 三模式架构（crush 只有两种）

pi 的 `packages/coding-agent/src/modes/`：
- `interactive/`：交互式 TUI（完整 UI）
- `print-mode.ts`：非交互单次输出（类似 `crush run`）
- `rpc/`：RPC 模式（远程控制，程序化接口）

对比 crush：只有 interactive + nonInteractive 两种，没有 RPC 模式。

### 5.3 agent 循环设计

pi 的 `packages/agent/src/types.ts`：
- `ToolExecutionMode`：`"sequential" | "parallel"`——工具可并行执行
- `QueueMode`：`"all" | "one-at-a-time"`——队列消息注入策略可配
- `BeforeToolCallResult`/`AfterToolCallResult`：工具调用前后钩子（可 block/replace）
- `ThinkingLevel`：7 级思考深度（off/minimal/low/medium/high/xhigh/max）
- `AgentTool<TParameters>`：泛型工具接口，带更新回调

对比 crush：工具顺序执行（除 parallel agent tool 外）、无思考级别、无工具前后钩子。

### 5.4 harness 分离

pi 的 `packages/agent/src/harness/`：
- `agent-harness.ts`：agent 运行容器
- `compaction/`：harness 层压缩（与 coding-agent 层分离）
- `session/`：harness 层会话管理
- `tools/`：harness 层工具集
- `system-prompt.ts`：harness 层 prompt 模板
- `skills.ts`：harness 层技能

对比 crush：所有逻辑塞在 `agent.go` 的 Run 函数（764 行）里，没有 harness 分离。

### 5.5 模型管理

pi 的 `packages/coding-agent/src/core/`：
- `model-config.ts`：模型配置
- `model-registry.ts`：模型注册表
- `model-resolver.ts`：模型解析器（支持 per-agent 模型）
- `model-runtime.ts`：模型运行时
- `models-store.ts`：模型存储

对比 crush：只有 large/small 两个全局槽位，无 per-agent 模型（我们的 PR 在解决这个问题）。

### 5.6 pi 生态（社区已建的大量扩展）

| 仓库 | stars | 功能 |
|---|---|---|
| `agegr/pi-web` | 2205 | Web UI |
| `badlogic/pi-skills` | 2235 | 技能包（兼容 Claude Code + Codex CLI） |
| `nicobailon/pi-mcp-adapter` | 1045 | MCP 适配器 |
| `svkozak/pi-acp` | 530 | ACP 适配器 |
| `nicobailon/pi-web-access` | 867 | 网页搜索扩展 |
| `minghinmatthewlam/pi-gui` | 692 | Electron GUI |
| `nicobailon/pi-messenger` | 651 | 多 agent 通信扩展 |
| `nicobailon/pi-powerline-footer` | 345 | Powerline 状态栏 |
| `jshachm/pi-rs` | 605 | Rust 轻量化版本 |

---

## 6. P0/P1 需求与 pi 对比

| 需求 | crush 现状 | pi 已有 | 我们要做 |
|---|---|---|---|
| 上下文不丢 | 摘要后全丢 | 三层压缩 + 会话分叉 | 抄 pi 的 compaction 设计 |
| 工具调用可靠 | Run 函数耦合 | 工具前后钩子 + 并行执行 | harness 分离 + 钩子 |
| 多模型分级 | 无（我们 PR 在解决） | per-agent 模型 + 7 级思考 | per-agent + 动态传参 |
| 权限可控 | 全有/全无 | — | 细粒度 allowlist |
| 后台执行 | 无 | — | 异步 sub-agent |
| 成本可见 | Copilot $0 | — | per-step 成本 |
| provider 友好 | 配错就挂 | model-registry + resolver | 配置校验 + 自动发现 |
| RPC 模式 | 无 | 有 | 后续 |
| 会话分叉 | 无 | branch-summarization | 抄 pi 设计 |

---

## 7. 当前仓库状态

- **仓库**：`sdgumdam/crushforyou`（非 fork，独立仓库）
- **LICENSE**：MIT（但代码仍是 FSL derivative，需全部重写）
- **CI**：`.github/workflows/build.yml` + `lint.yml`
- **分支**：main
- **本地路径**：`/Users/jingyuanrunsen/Project/AgentManager/crushforyou`
- **crush 参考仓库**：`/Users/jingyuanrunsen/Project/AgentManager/crush`（feat/sub-agent-pinned-models 分支有我们之前的多模型 PR）
- **参考文档**：
  - `knowledge/crushforyou测试护栏梳理.md`
  - `knowledge/crushforyou-TUI层映射与测试复用.md`
  - `knowledge/crush上游规范参考.md`
  - `knowledge/crush架构参考.md`
  - `knowledge/cli-agent-业务需求文档.md`

## 8. 下一步

1. **删除 crush 的 FSL 代码**（internal/ 全部 + 根目录 .go）
2. **保留 go.mod**（依赖 charm.land MIT 库）
3. **定义渲染层接口**（ToolRenderer/Versioned/Dialog/DiffView/list.Item）
4. **从 18 个可复用测试提取断言点写新测试**
5. **TDD 驱动实现**
