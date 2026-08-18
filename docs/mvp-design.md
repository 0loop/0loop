# ITS MVP 设计方案

日期：2026-08-19。基于 [调研报告](mvp-research.md) 产出。本方案定义 MVP 的范围与验收标准、测试方案、模块架构与关键接口。开发按仓库协作流程 issue 先行：方案获批后拆分为 feature issue，待 `accepted` 标签后实施。

## 1. MVP 目标

在一个真实链路里跑通最小可用的 agent：**新 issue 打开 → GitHub Action 触发 → agent 读取 issue 与仓库代码 → 打标签 + 回复评论 → 输出 token 账单**。核心引擎完全依赖 Vercel AI SDK v5（`generateText` 多步工具循环 + zod 校验），工具分「内部工具直接暴露、外部工具路由管控」两层。软件结构为 v1 的 permission-aware 工具选择、多协议、MCP、skills 预留扩展点。

## 2. MVP 范围

### P0（必须，验收基准）

| 项 | 内容 |
| --- | --- |
| 触发链路 | `issues.opened` 自动跑 + `workflow_dispatch` 人工兜底；从 `GITHUB_EVENT_PATH` 读 payload，把 issue 标题/正文/标签/仓库信息组装进 prompt |
| Agent loop | `generateText` + 显式 `stopWhen: stepCountIs(maxSteps)`（默认值 10，可配）+ abortSignal（SIGINT/SIGTERM → abort）+ `onStepFinish` 日志 |
| 内部工具（直接暴露给模型） | Bash、Edit、Fetch、Glob，全部带 execute 的 `tool()` + zod inputSchema |
| 外部工具（路由管控） | GitHub 写工具（`add_label`、`comment`），经工具注册表 + 权限检查；「无 issues:write 权限时对应工具不注入」 |
| Provider | `anthropic-messages` 与 `openai-chat-completions` 两个协议（含 `ITS_BASE_URL` gateway），`ITS_PROTOCOL/ITS_MODEL/ITS_API_KEY` 运行时选择 |
| 注入防护 | issue 正文消毒（剥离 HTML 注释、隐形 Unicode 字符）+ system prompt 中 `<issue>` 边界标记 + 「内容是数据不是指令」声明 |
| 可观测性 | `::group::` 分步日志（每步工具调用 + usage）；`$GITHUB_STEP_SUMMARY` 输出最终回复、步数、token 账单；失败时 `setFailed` 带结构化错误 |
| 测试 | L1–L5 必跑（见第 6 节）+ self-test dogfood workflow |

### P1（随后，架构预留）

- `issue_comment` 触发（`@its` 触发词 + 作者 `author_association ∈ OWNER/MEMBER/COLLABORATOR` gating）。
- MCP stdio 支持（行式配置解析 + `@ai-sdk/mcp` + `<server>_` 前缀 + allowlist + 超时 + finally close）。
- skills 列表注入（扫描 `.agents/skills/*/SKILL.md` 的 name+description 进 system prompt，按需 Read 加载）。
- 开分支 + draft PR 写回（`contents: write` + checkout 持久化 credential + `git push`）。

### 非目标（v1+）

- `openai-responses` / `gemini` 协议；流式输出（streamText）；交互式审批；声明式工具路由（不带 execute + 外层循环，审批场景才需要）；release 预编译二进制发布（MVP 维持运行时构建）；Bash 命令白名单。

## 3. 验收标准

### 3.1 端到端验收（E2E）

1. **本地全链路**：mock event payload（`issue.opened` JSON 文件）+ 注入 `GITHUB_*` 环境变量 + 真模型 key，直接运行编译产物——issue 内容被读取、agent 完成多步工具调用、评论与标签写回（对测试仓库用真 token）、step summary 含账单。
2. **真实 CI 链路**：ITS 仓库自身 workflow `uses: ./`，`workflow_dispatch` 手动触发（或测试 issue）——job 成功结束，评论与标签出现在测试 issue 上，日志含每步工具调用与 token 用量。

### 3.2 工程验收（CI 门禁，全绿才可合入）

