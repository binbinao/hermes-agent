# Hermes Agent 项目代码结构分析

## 整体架构

```
hermes-agent/
├── cli.py                 # 主CLI入口 (~392KB, 核心对话循环)
├── run_agent.py           # AIAgent类实现 (~487KB)
├── model_tools.py         # 工具编排与发现
├── toolsets.py            # 工具集定义
├── hermes_state.py        # 会话存储 (SQLite + FTS5)
├── hermes_constants.py    # 常量定义
├── hermes_logging.py      # 日志系统
├── utils.py               # 工具函数

├── hermes_cli/            # CLI子命令和配置
├── agent/                 # Agent核心模块
├── tools/                 # 工具实现 (60+工具)
├── gateway/               # 消息平台网关
├── acp_adapter/           # VS Code/Zed/JetBrains集成
├── cron/                  # 定时任务调度
├── environments/          # RL训练环境
├── skills/                # 内置技能
├── tests/                 # 测试套件 (~3000测试)
└── docs/                  # 文档
```

---

## 核心模块

### 1. cli.py (~392KB)
主CLI入口，负责：
- 交互式CLI界面
- 命令解析与分发
- KawaiiSpinner动画
- Rich panels显示
- prompt_toolkit输入

### 2. run_agent.py (~487KB)
AIAgent类，核心对话循环：
- `chat()` - 简单接口，返回最终响应
- `run_conversation()` - 完整接口，返回dict
- API调用循环，处理tool_calls
- 上下文压缩
- 迭代预算管理

### 3. model_tools.py (~23KB)
工具编排：
- `_discover_tools()` - 自动发现工具
- `handle_function_call()` - 工具调用分发
- `_last_resolved_tool_names` - 全局工具名缓存

### 4. toolsets.py (~21KB)
工具集定义：
- `_HERMES_CORE_TOOLS` - 核心工具列表
- 工具集分组与依赖

---

## agent/ 目录 - Agent核心模块

| 文件 | 大小 | 功能 |
|------|------|------|
| `prompt_builder.py` | 43KB | 系统提示组装 |
| `context_compressor.py` | 33KB | 自动上下文压缩 |
| `prompt_caching.py` | 2KB | Anthropic提示缓存 |
| `auxiliary_client.py` | 93KB | 辅助LLM客户端 (视觉/摘要) |
| `model_metadata.py` | 39KB | 模型上下文长度/元数据 |
| `models_dev.py` | 26KB | models.dev注册表 |
| `display.py` | 41KB | KawaiiSpinner/工具预览 |
| `skill_commands.py` | 14KB | 技能斜杠命令 |
| `anthropic_adapter.py` | 56KB | Anthropic API适配器 |
| `credential_pool.py` | 48KB | 凭证池管理 |
| `error_classifier.py` | 27KB | 错误分类 |
| `insights.py` | 34KB | 使用洞察 |
| `usage_pricing.py` | 24KB | 用量计费 |

---

## tools/ 目录 - 工具实现 (60+工具)

### 核心工具

| 文件 | 大小 | 功能 |
|------|------|------|
| `registry.py` | 13KB | 中央工具注册表 |
| `terminal_tool.py` | 75KB | 终端编排 |
| `file_tools.py` | 39KB | 文件读写/搜索 |
| `file_operations.py` | 42KB | 文件操作 (patch等) |
| `web_tools.py` | 87KB | Web搜索/提取 |
| `browser_tool.py` | 85KB | 浏览器自动化 |
| `code_execution_tool.py` | 52KB | 代码执行沙箱 |
| `delegate_tool.py` | 40KB | 子代理委托 |

### 消息与集成

| 文件 | 大小 | 功能 |
|------|------|------|
| `send_message_tool.py` | 41KB | 跨平台消息发送 |
| `homeassistant_tool.py` | 17KB | Home Assistant集成 |
| `cronjob_tools.py` | 21KB | 定时任务管理 |

### AI/ML 工具

| 文件 | 大小 | 功能 |
|------|------|------|
| `mcp_tool.py` | 84KB | MCP客户端 (~1050行) |
| `skills_tool.py` | 50KB | 技能管理 |
| `skills_hub.py` | 99KB | 技能市场 |
| `skill_manager_tool.py` | 27KB | 技能管理器 |
| `rl_training_tool.py` | 57KB | RL训练 |
| `image_generation_tool.py` | 28KB | 图像生成 |
| `vision_tools.py` | 22KB | 视觉分析 |
| `transcription_tools.py` | 27KB | 语音转文本 |
| `tts_tool.py` | 37KB | 文本转语音 |
| `voice_mode.py` | 31KB | 语音模式 |

