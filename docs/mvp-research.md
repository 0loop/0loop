# ITS MVP 调研报告

日期：2026-08-19。五路并行调研（GitHub Action 触发链路、Vercel AI SDK v5、MCP 生态与 CI 集成、AI agent 测试策略、同类项目先例与 skills 生态）的汇总。所有关键 API 均对照了实际发布的 npm 包类型声明、GitHub tag 上的官方文档原文或官方文档站点。

## 1. 摘要

1. **触发链路**：MVP 首选 `issues.opened`（受信输入、自动跑）+ `workflow_dispatch`（人工兜底）；`issue_comment` 是外来不可信输入，必须做作者 gating + 触发词，放第二阶段。写回用 `@actions/github` 的 octokit（评论/标签/开 PR），GITHUB_TOKEN 显式声明 `contents: write` + `issues: write` + `pull-requests: write` 即可覆盖全部 MVP 写回。
2. **AI SDK v5**：npm `latest` 已是 7.x，必须锁 `ai-v5` dist-tag（当前 5.0.238）。v5 中 `maxSteps` 已移除，**默认只跑一步**（`stopWhen: stepCountIs(1)`），多步循环必须显式设置 `stopWhen`。`toolRouter`/`toolProxy`/`handoffs` 是 v6 原语，v5 全系不存在；v5 的工具路由姿势是「不带 execute 的工具声明 + stopWhen + 外层检查 steps 手动注入工具结果」。
3. **MCP**：CI 里只做 stdio 传输 + npx/bunx 启动（托管 runner 无 Docker daemon）。`@ai-sdk/mcp@0.0.28` 的 `createMCPClient().tools()` 返回带 execute 的 ToolSet 可直接并入 `tools`，但无内置前缀参数——多 server 同名工具会静默覆盖，必须自己加前缀。
4. **测试**：`MockLanguageModelV2`（函数形式）+ 公开的 `doGenerateCalls` 数组可写出完全确定性的 loop 测试；注意 v5 数组形式有 off-by-one bug，必须用函数形式。nektos/act 支持 composite action，适合壳验证；真实行为靠仓库内 self-test workflow（dogfood）。
5. **先例**：`anthropics/claude-code-action` 是与 ITS 最同构的参照（composite + Bun + TypeScript），其 sanitizer、actor gating、permission-aware 工具选择、self-test workflow 均可直接借鉴。skills.sh 是 Vercel 的 skills 生态 CLI，`.agents/skills/` 目录选择与 OpenCode/Vercel 生态约定一致。

## 2. GitHub Action 触发链路与运行时

### 2.1 触发事件选择

| 事件 | 适用场景 | 关键事实 |
| --- | --- | --- |
| `issues` | 新 issue 自动分类/回答/打标签 | types 20 种；payload 含 `issue.title/body/number/labels/user` |
| `issue_comment` | 「@agent 帮我做 X」交互式 | **对 issue 和 PR 评论都触发**，用 `github.event.issue.pull_request` 判空区分；workflow 文件必须存在于默认分支 |
| `workflow_dispatch` | 人工/定时兜底 | 自定义 inputs 进 `github.event.inputs`（全为字符串） |
| `workflow_run` | 二段式：CI 跑完再执行特权动作 | 默认分支上下文，能访问 secrets 和 write token |

安全关键事实：fork PR 的 `pull_request` 事件 token 只读且 secrets 不传入，但 **fork PR 上的 `issue_comment` 事件在 base 仓库以完整权限 + secrets 运行**——这是先例们用它做 agent 入口的原因，代价是必须校验评论者身份（prompt 注入面）。

### 2.2 事件上下文与写回

- 进程内直接读 `GITHUB_EVENT_PATH`（完整 webhook payload 的 JSON 文件）、`GITHUB_EVENT_NAME`、`GITHUB_REPOSITORY`、`GITHUB_WORKSPACE`、`GITHUB_TOKEN` 等环境变量。
- `@actions/github` 的 `github.getOctokit(token)` 返回注入认证的 `@octokit/rest` 实例；`context.repo` 提供 `{owner, repo}`。
- 写回路由（已逐一探测确认）：`rest.issues.createComment`（issue 与 PR 通用）、`rest.issues.addLabels`、`rest.pulls.create`（同仓库分支开 PR）、`rest.pulls.createReview`。
- 开分支推代码：`actions/checkout` 默认把 token 持久化到本地 git config，`git push` 直接用 GITHUB_TOKEN 认证。
- **GITHUB_TOKEN 触发的动作不级联触发新 workflow**；GITHUB_TOKEN 创建的 PR 其 CI run 处于 approval-required 状态（需有写权限的人点批准才跑）——对 MVP 是合理的默认安全边界。

