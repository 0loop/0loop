# ITS

**把 GitHub Actions 变成你的 AI Agent。**

ITS 是一个 GitHub Action。它把每次 workflow 运行变成一次 AI Agent 执行:GitHub 提供环境(全新的 Linux 虚拟机、checkout 好的代码、自动注入的 GITHUB_TOKEN),你只负责写一句 `prompt`,Agent 在环境里读代码、跑命令、改文件,然后用 GitHub 自身的权限机制把结果写回仓库——评论、开 PR、打标签。

- **零基础设施**:公开仓库免费额度,不需要任何服务器
- **零鉴权设计**:Agent 的权限就是 workflow 里的 `permissions:` 声明,边界由 GitHub 强制执行
- **模型无关**:Anthropic / OpenAI / Google / 任意 OpenAI 兼容网关,换模型不改 prompt
- **生态直通**:仓库里的 AGENTS.md、skills、任意 MCP server 直接消费,不发明自己的配置格式

## 快速开始

三步给仓库配上一个 Issue 分诊 Agent:

1. 创建 `.github/workflows/triage.yml`:

```yaml
name: Issue Triage
on:
  issues:
    types: [opened]

permissions:
  contents: read
  issues: write

jobs:
  its:
    runs-on: ubuntu-latest
    steps:
      - uses: minorcell/its@v1
        with:
          prompt: |
            新 issue 已附在下方。判断它的类型(bug / 需求 / 提问)并打上对应 label;
            缺少复现步骤的评论请求补充;常见问题直接解答。
        env:
          ITS_PROTOCOL: anthropic-messages
          ITS_MODEL: claude-sonnet-5
          ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
```

2. 在仓库的 Settings → Secrets and variables → Actions 里添加 secret `ITS_API_KEY`,值为你模型供应商的 API key。`ITS_PROTOCOL`、`ITS_MODEL` 如需组织级统一管理,可把字面量换成 `${{ vars.XXX }}` 引用。

3. 打开一个新 issue。ITS 会在几分钟内完成运行:分类、打标签、必要时评论。整个过程在 Actions 页实时可见。

## 输入参考

`with:` 参数:

| 参数 | 必填 | 说明 |
|---|---|---|
| `prompt` | ✅ | 用自然语言描述这次运行要达成的结果。事件上下文(新 issue 正文、PR diff、触发者等)自动附加,无需解析 |
| `tools` | | 额外工具,数组形式 |
| `skills` | | 技能,数组形式。`github.com/<owner>/<repo>` 为远程 skill 仓库,经 skills.sh 安装;`.` 为本仓库 `.agents/skills/` 目录 |
| `mcp` | | MCP server 启动命令,数组形式。需要认证的凭证用 `$VAR` 插槽引用,由 `env:` 提供 |

`env:` 变量:

| 变量 | 必填 | 说明 |
|---|---|---|
| `ITS_PROTOCOL` | ✅ | 协议名(非厂商名):`anthropic-messages` \| `openai-chat-completions` \| `openai-responses` \| `gemini` |
| `ITS_MODEL` | ✅ | 模型 ID,如 `claude-sonnet-5`、`gpt-5`、`gemini-2.5-pro` |
| `ITS_API_KEY` | ✅ | 当前协议对应的 API key |
| `ITS_BASE_URL` | | 使用第三方网关时的地址;`ITS_PROTOCOL` 填该网关实现的协议(通常 `openai-chat-completions`) |

连接信息(协议、模型、密钥、网关地址)统一放在 `env:`,与密钥归为一组,便于用 `${{ vars.XXX }}` 由组织统一管理;`with:` 只描述行为。

`ITS_PROTOCOL` 的取值是协议自身的名称,不是厂商名:`openai-chat-completions` 覆盖所有 OpenAI 兼容网关(DeepSeek、Groq、OpenRouter、Ollama 等),网关地址由 `ITS_BASE_URL` 指定;`openai-responses` 是 OpenAI 的 Responses API,与 Chat Completions 是两套不同的线协议,不可混用;`anthropic-messages` 同样覆盖 Anthropic 兼容网关;`gemini` 对应 Gemini API。

## 更多示例

### PR 初审

需要读代码的场景,显式 checkout 即可:

```yaml
name: PR Review
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  its:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: minorcell/its@v1
        with:
          prompt: |
            工作目录已 checkout 当前 PR。按 正确性 > 安全性 > 可维护性 的顺序
            发行内评论,最后给一条总结评论。
        env:
          ITS_PROTOCOL: openai-chat-completions
          ITS_MODEL: gpt-5
          ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
```

### 定时周报

