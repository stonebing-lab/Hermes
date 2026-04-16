# Hermes 模型切换问题调试记录

**日期**：2026-04-16  
**问题**：Hermes 无法在 qwen-plus 和 claude-sonnet-4-6 之间正常切换  
**最终状态**：✅ 已解决

---

## 问题现象

用户在 Hermes 中尝试切换模型时遇到以下问题：

1. **切换到 qwen-plus 时报错**：
   ```
   Unknown provider 'dashscope'
   ```

2. **切换到 claude-sonnet-4-6 时报错**：
   ```
   Unknown provider 'renrenai'
   Provider resolver returned an empty API key
   HTTP 401: Missing Authentication header
   Endpoint: https://openrouter.ai/api/v1
   ```

---

## 调试过程

### 第一阶段：Provider 名称问题

**问题分析**：
- 用户配置中使用了 `dashscope` 和 `renrenai` 作为 provider 名称
- 这两个都不是 Hermes 内置的 provider

**发现**：
通过检查 `/Users/stonebing/.hermes/hermes-agent/hermes_cli/auth.py`，找到 Hermes 支持的内置 provider：
- `alibaba` ✅（支持 DashScope/Qwen）
- `openrouter` ✅（支持多种模型）
- `anthropic`, `deepseek`, `gemini` 等

**解决方案 1**：
- qwen-plus：使用 `alibaba` provider
- claude-sonnet-4-6：尝试使用 `openrouter` provider

**配置修改**：
```yaml
# config.yaml
providers:
  alibaba:
    base_url: https://dashscope.aliyuncs.com/compatible-mode/v1
    api_key: sk-5f9565da1beb4ceb8926efaf9c5c156f
  openrouter:
    base_url: https://renrenai.chat/v1
    api_key: sk-wtF8W3sjvjNzzrWJT74V6GL7HFDyi2ZaGzrGrPZCOCcCc8cN
```

**结果**：
- ✅ qwen-plus 切换成功
- ❌ claude-sonnet-4-6 仍然失败

---

### 第二阶段：OpenRouter Base URL 覆盖问题

**问题分析**：
虽然在 `config.yaml` 中配置了 `openrouter` 的 `base_url: https://renrenai.chat/v1`，但 Hermes 仍然连接到 `https://openrouter.ai/api/v1`。

**原因**：
Hermes 的内置 `openrouter` provider 会忽略用户配置的 `base_url`，强制使用内置的 OpenRouter endpoint。

**尝试的解决方案**：

#### 方案 A：直连模式（失败）
尝试不使用 provider，直接在 `model` 部分配置 `base_url` 和 `api_key`：
```yaml
model:
  default: claude-sonnet-4-6
  base_url: https://renrenai.chat/v1
  api_key: sk-wtF8W3sjvjNzzrWJT74V6GL7HFDyi2ZaGzrGrPZCOCcCc8cN
```
**结果**：❌ Hermes 根据模型名称 `claude-sonnet-4-6` 自动推断为 OpenRouter 模型，仍然连接到 openrouter.ai

#### 方案 B：自定义 Provider（失败）
尝试创建自定义 provider `custom-claude`：
```yaml
providers:
  custom-claude:
    base_url: https://renrenai.chat/v1
    api_key: sk-wtF8W3sjvjNzzrWJT74V6GL7HFDyi2ZaGzrGrPZCOCcCc8cN
```
**结果**：❌ Hermes 不认识自定义 provider

#### 方案 C：使用通用模型名称（失败）
尝试使用 Hermes 不认识的模型名称（如 `gpt-4`, `renrenai-claude`）：
**结果**：❌ Hermes 仍然会根据模型名称自动推断 provider

---

### 第三阶段：发现环境变量覆盖机制（成功）

**关键发现**：
通过阅读 `/Users/stonebing/.hermes/hermes-agent/hermes_cli/providers.py`，发现 Hermes 支持通过环境变量覆盖 provider 的 `base_url`：

```python
HERMES_OVERLAYS: Dict[str, HermesOverlay] = {
    "openrouter": HermesOverlay(
        transport="openai_chat",
        is_aggregator=True,
        extra_env_vars=("OPENAI_API_KEY",),
        base_url_env_var="OPENROUTER_BASE_URL",  # 🔑 关键！
    ),
    ...
}
```

**最终解决方案**：
在 `.env` 文件中添加 `OPENROUTER_BASE_URL` 环境变量：

```bash
# .env
OPENROUTER_API_KEY=sk-wtF8W3sjvjNzzrWJT74V6GL7HFDyi2ZaGzrGrPZCOCcCc8cN
OPENROUTER_BASE_URL=https://renrenai.chat/v1
```

**配置文件**：
```yaml
# config.yaml
model:
  default: claude-sonnet-4-6
  provider: openrouter
  base_url: https://renrenai.chat/v1
```

**结果**：✅ 成功！Hermes 现在连接到 `https://renrenai.chat/v1`

---

## 最终配置

