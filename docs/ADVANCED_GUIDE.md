# Hermes Agent 进阶指南

本文档面向已掌握基础用法的用户，深入讲解高级功能、工作流优化和生产环境部署。

---

## 1. 多会话与上下文管理

### 1.1 会话生命周期

Hermes 使用 SQLite FTS5 存储会话历史，支持跨会话上下文恢复：

```bash
hermes sessions list          # 列出最近会话
hermes sessions resume <id>   # 恢复指定会话
hermes sessions search <query> # 搜索历史会话
```

```python
# Python API 中管理会话
session_id = "my-session-123"
agent = AIAgent(session_id=session_id)
response = agent.chat("继续上次的工作")
```

### 1.2 上下文压缩策略

当对话接近 token 窗口限制时，Hermes 自动触发压缩：

```yaml
# config.yaml
compression:
  enabled: true
  threshold: 0.50      # 达到 50% 窗口时触发
  target_ratio: 0.20   # 压缩至 20% 原始大小
```

**压缩触发时的行为：**
- 中间工具调用结果被摘要替换
- 系统提示和初始化上下文保留
- 对话结构（用户/助手交替）保持

### 1.3 手动上下文控制

```bash
# 强制压缩当前上下文
/compact

# 查看当前上下文使用量
/hermes stats

# 清除并开始新会话
/new
```

---

## 2. 子代理委托模式

### 2.1 delegate_task 的使用场景

当任务具有以下特征时，优先使用 `delegate_task`：

| 场景 | 原因 |
|------|------|
| 独立并行研究 | 多个子任务同时执行，结果汇总 |
| 复杂调试任务 | 隔离上下文，避免干扰主对话 |
| 大型代码重构 | 子代理可以深入修改，主对话不被打断 |
| 需要不同工具集 | 每个子代理可配置独立工具集 |

### 2.2 并行工作流

```python
# 伪代码示例：同时研究三个主题
tasks = [
    {"goal": "研究 vLLM 推理优化技术", "toolsets": ["web"]},
    {"goal": "分析该项目性能瓶颈", "toolsets": ["terminal", "file"]},
    {"goal": "调研最新 RL 训练方法", "toolsets": ["web", "file"]},
]
results = delegate_task(tasks=tasks)  # 最多 3 个并行
```

### 2.3 上下文隔离与数据传递

子代理完全隔离，不继承父代理的对话历史。传递信息的推荐方式：

```python
# 通过 context 参数传递所有必要信息
delegate_task(
    goal="分析代码库并生成报告",
    context="""
    项目路径: /data/project
    关键文件: src/main.py, src/utils.py
    分析要求:
    - 统计代码行数
    - 识别主要模块依赖
    - 找出循环依赖
    输出格式: JSON 报告
    """,
    toolsets=["terminal", "file"]
)
```

### 2.4 与 execute_code 的选择

| 任务类型 | 推荐工具 | 原因 |
|----------|----------|------|
| 数据处理/分析 | `execute_code` | 直接结果，无需解析 |
| 需要多轮工具调用 | `delegate_task` | 完整 Agent 能力 |
| 简单脚本执行 | `execute_code` | 开销更小 |
| 研究类任务 | `delegate_task` | 可以搜索/阅读/推理 |

---

## 3. 终端后端进阶

### 3.1 Docker 后端配置

```yaml
# config.yaml
terminal:
  backend: docker
  docker:
    image: "python:3.11-slim"
    cpu: 2
    memory: "4g"
    disk: "10g"
    network: true
    # 危险命令在容器内执行，主机安全
```

### 3.2 SSH 远程执行

```yaml
terminal:
  backend: ssh
  ssh:
    host: "your-server.com"
    user: "developer"
    key: "~/.ssh/id_rsa"
    cwd: "/home/developer/projects"
```

```bash
# 远程执行命令示例
hermes chat -q "在远程服务器上运行 pytest tests/"
```

### 3.3 Modal 云沙箱（GPU 支持）

