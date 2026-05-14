# Claude Code 第三方 API 配置 SOP

标准操作流程：配置 `claude` CLI 使用非 Anthropic 官方的第三方 API 代理。

## 适用代理

- **LongCat** — `https://api.longcat.chat/anthropic`
- **小米 MiMo** — `https://token-plan-cn.xiaomimimo.com/anthropic`
- **MoMA（移动云）** — `https://zhenze-huhehaote.cmecloud.cn/v1`
- 其他兼容 Anthropic Messages API 的代理

## 使用方法

详见 [`SKILL.md`](./SKILL.md) — 完整的 5 步配置流程。

## 快速开始

```bash
# 1. 测试 API 连通性
curl -s --max-time 15 -X POST "https://你的地址/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 你的key" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"模型名","max_tokens":20,"messages":[{"role":"user","content":"hi"}]}'

# 2. 编辑 ~/.claude/settings.json
# 3. 验证
claude -p "回复OK" --max-turns 1 --output-format json
```

## 许可证

MIT