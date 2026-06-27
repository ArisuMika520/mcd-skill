---
name: mcd-mcp
description: 通过麦当劳中国官方 MCP 查询营养、活动日历、麦麦省优惠券、积分、麦麦商城商品与订单、门店和菜单，并完成到店自取、得来速、麦乐送、企业团餐的点餐计价下单、一键领券，以及积分兑换虚拟券或实物商品。用户提到麦当劳、麦乐送、得来速、麦麦省、麦麦商城、麦当劳优惠券或积分兑换，或要求查询/创建相关订单时使用。
---

# 麦当劳中国 MCP

使用你所在 Agent 框架中已配置的麦当劳中国官方 MCP 工具完成任务。本文档统一用官方工具名（连字符形式，如 `query-meals`、`calculate-price`）指代各工具；运行时不同框架可能给工具加前缀或把连字符转为下划线——例如 Claude Code 形如 `mcp__<服务名>__<工具名>`、Hermes 形如 `mcp_mcd_mcp_query_meals`，也有客户端直接暴露 `query-meals`。调用时以本框架工具列表里的实际名称为准，按官方名一一对应即可。不要用 `curl` 手写 JSON-RPC，也不要读取、打印或向用户展示 `MCD_MCP_TOKEN`。

服务器实时返回的工具描述和 input schema 优先于本文。需要确认参数、工具副作用或返回字段时，读取 [references/tools.md](references/tools.md)。

## 接入准备

- 服务由麦当劳中国提供，覆盖中国大陆（不含港澳台）。共 24 个工具，端点 `https://mcp.mcd.cn`，传输 Streamable HTTP，仅支持 MCP 协议 `2025-06-18` 及之前版本。
- Token：在 https://open.mcd.cn/mcp 用手机号登录 →「控制台」→「激活」→ 同意服务协议后复制。Token 保存在环境变量 `MCD_MCP_TOKEN`，请求头使用 `Authorization: Bearer <MCD_MCP_TOKEN>`。
- 接入方式由你的 Agent 框架决定（Claude Code、Cursor、OpenClaw、Hermes 等）：在框架的 MCP 配置中新增一个 Streamable HTTP 服务，核心三项是 `url: "https://mcp.mcd.cn"`、传输类型 Streamable HTTP（配置键名各框架不同，如 `"type": "http"` 或 `"streamablehttp"`）、请求头 `Authorization: Bearer <MCD_MCP_TOKEN>`。各框架的具体写法见 [README](README.md)。框架已注册该服务时直接调用其暴露的工具即可，无需手写配置。
- 用户没有可用 Token、或工具返回 401 时，提示其去上述地址获取/更新 Token，不要索要明文。

## 调用原则

1. 先确定业务类型，再沿用对应的 `beType`、`orderType` 和门店查询工具；`beCode` 一律使用门店查询返回值并回传：
   - 到店自取：`beType=1`、`orderType=1`，门店来自 `query-nearby-stores`。
   - 得来速（DT）：`beType=5`、`orderType=1`，门店来自 `query-nearby-stores`。
   - 麦乐送：`beType=2`、`orderType=2`，门店来自 `delivery-query-stores`。
   - 企业团餐：`beType=6`、`orderType=2`，门店来自 `delivery-query-stores`；计价、下单前先查 `query-meal-assistance` 并传 `gmServiceCode`。
   - `orderType` 只有 `1`（到店）或 `2`（外送）；不要把 `beType` 当作 `orderType`。
2. 只使用上游工具返回的 `storeCode`、`beCode`、餐品编码、券 ID/Code、`skuId`、`spuId`、`addressId`、`takeWayCode`、`gmServiceCode` 和订单号；禁止猜测或复用示例值。
3. 预约场景把同一个 `reservationDate`（`yyyy-MM-dd HH:mm`）传给查券、菜单、餐品详情、助餐、计价和创建订单。相对日期（如“明天中午”）不明确时先调用 `now-time-info`。
4. 价格单位**按工具区分**，不要一概除以 100：
   - 只有 `calculate-price` 返回的价格字段（`productPrice`、`deliveryPrice`、`price`、`originalPrice`、`discount` 等，均为无引号整数）以“分”为单位，展示前除以 100 换算为人民币元。
   - `query-meals`（`currentPrice`）、`query-meal-detail`（`price`）、`create-order`/`query-order`（`totalAmount`、`realTotalAmount`、`productPrice`、`deliveryPrice` 等）、以及麦麦商城订单（`price`、`realTotalAmount`、`goodsTotalPrice` 等）返回的金额**已是“元”字符串**，直接展示、不要再除以 100。
   - 所有积分字段（`points`、`totalPoints`、`goodsTotalPoints` 等）不按金额换算。
5. 读取响应中的 `success`、`code`、`message` 和 `data`，不要只凭 HTTP 成功判定业务成功。营养信息的 `data` 是 TOON 紧凑文本，不要误判为 JSON 数组。

## 点餐流程

### 选择门店

