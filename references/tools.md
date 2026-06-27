# 麦当劳中国 MCP 工具参考

来源：[麦当劳中国 MCP 官方接入指南](https://open.mcd.cn/mcp/doc) 与官方仓库 [M-China/mcd-mcp-server](https://github.com/M-China/mcd-mcp-server) README（v1.0.4，2026-05-21）。核对日期：2026-06-27。**服务器实时返回的 input schema 优先于本文件**——参数有出入时以运行时 schema 为准。

## 连接约束

- 端点：`https://mcp.mcd.cn`
- 传输：Streamable HTTP（MCP 配置 `type: "streamablehttp"`）
- 认证：`Authorization: Bearer <MCD_MCP_TOKEN>`
- 支持的最高 MCP 协议版本：`2025-06-18`
- 限流：每个 Token 每分钟 600 次，超限返回 `429`
- Token 申请：https://open.mcd.cn/mcp （登录 → 控制台 → 激活 → 同意协议 → 复制）
- 运行时工具名：本文档一律用官方连字符名（如 `query-meals`）。各 Agent 框架可能加前缀或把连字符转为下划线（如 Claude Code `mcp__<服务名>__<工具名>`、Hermes `mcp_mcd_mcp_query_meals`，或直接 `query-meals`）；调用时以框架工具列表中的实际名称为准。

## 业务类型与公共参数

| 场景 | `beType` | `orderType` | 门店查询工具 | `beCode` |
|---|---|---|---|---|
| 到店自取 | 1 | 1 | `query-nearby-stores` | 由门店查询返回，回传 |
| 得来速（DT） | 5 | 1 | `query-nearby-stores` | 由门店查询返回，回传 |
| 麦乐送 | 2 | 2 | `delivery-query-stores` | 由门店查询返回，回传 |
| 企业团餐 | 6 | 2 | `delivery-query-stores` | 由门店查询返回，回传 |

- `orderType` 仅 `1`（到店）或 `2`（外送）；`beType` 取 `1/2/5/6`。两者含义不同，不要混用。
- `beCode`（Business Entity Code）由门店查询工具对所有场景（含到店自取）返回，点餐链路上的查券、菜单、详情、计价、下单均应回传同一 `beCode`；`create-order` 将其标为必填。
- `reservationDate`：预约场景必填，格式 `yyyy-MM-dd HH:mm`，需在查券、菜单、详情、助餐、计价、下单间保持一致。

## 工具清单（24 个）

| 官方工具名 | 用途 | 操作类型 |
|---|---|---|
| `list-nutrition-foods` | 查询常见餐品营养数据 | 读取 |
| `delivery-query-addresses` | 查询用户配送地址 | 读取敏感数据 |
| `delivery-create-address` | 新增配送地址 | 写入、传输个人信息 |
| `delivery-query-stores` | 按配送地址查询麦乐送/团餐门店 | 读取 |
| `query-meal-assistance` | 查询团餐助餐服务 | 读取 |
| `query-nearby-stores` | 查询到店自取/得来速门店 | 读取 |
| `query-store-coupons` | 查询当前门店和订单可用券 | 读取 |
| `query-meals` | 查询实时菜单 | 读取 |
| `query-meal-detail` | 查询套餐组成与默认搭配 | 读取 |
| `calculate-price` | 计算商品、优惠和费用 | 读取/试算 |
| `create-order` | 创建餐饮订单并返回支付链接 | 写入、创建订单 |
| `query-order` | 查询餐饮订单详情/状态 | 读取 |
| `campaign-calendar` | 查询活动日历 | 读取 |
| `available-coupons` | 查询可领取的麦麦省券 | 读取 |
| `auto-bind-coupons` | 一键领取所有当前可领券 | 写入、领券 |
| `query-my-coupons` | 查询账户卡包中的券 | 读取 |
| `query-my-account` | 查询积分账户 | 读取 |
| `mall-points-products` | 查询积分可兑换商品 | 读取 |
| `mall-product-detail` | 查询兑换商品与 SKU 详情 | 读取 |
| `mall-create-order` | 积分兑换虚拟餐品券下单 | 写入、扣减积分 |
| `mall-create-order-physical` | 积分兑换实物商品下单 | 写入、扣减积分、传输地址 |
| `mall-order-list` | 查询麦麦商城订单列表 | 读取 |
| `mall-order-detail` | 查询麦麦商城订单详情 | 读取 |
| `now-time-info` | 查询服务器当前时间 | 读取 |

## 关键入参

### 无入参

`list-nutrition-foods`、`delivery-query-addresses`、`available-coupons`、`auto-bind-coupons`、`query-my-coupons`、`query-my-account`、`mall-points-products`（也可选 `catRuleIds`）、`now-time-info` 传空对象。

### 地址与门店

`delivery-create-address`

- 必填：`city`、`contactName`、`phone`（11 位纯数字）、`address`、`addressDetail`
- 可选：`gender`（如“先生”“女士”）

`delivery-query-stores`

- 必填：`addressId`、`beType`（仅 `2` 麦乐送或 `6` 企业团餐）
- 返回 `storeCode`、`beCode`、`reservation`、`reservationTimeOptions` 等。

`query-nearby-stores`

- 必填：`searchType`（`2` 按位置 / `1` 收藏门店，默认 `1`）、`beType`（仅 `1` 到店自取或 `5` 得来速）
- `searchType=2` 时另必填：`city`、`keyword`
- 返回 `storeCode`、`beCode`、`reservation`、`reservationTimeOptions` 等。

### 菜单、券、计价、下单

公共字段见上文「业务类型与公共参数」。以下列各工具的附加要点：

`query-store-coupons` / `query-meals`

- 必填：`storeCode`、`orderType`、`beType`；回传 `beCode`；预约必填 `reservationDate`。

`query-meal-detail`

- 必填：`code`（餐品编码）、`orderType`、`beType`；附 `storeCode`、`beCode`；预约必填 `reservationDate`。
- 当前版本暂不支持更换套餐内单品。返回的 `rounds/choices` 仅供展示套餐组成。

`query-meal-assistance`（仅团餐）

- 必填：`storeCode`、`beCode`、`beType=6`、`orderType=2`；预约可传 `reservationDate`。
- 返回 `mealAssistanceItems[].gmServiceCode` 供计价/下单使用。

`calculate-price`

- 必填：`storeCode`、`orderType`、`beType`、`items`；回传 `beCode`；团餐必填 `gmServiceCode`；预约必填 `reservationDate`。
- 返回价格字段（`productPrice`、`deliveryPrice`、`price` 等）以**分**为单位；`takeWayList` 供到店场景取 `takeWayCode`。

`create-order`

- 公共必填：`storeCode`、`beCode`、`beType`、`items`（与最终计价一致）。
- 外送/团餐必填：`addressId`；团餐另必填 `gmServiceCode`。
- 到店/得来速必填：`takeWayCode`，来自 `calculate-price` 返回的 `takeWayList` 中用户选定项。
- 预约场景必填：`reservationDate`。
- 返回 `orderId`、`payId`、`payH5Url`（当前为 `scanToPay` 扫码支付页）及订单明细。

`query-order`

- 必填：`orderId`。

#### `items[]` 结构（`calculate-price` 与 `create-order` 通用）

- `productCode`：来自 `query-meals` / `query-meal-detail`
- `quantity`
- 可选 `couponId`、`couponCode`（使用优惠券时）

`items` 必须是 JSON 对象数组，不能传字符串。结构是扁平的——**没有** `roundList`、`modification`、`comboItemList`、`selectedKey`、`unselectedKey` 等输入字段；套餐直接用其餐品编码下单，不能在下单接口内更换套餐组成。

### 活动

`campaign-calendar`

- 可选：`specifiedDate`（`yyyy-MM-dd`）。传入时返回该日期附近共三天的活动；不传时查询当前月。
- 返回为营销文案 Markdown，可能含表情符号。

### 积分与麦麦商城

`mall-points-products`

- 可选：`catRuleIds`（String，逗号分隔的类目筛选）。官方示例 `1>4,2`（商品券 + 实物商品）。更细类目可参考返回项的 `categoryRuleId`。
- 返回 `spuName`、`spuId`、`point` 等。

`mall-product-detail`

- 必填：`spuId`（Long）。
- 返回 `skuList[].skuId`、`spuCategory`（`"1"` 虚拟 / `"2"` 实物）、`categoryRuleId` 等。

`mall-create-order`（虚拟餐品券）

- 必填：`skuId`（Long）
- 可选：`count`（默认 1）
- 返回 `orderId`、`coupons[].couponCodes`、`orderStatus`/`status`。

`mall-create-order-physical`（实物商品）

- 必填：`skuId`（Long）
- 可选：`count`（默认 1）
- `spuCategory`（String，`"1"` 虚拟 / `"2"` 实物，取自商品列表或详情）
- `spuCategory="2"`（实物）时 `addressId` 必填
- 返回 `orderId`、`orderStatus`/`status`、`orderDetailVo`、`addressVO`。

> 单个商品单独下单；多个商品串行创建，前一单明确成功后再下一单。

`mall-order-list`

- 可选：`lastId`（Long，上一页最后订单 id，用于翻页）、`size`（Integer）
- 返回 `hasNext`、`lastId`、`list[]`。

`mall-order-detail`

- 必填：`orderId`（String）。

### 通用

`now-time-info`

- 无入参。返回 `timestamp`、`formatted`、`date`、`dayOfWeek`、`timezone` 等。相对日期/预约时间不明确时先调用。

## 返回值注意点

- 所有工具先检查顶层 `success`、`code`、`message`、`data`；失败响应可含 `traceId`，向用户说明问题时可提供 traceId，但不要提供 Token。
- `list-nutrition-foods.data` 为 TOON 紧凑文本，不是 JSON 数组。
- 价格单位**按工具区分**：仅 `calculate-price` 的价格字段（无引号整数）以分为单位，展示前除以 100；`query-meals`、`query-meal-detail`、`create-order`、`query-order` 及麦麦商城订单（`mall-create-order-physical`/`mall-order-list`/`mall-order-detail`）返回的金额（`currentPrice`、`price`、`totalAmount`、`realTotalAmount`、`goodsTotalPrice` 等）已是“元”字符串，直接展示，不要再除以 100。`points`、`totalPoints`、`goodsTotalPoints` 等积分字段不按金额换算。
- `query-my-coupons` 返回账户拥有的券，不能替代 `query-store-coupons` 的下单可用性校验。
- `create-order` 返回支付链接和订单号，支付由用户自行完成。
- `mall-create-order` / `mall-create-order-physical` 返回订单状态，不能只凭模型文字宣称兑换成功；状态不明时用 `mall-order-list` / `mall-order-detail` 查询。
