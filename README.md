# Claude Code 第三方 API 配置 SOP

标准操作流程：配置 `claude` CLI 使用非 Anthropic 官方的第三方 API 代理。

## 适用代理

- **LongCat** — `https://api.longcat.chat/anthropic`
- **小米 MiMo** — `https://token-plan-cn.xiaomimimo.com/anthropic`
- **MoMA（移动云）** — `https://<region>.cmecloud.cn/v1`
- 其他兼容 Anthropic Messages API 的代理

## 使用方法

详见 [`SKILL.md`](./SKILL.md) — 完整的 5 步配置流程。

## 快速开始

```bash
# 1. 前置条件检查
which claude && claude --version

# 2. 测试 API 连通性
curl -s --max-time 15 -X POST "https://你的地址/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 你的key" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"模型名","max_tokens":20,"messages":[{"role":"user","content":"hi"}]}'

# 3. 编辑 ~/.claude/settings.json（见 SKILL.md 模板）

# 4. 验证
claude -p "回复OK" --max-turns 1 --output-format json 2>/dev/null
```

## 主要变更 (v1.1.0)

- 增加前置条件检查步骤
- 修复 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 为字符串类型
- 补全 CloudCLI 补丁的具体 sed 命令
- 模糊化 MoMA 内网域名
- 增加 429 限频、HTTP_PROXY、配置恢复等 FAQ
- 增加 bashrc 权限安全提醒
- 统一 curl 路径与 settings.json 的说明

## 许可证

MIT