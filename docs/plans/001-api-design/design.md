# 0loop 顶层 API 设计

> 0loop 以 GitHub Action 的形式对外暴露 API。本文描述顶层 API。

## 1. 定位

**0loop 是一个 GitHub Action：把自然语言任务、模型连接、仓库上下文、GitHub 权限组合进一个 agent action step。**

用户在自己的 workflow 里写一步 0loop action，声明任务与权限，给出模型连接；0loop 组装事件上下文与仓库约定，施加权限边界，驱动 agent 在 runner 上完成任务并产出结果。

- **一步接入**：标准 GitHub Actions workflow，无新格式、无额外工具链；用法与 `actions/checkout` 相同——声明「这一步要 agent 完成什么」
- **自带基础设施**：模型连接走用户自己的 key 与网关，代码不出用户的 runner；无 SaaS、无 GitHub App、无托管
- **模型中立**：用户以 wire 协议 + 模型 id + 网关地址描述模型，不绑定任何厂商
- **底层封装开源 runtime**：agent loop、工具执行、沙箱、模型协议由开源 agent runtime 承担（首批 codex 与 pi），0loop 做产品层（GitHub 集成、上下文、权限、护栏）与统一层（消除 runtime 差异、模型中立）。runtime 不自研
- **run 一次性、无需值守**：创建即启动，结束即停止

## 2. 顶层 API

五个顶层对象。本次交付：`agent`、`env`、`permissions`。`memory` 与 `hook` 第一版不提供。

### 2.1 `agent`

一个 run 一个 agent，定义模型、运行时、任务与工具：

| 字段                     | 内容                                                                                                                                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `model`                  | 模型定义：`model`（模型 id）、`protocol`（wire 协议：anthropic-messages / openai-responses / openai-chat-completions / gemini）、`effort`（思考深度，默认使用模型默认值）、`meta`（模型元信息） |
| `runtime`                | 执行引擎：codex / pi，可指定版本（如 `codex@0.148.0`）；默认按 protocol 推导                                                                                                                    |
| `prompt` / `prompt-file` | 任务内容，二选一；过滤后的事件上下文自动附加；prompt-file 为仓库内纯静态文件，任务变更走 PR review                                                                                              |
| `mcp`                    | MCP server 列表。每项含 `command`（启动命令）与可选的 `env`（该进程内的变量名 → job 环境变量名）                                                                                                |
| `mcp-file`               | 第一版不提供                                                                                                                                                                                    |
| `skills`                 | 默认读取本仓库 `.agents/skills/`；`github.com/<owner>/<repo>[@ref]` = 远程技能仓库（兼容 skills.sh），准备环境阶段拉取到 runner 根目录的 `.agents/skills/`                                      |

注解：

- agent 不是持久资源：workflow 文件就是版本化的 agent 定义，git 承担版本管理
- 多个 MCP 的凭证靠各条目的 `env` 区分，不使用全局保留名，也不另设凭证库
- 未在某条 `env` 中列出的 job 环境变量不进入该 MCP 进程

### 2.2 `env`

密钥与配置的值只来自 GitHub secrets / variables，经 workflow 的 `env:` 传入，不写入 action `with:`。

`env` 使用保留变量名，可被多个 workflow 复用：

| 变量                | 必填 | 含义         |
| ------------------- | ---- | ------------ |
| `ZEROLOOP_BASE_URL` | 是   | 模型网关地址 |
| `ZEROLOOP_API_KEY`  | 是   | 凭证         |

MCP 不使用 `ZEROLOOP_` 保留名。每个 MCP 在该条目的 `env` 中声明对应关系：左边是该进程内的变量名，右边是 job 环境变量名。job 的 `env:` 中名字必须互不相同；不同 MCP 进程内可以出现相同的变量名，值分别来自不同的 job 变量。

分界：**env 放组织级共享的（网关地址与凭证），inputs 放任务级的（模型、思考深度、任务、MCP）。**

### 2.3 `permissions`

0loop 的权限聚合在 `permissions` input 下，三个字段：`tools`、`network`、`allow-users`。GitHub API 权限由 workflow 的 job 级 `permissions:` 管理，不在此处声明。

| 字段          | 管辖                         | 语义                                                                                                                                                           | 默认       |
| ------------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `tools`       | agent 在 runner 本地能做什么 | `read`（只读文件工具，不提供 shell）/ `write`（可编辑文件、可用 bash；bash 写文件与文件工具受同一 workspace 磁盘边界管辖）                                            | `read`     |
| `network`     | agent 能出网到哪里           | 主机列表。未配置则不限制；配置后仅允许列表中的主机。**模型端点（`ZEROLOOP_BASE_URL`）始终允许访问**。环境准备阶段（runtime 安装、skills 拉取、MCP 启动）不受限 | 不限制     |
| `allow-users` | 谁能启动一次 run             | GitHub 用户名数组。未配置则不额外限制（能触发该 workflow 的人均可）                                                                                            | 不额外限制 |