```yaml
terminal:
  backend: modal
  modal:
    gpu: "T4"          # T4/A10G/H100
    memory: "32Gi"
    timeout: 3600
```

适用场景：模型推理测试、大文件处理、需要 GPU 的任务。

---

## 4. MCP 服务器集成

### 4.1 本地 MCP 服务器

```yaml
# config.yaml
mcp_servers:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user"]
```

### 4.2 远程 MCP 服务器

```yaml
mcp_servers:
  remote-api:
    url: "https://api.example.com/mcp"
    auth: "bearer-token-xxx"
```

### 4.3 MCP 工具调用

连接后，MCP 工具与内置工具使用方式相同：

```
帮我用 filesystem MCP 读取 /home/user/documents/report.pdf 的内容
```

---

## 5. 定时任务与工作流自动化

### 5.1 Cron 任务创建

```bash
# 每小时执行一次信息收集
hermes cron create \
  --name "每小时新闻摘要" \
  --schedule "0 * * * *" \
  --prompt "搜索最新 AI/ML 新闻，生成简洁摘要"

# 每天早上 9 点发送日程提醒
hermes cron create \
  --name "每日技术资讯" \
  --schedule "0 9 * * *" \
  --prompt "搜索并总结当日 AI 技术进展"
```

### 5.2 Cron 任务管理

```bash
hermes cron list              # 列出所有任务
hermes cron pause <job_id>    # 暂停任务
hermes cron resume <job_id>   # 恢复任务
hermes cron remove <job_id>   # 删除任务
hermes cron run <job_id>      # 立即执行（测试用）
```

### 5.3 配合技能使用

```bash
hermes cron create \
  --name "每日 PR 审查" \
  --schedule "0 10 * * *" \
  --skills "github-code-review" \
  --prompt "审查 NousResearch/hermes-agent 的最新 PR，生成审查报告"
```

### 5.4 自动化工作流示例

```
┌─────────────────────────────────────────────────────────┐
│  定时触发 (每天 8:00)                                    │
│    ↓                                                    │
│  ┌─────────────────┐                                   │
│  │ 获取今日新闻    │ → web_search                       │
│  └────────┬────────┘                                   │
│           ↓                                             │
│  ┌─────────────────┐                                   │
│  │ 生成摘要        │ → LLM 处理                         │
│  └────────┬────────┘                                   │
│           ↓                                             │
│  ┌─────────────────┐                                   │
│  │ 发送到 Telegram │ → send_message                     │
│  └─────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
```

---

## 6. 记忆系统深入使用

### 6.1 记忆类型

| 类型 | 存储位置 | 内容 | 用途 |
|------|----------|------|------|
| 用户画像 | `USER.md` | 偏好、工作、习惯 | 个性化响应 |
| Agent 笔记 | `MEMORY.md` | 环境、约定、项目 | 跨会话一致性 |
| 会话历史 | SQLite FTS5 | 完整对话 | 搜索和恢复 |

### 6.2 记忆指令

```bash
# 记忆会在新会话开始时自动加载
# 你可以随时要求更新记忆

"记住我偏好 Python 3.11+ 语法"
"记住这个项目使用 uv 作为包管理器"
"记住我讨厌被问确认问题，直接执行"
```

### 6.3 记忆插件后端

支持多种记忆后端，通过 `memory.provider` 配置：

```yaml
# config.yaml
memory:
  provider: "mem0"      # mem0/honcho/holographic/...
  user_profile_enabled: true
  memory_enabled: true
```

---

## 7. 消息平台网关配置

### 7.1 Telegram Bot 部署

```bash
# 1. 创建 Bot：找 @BotFather 获取 Token
# 2. 配置环境变量
export TELEGRAM_BOT_TOKEN="your-token-here"

# 3. 启动网关
hermes gateway run --platform telegram
```

### 7.2 Discord 集成

```yaml
# .env
DISCORD_BOT_TOKEN=your-discord-token
DISCORD_GUILD_ID=your-guild-id
```

