# Hummingbot 下单通道机制

> 最后更新：2026-04-22

---

## 核心结论

**所有 CEX 连接器通过 REST API 下单，WebSocket 仅用于接收数据。**

### 下单调用链

```
策略 buy()/sell()
  → ExchangePyBase._create_order()     # 验证金额/价格
    → _place_order()                   # 抽象方法，各连接器实现
      → _api_post(path_url, data)      # REST POST 请求
        → RESTAssistant.execute_request()
          → aiohttp.ClientSession.post()
```

### 各连接器下单通道

| 类别 | 连接器 | 下单通道 | 说明 |
|------|--------|---------|------|
| CEX 现货 | Binance/KuCoin/OKX/Bybit/Gate.io/Bitget/MEXC/Kraken/Coinbase 等 | REST POST | `_api_post(ORDER_PATH_URL)` |
| CEX 永续 | Binance Perp/OKX Perp/Bybit Perp/KuCoin Perp/Bitget Perp 等 | REST POST | `_api_post(ORDER_URL)` |
| DEX（API） | Hyperliquid/Derive/Aevo/Vertex/GRVT/Cube/Architect | REST POST | 发送到 DEX API 端点 |
| DEX（链原生） | Injective/dYdX v4/XRPL/Dexalot/Evedex/Decibel | 链上交易广播 | 构造签名交易 → 广播到节点 |
| Gateway | Uniswap/Jupiter/PancakeSwap 等 | REST → Gateway → 链上 | HTTP POST → Gateway 构造链上交易 |

### WebSocket 角色（仅接收，不下单）

| 用途 | 方向 | 内容 |
|------|------|------|
| 公共数据流 | 交易所 → Hummingbot | OrderBook diff/snapshot、最新成交 |
| 用户流 | 交易所 → Hummingbot | 订单状态更新、余额变化 |

### 对套利策略的影响

REST 下单延迟 50-200ms（含网络往返 + 交易所处理），跨所套利双边 REST 总延迟 100-400ms。

| 场景 | 延迟 |
|------|------|
| 单交易所内 | 50-100ms |
| 跨所双边 REST | 100-400ms |
| WS 推送信号 → REST 下单 | 信号 ~1ms + 下单 50-200ms |

**优化方向**：部分交易所支持 `cancelReplace`（改单 API），将撤单+重挂合并为一次请求，延迟降至 50-100ms。当前 Hummingbot 未实现。