### 2.3 权限模型

- `permissions:` 一旦显式声明任一权限，其余未声明的全部置为 `none`。`write` 包含 `read`。
- MVP 写回（评论 + 标签）只需 `issues: write`；开分支/PR 需要 `contents: write` + `pull-requests: write`。
- 默认权限：新仓库只读；fork `pull_request` 强制只读、无 secrets。
- 需要跨仓库权限时才换 PAT/GitHub App token；首版不需要。

### 2.4 runner 限制与取消语义

- 单 job 硬上限 6 小时（`timeout-minutes` 默认 360）；私有仓库 Linux x64 $0.006/min。
- **取消时**：runner 对步骤入口进程发 SIGINT → 7.5s → SIGTERM → 2.5s → kill 进程树。agent 主进程必须捕获 SIGINT/SIGTERM，在 ~10 秒宽限期内 abort 当前模型调用并落 checkpoint。
- 产出设施：`$GITHUB_STEP_SUMMARY`（每 step 1MiB 上限）、`$GITHUB_OUTPUT`、artifact 上传。

### 2.5 形态取舍：composite + 运行时构建 vs release 预编译

- JS action 只支持 node20/node24 单文件，Bun 自编译二进制无法作为 JS action 跑；Docker action 只支持 Linux runner 且开销大。
- **composite 是唯一能把「装 Bun → 构建 → 跑二进制」串起来的形态**；`$GITHUB_ACTION_PATH` 仅 composite 支持，指向 action 自身源码目录。
- 运行时 `bun install --frozen-lockfile` 每次 job 多花 10–20s 且引入网络/供应链依赖；演进方向是 release 预编译二进制（`bun build --compile --target=bun-linux-x64`）+ 移动 major tag（`v1`）。**MVP 可维持现状（运行时构建），发布优化留到 v1。**

### 2.6 如何测试 GitHub Action

- **nektos/act**：支持 composite action、任意事件（`-e payload.json`）、workflow_dispatch inputs；但 **GITHUB_TOKEN 不自动可用**（`act -s GITHUB_TOKEN="$(gh auth token)"`），不能真实调用 GitHub API。
- **直接跑编译产物 + 注入 GITHUB_\* 环境变量 + mock event payload**：比 act 更贴近生产，可全链路跑通（octokit 用 fake token 或对测试仓库用真 token）。
- **仓库内 dogfooding**：workflow 里 `uses: ./` 引本地 action，加一个 smoke workflow 零成本验证。
- `@actions/core` 的输出本质是写 `$GITHUB_OUTPUT`/`$GITHUB_STEP_SUMMARY` 文件，单测把 env 指向临时文件断言内容即可。

## 3. Vercel AI SDK v5

### 3.1 版本锚定

- 5.x 最新 `ai@5.0.238`（npm dist-tag `ai-v5`，5.0.0 发布于 2025-07-31）；npm `latest` 已是 7.x，**必须用 `ai-v5` dist-tag 或精确版本安装**。
- 配套：`@ai-sdk/openai@2.0.118`、`@ai-sdk/anthropic@2.0.95`、`@ai-sdk/openai-compatible@1.0.48`、`@ai-sdk/mcp@0.0.28`（同样用 `ai-v5` dist-tag）。
- 全部包 engines `node >=18`，Bun 1.3.14 满足；CLI + generateText 场景无已知阻塞问题。

### 3.2 核心 loop（generateText）

```ts
generateText({
  model, system, prompt, messages,
  tools, toolChoice,             // toolChoice: 'auto'(默认)|'none'|'required'|{ type:'tool', toolName }
  stopWhen,                      // 默认 stepCountIs(1)——只跑一步！多步循环必须显式设置
  activeTools,                   // 数组：本调用可用的工具名
  prepareStep,                   // 按步裁剪工具集等
  maxOutputTokens,               // v4 的 maxTokens 已改名
  maxRetries, abortSignal, headers,
  onStepFinish,                  // 每步结束回调（日志、用量）
  experimental_context,          // 透传给工具 execute 的 options.experimental_context
  experimental_output,
})
```

