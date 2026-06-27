# 麦当劳中国 MCP Skill

用麦当劳中国官方 MCP 服务完成营养查询、活动日历、麦麦省优惠券、积分、麦麦商城商品与订单、门店与菜单查询，以及到店自取、得来速、麦乐送、企业团餐的点餐计价下单、一键领券、积分兑换虚拟券或实物商品。

[SKILL.md](SKILL.md) 与 [references/tools.md](references/tools.md) 全程用官方工具名（`query-meals`、`calculate-price` 等）描述流程

## 仓库结构

| 文件 | 作用 |
|---|---|
| [SKILL.md](SKILL.md) | 技能主体：业务类型、调用原则、点餐/领券/兑换流程、隐私与错误处理。Agent 运行时加载它 |
| [references/tools.md](references/tools.md) | 24 个工具的清单、入参要点与返回值注意点，供 Agent 按需查阅 |
| README.md | 本文：如何获取 Token、在各框架配置 MCP、装载技能 |

## 服务概况

- 提供方：麦当劳中国，覆盖中国大陆（不含港澳台）。
- 端点：`https://mcp.mcd.cn`，传输 **Streamable HTTP**，支持的最高 MCP 协议版本 `2025-06-18`。
- 工具数：24 个；限流：每个 Token 每分钟 600 次（超限返回 `429`）。

## 前置：获取 Token

1. 打开 <https://open.mcd.cn/mcp>，用手机号登录。
2. 进入「控制台」→「激活」→ 同意服务协议。
3. 复制 Token，建议存入环境变量 `MCD_MCP_TOKEN`。
4. 后续所有请求通过请求头 `Authorization: Bearer <MCD_MCP_TOKEN>` 鉴权。Token 过期或缺失时工具返回 `401`，回到上面地址重新获取即可。

> ⚠️ 不要把 Token 写进会被提交或分享的文件，也不要在对话、日志中展示明文。

## 第一步：配置 MCP 服务

核心三项各框架一致：`url = https://mcp.mcd.cn`、传输类型 Streamable HTTP、请求头携带 Bearer Token。下面是常见框架的写法，具体键名以你框架的 MCP 文档为准。

### Claude Code

命令行：

```bash
claude mcp add --transport http mcd-mcp https://mcp.mcd.cn \
  --header "Authorization: Bearer ${MCD_MCP_TOKEN}"
```

或写入 `.mcp.json` / `~/.claude.json`：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "http",
      "url": "https://mcp.mcd.cn",
      "headers": { "Authorization": "Bearer ${MCD_MCP_TOKEN}" }
    }
  }
}
```

### Cursor（`.cursor/mcp.json`）

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "url": "https://mcp.mcd.cn",
      "headers": { "Authorization": "Bearer YOUR_MCP_TOKEN" }
    }
  }
}
```

### Cline / Roo Code（`cline_mcp_settings.json`）

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamableHttp",
      "url": "https://mcp.mcd.cn",
      "headers": { "Authorization": "Bearer YOUR_MCP_TOKEN" }
    }
  }
}
```

### Claude Agent SDK（TypeScript）

```ts
const options = {
  mcpServers: {
    "mcd-mcp": {
      type: "http",
      url: "https://mcp.mcd.cn",
      headers: { Authorization: `Bearer ${process.env.MCD_MCP_TOKEN}` },
    },
  },
};
```

### 通用 MCP 客户端（含 OpenClaw 等）

任何实现了 MCP 的客户端（OpenClaw 及其他未单列的框架同理）：用 **Streamable HTTP** 传输连接 `https://mcp.mcd.cn`，并在 HTTP 头里附带 `Authorization: Bearer <MCD_MCP_TOKEN>`。连接后通过 `tools/list` 即可拿到 24 个工具。具体配置键名（如传输类型字段叫 `type`、`transport` 还是别的）以该框架自己的 MCP 文档为准。

### Hermes

已内置注册该服务，无需手写配置；工具以 `mcp_mcd_mcp_<tool_name>` 形式暴露（连字符转下划线）。

## 第二步：装载技能

技能内容（[SKILL.md](SKILL.md)）的作用是告诉模型**如何正确编排这些工具**——业务类型对应的 `beType`/`orderType`、价格单位陷阱、写操作前必须确认等。装载方式取决于框架是否原生支持 Agent Skills：

- **支持 Skills 的框架**（Claude Code、Claude Agent SDK、Claude.ai）：把整个 `mcd-mcp/` 目录放进框架的 skills 目录（如 Claude Code 的 `~/.claude/skills/` 或项目级 `.claude/skills/`）。框架会按 frontmatter 的 `description` 自动判断何时启用。
- **不支持 Skills 的框架**（Cursor、Cline 等）：把 [SKILL.md](SKILL.md) 的正文作为系统提示 / 项目规则使用（如 Cursor 的 `.cursorrules`、Cline 的 custom instructions），并把 [references/tools.md](references/tools.md) 作为补充资料随需提供给模型。

两种方式下，模型都按官方工具名调用，运行时自动对应到框架暴露的实际名称。

## 工具名对照

| 框架 | `query-meals` 的实际名称 | 说明 |
|---|---|---|
| 官方 / 通用 MCP 客户端 | `query-meals` | 原始连字符名 |
| Claude Code | `mcp__<服务名>__query-meals` | 前缀含你配置的服务名 |
| Hermes | `mcp_mcd_mcp_query_meals` | 连字符转下划线 |

> 服务器实时返回的工具描述与 input schema **优先于文档**。参数有出入时以运行时 schema 为准。

## 安全与隐私要点

- 写操作会改变外部账户状态——`delivery-create-address`、`auto-bind-coupons`、`create-order`、`mall-create-order`、`mall-create-order-physical`——必须在用户明确确认后才调用；浏览/咨询阶段不要触发。
- 新增地址会向麦当劳发送姓名、手机号、详细地址；只收集任务必需数据，展示手机号时脱敏（如 `152****6666`）。
- 支付由用户在返回的链接或麦当劳 App 自行完成，Agent 不代付。
- 任何情况下都不在回答、日志、命令参数或文件中暴露 `MCD_MCP_TOKEN`。

## 价格单位（常见坑）

价格单位**按工具区分**，不要一概除以 100：

- **仅** `calculate-price` 返回的价格字段是「分」（无引号整数），展示前除以 100。
- `query-meals`、`query-meal-detail`、`create-order`、`query-order` 及麦麦商城订单返回的金额已是「元」字符串，直接展示。
- 所有积分字段（`points`、`totalPoints` 等）不按金额换算。

## 参考

- 官方接入指南：<https://open.mcd.cn/mcp/doc>
- 官方仓库：<https://github.com/M-China/mcd-mcp-server>
