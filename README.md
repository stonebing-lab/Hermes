# Hermes Agent 安装配置指南

## 项目简介

**Hermes Agent** 是 Nous Research 开发的自我改进型 AI Agent，具有内置学习循环、技能系统、跨平台消息支持等特性。

- **官方仓库**: https://github.com/nousresearch/hermes-agent
- **文档**: https://hermes-agent.nousresearch.com/docs/
- **版本**: v0.8.0 (2026.4.8)

## 系统要求

- **操作系统**: macOS / Linux / WSL2
- **Python**: 3.11+
- **包管理器**: uv (自动安装)
- **依赖**: Node.js, Git, ripgrep, ffmpeg

## 安装步骤

### 1. 一键安装

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

安装脚本会自动：
- 安装 Python 3.11（通过 uv）
- 克隆仓库到 `~/.hermes/hermes-agent`
- 安装依赖（ripgrep, ffmpeg, Playwright）
- 创建配置文件模板
- 同步 78 个内置 skills

### 2. 添加 PATH

安装后需要将 `~/.local/bin` 加入 PATH：

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

验证安装：

```bash
hermes --version
# 输出: Hermes Agent v0.8.0 (2026.4.8)
```

## 配置 Qwen 模型

### 问题：Intel Mac 不支持 qwen-cli

在 Intel Mac (x86_64) 上，`qwen-cli` 只有 ARM64 版本，无法直接使用 Qwen OAuth。

### 解决方案：使用 DashScope API

通过阿里云 DashScope 的 OpenAI 兼容接口访问 Qwen 模型。

#### 1. 配置环境变量

编辑 `~/.hermes/.env`：

```bash
# Alibaba Cloud DashScope (Qwen)
DASHSCOPE_API_KEY=sk-your-api-key-here
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1

# Terminal 配置
TERMINAL_ENV=local
TERMINAL_TIMEOUT=60
TERMINAL_LIFETIME_SECONDS=300
```

**注意**：
- 使用 `DASHSCOPE_API_KEY`，不是 `OPENAI_API_KEY`
- Base URL 必须是国内版 `dashscope.aliyuncs.com`，不是国际版 `dashscope-intl.aliyuncs.com`

#### 2. 配置模型

编辑 `~/.hermes/config.yaml`：

```yaml
model:
  default: qwen3.5-plus
  provider: alibaba
  base_url: https://dashscope.aliyuncs.com/compatible-mode/v1
```

**关键点**：
- `provider` 必须设置为 `alibaba`
- `base_url` 必须显式指定国内版地址（覆盖 models.dev 的默认国际版）

#### 3. 验证配置

```bash
hermes model
# 应显示:
# Current model: qwen3.5-plus
# Active provider: alibaba
```

## 常见问题

### 1. HTTP 401 认证失败

**错误信息**：
```
HTTP 401: Incorrect API key provided
Endpoint: https://dashscope-intl.aliyuncs.com/compatible-mode/v1
```

**原因**：Hermes 连接了国际版 endpoint，但 API key 是国内版的。

**解决**：
1. 检查 `~/.hermes/.env` 中 `DASHSCOPE_BASE_URL` 是否为国内版
2. 检查 `~/.hermes/config.yaml` 中 `model.base_url` 是否为国内版
3. 两处都必须是 `https://dashscope.aliyuncs.com/compatible-mode/v1`

### 2. hermes 命令找不到

**错误信息**：
```
zsh: command not found: hermes
```

**解决**：
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 3. setup wizard 无法运行

**错误信息**：
```
/dev/tty: Device not configured
```

**原因**：在非交互式环境（如通过 exec 工具）运行，没有 TTY。

**解决**：在真实终端（Terminal.app / iTerm2）中运行 `hermes setup`，或直接手动编辑配置文件。

## 从 OpenClaw 迁移

Hermes 支持从 OpenClaw 自动导入配置：

```bash
hermes claw migrate --dry-run   # 预览迁移内容
hermes claw migrate             # 正式迁移
```

会导入：
- SOUL.md（persona）
- 记忆文件（MEMORY.md, USER.md）
- Skills
- API keys（Telegram, OpenRouter, OpenAI, Anthropic 等）
- 命令白名单
- 消息平台配置

## 使用

### 启动对话

```bash
hermes
```

### 常用命令

```bash
hermes model              # 查看/切换模型
hermes setup              # 配置向导
hermes gateway            # 启动消息网关（Telegram/Discord 等）
hermes skills             # 管理 skills
hermes config set         # 修改配置
hermes doctor             # 诊断问题
hermes update             # 更新到最新版本
```

### 对话中的命令

- `/new` - 开始新对话
- `/model [provider:model]` - 切换模型
- `/skills` - 浏览 skills
- `/retry` - 重试上一轮
- `/compress` - 压缩上下文
- `/usage` - 查看 token 使用

## 配置文件位置

- **主配置**: `~/.hermes/config.yaml`
- **环境变量**: `~/.hermes/.env`
- **Persona**: `~/.hermes/SOUL.md`
- **Skills**: `~/.hermes/skills/`
- **日志**: `~/.hermes/logs/`

## 支持的 LLM 提供商

- OpenRouter（200+ 模型）
- Alibaba Cloud / DashScope（Qwen 系列）
- Google AI Studio（Gemini）
- Z.AI / GLM（智谱）
- Kimi / Moonshot
- MiniMax
- OpenAI
- Anthropic
- GitHub Copilot
- 自定义 OpenAI 兼容端点

## 技术架构

- **包管理**: uv（比 pip 更快）
- **Python**: 3.11.15
- **浏览器**: Playwright Chromium
- **终端后端**: local / docker / ssh / modal / singularity
- **Skills**: 78 个内置 + 用户自定义

## 安装日期

2026-04-13

## 参考资源

- 官方文档: https://hermes-agent.nousresearch.com/docs/
- GitHub: https://github.com/nousresearch/hermes-agent
- Discord: https://discord.gg/NousResearch
- Skills Hub: https://agentskills.io