### 7.3 消息平台对比

| 平台 | 适合场景 | 限制 |
|------|----------|------|
| Telegram | 快速原型、私人使用 | 需要 Bot Token |
| Discord | 社区、团队协作 | 需要服务器管理权限 |
| Slack | 企业环境 | 需要Workspace管理 |
| WhatsApp | 实时通知 | 需要手机在线 |
| Home Assistant | 智能家居控制 | 需要 HA 访问 |

### 7.4 网关高可用配置

```yaml
# config.yaml (网关部分)
gateway:
  auto_reconnect: true
  reconnect_interval: 5
  max_retries: 3
  token_lock: true    # 防止多实例使用同一凭证
```

---

## 8. 技能系统高级用法

### 8.1 技能搜索和安装

```bash
hermes skills search "code review"    # 搜索相关技能
hermes skills browse                  # 浏览技能市场
hermes skills install github-code-review  # 安装技能
```

### 8.2 自定义技能创建

```bash
# 在 ~/.hermes/skills/ 下创建技能
mkdir -p ~/.hermes/skills/my-custom-skill
```

创建 `SKILL.md`：

```markdown
---
name: my-custom-skill
description: 处理特定业务逻辑
platforms: [linux, macos]
---

# 自定义技能名称

## 使用时机
当用户请求 XXX 时使用此技能。

## 快速参考
```
/my-skill <参数>
```

## 步骤

1. 解析输入参数
2. 执行核心逻辑
3. 返回格式化结果
```

### 8.3 技能元数据

技能文件头的 YAML 元数据控制其行为：

```yaml
---
name: github-code-review
description: 代码审查技能
version: 1.0.0
platforms: [linux, macos]           # 限制运行平台
metadata:
  hermes:
    tags: [github, code-review]     # 搜索标签
    requires_toolsets: [terminal]    # 需要 terminal 工具集
    fallback_for_toolsets: []        # 备用工具集
---
```

---

## 9. 性能优化与调试

### 9.1 Token 使用分析

```bash
/hermes stats    # 查看当前会话 token 消耗
```

```yaml
# config.yaml - 限制最大消耗
agent:
  max_turns: 60          # 最大对话轮次
  max_context_tokens: 100000  # 限制上下文大小
```

### 9.2 推理 Effort 控制

```yaml
agent:
  reasoning_effort: "medium"  # xhigh/high/medium/low/minimal/none
```

| 级别 | 适用场景 | Token 消耗 |
|------|----------|-----------|
| xhigh | 复杂推理、证明 | 最高 |
| medium | 日常对话（默认） | 适中 |
| minimal | 快速响应、简单任务 | 最低 |

### 9.3 日志调试

```bash
# 查看详细日志
hermes logs --tail 100 --level DEBUG

# 分析特定会话
hermes logs --session <session_id> --level INFO
```

```yaml
# config.yaml - 详细日志配置
logging:
  level: INFO           # DEBUG/INFO/WARNING/ERROR
  file: "~/.hermes/logs/hermes.log"
  max_size: "10MB"
  backup_count: 5
```

### 9.4 常见问题排查

| 问题 | 排查方法 |
|------|----------|
| 工具无响应 | `hermes doctor` 检查配置 |
| API 超时 | 增加 `terminal.timeout` |
| 记忆丢失 | 检查 `memory.provider` 配置 |
| 会话无法恢复 | `hermes sessions list` 确认存在 |
| MCP 连接失败 | 检查网络和 Token 配置 |

---

## 10. 生产环境部署

### 10.1 Docker 部署

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install -e .
CMD ["hermes", "gateway", "run"]
```

```bash
# 构建和运行
docker build -t hermes-prod .
docker run -d \
  --name hermes \
  -v ~/.hermes:/root/.hermes \
  -e TELEGRAM_BOT_TOKEN=${TELEGRAM_TOKEN} \
  hermes-prod
