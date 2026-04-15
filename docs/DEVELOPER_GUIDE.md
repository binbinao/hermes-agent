# Hermes Agent 开发文档

面向开发者和 AI 编码助手的项目架构与开发指南。

## 项目结构

```
hermes-agent/
├── run_agent.py          # AIAgent 类 — 核心对话循环
├── model_tools.py        # 工具编排, _discover_tools(), handle_function_call()
├── toolsets.py           # 工具集定义, _HERMES_CORE_TOOLS
├── cli.py                # HermesCLI 类 — 交互式 CLI
├── hermes_state.py       # SessionDB — SQLite 会话存储(FTS5搜索)
├── hermes_constants.py   # 共享常量(HERMES_HOME, URL等)
├── hermes_logging.py     # 集中日志(RotatingFileHandler + 密钥脱敏)
├── hermes_time.py        # 时区感知时钟
├── batch_runner.py       # 并行批处理
├── mcp_serve.py          # MCP 服务器模式(暴露消息会话为MCP工具)
│
├── agent/                # Agent 内部模块
│   ├── prompt_builder.py     # 系统提示组装
│   ├── context_compressor.py # 自动上下文压缩
│   ├── prompt_caching.py     # Anthropic prompt 缓存(system_and_3策略)
│   ├── auxiliary_client.py   # 辅助LLM客户端路由(vision/compression/web等)
│   ├── model_metadata.py     # 模型上下文长度, token估算
│   ├── models_dev.py         # models.dev注册表(4000+模型, 109+提供商)
│   ├── display.py            # KawaiiSpinner, 工具预览格式化
│   ├── skill_commands.py     # 技能斜杠命令
│   ├── trajectory.py         # 轨迹保存
│   └── redact.py             # 密钥脱敏
│
├── hermes_cli/           # CLI 子命令
│   ├── main.py           # 入口点 — 所有 hermes 子命令
│   ├── config.py         # DEFAULT_CONFIG, OPTIONAL_ENV_VARS, 迁移
│   ├── commands.py       # 斜杠命令注册表(CommandDef)
│   ├── callbacks.py      # 终端回调(clarify/sudo/approval)
│   ├── setup.py          # 交互式设置向导
│   ├── skin_engine.py    # 皮肤/主题引擎
│   ├── skills_config.py  # 技能启用/禁用
│   ├── tools_config.py   # 工具启用/禁用
│   ├── skills_hub.py     # 技能仓库(/skills命令)
│   ├── models.py         # 模型目录, 提供商模型列表
│   ├── model_switch.py   # /model 切换管道
│   └── auth.py           # 提供商凭据解析
│
├── tools/                # 工具实现(每文件自注册)
│   ├── registry.py       # 中央工具注册表(schema/handler/dispatch)
│   ├── approval.py       # 危险命令检测+审批
│   ├── terminal_tool.py  # 终端编排(~1750行)
│   ├── process_registry.py # 后台进程管理
│   ├── file_tools.py     # 文件读写/搜索/补丁
│   ├── web_tools.py      # Web搜索/提取(Parallel+Firecrawl)
│   ├── browser_tool.py   # 浏览器自动化(~2183行)
│   ├── code_execution_tool.py # execute_code沙箱
│   ├── delegate_tool.py  # 子代理委托
│   ├── mcp_tool.py       # MCP客户端(~2187行)
│   └── environments/     # 终端后端
│       ├── base.py       # BaseEnvironment ABC
│       ├── local.py, docker.py, ssh.py, singularity.py, modal.py, daytona.py
│
├── gateway/              # 消息平台网关
│   ├── run.py            # GatewayRunner — 平台生命周期, 消息路由
│   ├── session.py        # SessionStore — 会话持久化
│   └── platforms/        # 18+适配器: telegram, discord, slack, whatsapp, signal,
│                         # matrix, mattermost, homeassistant, feishu, dingtalk,
│                         # wecom, bluebubbles, email, sms, webhook, api_server...
│
├── plugins/memory/       # 记忆插件(honcho/mem0/holographic/byterover等8种)
├── skills/               # 77+内置技能(SKILL.md格式)
├── optional-skills/      # 45+可选技能
├── environments/         # RL训练环境(Atropos集成)
├── acp_adapter/          # ACP服务器(IDE集成)
├── cron/                 # 定时调度(jobs.py + scheduler.py)
└── tests/                # ~3000测试
```

## 核心架构

### 文件依赖链

```
tools/registry.py  (无依赖 — 被所有工具文件导入)
       ↑
tools/*.py  (每个工具在import时调用registry.register())
       ↑
model_tools.py  (导入tools/registry + 触发工具发现)
       ↑
run_agent.py, cli.py, batch_runner.py
```

### Agent 循环

```python
# run_agent.py → AIAgent.run_conversation()
while api_call_count < max_iterations:
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
    else:
        return response.content  # 最终文本响应
```