- 多步语义：设置 `stopWhen` 后模型产生工具调用时，SDK 自动把工具结果传入下一轮生成，直到无工具调用或满足停止条件。
- `abortSignal` 中止整个 loop 并**转发给每个工具的 execute**——GitHub runner 取消的接法：捕获 SIGINT/SIGTERM → `controller.abort()`。
- `result.steps`（每步 text/toolCalls/toolResults/usage）、`result.response.messages`（可直接复用的消息数组）、`result.totalUsage`（全部步累计）。
- 选型：**MVP 用 `generateText`**（Action 日志非交互，无流式需求；直接拿 steps/totalUsage 做账单与续跑），`onStepFinish` 打 `::group::` 日志。

### 3.3 工具定义

```ts
const bash = tool({
  description: '...',            // 直接影响模型选择
  inputSchema: z.object({ ... }),// zod 参数校验
  outputSchema: z.object({ ... }),// 可选，校验与类型推断
  execute: async (input, { toolCallId, messages, abortSignal, experimental_context }) => { ... },
});
```

- **不带 execute 的 tool 是合法的**：模型可调用、SDK 不执行、循环在该步停下（finishReason `'tool-calls'`）——官方定位即「转发到客户端/队列执行」，这是 v5 工具路由的接入点。
- `dynamicTool` 用于运行时 schema 未知的工具（MCP 工具、用户自定义）。

### 3.4 工具路由：v5 有什么、没有什么

**v5 全版本（5.0.0 / 5.0.238 / 5.1.0-beta.28）均不存在 `toolRouter`、`toolProxy`、`declareTool`、`handoffs`、`experimental_toolRouting`**（case-insensitive 全量 grep 0 命中）——这些是 v6（`ToolLoopAgent`）才稳定的原语。v5 拥有的路由相关能力：

| v5 原语 | 语义 |
| --- | --- |
| `tool()` 不带 `execute` | 声明式路由：模型可调用，SDK 不执行，外层接管 |
| `stopWhen`（`stepCountIs`/`hasToolCall`/自定义） | 声明停止条件，检查 `steps[last].toolCalls` |
| `activeTools: ['a','b']` | 按次调用裁剪可见工具集 |
| `experimental_context` | 把运行时上下文注入工具 execute |

两条落地路径：
- **（a）带 execute 直接参与 loop**：权限检查、超时等管控逻辑放在 execute 内部——loop 完全由 SDK 驱动，零手搓。
- **（b）声明式 + 外层循环**：对外部工具只声明 `tool({ description, inputSchema })` 不带 execute；generateText 在该步停下后，外层检查 `result.steps.at(-1).toolCalls`，路由到注册表执行（审批/懒加载/MCP 启动），把结果作为 `{ role: 'tool', content: [{ type: 'tool-result', toolCallId, toolName, output }] }` 追加进 messages 再次调用。适合需要「人审批」或「跨进程转发」的场景。

`experimental_agent`（`Agent` 类）只是 settings 封装（`generate()/stream()/respond()`，无 stop()/handoffs），**MVP 不建议用**，直接 generateText。

### 3.5 MCP 集成（@ai-sdk/mcp@0.0.28）

```ts
import { experimental_createMCPClient as createMCPClient } from '@ai-sdk/mcp';
import { Experimental_StdioMCPTransport as StdioClientTransport } from '@ai-sdk/mcp/mcp-stdio';

const mcpClient = await createMCPClient({
  transport: new StdioClientTransport({ command: 'bunx', args: [...], env: { GITHUB_TOKEN } }),
});
const tools = await mcpClient.tools();  // McpToolSet，每个工具带 execute，可直接进 generateText
```

- 创建 client 时完成 initialize 握手；`tools({ schemas })` 可取白名单子集。
- **无内置 prefix/namespace 参数**，多 server 同名工具 spread 合并时后者静默覆盖——必须自己加 `<server>_` 前缀。
- 工具执行抛错走 `tool-error` content part（`steps` 里可过滤 `type === 'tool-error'`），模型可自愈，loop 不崩；stdio 进程整体退出后 client 无法自恢复，需重建 transport。
- 生命周期：`finally` 里 `client.close()`，否则子进程泄漏挂起 CI job。