```

### 10.2 Systemd 服务（Linux）

```ini
# /etc/systemd/system/hermes.service
[Unit]
Description=Hermes Agent Gateway
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/home/your-user
ExecStart=/home/your-user/.local/bin/hermes gateway run
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable hermes
sudo systemctl start hermes
sudo journalctl -u hermes -f
```

### 10.3 多实例部署

```yaml
# config.yaml - 多实例配置
gateway:
  port: 8080
  workers: 4          # 4 个 worker 进程
  token_lock: true    # 凭证锁定防止冲突
```

---

## 11. 安全配置

### 11.1 危险命令审批

```yaml
# config.yaml
approval:
  mode: "smart"    # disabled/manual/smart
  dangerous_patterns:
    - "^rm -rf /"
    - "^dd if=.*of=/dev/"
    - "^mkfs\\.\\w+"
```

| 模式 | 行为 |
|------|------|
| disabled | 不检测危险命令 |
| manual | 所有危险命令需用户确认 |
| smart | 智能判断，仅高风险命令需确认 |

### 11.2 受保护路径

```yaml
approval:
  protected_paths:
    - "/etc"
    - "/sys"
    - "/proc"
    - "/boot"
```

### 11.3 API 密钥安全

```bash
# 敏感变量放在 .env，不提交到版本控制
# .env 文件权限
chmod 600 ~/.hermes/.env
```

---

## 12. 高级配置技巧

### 12.1 模型提供商配置

```yaml
model:
  default: "anthropic/claude-opus-4.6"
  provider: "auto"
  providers:
    anthropic:
      api_key: "${ANTHROPIC_API_KEY}"
    openrouter:
      api_key: "${OPENROUTER_API_KEY}"
      base_url: "https://openrouter.ai/api/v1"
```

### 12.2 自定义模型端点

```yaml
model:
  providers:
    custom:
      base_url: "http://your-custom-endpoint.com/v1"
      api_key: "${CUSTOM_API_KEY}"
```

### 12.3 皮肤/主题自定义

```yaml
# ~/.hermes/skins/cyberpunk.yaml
name: cyberpunk
description: 赛博朋克风格
colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
spinner:
  thinking_verbs: ["jacking in", "decrypting", "uploading"]
branding:
  agent_name: "CYBER"
  prompt_symbol: "❯ "
tool_prefix: "╎"
```

使用皮肤：

```
/skin cyberpunk
```

---

## 13. 进阶案例

### 案例 1：自动化代码审查流水线

```
触发条件: GitHub PR 创建
    ↓
步骤 1: 获取 PR 差异 (delegate_task + web)
    ↓
步骤 2: 静态代码分析 (execute_code)
    ↓
步骤 3: 生成审查报告 (LLM)
    ↓
步骤 4: 发布评论到 GitHub (github API)
```

### 案例 2：多源信息聚合

```
用户请求: "给我今日技术摘要"
    ↓
并行执行:
  - 新闻搜索 (web_search)
  - Reddit/HN 热门 (web_extract)
  - arXiv 新论文 (arXiv skill)
    ↓
聚合整理 (LLM)
    ↓
输出格式化的每日简报
```

### 案例 3：智能开发环境

```yaml
# 配置开发专用皮肤
terminal:
  backend: docker
  docker:
    image: "my-dev-env:latest"
    volumes:
      - "/home/user/projects:/workspace"
```

---

## 14. 资源与支持

| 资源 | 位置 |
|------|------|
| 用户指南 | `docs/USER_GUIDE.md` |
| 开发者指南 | `docs/DEVELOPER_GUIDE.md` |
| 代码结构 | `docs/CODE_STRUCTURE.md` |
| 快速入门 | `docs/QUICK_START.md` |
| 斜杠命令 | `/help` |
| 环境诊断 | `hermes doctor` |
| 官方文档 | https://hermes-agent.dev |

---

进阶功能的核心原则：先理解基础架构，再根据实际需求逐步引入高级特性。建议从定时任务或子代理开始尝试，逐步深入到自定义技能和生产部署。