规则：

- `tools` 为 `read` / `write` 两级：**默认只读，需要写文件或使用 shell 时指定 `write`。** `read` 仅提供只读文件工具，不提供 shell；`write` 指写工作区文件并允许 bash，不表示 GitHub API 权限。
- GitHub API 权限由 job 级 `permissions:` 管理，不在 0loop 的 `permissions` 中重复声明。
- 凭证隔离：`ZEROLOOP_*` 只进入 0loop / 模型客户端；MCP 条目 `env` 中列出的变量只进入该 MCP 进程；上述变量均不进入 agent 可见环境。
- fork PR 遵循 GitHub 平台降级（只读 token、无 secrets）；`GITHUB_TOKEN` 默认不含 workflow scope。

### 2.4 `memory`

跨 run 持久状态。第一版不提供。

后期方向：对象存储；agent 通过 0loop 提供的 skills 或工具同步读取和更新。

### 2.5 `hook`

agent 生命周期钩子。第一版不提供。

Action 前后的 workflow step 即前后钩子。runtime 自带的 hooks 属于 runtime 配置，不在本层提供。

## 3. 示例

GitHub API 权限写在 job 的 `permissions:`，不写在 0loop 的 `permissions` 里。`mcp-file`、`memory`、`hook` 第一版不提供，不出现在示例中。Action 前后的 workflow step 即前后钩子。

```yaml
# PR 审查：默认 tools: read；GitHub 写评论由 job 的 permissions 控制
permissions:
  contents: read
  pull-requests: write
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: 0loop/0loop@v1
        with:
          agent: |
            protocol: openai-responses
            model: your-model-id
            prompt: |
              Review this pull request. Report findings in the order
              correctness, security, and maintainability, then publish a summary.
        env:
          ZEROLOOP_BASE_URL: ${{ vars.ZEROLOOP_BASE_URL }}
          ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```

```yaml
# 修 CI：指定 runtime、effort、meta，写文件，远程 skills
- uses: 0loop/0loop@v1
  with:
    agent: |
      protocol: openai-responses
      model: your-model-id
      runtime: codex@0.148.0
      effort: high
      meta: |
        context-window: 272000
      prompt: |
        The CI failed. Read the failure, fix the code, verify with tests.
      skills: |
        - github.com/owner/ci-fix-skills@v1
    permissions: |
      tools: write
  env:
    ZEROLOOP_BASE_URL: ${{ vars.ZEROLOOP_BASE_URL }}
    ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```

```yaml
# Issue 分诊：prompt-file、runtime pi、限制 allow-users
- uses: 0loop/0loop@v1
  with:
    agent: |
      protocol: anthropic-messages
      model: your-model-id
      runtime: pi
      prompt-file: .github/prompts/issue-triage.md
    permissions: |
      allow-users:
        - alice
        - bob
  env:
    ZEROLOOP_BASE_URL: ${{ vars.ZEROLOOP_BASE_URL }}
    ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```

```yaml
# 查外部系统再回复 Issue：多个 MCP；有凭证的用 env 对应，无凭证的只写 command
- uses: 0loop/0loop@v1
  with:
    agent: |
      protocol: gemini
      model: your-model-id
      prompt: |
        Query Slack and Jira, then reply on the issue.
      mcp: |
        - command: npx -y @slack/mcp
          env:
            API_KEY: SLACK_BOT_TOKEN
        - command: npx -y @jira/mcp
          env:
            API_KEY: JIRA_API_TOKEN
        - command: npx -y @org/docs-mcp
    permissions: |
      tools: read
  env:
    ZEROLOOP_BASE_URL: ${{ vars.ZEROLOOP_BASE_URL }}
    ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
    SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
    JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}
```

```yaml
# 发版：本仓库 skills + 远程 skills，限制 network 与 allow-users
- uses: 0loop/0loop@v1
  with:
    agent: |
      protocol: openai-chat-completions
      model: gateway-model-id
      prompt: |
        Prepare a release according to the repository's release skill.
      skills: |
        - .
        - github.com/owner/release-skills@v1
    permissions: |
      tools: write
      network:
        - registry.npmjs.org
        - pypi.org
      allow-users:
        - alice
  env:
    ZEROLOOP_BASE_URL: ${{ vars.ZEROLOOP_BASE_URL }}
    ZEROLOOP_API_KEY: ${{ secrets.ZEROLOOP_API_KEY }}
```