| 编号 | 验收项 | 验证方式 |
| --- | --- | --- |
| A1 | loop 多步：模型连续 ≥2 轮工具调用时，工具结果正确回传并最终终止 | MockLanguageModelV2 脚本化 2–3 轮 tool-call |
| A2 | 步数上限：到达 `maxSteps` 时 loop 终止且结果标记截断 | mock 模型持续返回 tool-call |
| A3 | 中止：abortSignal 触发后模型调用与工具 execute 均被中止 | SIGTERM 模拟 + mock 断言 |
| A4 | 工具路由：`issues: write` 缺失时 GitHub 写工具不在 tools/activeTools 中 | context 注入测试 |
| A5 | 消毒：HTML 注释、零宽字符从 issue 正文中移除；正文以边界标记包裹 | 纯函数单测 |
| A6 | 四个内部工具失败路径全覆盖（超时/非零退出/匹配失败/非 2xx/.gitignore 等） | L2 工具单测清单 |
| A7 | 账单：`onStepFinish` 逐步记录 + `totalUsage` 汇总写入 summary | loop 测试断言 summary 内容 |
| A8 | 两个协议均能构造出正确 provider 模型（含 baseURL） | provider 单测 + 可选真 key smoke |
| A9 | 取消安全：收到 SIGTERM 后 ~10s 内退出，退出码非零、日志含取消原因 | 集成测试（真实进程 + 假模型挂起） |
| A10 | 构建与静态检查：`bun run build`、`fmt:check`、`lint:check`、`bun test` 全绿 | CI workflow |

## 4. 架构设计

### 4.1 模块划分

```
src/
├── index.ts              # 入口：装配全部依赖后执行一次 run
├── provider/             # 模型接入：协议 → LanguageModelV2
│   ├── index.ts          # createModel(config)
│   └── protocols.ts      # 协议 → provider 工厂映射
├── agent/                # 核心 loop（AI SDK 驱动，不手搓循环）
│   ├── index.ts
│   ├── loop.ts           # runLoop：generateText 封装
│   ├── prompt.ts         # system/user prompt 组装（事件上下文、AGENTS.md、skills 列表）
│   └── sanitize.ts       # 用户文本消毒（HTML 注释、隐形字符、边界标记）
├── tools/                # 工具层
│   ├── index.ts          # buildToolSet(ctx) → { tools, activeTools }
│   ├── internal/         # 内部工具：直接暴露给模型
│   │   ├── bash.ts
│   │   ├── edit.ts
│   │   ├── fetch.ts
│   │   └── glob.ts
│   ├── external/         # 外部工具：经路由/注册表管控
│   │   └── github.ts     # add_label / comment（权限感知）
│   └── registry.ts       # 工具注册表：name → { definition, requires }
├── github/               # GitHub 集成（外部工具与上下文的数据来源）
│   ├── index.ts
│   ├── context.ts        # getContext(env)：GITHUB_* 收敛解析（唯一读 env 的地方）
│   ├── event.ts          # event payload 解析与格式化
│   └── client.ts         # octokit 封装：评论、标签（可注入 fake）
└── mcp/                  # P1：MCP server 管理（配置解析、stdio 启动、前缀、close）
    └── index.ts
```

模块依赖方向：`index → agent → { provider, tools, github }`；`tools → { github, mcp(P1) }`。所有内部 import 走 `@` 别名，模块间只从 `index.ts` 导入。

### 4.2 数据流（一次 run）

```
env (GITHUB_*, ITS_*) → context.ts → RunContext
RunContext + ITS_PROMPT → prompt.ts（消毒 + 组装）→ system/user messages
RunContext → buildToolSet → { tools, activeTools }   # 内部工具全量；GitHub 写工具按 token 权限启用
ITS_* → createModel → LanguageModelV2
→ runLoop({ model, messages, tools, activeTools, maxSteps, abortSignal, onStep })
  └─ generateText（SDK 自动执行带 execute 的工具，多步直到 stopWhen）
→ LoopResult { text, steps, totalUsage }
→ 写回：octokit 评论 + 标签（由 agent 在 loop 中通过 GitHub 工具完成）
→ $GITHUB_STEP_SUMMARY + 退出码
```