### 记忆与状态

| 文件 | 大小 | 功能 |
|------|------|------|
| `memory_tool.py` | 23KB | 长期记忆 |
| `session_search_tool.py` | 21KB | 会话搜索 |
| `todo_tool.py` | 10KB | 任务列表 |
| `process_registry.py` | 41KB | 后台进程管理 |

### 安全与审批

| 文件 | 大小 | 功能 |
|------|------|------|
| `approval.py` | 35KB | 危险命令检测 |
| `tirith_security.py` | 25KB | 安全检查 |
| `skills_guard.py` | 43KB | 技能防护 |

### 工具子目录

```
tools/
├── environments/          # 终端后端 (local/docker/ssh/modal/daytona)
├── browser_providers/     # 浏览器提供者
└── neutts_samples/       # TTS样本
```

---

## hermes_cli/ 目录 - CLI子命令

| 文件 | 大小 | 功能 |
|------|------|------|
| `main.py` | - | 入口点，`hermes`所有子命令 |
| `config.py` | 44KB | 配置管理 + DEFAULT_CONFIG |
| `setup.py` | - | 交互式设置向导 |
| `commands.py` | - | 斜杠命令定义 (CommandDef) |
| `callbacks.py` | - | 终端回调 (clarify/sudo/approval) |
| `banner.py` | - | Banner显示 |
| `skin_engine.py` | - | 皮肤/主题引擎 |
| `models.py` | - | 模型目录 |
| `model_switch.py` | - | 模型切换 |
| `auth.py` | - | 认证管理 |
| `tools_config.py` | - | 工具配置 |
| `skills_config.py` | - | 技能配置 |
| `skills_hub.py` | - | 技能市场 |
| `profiles.py` | - | 多配置文件管理 |
| `doctor.py` | - | 环境诊断 |
| `logs.py` | - | 日志管理 |

---

## gateway/ 目录 - 消息平台网关

| 文件 | 大小 | 功能 |
|------|------|------|
| `run.py` | 355KB | 主循环/斜杠命令/消息分发 |
| `session.py` | 42KB | 会话持久化 |
| `config.py` | 44KB | 网关配置 |
| `status.py` | 12KB | 连接状态 |
| `delivery.py` | 11KB | 消息投递 |
| `hooks.py` | 6KB | 事件钩子 |
| `pairing.py` | 11KB | 设备配对 |
| `channel_directory.py` | 9KB | 频道目录 |
| `stream_consumer.py` | 16KB | 流式消费 |

### gateway/platforms/ - 平台适配器

```
gateway/platforms/
├── telegram.py
├── discord.py
├── slack.py
├── whatsapp.py
├── homeassistant.py
└── signal.py
```

---

## 配置文件

| 文件 | 功能 |
|------|------|
| `config.yaml` | 主配置 (~43KB 示例) |
| `.env` | API密钥和环境变量 |
| `pyproject.toml` | 项目元数据/依赖 |
| `requirements.txt` | pip依赖 |
| `uv.lock` | uv锁文件 |

---

## 用户配置目录 (~/.hermes/)

```
~/.hermes/
├── config.yaml          # 主配置
├── auth.json            # 认证信息
├── sessions/            # 会话记录
├── memories/            # 记忆数据
├── skills/              # 用户技能
├── cron/                # 定时任务
├── checkpoints/         # 检查点
├── logs/                # 日志
└── skins/               # 自定义皮肤
```

---

## 依赖关系图

```
tools/registry.py       # 无依赖 - 所有工具文件导入
    ↑
tools/*.py              # 每个调用 registry.register()
    ↑
model_tools.py          # 导入tools/registry + 触发工具发现
    ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

---

## 主要入口点

1. **CLI模式**: `python cli.py` 或 `hermes` 命令
2. **Agent模式**: `python run_agent.py`
3. **网关模式**: `python -m gateway.run`
4. **批处理模式**: `python batch_runner.py`
5. **MCP服务**: `python mcp_serve.py`
6. **RL训练**: `python rl_cli.py`

---

## 技术栈

- **HTTP客户端**: httpx, aiohttp
- **数据库**: SQLite + FTS5
- **CLI界面**: Rich, prompt_toolkit
- **异步**: asyncio, threading
- **ML/AI**: anthropic, openai, transformers
- **浏览器自动化**: playwright, selenium
- **容器**: docker, modal
