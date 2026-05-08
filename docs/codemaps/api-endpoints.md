# Hummingbot API 端点文档

> 自动生成，基于项目源码扫描。最后更新：2026-04-22

## 目录

- [1. Gateway HTTP API](#1-gateway-http-api)
  - [1.1 基础/状态](#11-基础状态)
  - [1.2 配置管理](#12-配置管理)
  - [1.3 钱包管理](#13-钱包管理)
  - [1.4 余额与授权](#14-余额与授权)
  - [1.5 交易/兑换](#15-交易兑换)
  - [1.6 AMM/CLMM 流动性](#16-ammclmm-流动性)
  - [1.7 Token 管理](#17-token-管理)
  - [1.8 池子管理](#18-池子管理)
  - [1.9 链上工具](#19-链上工具)
  - [1.10 错误码](#110-错误码)
- [2. MQTT RPC 接口](#2-mqtt-rpc-接口)
- [3. MQTT Pub/Sub 主题](#3-mqtt-pubsub-主题)
- [4. OMS Connector API](#4-oms-connector-api)
  - [4.1 REST 端点](#41-rest-端点)
  - [4.2 WebSocket 端点](#42-websocket-端点)
  - [4.3 WebSocket 事件](#43-websocket-事件)

---

## 1. Gateway HTTP API

网关 HTTP 客户端，用于与 Gateway 服务通信。

- **源文件**: `hummingbot/core/gateway/gateway_http_client.py`
- **基址**: `{http|https}://{gateway_api_host}:{gateway_api_port}`
- **SSL**: 可选（由 `gateway_use_ssl` 配置控制）
- **超时**: 30 秒

### 1.1 基础/状态

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| GET | `/` | Ping 网关（健康检查） | 无 | `{"status": "ok"}` |
| GET | `/chains/{chain}/status` | 获取链/网络状态 | `network: str` (query) | 网络状态信息 |
| POST | `/restart` | 重启网关服务 | 无 | 重启结果 |

### 1.2 配置管理

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| GET | `/config` | 获取网关配置 | `namespace: str` (query, 可选) | 配置字典 |
| GET | `/config/connectors` | 获取连接器列表 | 无 | `{"connectors": [{name, chain, trading_types}]}` |
| GET | `/config/chains` | 获取链列表 | 无 | `{"chains": [{chain, networks}]}` |
| GET | `/config/namespaces` | 获取命名空间列表 | 无 | `{"namespaces": [str]}` |
| POST | `/config/update` | 更新配置项 | `namespace: str, path: str, value: Any` (body) | 更新结果 |

### 1.3 钱包管理

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| GET | `/wallet` | 获取钱包列表 | `showHardware: str` (query) | 钱包列表 |
| POST | `/wallet/add` | 添加钱包 | `chain: str, privateKey: str, setDefault: bool` (body) | 添加结果 |
| POST | `/wallet/add-hardware` | 添加硬件钱包 | `chain: str, address: str, setDefault: bool` (body) | 添加结果 |
| DELETE | `/wallet/remove` | 移除钱包 | `chain: str, address: str` (body) | 移除结果 |
| POST | `/wallet/setDefault` | 设置默认钱包 | `chain: str, address: str` (body) | 设置结果 |

### 1.4 余额与授权

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| POST | `/chains/{chain}/balances` | 获取代币余额 | `network: str, address: str, tokens: List[str]` (body) | `{"balances": {symbol: amount}}` |
| POST | `/chains/ethereum/allowances` | 获取授权额度 | `network: str, address: str, tokens: List[str], spender: str` (body) | 授权信息 |
| POST | `/chains/ethereum/approve` | 授权代币 | `network: str, address: str, token: str, spender: str, amount?: int` (body) | 授权结果 |
| POST | `/chains/{chain}/poll` | 查询交易状态 | `network: str, signature: str` (body) | 交易状态 |

### 1.5 交易/兑换

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| GET | `/connectors/{dex}/{trading_type}/quote-swap` | 获取兑换报价 | `network, baseToken, quoteToken, amount, side, slippagePct?, poolAddress?` (query) | 报价信息 (price, amountIn, amountOut) |
| POST | `/connectors/{dex}/{trading_type}/execute-swap` | 执行兑换 | `network, baseToken, quoteToken, amount, side, slippagePct?, poolAddress?, walletAddress?` (body) | 交易详情 |
| POST | `/connectors/{dex}/{trading_type}/execute-quote` | 按报价ID执行 | `quoteId, network?, walletAddress?` (body) | 交易详情 |

### 1.6 AMM/CLMM 流动性

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| GET | `/connectors/{dex}/{trading_type}/pool-info` | 获取池子信息 | `network, poolAddress` (query) | 池子详情 |
| GET | `/connectors/{dex}/clmm/position-info` | 获取 CLMM 头寸信息 | `network, positionAddress, walletAddress` (query) | 头寸详情 |
| GET | `/connectors/{dex}/amm/position-info` | 获取 AMM 头寸信息 | `network, walletAddress, poolAddress` (query) | 头寸详情 |
| GET | `/connectors/{dex}/clmm/quote-position` | CLMM 开仓报价 | `network, poolAddress, lowerPrice, upperPrice, baseTokenAmount?, quoteTokenAmount?, slippagePct?` (query) | 报价信息 |
| GET | `/connectors/{dex}/amm/quote-liquidity` | AMM 添加流动性报价 | `network, poolAddress, baseTokenAmount, quoteTokenAmount, slippagePct?` (query) | 报价信息 |
| POST | `/connectors/{dex}/clmm/open-position` | CLMM 开仓 | `network, walletAddress, poolAddress, lowerPrice, upperPrice, baseTokenAmount?, quoteTokenAmount?, slippagePct?` (body) | 开仓结果 |
| POST | `/connectors/{dex}/clmm/close-position` | CLMM 平仓 | `network, walletAddress, positionAddress` (body) | 平仓结果 |
| POST | `/connectors/{dex}/clmm/add-liquidity` | CLMM 添加流动性 | `network, walletAddress, positionAddress, baseTokenAmount?, quoteTokenAmount?, slippagePct?` (body) | 添加结果 |
| POST | `/connectors/{dex}/clmm/remove-liquidity` | CLMM 移除流动性 | `network, walletAddress, positionAddress, percentageToRemove` (body) | 移除结果 |
| POST | `/connectors/{dex}/clmm/collect-fees` | CLMM 收取手续费 | `network, walletAddress, positionAddress` (body) | 收取结果 |
| GET | `/connectors/{dex}/clmm/positions-owned` | 查询拥有的 CLMM 头寸 | `network, walletAddress` (query) | 头寸列表 |
| POST | `/connectors/{dex}/amm/add-liquidity` | AMM 添加流动性 | `network, walletAddress, poolAddress, baseTokenAmount, quoteTokenAmount, slippagePct?` (body) | 添加结果 |
| POST | `/connectors/{dex}/amm/remove-liquidity` | AMM 移除流动性 | `network, walletAddress, poolAddress, percentageToRemove` (body) | 移除结果 |

### 1.7 Token 管理

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| GET | `/tokens` | 获取代币列表 | `chain, network, search?` (query) | `{"tokens": [{symbol, address, decimals, name}]}` |
| GET | `/tokens/{symbol_or_address}` | 获取代币详情 | `chain, network` (query) | 代币详情 |
| POST | `/tokens` | 添加代币 | `chain, network, token: Dict` (body) | 添加结果 |
| DELETE | `/tokens/{address}` | 移除代币 | `chain, network` (body) | 移除结果 |

### 1.8 池子管理

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| GET | `/pools/{trading_pair}` | 获取池子信息 | `connector, network, type` (query) | 池子信息（含地址） |
| POST | `/pools` | 添加池子 | `connector, network, address, type, baseTokenAddress, quoteTokenAddress, ...` (body) | 添加结果 |
| DELETE | `/pools/{address}` | 移除池子 | `connector, network, type` (body) | 移除结果 |

### 1.9 链上工具

| 方法 | 路径 | 用途 | 请求参数 | 响应格式 |
|------|------|------|---------|---------|
| GET | `/chains/{chain}/estimate-gas` | 估算 Gas 费用 | `network` (query) | `{feePerComputeUnit, computeUnits, fee, feeAsset, denomination, gasType?, maxFeePerGas?, maxPriorityFeePerGas?}` |

### 1.10 错误码

| 错误码 | 枚举名 | 说明 |
|--------|--------|------|
| 1001 | Network | 网络错误 |
| 1002 | RateLimit | 请求频率限制 |
| 1003 | OutOfGas | Gas 不足 |
| 1004 | TransactionGasPriceTooLow | Gas 价格太低 |
| 1005 | LoadWallet | 钱包加载失败 |
| 1006 | TokenNotSupported | 不支持的代币 |
| 1007 | TradeFailed | 交易失败 |
| 1008 | SwapPriceExceedsLimitPrice | 兑换价格超过限价 |
| 1009 | SwapPriceLowerThanLimitPrice | 兑换价格低于限价 |
| 1010 | ServiceUnitialized | 服务未初始化 |
| 1011 | UnknownChainError | 未知链错误 |
| 1012 | InvalidNonceError | 无效 Nonce |
| 1013 | PriceFailed | 价格查询失败 |
| 1022 | InsufficientBaseBalance | 基础代币余额不足 |
| 1023 | InsufficientQuoteBalance | 报价代币余额不足 |
| 1024 | SimulationError | 交易模拟失败 |
| 1025 | SwapRouteFetchError | 兑换路由获取失败 |
| 1099 | UnknownError | 未知错误 |

---

## 2. MQTT RPC 接口

MQTT 远程控制接口，基于 RPC 模式实现请求-响应通信。

- **源文件**: `hummingbot/remote_iface/mqtt.py`, `hummingbot/remote_iface/messages.py`
- **主题前缀**: `{namespace}/{instance_id}`
- **传输协议**: MQTT（通过 commlib 库）

### 2.1 启动策略

| 字段 | 值 |
|------|---|
| **RPC 路径** | `{prefix}/start` |
| **用途** | 启动交易策略 |
| **请求参数** | `log_level?: str, script?: str, conf?: str, is_quickstart?: bool, async_backend?: bool` |
| **响应格式** | `status: int (200/400), msg: str` |

### 2.2 停止策略

| 字段 | 值 |
|------|---|
| **RPC 路径** | `{prefix}/stop` |
| **用途** | 停止当前运行的策略 |
| **请求参数** | `skip_order_cancellation?: bool, async_backend?: bool` |
| **响应格式** | `status: int (200/400), msg: str` |

### 2.3 配置管理

| 字段 | 值 |
|------|---|
| **RPC 路径** | `{prefix}/config` |
| **用途** | 读取/修改策略配置 |
| **请求参数** | `params: List[Tuple[str, Any]]` — 键值对列表 |
| **响应格式** | `status: int, msg: str, changes: List[Tuple[str, Any]], config: {client: Dict, strategy: Dict}` |

### 2.4 导入配置

| 字段 | 值 |
|------|---|
| **RPC 路径** | `{prefix}/import` |
| **用途** | 导入策略配置文件 |
| **请求参数** | `strategy: str` — 策略名称 |
| **响应格式** | `status: int (200/400), msg: str` |

### 2.5 查询状态

| 字段 | 值 |
|------|---|
| **RPC 路径** | `{prefix}/status` |
| **用途** | 查询当前策略运行状态 |
| **请求参数** | `async_backend?: bool` |
| **响应格式** | `status: int (200/400), msg: str, data: Any` |

### 2.6 查询历史

| 字段 | 值 |
|------|---|
| **RPC 路径** | `{prefix}/history` |
| **用途** | 查询交易历史 |
| **请求参数** | `days?: float, verbose?: bool, precision?: int, async_backend?: bool` |
| **响应格式** | `status: int (200/400), msg: str, trades: List[Any]` |

### 2.7 设置余额限制

| 字段 | 值 |
|------|---|
| **RPC 路径** | `{prefix}/balance/limit` |
| **用途** | 设置限额交易模式的余额 |
| **请求参数** | `exchange: str, asset: str, amount: float` |
| **响应格式** | `status: int (200/400), msg: str, data: str` |

### 2.8 设置模拟余额

| 字段 | 值 |
|------|---|
| **RPC 路径** | `{prefix}/balance/paper` |
| **用途** | 设置模拟交易模式的余额 |
| **请求参数** | `asset: str, amount: float` |
| **响应格式** | `status: int (200/400), msg: str, data: str` |

---

## 3. MQTT Pub/Sub 主题

MQTT 发布/订阅接口，用于接收 Hummingbot 的实时事件推送。

- **源文件**: `hummingbot/remote_iface/mqtt.py`, `hummingbot/remote_iface/messages.py`

### 3.1 日志主题

| 字段 | 值 |
|------|---|
| **主题** | `{prefix}/log` |
| **方向** | 发布（Hummingbot -> 外部） |
| **消息类型** | `LogMessage` |
| **消息格式** | `timestamp: float, msg: str, level_no: int, level_name: str, logger_name: str` |

### 3.2 内部事件主题

| 字段 | 值 |
|------|---|
| **主题** | `{prefix}/events` |
| **方向** | 发布（Hummingbot -> 外部） |
| **消息类型** | `InternalEventMessage` |
| **消息格式** | `timestamp: int, type: str, data: Dict` |
| **事件类型** | `BuyOrderCreated`, `BuyOrderCompleted`, `SellOrderCreated`, `SellOrderCompleted`, `OrderFilled`, `OrderCancelled`, `OrderExpired`, `OrderFailure`, `FundingPaymentCompleted`, `RangePositionLiquidityAdded`, `RangePositionLiquidityRemoved`, `RangePositionUpdateFailure` |

### 3.3 通知主题

| 字段 | 值 |
|------|---|
| **主题** | `{prefix}/notify` |
| **方向** | 发布（Hummingbot -> 外部） |
| **消息类型** | `NotifyMessage` |
| **消息格式** | `seq: int, timestamp: int, msg: str` |

### 3.4 状态更新主题

| 字段 | 值 |
|------|---|
| **主题** | `{prefix}/status_updates` |
| **方向** | 发布（Hummingbot -> 外部） |
| **消息类型** | `StatusUpdateMessage` |
| **消息格式** | `timestamp: int, type: str, msg: str` |

### 3.5 心跳主题

| 字段 | 值 |
|------|---|
| **主题** | `{prefix}/hb` |
| **方向** | 发布（Hummingbot -> 外部） |
| **消息类型** | 内置心跳 |
| **说明** | 由 commlib Node 自动发送 |

### 3.6 外部事件主题

| 字段 | 值 |
|------|---|
| **主题** | `{prefix}/external/event/*` |
| **方向** | 订阅（外部 -> Hummingbot） |
| **消息类型** | `ExternalEventMessage` |
| **消息格式** | `timestamp: int, sequence: int, type: str, data: Dict` |
| **说明** | 支持通配符 `*`，外部系统可向此主题发送自定义事件触发 Hummingbot 内部回调 |

---

## 4. OMS Connector API

订单管理系统（OMS）连接器，用于与外部 OMS 服务通信。

- **源文件**: `hummingbot/connector/utilities/oms_connector/`
- **通信方式**: REST + WebSocket
- **消息格式**: JSON-RPC 风格（含 `m`, `i`, `n`, `o` 字段）

### 4.1 REST 端点

| 端点名 | 用途 | 说明 |
|--------|------|------|
| `Authenticate` | 认证 | 获取会话令牌 |
| `GetInstruments` | 获取交易对 | 获取可用交易品种信息 |
| `GetLevel1` | 获取 L1 行情 | 最优买卖价 |
| `GetL2Snapshot` | 获取 L2 深度 | 深度订单簿快照（最大 400 档） |
| `Ping` | 心跳检测 | 保持连接活跃 |
| `SendOrder` | 下单 | 提交限价单 |
| `GetOrderStatus` | 查询订单状态 | 按订单 ID 查询 |
| `CancelOrder` | 撤单 | 取消指定订单 |
| `GetAccountPositions` | 查询账户持仓 | 获取当前持仓 |
| `GetTradesHistory` | 查询成交历史 | 获取历史成交记录 |

**频率限制**: REST 5000 请求/分钟

### 4.2 WebSocket 端点

| 端点名 | 用途 | 方向 |
|--------|------|------|
| `AuthenticateUser` | WS 认证 | 客户端 -> 服务端 |
| `SubscribeAccountEvents` | 订阅账户事件 | 客户端 -> 服务端 |
| `SubscribeTrades` | 订阅成交 | 客户端 -> 服务端 |
| `SubscribeLevel2` | 订阅 L2 深度 | 客户端 -> 服务端 |
| `UnsubscribeLevel2` | 取消订阅 L2 | 客户端 -> 服务端 |
| `Ping` | 心跳 | 客户端 -> 服务端 |

**频率限制**: WS 500000 请求/分钟

### 4.3 WebSocket 事件

| 事件名 | 说明 |
|--------|------|
| `Level2UpdateEvent` | L2 深度增量更新 |
| `AccountPositionEvent` | 账户持仓变更 |
| `OrderStateEvent` | 订单状态变更 |
| `OrderTradeEvent` | 订单成交事件 |
| `CancelOrderRejectEvent` | 撤单被拒绝 |

---

## 附录：关键源文件索引

| 文件路径 | 说明 |
|---------|------|
| `hummingbot/core/gateway/gateway_http_client.py` | Gateway HTTP 客户端，所有 Gateway API 的调用入口 |
| `hummingbot/remote_iface/mqtt.py` | MQTT 网关实现，含 RPC 命令、事件转发、通知、日志、外部事件 |
| `hummingbot/remote_iface/messages.py` | MQTT 消息类型定义（Request/Response/PubSub） |
| `hummingbot/connector/utilities/oms_connector/oms_connector_constants.py` | OMS 连接器常量（端点、频率限制、字段映射） |
| `hummingbot/connector/utilities/oms_connector/oms_connector_web_utils.py` | OMS WebSocket 消息预/后处理器 |
| `hummingbot/logger/log_server_client.py` | 远程日志上报客户端（Coinalpha API） |