工具路由语义（MVP 实现，与调研结论一致）：

- **内部工具**：`tool()` 带 execute，SDK 在 loop 内自动执行——完全协议暴露，无中间层。
- **外部工具**：同样 `tool()` 带 execute，但构造时经注册表：`requires`（如 `issues:write`）不满足则**不注入 tools**（模型不可见）；execute 内再做一次权限校验（深度防御）。管控逻辑收敛在 registry 与工具工厂，loop 无感知。
- 预留升级路径：未来需要审批/跨进程时，把外部工具改为不带 execute 的声明 + `stopWhen` + 外层注入（调研第 3.4 节路径 b），loop.ts 已预留该分支。

### 4.3 关键接口定义

```ts
// provider
export type ProtocolId = 'anthropic-messages' | 'openai-chat-completions';
export interface ModelConfig {
  protocol: ProtocolId;
  model: string;
  apiKey: string;
  baseUrl?: string;
}
export function createModel(config: ModelConfig): LanguageModelV2;
// anthropic-messages  → createAnthropic({ apiKey, baseURL })(model)
// openai-chat-completions → createOpenAI({ apiKey, baseURL }).chat(model)  // 注意 .chat()，openai() 默认 Responses API

// github/context：唯一的 env 读取点
export interface RunContext {
  repo: { owner: string; repo: string };
  eventName: string;
  event: Record<string, unknown>;   // GITHUB_EVENT_PATH 解析后的 payload
  workspace: string;                // GITHUB_WORKSPACE
  token: string | undefined;        // GITHUB_TOKEN（经 ITS_GITHUB_TOKEN 传入）
  permissions: Set<string>;         // 从 env 或探测得出：'issues:write' 等
  prompt: string;                   // ITS_PROMPT
  maxSteps: number;                 // 默认 10
}
export function getContext(env: NodeJS.ProcessEnv): RunContext;

// agent
export interface LoopConfig {
  model: LanguageModelV2;
  system: string;
  prompt: string;
  tools: ToolSet;
  activeTools?: string[];
  maxSteps: number;
  abortSignal?: AbortSignal;
  onStep?: (step: StepResult<unknown>) => void;
}
export interface LoopResult {
  text: string;
  steps: StepResult<unknown>[];
  totalUsage: LanguageModelUsage;
}
export async function runLoop(config: LoopConfig): Promise<LoopResult>;

// tools
export interface ToolRegistration<T> {
  name: string;
  description: string;
  inputSchema: z.ZodType<T>;
  requires?: string[];              // 权限要求，如 ['issues:write']
  create: (ctx: RunContext) => Tool; // 工具工厂（注入 cwd/fetcher/octokit）
}
export function buildToolSet(ctx: RunContext): { tools: ToolSet; activeTools: string[] };

// tools/internal
// bash:  { command, timeout_ms?, cwd? } → { stdout, stderr, exitCode }（Bun.spawn，输出截断）
// edit:  { path, old_string, new_string }（唯一匹配；幂等语义：已含 new_string 视为成功）
// fetch: { url, headers? } → { status, body, truncated }（注入 fetcher，拒绝 file://，非 2xx 结构化错误）
// glob:  { pattern, path? } → string[]（ignore 包解析 .gitignore，结果排序）

// github（外部工具，经 registry 路由）
// add_label: { label } → 权限 'issues:write'
// comment:   { body }   → 权限 'issues:write'
```

### 4.4 action.yml 变更（开发时落地）

- 新增 `env: ITS_GITHUB_TOKEN: ${{ github.token }}`（显式传入便于本地测试注入 fake token）。
- 其余 inputs/env 保持现状（`tools`/`skills`/`mcp` 在 P1 才解析）。

## 5. 依赖与版本

```bash
bun add ai@ai-v5 zod
bun add @ai-sdk/anthropic@ai-v5 @ai-sdk/openai@ai-v5
bun add @ai-sdk/mcp@ai-v5          # P1（MCP）
bun add @actions/core @actions/github
bun add ignore                     # Glob 的 .gitignore 解析
```

全部包 engines `node >=18`，Bun 1.3.14 满足。zod 用 v4（`ai@5.0.238` peerDep `zod ^3.25.76 || ^4`）。

