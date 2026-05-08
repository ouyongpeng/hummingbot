# Hummingbot 性能瓶颈与价差监控能力

> 最后更新：2026-04-22

---

## 一、价差监控能力

### 现有机制

所有价差计算依赖 **OrderBook** 数据：

```
Connector.get_quote_price() → ExchangeBase.get_vwap_for_volume() → OrderBook.c_get_price_for_volume()
```

- ArbitrageExecutor：`(sell_price - buy_price) / buy_price - tx_cost`
- XEMMExecutor：`taker_price / (1 ± target_profitability ± tx_cost_pct)`
- V1 XEMM：`taker_VWAP / (1 + min_profitability)` → Maker 挂单价格

### 能否同时监控多个交易所的所有交易对？

| 维度 | 支持情况 | 限制 |
|------|---------|------|
| 多交易所并行 | ✅ 每交易所独立连接器 | 无代码限制 |
| 单连接器多交易对 | ✅ 一个 WS 订阅多个频道 | 受交易所 WS 限制 |
| 全交易对扫描 | ❌ 无内置 screener | 必须手动指定 trading_pairs |

**交易所 WS 限制（硬约束）**：

| 交易所 | WS 连接数 | 每连接订阅数 |
|--------|----------|------------|
| Binance | 5/IP | 200 频道（约 100 交易对） |
| KuCoin | 5/IP | 100 topic |
| OKX | 5/IP | 无明确限制 |
| Bybit | 5/IP | 10 topic（最严格） |

### 实际可监控交易对数

| 场景 | 可行数量 |
|------|---------|
| 2 所 × 1 对（默认 XEMM） | 1 |
| 2 所 × 5 对 | 5 |
| 3 所 × 20 对 | 接近极限 |
| 5 所 × 50 对 | **不可行** |

### 全交易对扫描建议

```
方案 A：轻量级扫描器（推荐）
├── REST API 批量获取所有交易对 ticker
│   （Binance: GET /api/v3/ticker/bookTicker，1 次请求返回 2000+ 对）
├── 粗筛价差 > 阈值的交易对
├── 仅对筛选结果启动 Hummingbot 策略
└── 周期性重新扫描（如每 5 分钟）

方案 B：多实例部署
├── 每实例监控 5-10 个交易对
├── MQTT 消息队列共享价差信息
└── 水平扩展
```

---

## 二、性能瓶颈

### 瓶颈全景

```
交易所 API ← 瓶颈1: API 限流
    │
WS 消息解析 ← 瓶颈2: 单线程处理
    │
OrderBook ← 非瓶颈（Cython 微秒级）
    │
策略 tick ← 瓶颈3: 串行执行
    │
SQLite 写入 ← 瓶颈4: 单文件锁
```

### 量化分析（N 交易对 × M 交易所）

| 指标 | 公式 | 5×2 | 10×3 | 50×5 |
|------|------|-----|------|------|
| WS 连接数 | 2M | 4 | 6 | 10 |
| asyncio.Task 数 | ~17M+NM | 27 | 81 | 335 |
| OrderBook 内存 | NM×240KB | 2.4MB | 7.2MB | 60MB |
| WS 消息/秒 | NM×10 | 100 | 300 | 2500 |
| Diff CPU/秒 | NM×50μs | 0.5ms | 1.5ms | 12.5ms |
| 初始化时间 | N 秒/交易所 | 5s | 10s | 50s |

### 5 个关键瓶颈

**1. WS 消息串行处理**

所有交易对的 diff 消息在同一条协程中串行解析和路由。50 对时约 5000 msg/s，每条 10-50μs，总耗时 50-750ms/s，开始挤压 tick 时间。

**2. OrderBook 初始化串行化**

逐个获取 REST 快照，每对间隔 1 秒。50 对需 50 秒。Binance 限流 6000 权重/分钟，50 对需 5000 权重 → 接近限额。

**3. API 限流器锁竞争**

全局 asyncio.Lock，高并发请求时串行化。`within_capacity()` O(T×L) 复杂度。

**4. 策略 tick 串行**

Clock 串行调用所有 child_iterator.tick()。150 个策略实例 × 0.5-2ms = 75-300ms，占用 7.5%-30% 的 1 秒窗口。

**5. SQLite 同步写入**

`session.commit()` 在主事件循环中同步执行，高频交易时可能阻塞数百毫秒。

### OrderBook 性能（非瓶颈）

| 操作 | 复杂度 | 耗时 |
|------|--------|------|
| apply_diffs() | O(D×log(B)) | 1-10μs |
| apply_snapshot() | O(S×log(S)) | 10-100μs |
| get_price() | O(1) 缓存 | <1μs |
| get_vwap_for_volume() | O(K) | 5-50μs |