- 到店自取或得来速：调用 `query-nearby-stores`。按位置搜索用 `searchType=2` 并从用户输入获取 `city`、`keyword`；查收藏门店用 `searchType=1`。
- 麦乐送或企业团餐：先调用 `delivery-query-addresses` 让用户选已有地址；无合适地址时，取得完整联系人和地址后调用 `delivery-create-address`，再调用 `delivery-query-stores` 获取 `storeCode` 和 `beCode`。
- 展示候选门店让用户选择，不要擅自决定。门店 `reservation=true` 时完整展示 `reservationTimeOptions` 供用户选预约时段。

### 选品与计价

1. 调用 `query-meals` 获取当前门店、业务类型、时段的实时菜单（含分类、餐品编码、价格、标签）。
2. 用户要看套餐组成或默认搭配时调用 `query-meal-detail`。注意当前版本不支持更换套餐内单品；套餐用其自身餐品编码下单即可。若某促销/买一送一编码在 `query-meal-detail` 报“未匹配到商品”，回退使用 `query-meals` 的名称和价格，直接用该编码下单。
3. 当前订单可用券必须调用 `query-store-coupons`；`query-my-coupons` 只表示账户拥有的券，不能据此承诺当前订单可用。
4. 团餐在计价前调用 `query-meal-assistance`，让用户选择助餐服务并取得 `gmServiceCode`。
5. 按用户最终选择调用 `calculate-price`。`items` 是 JSON 对象数组，每项仅含 `productCode`、`quantity`，需要用券时加 `couponId`/`couponCode`；不要构造 `roundList`、`modification` 等字段。
6. 向用户汇总门店、取餐/配送方式、预约时间、商品与数量、优惠、费用明细和应付总额；金额取自 `calculate-price` 时把分除以 100 换算为元，取自 `create-order`/`query-order` 时已是元、直接展示。

### 创建订单

`create-order` 会创建真实订单。必须在计价成功并取得用户对上述摘要的明确确认后，调用一次：

- 公共必填：`storeCode`、`beCode`、`beType`、`items`（与最终计价一致）。
- 到店/得来速：传 `takeWayCode`，取自 `calculate-price` 返回的 `takeWayList` 中由用户选择的取餐方式。
- 麦乐送/团餐：传用户选定的 `addressId`；团餐还要传 `gmServiceCode`。
- 预约场景传 `reservationDate`。
- 返回 `payH5Url`（当前为 `scanToPay` 扫码支付页）和 `orderId`：告知用户自行在链接或麦当劳 App 完成支付，不要代付。
- 超时或结果不明确时不要盲目重试，先用已返回的 `orderId` 调用 `query-order`，避免重复下单。

## 领券与积分兑换

- 查可领券用 `available-coupons`；仅当用户明确要求一键领取时调用一次 `auto-bind-coupons`。
- 查积分用 `query-my-account`；查卡包内券用 `query-my-coupons`（无入参）。
- 积分兑换：先 `mall-points-products`（可用 `catRuleIds` 筛选，如 `1>4,2` 为商品券+实物商品），需要时 `mall-product-detail` 取 `skuId`、`spuCategory`。
- 按商品类型选择下单工具：
  - 虚拟餐品券（`spuCategory="1"`）用 `mall-create-order`（仅传 `skuId`，可选 `count`）。
  - 实物商品（`spuCategory="2"`）用 `mall-create-order-physical`，传 `skuId`、`spuCategory`，并传用户确认的收货 `addressId`。
  两者都会扣减积分并创建真实兑换订单。调用前向用户确认商品、SKU、数量、所需积分；实物还需确认配送地址。
- 每个商品单独下单；多个商品串行调用，前一单明确成功后再下一单。结果不明时先用 `mall-order-list`/`mall-order-detail` 查询，不要直接重试。

## 隐私与副作用

- 新增地址会向麦当劳发送姓名、手机号和详细地址。只收集任务必需的数据；调用前让用户核对，展示手机号时脱敏（如 `152****6666`）。
- 查账户、门店、菜单、活动、营养、券和订单属读取操作，可按用户请求直接调用。
- 写操作会改变外部账户状态：`delivery-create-address`、`auto-bind-coupons`、`create-order`、`mall-create-order`、`mall-create-order-physical`。用户仅在咨询或浏览时不得调用；缺少明确操作意图或关键字段时先询问。
- 不在回答、日志、命令参数或 skill 文件中暴露 Token。

## 错误处理

- `401`：MCP Token 无效、过期或未提供。请用户在 https://open.mcd.cn/mcp 重新获取并更新 `MCD_MCP_TOKEN`，不要索要明文。
- `429`：触发限流（每 Token 每分钟 600 次）。停止连续重试，降低频率。
- 参数错误：重新读取实时 input schema 和上游返回值补齐，不要用猜测值；尤其 `items` 必须是对象数组、`beCode`/`takeWayCode`/`gmServiceCode` 必须来自上游。
- 创建类操作超时：餐饮订单先 `query-order`，商城订单先 `mall-order-list`/`mall-order-detail`；确认未创建且用户仍同意时才重试。
- 失败响应中的 `traceId` 可在向用户说明问题时提供，但绝不提供 Token。
