---
name: claude-code-third-party-api-setup
description: "SOP: 配置 Claude Code CLI 使用第三方 API（LongCat、小米 MiMo、MoMA 等），含 settings.json 模板、认证方式排查、模型名覆盖、验证步骤"
version: 1.0.0
author: Hermes Agent
---

# Claude Code 第三方 API 配置 SOP

## 适用场景

需要让 `claude` CLI 或 CloudCLI（Web UI）使用非 Anthropic 官方的 API 代理，例如：
- LongCat（国内代理）
- 小米 MiMo（国内代理）
- 其他兼容 Anthropic Messages API 的代理

## 核心原则

1. **settings.json 优先于 shell 环境变量** — 持久化到 tmux、子代理、`--bare` 模式
2. **模型名必须显式指定** — 第三方代理几乎都拒绝标准 Claude 模型名
3. **认证方式因代理而异** — 不要假设 `x-api-key` 通用

---

## 第一步：确认 API 兼容性

先直接 curl 测试 API 端点，**不要跳过这一步**。

```bash
# 测试 Anthropic Messages API 格式
curl -s --max-time 15 -X POST "https://你的地址/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "x-api-key: 你的key" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"模型名","max_tokens":20,"messages":[{"role":"user","content":"hi"}]}'
```

如果返回 `missing_api_key` 或 `invalid_api_key`，换 Bearer 认证再试：

```bash
curl -s --max-time 15 -X POST "https://你的地址/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 你的key" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"模型名","max_tokens":20,"messages":[{"role":"user","content":"hi"}]}'
```

**预期结果：** 返回 JSON 含 `"type":"message"` 和 `"content"`。如果失败，排查：
- URL 路径是否需要 `/v1` 后缀
- 认证头格式（`x-api-key` vs `Bearer`）
- 模型名是否被代理接受

---

## 第二步：配置 settings.json

编辑 `~/.claude/settings.json`：

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "你的key",
    "ANTHROPIC_AUTH_TOKEN": "你的key",
    "ANTHROPIC_BASE_URL": "https://你的地址/anthropic",
    "ANTHROPIC_MODEL": "代理的主模型名",
    "ANTHROPIC_SMALL_FAST_MODEL": "代理的轻量模型名",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "代理的主模型名",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "代理的主模型名",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "6000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": 1
  },
  "permissions": {
    "allow": [],
    "deny": []
  },
  "model": "代理的主模型名",
  "theme": "dark-daltonized"
}
```

### 关键说明

| 字段 | 必须？ | 说明 |
|------|--------|------|
| `ANTHROPIC_API_KEY` | 是 | Claude CLI 标准认证，发 `x-api-key` 头 |
| `ANTHROPIC_AUTH_TOKEN` | 视代理而定 | 某些代理（如 LongCat）用 Bearer token，必须额外加这个 |
| `ANTHROPIC_BASE_URL` | 是 | 代理的 Anthropic 兼容端点 |
| `ANTHROPIC_MODEL` | 是 | 覆盖默认模型名，避免 CLI 传标准名被拒 |
| `model`（顶层） | 是 | 某些场景（如 CloudCLI）读这个字段 |

**为什么 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN` 都要设？**
- Claude CLI 读 `ANTHROPIC_API_KEY` 发 `x-api-key` 头
- 某些代理（LongCat）只认 `Authorization: Bearer`，需要 `ANTHROPIC_AUTH_TOKEN`
- 两个都设确保兼容性，不会造成冲突

---

## 第三步：配置 shell 环境变量（可选但推荐）

编辑 `~/.bashrc`，追加：

```bash
# Claude Code - 第三方 API
export ANTHROPIC_API_KEY="你的key"
export ANTHROPIC_AUTH_TOKEN="你的key"
export ANTHROPIC_BASE_URL="https://你的地址/anthropic"
export ANTHROPIC_MODEL="代理的主模型名"
export ANTHROPIC_SMALL_FAST_MODEL="代理的轻量模型名"
```

然后 `source ~/.bashrc` 或开新终端。

---

## 第四步：验证

```bash
claude -p "回复OK" --max-turns 1 --output-format json 2>&1 | \
  python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print('model:', list(d.get('modelUsage',{}).keys())); print('status:', d.get('subtype')); print('cost:', d.get('total_cost_usd'))"
```

**预期输出：**
```
model: ['代理模型名']
status: success
cost: 0.xxxxx
```

如果超时（60s+），先测 curl 确认网络连通性。

---

## 第五步（可选）：CloudCLI Web UI 特殊处理

CloudCLI 有自己的模型解析逻辑，settings.json 的 `model` 键对它无效。需要打补丁：

### 5a. 修改模型下拉菜单
```bash
# 找到 modelConstants.js
MODEL_FILE="$(npm root -g)/@cloudcli-ai/cloudcli/shared/modelConstants.js"
# 添加模型到 OPTIONS 并设置 DEFAULT
```

### 5b. 强制 SDK 模型名
```bash
SDK_FILE="$(npm root -g)/@cloudcli-ai/cloudcli/server/claude-sdk.js"
# 硬编码 model 字段
```

> ⚠️ 这两个文件会被 `cloudcli update` 覆盖，更新后需重新打补丁。

---

## 已知代理速查表

| 代理 | Base URL | 认证方式 | 模型名 | 特殊说明 |
|------|----------|----------|--------|----------|
| **LongCat** | `https://api.longcat.chat/anthropic` | Bearer token（需 `AUTH_TOKEN`） | `LongCat-Flash-Chat` / `LongCat-Flash-Lite` | 必须同时设 `API_KEY` + `AUTH_TOKEN` |
| **小米 MiMo** | `https://token-plan-cn.xiaomimimo.com/anthropic` | `x-api-key` | `mimo-v2.5-pro` | 标准 Claude 名全拒，必须传 `--model mimo-v2.5-pro` |
| **MoMA（移动云）** | `https://zhenze-huhehaote.cmecloud.cn/v1` | Bearer token | `deepseek-v3.1` 等 | OpenAI 兼容格式，需用 `tool_use_enforcement: manual` |

---

## 常见问题排查

### Q: curl 通了但 claude CLI 超时
- 检查 settings.json 的 `env` 块是否写对了（JSON 格式错误会导致被忽略）
- 检查 `ANTHROPIC_BASE_URL` 路径是否以 `/anthropic` 结尾
- 尝试加 `--bare` 模式排除插件干扰

### Q: 返回 400 Bad Request
- 模型名不对 — 代理不接受标准 Claude 模型名
- 在 settings.json 设 `ANTHROPIC_MODEL` 并用 `--model` 显式指定

### Q: 返回 missing_api_key
- 认证头格式不对 — 尝试加 `ANTHROPIC_AUTH_TOKEN`
- 用 curl 分别测试 `x-api-key` 和 `Bearer` 两种方式

### Q: 之前能用的配置突然不行了
- 检查 `cloudcli update` 是否覆盖了补丁文件
- 检查 API key 是否过期
- 重新执行 curl 测试确认 API 本身正常

### Q: 开了新终端但环境变量没生效
- 执行 `source ~/.bashrc` 或开全新终端
- settings.json 的 env 块不受此限制，优先用 settings.json