### 3.6 Provider 抽象

```ts
createAnthropic({ apiKey, baseURL })(modelId);                    // Messages API
createOpenAI({ apiKey, baseURL }).chat(modelId);                  // 注意：openai() 默认走 Responses API，必须 .chat()！
createOpenAICompatible({ name, baseURL, apiKey }).chatModel(modelId); // 第三方 OpenAI 兼容端点
```

映射到 ITS 环境变量：`ITS_PROTOCOL`（`anthropic-messages` / `openai-chat-completions` / `openai-responses` / `gemini`）+ `ITS_MODEL` + `ITS_API_KEY` + `ITS_BASE_URL`。`LanguageModelV2` 是自定义 provider 的契约接口，ITS 无需自实现——`createOpenAICompatible` 已覆盖 gateway 场景。

### 3.7 用量与可观测性

- `LanguageModelUsage`：`{ inputTokens, outputTokens, totalTokens, reasoningTokens?, cachedInputTokens? }`（v5 起 `totalTokens` 必填）。
- 账单：`onStepFinish(stepResult)` 里 `stepResult.usage` 是该步用量；`result.totalUsage` 是累计。

## 4. MCP 生态与 CI 集成

### 4.1 协议现状

- 当前 spec 版本 2025-11-25（Tasks 抽象、Streamable HTTP 成为规范远程传输）；HTTP+SSE 已弃用。
- **CI 里选 stdio**：无端口、无鉴权面、生命周期跟 agent 进程绑定；坑是进程管理（崩溃检测、stdout 只能走协议消息）和挂起无心跳（需客户端超时兜底）。
- 托管 runner **无 Docker daemon**——github-mcp-server 官方推荐的 docker 方式跑不了，用 stdio 或远程 HTTP。

### 4.2 配置形态

- Claude Code 的 `.mcp.json`（`mcpServers` → `command/args/env` + `${VAR}` 占位符）是社区事实标准。
- ITS 的「每行一个启动命令 + `$VAR` 占位符」覆盖 90% 场景，建议 MVP 行式为主、允许 `allow:` 后缀、预留 JSON 对象兼容解析。
- 启动器：npx/bunx 优先（runner 预装 Node、ITS 自带 Bun）；uvx 需 setup-uv 且注意 `UV_EXCLUDE_NEWER` 缓存坑。
- 推荐的 4 个 server：github（`gh mcp` 或远程 Streamable HTTP + Bearer）、filesystem（只开读工具）、fetch、context7。

### 4.3 安全

- MCP server 是任意外部代码：工具名/描述/返回内容全部进入模型上下文（tool poisoning、供应链投毒是真实攻击面）。
- 收敛做法：**工具 allowlist fail-closed**（denylist 挡不住变体）、**锁版本**（`npx -y pkg@1.2.3`）、凭证只经显式 env 注入（绝不写进 prompt）、工具名加 server 前缀、write 类工具默认不给（github server 用 `--read-only`）。

## 5. AI agent 测试策略

### 5.1 AI SDK 官方测试设施（v5，已核对源码）

- `ai/test` 导出 `MockLanguageModelV2`（另有 Embedding/Image/Provider/Speech/Transcription 的 mock）、`mockId`、`simulateReadableStream`、`convertArrayToReadableStream` 等。
- **`doGenerateCalls` / `doStreamCalls` 是公开数组**，每次调用把完整 `LanguageModelV2CallOptions` push 进去——可直接断言「模型每一轮收到了哪些消息和工具结果」。
- ⚠️ **v5 数组形式有 off-by-one bug**（首元素被跳过，5.0.238 仍存在）：脚本化多轮对话用**函数形式 + 闭包计数器**。
- `TestingLanguageModel` 在 v5 不存在（ai 2.x/3.x 的 API）。
- v5 无内置 record/replay：录制用 `wrapLanguageModel` 的 `transformParams` 钩子（params 含完整消息历史），回放把录制的响应喂给 MockLanguageModelV2——自建约 50 行，零依赖。

### 5.2 建议测试金字塔（L1–L7）

