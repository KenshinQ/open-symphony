# Symphony TypeScript 移植方案

## 背景 / 目标

Symphony 是一个多代理编排服务：持续轮询 Linear 看板，为每张 ticket 创建隔离工作区，并在工作区内运行 Codex App-Server 编码代理。当前仓库 `d:\symphony\open-symphony` 中 `src/` 目录为空，仅有语言无关的 [SPEC.md](file:///d:/symphony/open-symphony/SPEC.md)（2 111 行）和配置文件 [WORKFLOW.md](file:///d:/symphony/open-symphony/WORKFLOW.md)。本方案在该仓库下使用 **TypeScript + Node.js** 从零实现符合规范的服务。

---

## 技术栈选型

| 关注点 | 选择 | 理由 |
|---|---|---|
| 运行时 | **Node.js 20 LTS** | 无需额外依赖，子进程/文件系统 API 完备 |
| 语言 | **TypeScript 5** | 严格类型覆盖所有域模型；与 Codex / eslint 生态契合 |
| 包管理 | **npm workspaces** | 仓库自身已使用 npm |
| YAML 解析 | **js-yaml** | 解析 WORKFLOW.md front matter |
| Liquid 模板 | **liquidjs** | 严格模式支持；未知变量/过滤器报错 |
| HTTP 服务 | **fastify** | 轻量、类型友好；用于可选 Dashboard API |
| 测试 | **vitest** | 速度快；无需 Babel/Babel-register；与 TS 开箱即用 |
| 日志 | **pino** | 结构化 JSON 日志；支持多 sink；高性能 |
| 文件监听 | **chokidar** | 跨平台 WORKFLOW.md 热重载 |
| GraphQL 请求 | **node fetch（内置）** | 无额外依赖调用 Linear API |

---

## 目录结构（规划）

```
src/
  config/         # Workflow 加载 + 类型化 Config 层
  tracker/        # Linear Issue Tracker 适配器
  workspace/      # 工作区生命周期管理
  runner/         # Agent Runner（App-Server 子进程协议）
  orchestrator/   # 调度主循环、并发、重试、协调
  server/         # 可选 HTTP Dashboard（/api/v1/*）
  observability/  # 结构化日志 + Snapshot
  cli.ts          # 入口：解析参数、启动服务
  types.ts        # 共享域模型类型（Issue、Session、RetryEntry …）
tests/
  unit/           # 纯函数单元测试（vitest）
  integration/    # 集成测试（真实 Linear 凭证可选）
```

---

## 各层设计概要

### 1. 域模型（`src/types.ts`）

定义 SPEC §4 所有实体的 TypeScript 接口：
- `Issue`, `WorkflowDefinition`, `ServiceConfig`
- `Workspace`, `RunAttempt`, `LiveSession`, `RetryEntry`
- `OrchestratorState`（单一可变对象，所有调度操作通过 orchestrator 串行执行）

---

### 2. Config 层（`src/config/`）

#### [NEW] `workflowLoader.ts`
- 解析 [WORKFLOW.md](file:///d:/symphony/open-symphony/WORKFLOW.md)：提取 YAML front matter（js-yaml）+ prompt body（liquidjs 模板对象）
- 返回 `WorkflowDefinition`；缺文件 → `missing_workflow_file`；YAML 非 map → `workflow_front_matter_not_a_map`

#### [NEW] `serviceConfig.ts`
- 将 raw config map 转换为强类型 `ServiceConfig`
- 支持所有默认值（见 SPEC §6.4）
- `$VAR` 环境变量展开、`~` home 展开
- 暴露校验函数：`validateDispatchConfig()`

#### [NEW] `workflowWatcher.ts`
- 用 **chokidar** 监听 [WORKFLOW.md](file:///d:/symphony/open-symphony/WORKFLOW.md)
- 变更时回调 `onReload`，失败时保留上一次有效配置并输出 error 日志

---

### 3. Issue Tracker 适配器（`src/tracker/`）

#### [NEW] `linearClient.ts`
- `fetchCandidateIssues()` — 分页（默认 50 条/页），按 `active_states` + `project.slugId` 过滤
- `fetchIssuesByStates(stateNames[])` — 用于启动清理
- `fetchIssueStatesByIds(ids[])` — 用于协调 reconciliation
- 全部操作的 GraphQL 查询与 SPEC §11 对齐；网络超时 30 s
- `normalizeIssue()` — SPEC §11.3 规范化（lowercase labels、blockers、timestamp 解析）

---

### 4. 工作区管理（`src/workspace/`）

#### [NEW] `workspaceManager.ts`
- `sanitizeKey(identifier)` — 只保留 `[A-Za-z0-9._-]`
- `createForIssue(identifier)` — 确保目录存在；返回 `{path, createdNow}`
- `removeForIssue(identifier)` — 运行 `before_remove` hook 后删除目录
- 路径安全检查 (Invariant 1-3, SPEC §9.5)

#### [NEW] `hookRunner.ts`
- `runHook(hookScript, cwd, timeoutMs)` — `sh -lc` 或 `bash -lc` 执行；Windows 上改用 `cmd /c`
- 超时 kill；`after_create`/`before_run` 失败为 fatal；`after_run`/`before_remove` 失败仅 log

---

### 5. Agent Runner（`src/runner/`）—  长时任务核心

#### 关键设计：进程生命周期

Elixir 版通过 `Erlang Port` 保持 Codex 子进程在整个 session 存活，仅在 session 结束时 `stop_port`。Node.js 移植采用相同原则：

```
session 开始
  └─ spawn child_process (bash -lc codex app-server)
       ├─ turn 1: send turn/start → readline 读取事件流 → turn/completed
       │     ↓ 检查 issue 状态，仍 active 且 turn < max_turns
       ├─ turn 2: 同一子进程，复用 thread_id，发 turn/start
       │     ↓ ...
       └─ session 结束: kill(subprocess) + after_run hook
```

**关键实现点**：

- 子进程用 `child_process.spawn()` 启动，不复用（每次 `startSession` 启动新进程，一次 session 只有一个进程）
- stdout 通过 `readline.createInterface` 逐行读取，支持最大 10 MB 行大小（SPEC §10.1）
- stderr 单独监听仅打 debug 日志，不解析协议帧
- 每轮 turn 独立设置 `turn_timeout_ms` 计时，用 `AbortSignal` + `clearTimeout` 管理；stall 超时由 orchestrator 层统一检测
- turn_timeout 触发时：向 orchestrator 报告 `turn_timeout` 事件，然后 kill 子进程

#### [NEW] `appServerClient.ts`

```typescript
interface AppServerSession {
  proc: ChildProcess;       // 在 startSession → stopSession 全程存活
  rl: readline.Interface;   // stdout 行读器
  threadId: string;         // start_thread 后确定，复用于所有 turns
  workspace: string;
}

// 生命周期
startSession(workspace): AppServerSession
  → validateWorkspaceCwd()     // 安全检查
  → spawn(bash, ['-lc', codex.command], { cwd: workspace })
  → readline.createInterface(proc.stdout)
  → sendInitialize() → sendInitialized() → sendThreadStart()
  → 返回 session（含 threadId）

runTurn(session, prompt, issue, onMessage): Promise<TurnResult>
  → sendTurnStart(threadId, prompt, issue)
  → 设置 AbortController（turn_timeout_ms）
  → runReceiveLoop(rl, abortSignal, onMessage)
    ├─ 每行 JSON → dispatch handler
    ├─ turn/completed → resolve
    ├─ turn/failed / turn/cancelled → reject
    ├─ item/commandExecution/requestApproval → auto-approve（send result）→ continue loop
    ├─ item/fileChange/requestApproval → auto-approve → continue loop
    ├─ item/tool/call → executeTool() → send result → continue loop
    ├─ item/tool/requestUserInput → non-interactive answer → continue loop
    └─ abort signal 触发 → reject(turn_timeout)

stopSession(session): void
  → session.proc.kill('SIGTERM')   // 优雅退出
  → session.rl.close()
```

#### [NEW] `agentRunner.ts`

- `runAgentAttempt(issue, attempt, onEvent)` — 完整一次 worker 执行
- 多轮循环：`startSession` 一次，`runTurn` 多次，最后 `stopSession`
- 每轮后用 `fetchIssueStatesByIds` 刷新 issue 状态，决定继续或退出
- 任何异常 → `after_run` hook（best-effort）→ throw（由 orchestrator 捕获）

---

### 6. 编排器（`src/orchestrator/`）

#### [NEW] `orchestrator.ts`
- 单例，持有 `OrchestratorState`
- `start()` → 校验配置 → 启动清理 → `scheduleTickImmediate()`
- `onTick()` → reconcile → validate → fetch → sort → dispatch
- `dispatchIssue()`, `onWorkerExit()`, `onRetryTimer()`, `onCodexUpdate()`
- 并发控制：全局 + per-state（SPEC §8.3）
- 重试退避：continuation=1 000 ms；failure=`min(10000×2^(attempt-1), max_retry_backoff_ms)`
- Stall 检测：每 tick 检查 `last_codex_timestamp`

---

### 7. HTTP Dashboard（`src/server/`, 可选）— 实时推送设计

Elixir 版使用 **Phoenix LiveView（WebSocket）+ PubSub** 实现服务端推送。  
TypeScript 移植等价方案：**Server-Sent Events（SSE）+ 进程内 EventEmitter**。

#### 为什么选 SSE 而非 WebSocket / 轮询

| 方案 | 优点 | 缺点 |
|---|---|---|
| 客户端轮询 | 实现最简单 | 延迟高、浪费请求；Dashboard 跳字感 |
| WebSocket | 双向实时 | Dashboard 只需单向推送，复杂度过高 |
| **SSE（选用）** | 单向推送、原生 HTTP/1.1 + HTTP/2 支持、自动重连 | 单向 |

#### 架构设计

```
┌─────────────────────────────────┐
│  Orchestrator                   │
│  onStateChange() →              │
│    observabilityBus.emit('upd') │
└──────────────┬──────────────────┘
               │ EventEmitter（进程内，同步）
┌──────────────▼──────────────────┐
│  ObservabilityBus               │
│  (Node.js EventEmitter 单例)    │
└──────────────┬──────────────────┘
               │ fan-out
     ┌─────────┼─────────┐
     ▼         ▼         ▼
  SSE conn  SSE conn  SSE conn   （浏览器 dashboard tab）
```

**进程内 EventEmitter**（对应 Elixir PubSub）：
- `observabilityBus.ts` — 单例 `EventEmitter`；orchestrator 在每次状态变更后调用 `bus.emit('update')`
- 不引入 Redis / 消息队列；单机部署足够

**SSE 端点**（对应 Phoenix LiveView mount + subscribe）：
- `GET /api/v1/events`：Fastify 路由设置 `Content-Type: text/event-stream`，长连接保持不关闭
- 连接建立时立即发送一次当前快照（避免首屏空白）
- 订阅 `observabilityBus`；每次 `update` 事件 → `res.raw.write('data: {...}\n\n')`
- 连接断开（`req.raw.on('close')`）时取消订阅，防止内存泄漏
- Fastify 需禁用该路由的 response 超时：`reply.raw.setTimeout(0)`

**前端 Dashboard**（`GET /`，静态 HTML + 内联 JS）：

```html
<!-- 对应 Elixir LiveView render，用原生 JS 实现 -->
<script>
  const src = new EventSource('/api/v1/events');
  src.onmessage = (e) => render(JSON.parse(e.data)); // 增量更新 DOM
  src.onerror = () => updateStatusBadge('offline');   // 断线提示
</script>
```

- 每 1 s 更新运行时秒数（纯前端 `setInterval`，对应 Elixir `@runtime_tick_ms 1_000`）
- 状态指示徽章：绿色 Live / 红色 Offline（SSE 断线时自动切换）
- 表格列：Issue、State 色标、Session（Copy ID 按钮）、Runtime/Turns、最后事件、Tokens

#### [NEW] `httpServer.ts`

- `fastify` app 在 `127.0.0.1:<port>` 启动
- `GET /` — 内嵌 Dashboard HTML（SSE 客户端）
- `GET /api/v1/events` — SSE 长连接推送，对接 `observabilityBus`
- `GET /api/v1/state` — 快照 JSON（供调试/监控）
- `GET /api/v1/:identifier` — 按 issue identifier 查详情（404 on unknown）
- `POST /api/v1/refresh` — 触发立即 tick（202 Accepted）

#### [NEW] `observabilityBus.ts`

- `class ObservabilityBus extends EventEmitter` 单例
- `notifyUpdate()` — orchestrator 状态变更时调用
- `getSnapshot()` — 返回当前 `OrchestratorState` 的 JSON 快照
- 维护最近 N 条事件日志（可选，用于 `recent_events` 字段）

---

### 8. CLI 入口（`src/cli.ts`）

- `node dist/cli.js [path-to-WORKFLOW.md] [--port <n>]`
- 默认 workflow path = [./WORKFLOW.md](file:///d:/symphony/open-symphony/WORKFLOW.md)；不存在则非零退出
- 启动失败（config validation 失败）→ 非零退出 + error log
- 正常关机（SIGTERM/SIGINT）→ 零退出

---

### 9. 可选 `linear_graphql` 工具扩展

在 `thread/start` 时将工具声明注入 Codex session；
处理 `item/tool/call` 事件时验证 query（单操作限制）并转发给 Linear API，返回结构化结果。

---

## 项目配置文件

| 文件 | 内容 |
|---|---|
| `package.json` | scripts: `dev`, `build`, `start`, `test`, `lint` |
| `tsconfig.json` | `strict: true`, `target: ES2022`, `module: Node16` |
| `.eslintrc.json` | TypeScript + import 规则 |
| `vitest.config.ts` | 测试配置 |

---

## 验证计划

### 单元测试（vitest）

运行命令：
```bash
npm test
```

覆盖范围（新增测试文件）：

| 测试文件 | 覆盖内容 |
|---|---|
| `tests/unit/workflowLoader.test.ts` | front matter 解析、缺文件错误、非 map 错误、默认值 |
| `tests/unit/serviceConfig.test.ts` | `$VAR` 展开、`~` 展开、全部默认值 |
| `tests/unit/workspaceManager.test.ts` | 路径 sanitize、目录创建/重用、安全 invariants |
| `tests/unit/linearNormalize.test.ts` | label lowercase、blocker 提取、timestamp 解析 |
| `tests/unit/orchestrator.test.ts` | dispatch sort、并发控制、blocker 规则、重试退避公式 |
| `tests/unit/promptRenderer.test.ts` | 严格模式：未知变量报错、attempt 渲染 |

### 集成测试（可选，需真实凭证）

```bash
LINEAR_API_KEY=<your-key> npm run test:integration
```

检查：Linear candidate fetch、workspace 钩子执行、App-Server 握手。

### 手动冒烟测试

1. 配置 [WORKFLOW.md](file:///d:/symphony/open-symphony/WORKFLOW.md) 中的 `tracker.project_slug` 和 `LINEAR_API_KEY`
2. 运行 `npm run dev WORKFLOW.md --port 3000`
3. 访问 `http://localhost:3000` 确认 Dashboard 显示运行状态
4. 在 Linear 创建一张 Todo 状态 ticket，等待约 5 s，观察日志中出现 `dispatch issue`
5. 修改 [WORKFLOW.md](file:///d:/symphony/open-symphony/WORKFLOW.md) 中 `polling.interval_ms`，确认无需重启即生效（热重载）

---

## 实现顺序

1. `types.ts` + `config/` + `tracker/` — 核心域层
2. `workspace/` — 工作区与钩子
3. `runner/` — App-Server 子进程协议
4. `orchestrator/` — 主调度循环
5. `cli.ts` — 入口 + 日志配置
6. `server/` — 可选 HTTP Dashboard
7. 全量单元测试
