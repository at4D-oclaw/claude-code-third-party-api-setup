---
name: claude-code-third-party-api-setup
description: "SOP: 配置 Claude Code CLI 使用第三方 API（LongCat、小米 MiMo、MoMA 等），含 settings.json 模板、认证方式排查、模型名覆盖、验证步骤"
version: 1.1.0
author: Hermes Agent
---

# Claude Code 第三方 API 配置 SOP

## 适用场景

需要让 `claude` CLI（命令行工具）或 **CloudCLI**（`claude code` 的 Web UI 版本）使用非 Anthropic 官方的 API 代理，例如：
- LongCat（国内代理）
- 小米 MiMo（国内代理）
- 其他兼容 Anthropic Messages API 的代理

## 前置条件

```bash
# 确认 claude CLI 已安装
which claude
claude --version    # 需要 v2.x+

# 确认 npm 全局路径可用
npm root -g
```

## 核心原则

1. **settings.json 优先于 shell 环境变量** — 持久化到 tmux、子代理、`--bare`（跳过插件/钩子加载的极简启动模式）模式
2. **模型名必须显式指定** — 第三方代理几乎都拒绝标准 Claude 模型名
3. **认证方式因代理而异** — 不要假设 `x-api-key` 通用
4. **优先级：CLI 参数 > settings.json env > shell 环境变量**

---

## 第一步：确认 API 兼容性

先直接 curl 测试 API 端点，**不要跳过这一步**。

> ⚠️ 注意：curl 测试的 URL 需包含 `/v1/messages` 路径。settings.json 中 `ANTHROPIC_BASE_URL` 只需配到 `/anthropic`，Claude CLI **会自动追加** `/v1/messages`。两者不冲突。

```bash
# 测试 Anthropic Messages API 格式（x-api-key 认证）
curl -s --max-time 15 -X POST "https://你的地址/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "x-api-key: 你的key" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"模型名","max_tokens":20,"messages":[{"role":"user","content":"hi"}]}'
```

如果返回 `missing_api_key` 或 `invalid_api_key`，换 Bearer 认证再试：

```bash
# Bearer token 认证
curl -s --max-time 15 -X POST "https://你的地址/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 你的key" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"模型名","max_tokens":20,"messages":[{"role":"user","content":"hi"}]}'
```

**预期结果：** 返回 JSON 含 `"type":"message"` 和 `"content"`。如果失败，排查：
- URL 路径是否需要 `/v1` 后缀
- 认证头格式（`x-api-key` vs `Bearer`）
- 模型名是否被代理接受（部分代理只认小写或特定格式）

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
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
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
| `ANTHROPIC_BASE_URL` | 是 | 代理的 Anthropic 兼容端点（CLI 自动追加 `/v1/messages`） |
| `ANTHROPIC_MODEL` | 是 | 覆盖默认模型名，避免 CLI 传标准名被拒 |
| `ANTHROPIC_SMALL_FAST_MODEL` | 推荐 | CLI 处理简单任务时使用的轻量模型 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` / `ANTHROPIC_DEFAULT_OPUS_MODEL` | 推荐 | 控制不同能力等级的后备模型。当用户请求 sonnet/opus 级别能力时使用 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 推荐 | 设为 `"1"`（字符串）禁用遥测等非必要网络请求，国内网络建议开启 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | 可选 | 限制单次输出 token 数 |
| `model`（顶层） | 是 | Claude CLI 读取此字段作为默认模型名 |

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

> ⚠️ 安全提醒：`~/.bashrc` 包含明文 API key，建议设置权限：`chmod 600 ~/.bashrc`。优先使用 settings.json（不暴露给子进程和 `ps aux`）。

---

## 第四步：验证

```bash
claude -p "回复OK" --max-turns 1 --output-format json 2>/dev/null | \
  python3 -c "
import sys, json
d = json.loads(sys.stdin.read())
models = list(d.get('modelUsage', {}).keys())
print('model:', models)
print('status:', d.get('subtype'))
print('cost:', d.get('total_cost_usd'))
"
```

**预期输出：**
```
model: ['代理模型名']
status: success
cost: 0.xxxxx
```

如果超时（60s+），先测 curl 确认网络连通性。如果 JSON 解析失败，先单独跑 `claude -p "OK" --max-turns 1 --output-format json 2>/dev/null` 看原始输出。

---

## 第五步（可选）：CloudCLI Web UI 特殊处理

**CloudCLI** 是 `claude code` 的 Web UI 包装，有自己的模型解析逻辑。settings.json 的 `model` 键对它无效，必须直接修改源文件。

### 5a. 修改模型下拉菜单

在 `modelConstants.js` 中添加模型并设为默认：

```bash
MODEL_FILE="$(npm root -g)/@cloudcli-ai/cloudcli/shared/modelConstants.js"

