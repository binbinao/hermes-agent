# Hermes 快速入门指南

## 首次使用完整案例

### 环境检查

你的配置已经就绪，关键配置项：

```yaml
model:
  default: minimax-m2.7
  provider: custom
  base_url: http://v2.open.venus.oa.com/llmproxy/v1
```

---

## 完整使用案例

### 案例 1：文件操作 + 代码任务

让 Hermes 帮你读写文件、编写代码：

```
你可以对我说：
- "帮我创建一个 Python 文件，实现快速排序"
- "分析这个项目的代码结构"
- "在当前目录创建一个 test.py 文件"
```

**示例命令：**
```bash
# Hermes 可以直接帮你操作文件
# 读取文件内容
# 写入新文件
# 搜索文件内容
```

### 案例 2：网络搜索 + 信息查询

```
- "搜索一下最新的大模型新闻"
- "帮我查找关于 Python 异步编程的教程"
- "搜索今天有什么科技新闻"
```

### 案例 3：终端命令执行

直接让 Hermes 执行 shell 命令：

```
- "用 git status 查看当前目录的 git 状态"
- "帮我运行 python --version"
- "创建一个新目录叫 projects"
```

### 案例 4：工具集（Toolsets）

查看你启用的工具集：

| 工具集 | 功能 |
|--------|------|
| `hermes-cli` | CLI 核心命令（/help, /model, /skin 等） |
| `terminal` | 执行终端命令 |
| `file` | 文件读写、搜索 |
| `web` | 网络搜索、浏览器自动化 |

### 案例 5：快捷命令（Slash Commands）

常用的斜杠命令：

| 命令 | 功能 |
|------|------|
| `/help` | 查看所有可用命令 |
| `/model` | 切换 AI 模型 |
| `/skin` | 切换主题外观 |
| `/skills` | 查看和管理技能 |
| `/resolve` | 分析并修复错误 |
| `/clear` | 清除当前会话 |
| `/background` | 后台执行任务 |
| `/retry` | 重试上一次请求 |

---

## 建议下一步

你可以尝试对我说：

1. **"帮我写一个计算器 Python 程序"** → 文件操作 + 代码生成
2. **"搜索一下最新的大模型新闻"** → 网络搜索
3. **"/help"** → 查看所有可用命令
4. **"给我介绍下你的能力"** → 了解完整功能
5. **"帮我分析这个项目的代码结构"** → 代码分析

---

## 配置文件位置

- **主配置**: `~/.hermes/config.yaml`
- **认证信息**: `~/.hermes/auth.json`
- **会话记录**: `~/.hermes/sessions/`
- **记忆数据**: `~/.hermes/memories/`
- **技能目录**: `~/.hermes/skills/`
- **定时任务**: `~/.hermes/cron/`

---

## 获取更多帮助

- `/help` - 查看所有命令
- `docs/USER_GUIDE.md` - 用户指南
- `docs/DEVELOPER_GUIDE.md` - 开发者指南