### 1. 环境变量（`.env`）
```bash
# Alibaba Cloud DashScope (Qwen)
DASHSCOPE_API_KEY=sk-5f9565da1beb4ceb8926efaf9c5c156f
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1

# OpenRouter (renrenai.chat) - 使用环境变量覆盖 base_url
OPENROUTER_API_KEY=sk-wtF8W3sjvjNzzrWJT74V6GL7HFDyi2ZaGzrGrPZCOCcCc8cN
OPENROUTER_BASE_URL=https://renrenai.chat/v1
```

### 2. 配置文件（`config.yaml`）
```yaml
model:
  default: qwen-plus  # 或 claude-sonnet-4-6
  provider: alibaba   # 或 openrouter
  base_url: https://dashscope.aliyuncs.com/compatible-mode/v1
  tool_progress: 'off'
  background_process_notifications: all

providers:
  alibaba:
    base_url: https://dashscope.aliyuncs.com/compatible-mode/v1
    api_key: sk-5f9565da1beb4ceb8926efaf9c5c156f
  openrouter:
    base_url: https://renrenai.chat/v1
    api_key: sk-wtF8W3sjvjNzzrWJT74V6GL7HFDyi2ZaGzrGrPZCOCcCc8cN
```

### 3. 切换脚本

创建了两个便捷脚本用于快速切换：

**`hermes-switch-qwen`**：
```python
#!/usr/bin/env python3
config_path = '/Users/stonebing/.hermes/config.yaml'

# 修改 model 部分
model:
  default: qwen-plus
  provider: alibaba
  base_url: https://dashscope.aliyuncs.com/compatible-mode/v1
  tool_progress: 'off'
  background_process_notifications: all
```

**`hermes-switch-claude`**：
```python
#!/usr/bin/env python3
config_path = '/Users/stonebing/.hermes/config.yaml'

# 修改 model 部分
model:
  default: claude-sonnet-4-6
  provider: openrouter
  base_url: https://renrenai.chat/v1
  tool_progress: 'off'
  background_process_notifications: all
```

---

## 使用方法

### 方法 1：使用切换脚本（推荐）
```bash
# 切换到 qwen-plus
hermes-switch-qwen

# 切换到 claude-sonnet-4-6
hermes-switch-claude
```

### 方法 2：在 Hermes 会话中使用 /model 命令
```bash
# 切换到 qwen-plus
/model qwen-plus --provider alibaba --global

# 切换到 claude-sonnet-4-6
/model claude-sonnet-4-6 --provider openrouter --global
```

---

## 关键经验总结

### 1. Hermes Provider 机制
- Hermes 只认识内置的 provider（`alibaba`, `openrouter`, `anthropic` 等）
- 自定义 provider 名称不被支持
- 模型名称会触发自动 provider 推断（如 `claude-sonnet-4-6` → `openrouter`）

### 2. Base URL 覆盖优先级
```
环境变量 (OPENROUTER_BASE_URL) > 内置 provider 配置 > config.yaml
```

### 3. 环境变量命名规则
每个 provider 都有对应的环境变量：
- `OPENROUTER_BASE_URL` - OpenRouter
- `DASHSCOPE_BASE_URL` - Alibaba/DashScope
- `DEEPSEEK_BASE_URL` - DeepSeek
- `MINIMAX_BASE_URL` - MiniMax
- 等等

可以在 `hermes_cli/providers.py` 的 `HERMES_OVERLAYS` 中查看完整列表。

### 4. 为什么 OpenClaw 更灵活
OpenClaw 的配置更灵活，支持完全自定义 provider：
```json
{
  "providers": {
    "renrenai": {
      "baseURL": "https://renrenai.chat/v1",
      "apiKey": "sk-xxx"
    }
  }
}
```
然后可以直接用 `/model renrenai/claude-sonnet-4-6` 切换，不会有自动推断的问题。

---

## 故障排查清单

如果模型切换失败，按以下顺序检查：

1. **Provider 名称是否正确**
   ```bash
   hermes config show | grep -A 2 "◆ Model"
   ```
   确认 `provider` 是 Hermes 内置支持的名称

2. **环境变量是否设置**
   ```bash
   cat ~/.hermes/.env | grep -E "API_KEY|BASE_URL"
   ```
   确认对应 provider 的 API key 和 base_url 环境变量已设置

3. **API key 是否有效**
   ```bash
   hermes doctor
   ```
   查看 API 连接状态

4. **Base URL 是否生效**
   查看错误日志中的 `Endpoint:` 字段，确认是否连接到正确的 URL

5. **模型名称是否正确**
   ```bash
   curl -s https://renrenai.chat/v1/models \
     -H "Authorization: Bearer sk-xxx" | \
     jq -r '.data[].id' | grep claude
   ```
   确认 API 支持该模型名称

---

## 参考资料

- Hermes 源码：`/Users/stonebing/.hermes/hermes-agent/`
- Provider 配置：`hermes_cli/providers.py`
- 模型切换逻辑：`hermes_cli/model_switch.py`
- 配置文件：`~/.hermes/config.yaml`
- 环境变量：`~/.hermes/.env`

---

**调试时间**：约 1 小时  
**尝试方案数**：6 个  
**最终成功方案**：环境变量覆盖机制

🦞