| 层 | 测什么 | 设施 | CI 必跑 |
| --- | --- | --- | --- |
| L1 纯逻辑单测 | 工具路由、prompt 组装、终止判定、context 解析 | bun test，无 mock | ✅ |
| L2 工具单测 | Bash/Edit/Fetch/Glob 全失败路径 + 幂等性 | mkdtemp fixture + fetcher 注入 + `Bun.spawn` timeout | ✅ |
| L3 loop 测试 | 伪造模型 tool-call 序列 → 断言 execute 被调、tool-result 回传、轮数上限 | `MockLanguageModelV2`（函数形式）+ `doGenerateCalls` | ✅ |
| L4 action 壳测试 | @actions/core 输出、context 读取 | spyOn(@actions/core)；或 `github-action-ts-run-api` | ✅ |
| L5 record/replay | 真实模型消息格式回归 | 自建 cassette + Mock 回放 | ✅ |
| L6 真 API smoke | 端到端最小链路 | env 门控 + 小模型 + 短 prompt | ⚠️ 可选（cron/manual） |
| L7 workflow e2e | action.yml 结构、构建产物可跑 | nektos/act（壳）+ self-test dogfood workflow | ⚠️ self-test 进 PR 门禁 |

### 5.3 可测试性架构硬约束

1. **model 可注入**：agent 入口接受 `model: LanguageModelV2` 参数（默认从 env 构造真实 provider），loop 内不 new provider。
2. **fetcher 可注入**：Fetch 工具构造参数 `fetchFn?: typeof fetch`。
3. **cwd 可注入**：Bash/Edit/Glob 基于参数传入的 cwd，工具内部不读 `process.cwd()`。
4. **env 读取收敛**：所有 `GITHUB_*`/`INPUT_*` 只在 context 模块读取（`getContext(env = process.env)`）。
5. **工具路由是纯函数**：`toolName + input → 执行器选择 + 参数校验`，无 I/O。
6. **loop 终止条件参数化**：最大步数、超时、错误策略经配置注入。

### 5.4 Bun test 注意点

- `mock.module` 支持 live bindings（已 import 的模块也能生效）；`mock.restore()` 不重置 `mock.module` 覆盖。
- 全局 fetch mock 优先用注入（`globalThis.fetch` 直接赋值在 Bun 上不可靠）。
- 子进程用 `Bun.spawn`（原生 `timeout`/`killSignal`/`env`/`cwd`）。
- 工具测试清单要点：Bash（超时、非零退出码、stderr、无 shell 模式 argv 直传防注入、输出截断）；Edit（old_string 唯一匹配、幂等语义、0/多匹配报错、行号越界）；Fetch（非 2xx、网络错误、URL 规范化拒绝 file://、body 截断）；Glob（**fast-glob 不自动读 .gitignore**，需 `ignore` 包解析各级 .gitignore 后传入 ignore 选项 + `dot: true`，或用 `Bun.Glob` 自建；结果排序保证确定性）。

## 6. 同类项目先例

### 6.1 anthropics/claude-code-action（最核心参照，与 ITS 最同构）

- composite action + TypeScript + Bun；事件驱动（issue_comment `@claude` 提及 / issue opened / PR / schedule）+ automation mode（prompt input）。
- GitHub 能力实现为四个自研 **MCP server**（comment/file-ops/inline-comment/actions），而非散装自定义工具。
- 安全样板（docs/security.md + `src/github/`）：`sanitizer.ts` 剥离 HTML 注释/隐形字符/图片 alt text/隐藏 HTML 属性/HTML 实体；默认只有 write 权限用户可触发 + bot 拦截 + `allowed_non_write_users`（触发时 scrub 子进程环境 + 沙箱）；pull_request_target 下恢复配置文件防 pwn-request。
- 结构：顶层 composite action → `base-action/`（装运行时 + 跑 agent，无权限逻辑）→ 上层 `src/`（entrypoints / github(api, context, validation, sanitizer) / mcp / modes）。
- 事故史：2026 年 CVSS 9.4 "Comment and Control" 注入（issue 内容窃取 API key）、bot 名 `[bot]` 后缀绕过鉴权（CVSS 7.8）——注入防护和 actor 校验不是可选项。

### 6.2 其他先例