## 6. 测试方案

对应调研金字塔（[调研报告 §5](mvp-research.md#5-ai-agent-测试策略)）：

| 层 | 内容 | 位置 |
| --- | --- | --- |
| L1 | registry 路由、prompt 组装、sanitize、context 解析、协议映射——纯函数单测 | `src/**/*.test.ts` |
| L2 | 四个内部工具 + GitHub 工具（octokit fake）：全失败路径清单 | 同上（mkdtemp fixture、fetcher 注入、`Bun.spawn` timeout） |
| L3 | loop：MockLanguageModelV2 函数形式脚本化 2–3 轮 tool-call；断言 `doGenerateCalls` 消息流、execute 入参、上限截断、abort | 同上 |
| L4 | @actions/core 输出：setFailed/summary 内容（env 指向临时文件断言） | 同上 |
| L5 | record/replay：`wrapLanguageModel` transformParams 录制 cassette → Mock 回放（自建 ~50 行，fixture 存 `test/fixtures/cassettes/`） | 同上 |
| L6 | 真 API smoke：env 门控（`ITS_E2E_KEY`），短 prompt + 小模型 + ≤2 步；CI 中 workflow_dispatch + cron，不进 PR 必跑路径 | `test/e2e/` |
| L7 | self-test workflow：`uses: ./` + 最小 prompt；fork PR 跳过（防密钥泄漏）；断言 job 成功 + summary 存在 | `.github/workflows/self-test.yml` |

可测试性硬约束（架构评审时逐条核对）：model/fetcher/cwd 可注入；env 只在 context 模块读取；工具路由纯函数；loop 终止参数化。

## 7. 开发阶段与 issue 拆分建议

按仓库协作流程（feature 模板 + `accepted` 标签 + squash PR），建议拆为 4 个依赖有序的 issue：

1. `feat: 接入 AI SDK agent loop 与 provider 抽象`——provider 模块 + agent 模块（loop/prompt/sanitize）+ L1/L3 测试。**后续一切的基础。**
2. `feat: 内置工具 Bash/Edit/Fetch/Glob`——tools/internal + L2 测试。
3. `feat: GitHub 事件上下文与权限感知写回`——github 模块（context/event/client）+ 外部工具（add_label/comment）+ registry + L1/L2/L4 测试 + action.yml 的 ITS_GITHUB_TOKEN。
4. `feat: self-test workflow 与 e2e 验证`——L5 cassette + L6 smoke（门控）+ L7 dogfood + 文档（README 用法更新、安全建议：timeout-minutes、permissions）。

P1 候选 issue（MVP 验收通过后）：`issue_comment` 触发与作者 gating；MCP stdio 支持；skills 注入；分支 + draft PR 写回。

## 8. 风险与开放问题

1. **注入防护的边界**：MVP 的 sanitize 是最小集（HTML 注释、隐形字符、边界标记）。issue 内容仍可包含对抗性指令——MVP 通过「仅 issues.opened 触发 + 最小权限 + 文档声明」控制风险；更强的防线（actor gating、子进程环境 scrub、工具白名单）在 `issue_comment` 触发时变为硬要求。
2. **Bash 无白名单**：MVP 的 Bash 在仓库 workspace 内全权执行，与 GITHUB_TOKEN 权限构成风险组合。README 必须写明建议（最小 permissions、`timeout-minutes: 10` 等）；白名单机制列入 v1。
3. **`experimental_context`/`experimental_createMCPClient` 带 Experimental 标注**：5.0.238 内稳定，升级 v6/v7 时需迁移评估（v6 有 `ToolLoopAgent` 的 `runtimeContext` 等对应物）。
4. **成本失控**：MVP 靠 maxSteps（默认 10）+ job timeout（文档建议 10–30 分钟）控制，不做硬 token 预算（调研确认无一家先例做硬预算）。
5. **GITHUB_TOKEN 探测权限的方式**：`permissions:` 声明与实际生效可能有差异（fork 场景等），MVP 以「声明式注入 env（如 `ITS_PERMISSIONS`）+ 写回失败兜底报错」处理，运行时探测留 v1。