# 添加模型到 OPTIONS 数组（找到 OPTIONS 行，在数组末尾添加）
sed -i "s/const OPTIONS = \[/const OPTIONS = [\n  { value: 'LongCat-Flash-Chat', label: 'LongCat-Flash-Chat' },\n  { value: 'LongCat-Flash-Lite', label: 'LongCat-Flash-Lite' },\n/" "$MODEL_FILE"

# 设置 DEFAULT 为你用的模型
sed -i 's/const DEFAULT = .*/const DEFAULT = "LongCat-Flash-Chat";/' "$MODEL_FILE"
```

### 5b. 强制 SDK 模型名

在 `claude-sdk.js` 中硬编码模型名：

```bash
SDK_FILE="$(npm root -g)/@cloudcli-ai/cloudcli/server/claude-sdk.js"

# 将 model 赋值语句替换为固定值
sed -i 's/sdkOptions.model = options.model || CLAUDE_MODELS.DEFAULT;/sdkOptions.model = "LongCat-Flash-Chat";/' "$SDK_FILE"
```

> ⚠️ 这两个文件会被 `cloudcli update` 覆盖，更新后需重新打补丁。

---

## 已知代理速查表

| 代理 | Base URL | 认证方式 | 模型名 | 特殊说明 |
|------|----------|----------|--------|----------|
| **LongCat** | `https://api.longcat.chat/anthropic` | Bearer token（需 `AUTH_TOKEN`） | `LongCat-Flash-Chat` / `LongCat-Flash-Lite` | 必须同时设 `API_KEY` + `AUTH_TOKEN` |
| **小米 MiMo** | `https://token-plan-cn.xiaomimimo.com/anthropic` | `x-api-key` | `mimo-v2.5-pro` | 标准 Claude 名全拒，必须传 `--model mimo-v2.5-pro` |
| **MoMA（移动云）** | `https://<region>.cmecloud.cn/v1` | Bearer token | `deepseek-v3.1` 等 | OpenAI 兼容格式，需在 settings.json 加 `"tool_use_enforcement": "manual"`（顶层字段） |

### MoMA 完整配置示例

MoMA 使用 OpenAI 兼容格式而非 Anthropic 格式，需要特殊处理：

```json
{
  "env": {
    "OPENAI_API_KEY": "你的key",
    "OPENAI_BASE_URL": "https://<region>.cmecloud.cn/v1",
    "ANTHROPIC_BASE_URL": "https://<region>.cmecloud.cn/v1/chat/completions",
    "ANTHROPIC_MODEL": "deepseek-v3.1"
  },
  "tool_use_enforcement": "manual",
  "model": "deepseek-v3.1"
}
```

---

## 常见问题排查

### Q: curl 通了但 claude CLI 超时
- 检查 settings.json 的 `env` 块是否写对了（JSON 格式错误会导致被忽略）
- 检查 `ANTHROPIC_BASE_URL` 路径是否以 `/anthropic` 结尾（CLI 自动追加 `/v1/messages`）
- 尝试加 `--bare` 模式排除插件/钩子干扰：`claude --bare -p "test" --max-turns 1`
- 检查网络代理环境：`export HTTP_PROXY` / `HTTPS_PROXY` 是否配置正确

### Q: 返回 400 Bad Request
- 模型名不对 — 代理不接受标准 Claude 模型名
- 在 settings.json 设 `ANTHROPIC_MODEL` 并用 `--model` 显式指定
- 部分代理要求模型名全小写，尝试 `longcat-flash-chat` 等

### Q: 返回 missing_api_key
- 认证头格式不对 — 尝试加 `ANTHROPIC_AUTH_TOKEN`
- 用 curl 分别测试 `x-api-key` 和 `Bearer` 两种方式

### Q: 返回 429 Too Many Requests
- API 限频了 — 等待一段时间重试
- 检查是否有多个进程同时使用同一个 API key
- 部分代理对免费/低频账户有请求限制

### Q: 之前能用的配置突然不行了
- 检查 `cloudcli update` 是否覆盖了补丁文件
- 检查 API key 是否过期
- 重新执行 curl 测试确认 API 本身正常

### Q: 开了新终端但环境变量没生效
- 执行 `source ~/.bashrc` 或开全新终端
- settings.json 的 env 块不受此限制，优先用 settings.json

### Q: 配置改坏了怎么恢复？
```bash
# 备份当前配置
cp ~/.claude/settings.json ~/.claude/settings.json.bak

# 恢复出厂（删除后 claude 会重新生成默认配置）
rm ~/.claude/settings.json

# 从备份恢复
cp ~/.claude/settings.json.bak ~/.claude/settings.json
```