- **qodo-ai/pr-agent**：Docker action，PR 事件 + 评论命令（`/review` 等）；`if: github.event.sender.type != 'Bot'` 防自循环。反面教材：Kudelski 披露其 `/ask` 把用户评论直接拼进 prompt 导致注入。
- **OpenHands resolver**：issue 加 `fix-me` 标签触发，沙箱内 git 操作，必须 PAT。教科书 bug（#7860）：token 识别逻辑忽略 GITHUB_TOKEN 导致写回失败——**ITS 必须原生兼容 GITHUB_TOKEN**。
- **gptme bot**：`@gptme` 评论触发；分层授权（agent 只能评论、只能开 draft PR、永不 merge）+ 用户 allowlist；失败时脏树推上分支 + artifacts 保底；复用脚本从远端固定 commit 下载并校验 SHA256。
- **aider 系列**：issue 加标签 → 跑 CLI → push 分支 → 开 PR；文档明确建议 `timeout-minutes: 10` 防烧钱。
- **GitHub Copilot coding agent**（官方托管）：仓库只读、只能推 `copilot/` 前缀分支、开 draft PR、不接触 Actions secrets——自托管 agent 可借鉴其「只读 + 前缀分支 + draft PR」约束思路。

### 6.3 MVP 阶段划分共性

1. **Scaffold**：action 能装上运行时、读到 prompt（ITS 现在的位置）。
2. **MVP v0**：单事件 + 单写回（评论级）+ 单协议 + GITHUB_TOKEN，最小权限；触发显式授权（label/mention）；处理外来内容时「建议给人」或严格消毒。
3. **v1**：第二种事件/写回、permission-aware tool selection、超时与预算上限。
4. **v2+**：多 provider、MCP、skills 生态、App 托管。

## 7. skills 生态

- **skills.sh 是 Vercel 的开放 skills 生态**（目录网站 + `npx skills` CLI，仓库 vercel-labs/skills）。`npx skills add <owner/repo>` 实现 = git 拉取 + 扫描 `SKILL.md` + 复制到 agent skills 目录。
- skill 本质 = **带 YAML frontmatter（`name` + `description`）的 markdown 指令 + 可选 scripts/ 目录**，遵循 Agent Skills 开放标准。
- `.agents/skills/` 是 OpenCode 与 Vercel skills 共同认可的跨 agent 目录——ITS 的选择与生态一致。
- **MVP 最小注入**（零新工具）：扫描 `.agents/skills/*/SKILL.md` 解析 name+description → system prompt 注入「可用 skills 列表」→ agent 需要时用现有 Read 工具加载全文、用 Bash 执行 scripts/。无需实现专用 skill tool（OpenCode 的 tool 方案是优化，不是必需）。
- 远程 skill 仓库：`git clone --depth 1` 到临时目录后同扫，或直接 `npx skills add`。

## 8. 主要来源

GitHub Actions：官方文档（Events that trigger workflows、Workflow syntax、Contexts、Automatic token authentication、Triggering a workflow、Webhook events、Metadata syntax、Limits、Workflow cancellation、Workflow commands、Billing）· nektosact.com 用户指南 · actions/toolkit · actions/checkout · actions/runner-images

AI SDK：npm registry dist-tags 与各包 `.d.ts`（ai@5.0.238 / @ai-sdk/openai@2.0.118 / @ai-sdk/anthropic@2.0.95 / @ai-sdk/openai-compatible@1.0.48 / @ai-sdk/mcp@0.0.28）· GitHub tag `ai@5.0.238` 官方文档（tools-and-tool-calling、mcp-tools、generate-text、tool、create-mcp-client 等）

MCP：modelcontextprotocol.io spec（2025-11-25 / 2025-06-18）· github/github-mcp-server · 官方 MCP examples · Simon Willison uvx 缓存实践 · ai-sdk.dev MCP 文档

测试：vercel/ai `packages/ai/test/` 与 `mock-language-model-v2.ts` 源码 · anthropics/claude-code-action 测试（run-claude-sdk.test.ts、test-base-action.yml、src/github/context.ts）· bun.sh 文档（test/mocks、spawn）· fast-glob README

先例：anthropics/claude-code-action（含 security.md）· qodo-ai/pr-agent · OpenHands（#7860、#5219）· gptme/gptme（bot 文档、action.yml）· mirrajabi/aider-github-action · kirodotdev/Kiro · cline/cline · vercel-labs/skills · SecurityWeek 与 Mallory 的 claude-code-action 漏洞分析 · Kudelski PR-Agent 漏洞研究