消息格式遵循 OpenAI 标准: `{"role": "system/user/assistant/tool", ...}`

### 工具注册模式

每个工具文件在 import 时自注册:

```python
# tools/your_tool.py
from tools.registry import registry

def your_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"result": "..."})

registry.register(
    name="your_tool",
    toolset="your_toolset",
    schema={"name": "your_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: your_tool(param=args.get("param"), task_id=kw.get("task_id")),
    check_fn=lambda: True,  # 可用性检查
    emoji="⚡",
)
```

### 添加新工具(3个文件)

1. **创建 `tools/your_tool.py`** — 实现+注册(如上)
2. **在 `model_tools.py` 的 `_discover_tools()` 列表中添加 import**
3. **在 `toolsets.py` 中添加** — `_HERMES_CORE_TOOLS` 或新工具集

所有handler必须返回JSON字符串。

### 添加斜杠命令

1. 在 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 添加 `CommandDef`
2. 在 `cli.py` 的 `process_command()` 添加处理器
3. 如需网关支持, 在 `gateway/run.py` 添加处理器

所有下游(自动补全/帮助/Telegram菜单/Slack子命令)自动更新。

## 关键设计原则

### Prompt 缓存不可破坏

- 不要在对话中途修改历史上下文
- 不要在对话中途切换工具集
- 不要在对话中途重新加载记忆或重建系统提示
- 唯一修改上下文的时机是上下文压缩

### 工作目录行为

- **CLI**: 使用当前目录(`.` → `os.getcwd()`)
- **消息网关**: 使用 `MESSAGING_CWD` 环境变量(默认: home目录)

### 安全层

| 层 | 实现 |
|----|------|
| Sudo密码 | `shlex.quote()` 防止Shell注入 |
| 危险命令检测 | `tools/approval.py` 正则模式 + 用户审批流 |
| 写入拒绝列表 | 受保护路径通过 `os.path.realpath()` 解析(防符号链接绕过) |
| 技能安全 | `tools/skills_guard.py` 扫描hub安装的技能 |
| 代码沙箱 | `execute_code` 子进程移除API密钥环境变量 |
| Docker加固 | 丢弃所有capabilities, 禁止提权, PID限制 |
| 密钥脱敏 | `agent/redact.py` 从日志和工具输出中脱敏 |

## 配置系统

### DEFAULT_CONFIG (hermes_cli/config.py)

关键默认值:
- `agent.max_turns`: 90
- `terminal.backend`: "local", `terminal.timeout`: 180
- `compression.threshold`: 0.50, `compression.target_ratio`: 0.20
- `file_read_max_chars`: 100,000
- `display.skin`: "default"

### 添加配置项

1. 在 `hermes_cli/config.py` 的 `DEFAULT_CONFIG` 添加
2. 增加 `_config_version` 以触发现有用户的迁移

### 添加环境变量

在 `OPTIONAL_ENV_VARS` 添加条目:
```python
"NEW_API_KEY": {
    "description": "用途说明",
    "prompt": "显示名称",
    "url": "https://...",
    "password": True,
    "category": "tool",  # provider/tool/messaging/setting
},
```

## 技能格式

```markdown
---
name: my-skill
description: 简短描述
version: 1.0.0
platforms: [macos, linux]  # 可选, 限制平台
metadata:
  hermes:
    tags: [Category, Keywords]
    fallback_for_toolsets: [web]     # 仅当工具集不可用时显示
    requires_toolsets: [terminal]    # 仅当工具集可用时显示
---
# 技能标题
## 使用时机
## 快速参考
## 步骤
```

## 记忆插件

支持8种记忆后端(plugins/memory/): honcho, mem0, holographic, byterover, hindsight, openviking, retaindb, supermemory. 通过 `memory.provider` 配置激活。

## 皮肤系统

数据驱动, 无需代码改动:

```yaml
# ~/.hermes/skins/<name>.yaml
name: mytheme
colors: { banner_border: "#HEX", banner_title: "#HEX" }
spinner: { thinking_verbs: ["forging"], wings: [["⟪⚔", "⚔⟫"]] }
branding: { agent_name: "My Agent", prompt_symbol: "⚔ ❯ " }
tool_prefix: "╎"
```

内置: default, ares, mono, slate. 代码: `hermes_cli/skin_engine.py`

## 测试

```bash
pytest tests/ -v  # ~3000测试
```

## 代码规范

- PEP 8, 注释仅解释非显而易见的意图
- 捕获具体异常, 使用 `logger.warning()`/`logger.error()`
- 跨平台: 不假设 Unix (termios/fcntl 需 try/except)
- 使用 `pathlib.Path` 而非字符串拼接路径
- Conventional Commits: `fix(cli):`, `feat(gateway):`, `docs(tools):`
