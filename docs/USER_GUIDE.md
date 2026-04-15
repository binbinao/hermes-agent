# Hermes Agent 用户指南

Hermes Agent 是 Nous Research 的开源 AI 编程助手。

## 安装

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent && ./setup-hermes.sh
source ~/.zshrc && hermes setup
```

要求: Python 3.11+, Git. 可选: Node.js 18+, ripgrep.
配置目录: `~/.hermes/` (config.yaml, .env, SOUL.md, skills/, memories/).

## 使用

```bash
hermes                         # 交互式CLI
hermes chat -q "写快排"         # 单次对话
hermes chat --model anthropic/claude-sonnet-4.6
hermes doctor                  # 诊断
```

CLI斜杠命令: `/new`(新会话) `/model`(切换模型) `/skills`(技能) `/skin`(主题) `/help` `/verbose` `/background`(后台任务) `/sessions`(历史) `/search`(搜索历史) `/reset` `/resume` `/personality`

## 工具一览

| 类别 | 工具 | 说明 |
|------|------|------|
| 终端 | terminal | Shell执行, 6后端: local/docker/ssh/modal/daytona/singularity |
| 终端 | process | 后台进程管理(poll/wait/kill/stdin) |
| 文件 | read_file | 带行号分页读取(最大100K字符) |
| 文件 | write_file | 文件写入(完整覆盖) |
| 文件 | patch | 精准编辑(9种模糊匹配 + V4A多文件补丁) |
| 文件 | search_files | ripgrep驱动的内容/文件名搜索 |
| Web | web_search | 搜索(Exa/Parallel/Firecrawl/Tavily后端) |
| Web | web_extract | 网页提取+LLM摘要, 支持PDF |
| 浏览器 | browser_* | 导航/点击/输入/滚动/截图+视觉AI/控制台/JS执行 |
| 高级 | execute_code | 沙箱Python, RPC调用工具, 中间结果不进上下文 |
| 高级 | delegate_task | 子代理委托, 最多3并行, 隔离上下文 |
| 多媒体 | vision_analyze, image_generate, text_to_speech | 图像分析/生成, TTS |
| 管理 | cronjob, memory, todo, session_search | 定时任务, 记忆, 任务规划, 历史搜索 |

## 技能系统

77+内置 + 45+可选技能, 分类: 研究(arXiv/论文), 开发(TDD/调试/审查), GitHub(PR/Issue/仓库), MLOps(Axolotl/vLLM/W&B), 创意(ASCII/P5.js/AI音乐), Apple(iMessage/提醒), 生产力(Notion/Google/OCR)等.

```bash
hermes skills list / browse / search <query> / install <name> / remove <name>
```

## 消息平台网关

支持18+平台: Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Mattermost, Home Assistant, 飞书, 钉钉, 企业微信, BlueBubbles(iMessage), Email, SMS, Webhook, API Server.

```bash
hermes gateway install   # 安装systemd服务
hermes gateway run       # 直接运行
```

在 ~/.hermes/.env 配置平台Token (TELEGRAM_BOT_TOKEN, DISCORD_BOT_TOKEN等).

## 终端后端

config.yaml -> terminal.backend:
- **local** (默认): 直接在主机执行
- **docker**: 容器隔离, 可配镜像/CPU/内存/磁盘
- **ssh**: 远程执行 (ssh_host/ssh_user/ssh_key)
- **modal**: Modal云沙箱 (GPU支持)
- **singularity**: HPC环境
- **daytona**: 云开发环境

## 关键配置

```yaml
model:
  default: "anthropic/claude-opus-4.6"
  provider: "auto"   # auto/openrouter/nous/anthropic/gemini/zai/custom...
agent:
  max_turns: 60
  reasoning_effort: "medium"  # xhigh/high/medium/low/minimal/none
compression:
  enabled: true
  threshold: 0.50
memory:
  memory_enabled: true
  user_profile_enabled: true
display:
  skin: default   # default/ares/mono/slate
  streaming: true
```

## MCP服务器

```yaml
mcp_servers:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user"]
  remote:
    url: "https://my-mcp-server.example.com/mcp"
```

支持stdio/HTTP传输, OAuth认证, Sampling, 动态工具刷新.

## 其他功能

- **持久化记忆**: MEMORY.md(Agent笔记) + USER.md(用户画像), 跨会话保留
- **定时任务**: `hermes cron list/create`, Agent可通过cronjob工具管理
- **主题皮肤**: 4款内置(default/ares/mono/slate) + YAML自定义皮肤
- **IDE集成**: ACP协议支持VS Code, Zed, JetBrains系列
- **Docker部署**: `docker build -t hermes . && docker run -v /opt/data:/opt/data hermes`
- **上下文压缩**: 接近窗口限制时自动摘要中间轮次
- **Prompt缓存**: Anthropic模型节省约75%输入token费用
- **危险命令审批**: 自动检测危险Shell命令, 支持手动/智能审批模式
- **子代理委托**: delegate_task生成隔离子代理, 支持批量并行
- **代码沙箱**: execute_code在安全沙箱中运行Python, API密钥不暴露