```yaml
name: Weekly Digest
on:
  schedule:
    - cron: "0 9 * * 1"   # 每周一 9:00

permissions:
  contents: read
  discussions: write

jobs:
  its:
    runs-on: ubuntu-latest
    steps:
      - uses: minorcell/its@v1
        with:
          prompt: |
            汇总过去 7 天的 commits、PR 和 issue,
            生成周报发到 Discussion,按 进展 / 问题 / 下周计划 组织。
        env:
          ITS_PROTOCOL: gemini
          ITS_MODEL: gemini-2.5-pro
          ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
```

### 第三方网关(OpenAI 兼容)

任何 OpenAI 兼容的第三方网关(DeepSeek、私有代理等),`ITS_PROTOCOL` 填它实现的协议,地址通过 `ITS_BASE_URL` 配置:

```yaml
      - uses: minorcell/its@v1
        with:
          prompt: |
            阅读本仓库的 README 和最近 20 条 commit,
            补充缺失的文档章节。
        env:
          ITS_PROTOCOL: openai-chat-completions
          ITS_MODEL: deepseek-chat
          ITS_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
          ITS_BASE_URL: https://api.deepseek.com/v1
```

### Skills:本仓库与远程仓库

```yaml
      - uses: minorcell/its@v1
        with:
          prompt: |
            按仓库的发布流程 skill 完成一次发布准备:
            检查 changelog、更新版本号、创建发布 PR。
          skills:
            - .                                # 本仓库 .agents/skills/ 下的全部 skill
            - github.com/minorcell/skills      # 远程 skill 仓库,经 skills.sh 安装
        env:
          ITS_PROTOCOL: anthropic-messages
          ITS_MODEL: claude-sonnet-5
          ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
```

### 带认证的 MCP server

需要凭证的远程 MCP 调用,密钥不写进 `with:`,用 `$VAR` 插槽从 `env:` 注入:

```yaml
      - uses: minorcell/its@v1
        with:
          prompt: |
            新 issue 已附在下方。查询内部统计工具后,
            用数据回复用户,并打上对应 label。
          mcp:
            - "npx -y @team/stats-server --token $MCP_TOKEN"
        env:
          ITS_PROTOCOL: openai-chat-completions
          ITS_MODEL: gpt-5
          ITS_API_KEY: ${{ secrets.ITS_API_KEY }}
          MCP_TOKEN: ${{ secrets.MCP_TOKEN }}
```

## 仓库生态

ITS 不发明自己的配置格式,直接消费仓库里已有的东西:

- **AGENTS.md**:仓库根目录存在时自动加载,作为 Agent 的仓库上下文(同 Claude Code 读取 CLAUDE.md 的方式)。
- **`.agents/skills/`**:本仓库的 skills 目录,`skills: ['.']` 即可启用。
- **远程 skill 仓库**:声明 `github.com/<owner>/<repo>` 后经 skills.sh 安装。
- **MCP server**:以命令数组声明,任何生态的 server 都能接入。

## 权限模型

ITS 不引入自己的权限体系。Agent 能做什么,由工作文件里的 `permissions:` 声明决定,工具集随声明自动收放:

- 没写 `issues: write`,Agent 的工具箱里就不存在"发评论";
- fork 的 PR 触发时,GitHub 自动将 token 降级为只读,Agent 相应地只剩读工具;
- API key 只存在你的 secrets 里,ITS 不接触、不转售。

这条边界由 GitHub 强制执行,与 ITS 无关地成立。

## 注意事项

- **异步执行**:从触发到运行完成有分钟级延迟(运行排队 + 环境启动),ITS 适合异步任务,不是实时对话。
- **成本**:每次运行消耗你模型账号的 token,建议 prompt 里明确范围。
- **幂等**:同一事件重复触发时 ITS 会检查是否已处理过,避免重复回复。
- **可审计**:每一步(工具调用、模型输出)都在 Actions 日志里,产物是普通 GitHub 对象。
- **prompt 注入**:issue 正文等事件内容被视为不可信数据,仅作为数据处理。

## Roadmap

- **Phase 1**:任何 GitHub 用户只靠一个工作文件和一个 secret,就能让 ITS 在自己的仓库里完成一件真实的事,全程不离开 GitHub。
- **Phase 2**:用户不需要为 ITS 学习任何新格式——仓库里已有的 AGENTS.md、skills、任意 MCP server 都被直接消费;换模型或换协议,prompt 一个字不改。
- **Phase 3**:一个团队能用多个 ITS 组成流水线(规划、执行、复核各司其职),复用的不是代码,而是别人写好的工作文件。
- **Phase 4(待验证)**:企业环境(自托管 runner、私有网关、私有模型)下,体验与公网完全一致。

## 状态

🚧 开发中,接口可能变化。
