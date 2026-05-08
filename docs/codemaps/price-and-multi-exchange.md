# Hummingbot 价格获取与多交易所并行机制

> 最后更新：2026-04-22

---

## 一、价格获取三层架构

| 层次 | 组件 | 职责 |
|------|------|------|
| 1. 连接器层 | ExchangePyBase + OrderBookTrackerDataSource | WebSocket/REST 连接交易所，获取原始 OrderBook 数据 |
| 2. OrderBook 层 | OrderBook (Cython + C++ std::set) | 本地维护买卖盘，提供 VWAP/最优价查询 |
| 3. RateOracle 层 | RateOracle (全局单例) | 跨交易所汇率聚合，支持链式匹配 |

### 数据流

```
交易所 API
    ├── REST API ──> 初始订单簿快照（全量）
    └── WebSocket ──> 实时增量更新（100ms 级别）
            │
            ▼
    OrderBookTracker（每交易对独立 Task）
            │
            ▼
    OrderBook.apply_snapshot() / apply_diffs() / apply_trade()
            │
            ▼
    OrderBook 查询 API（Cython，微秒级）
    ├── get_price(is_buy)          → O(1) 缓存最优价
    ├── get_vwap_for_volume()      → O(K) 加权平均价
    └── get_price_for_volume()     → O(K) 完全成交价
```

### XEMM 策略中的价格使用

```
Maker 挂单价格 = Taker VWAP / (1 + min_profitability)
```

1. 从 Taker 连接器 OrderBook 取 `get_vwap_for_volume()`（考虑对冲量滑点）
2. 通过 RateOracle 做跨币种汇率转换
3. 加上最小利润率，算出 Maker 端挂单价
4. `adjust_order_enabled=True` 时微调到盘口最优价附近

---

## 二、多交易所并行机制

### 核心设计：连接器实例隔离

每个交易所一个独立连接器实例，状态完全隔离：

| 组件 | 每实例数量 | 说明 |
|------|----------|------|
| WS 连接 | 2 个（公共数据 + 用户流） | 独立维护 |
| Auth | 1 个 | 独立 API Key 签名 |
| ClientOrderTracker | 1 个 | 仅管本交易所订单 |
| OrderBookTracker | 1 个 | 仅管本交易所 OrderBook |
| _account_balances | 1 个字典 | 仅管本交易所余额 |

### 初始化流程

```python
TradingCore.initialize_markets([
    ("binance", ["BTC-USDT", "ETH-USDT"]),
    ("kucoin",  ["BTC-USDT"]),
])
```

1. `Security.api_keys(name)` → 解密各交易所 API Key
2. `ConnectorManager.create_connector()` → 创建独立实例
3. `Clock.add_iterator(connector)` → 加入调度循环

### 余额跟踪（双通道）

| 通道 | 模式 | 时效 |
|------|------|------|
| WS 用户流 | `real_time=True`（默认） | 即时 |
| REST 轮询 | `real_time=False`（备用） | 1-15 秒 |

可用余额 = 账户余额 - 在途订单锁定 - 余额限制配置

### 持久化

`MarketsRecorder` 监听所有连接器事件，写入 SQLite 时用 `market` 字段区分交易所。

---

## 三、RateOracle 汇率转换

### 查找优先级

1. 本地缓存（RateSource 获取的价格）
2. 已注册连接器 OrderBook 的中间价
3. 反向查找（BTC-USDT → USDT-BTC 取倒数）
4. 链式匹配（BTC-AAVE → BTC-USDT × USDT-AAVE）

### 价格刷新

每秒从配置的 RateSource（如 Binance `/ticker/bookTicker`）刷新汇率，30 秒缓存。

---

## 四、关键代码路径

| 类 | 文件 | 关键方法 |
|----|------|---------|
| ExchangePyBase | `connector/exchange_py_base.py` | `start_network()`, `_create_order()` |
| OrderBookTracker | `core/data_type/order_book_tracker.py` | `start()`, `_track_single_book()` |
| OrderBook | `core/data_type/order_book.pyx` | `apply_snapshot()`, `apply_diffs()`, `c_get_vwap_for_volume()` |
| RateOracle | `core/rate_oracle/rate_oracle.py` | `get_pair_rate()`, `_fetch_price_loop()` |
| ConnectorManager | `core/connector_manager.py` | `create_connector()` |
| TradingCore | `core/trading_core.py` | `initialize_markets()` |
| Security | `client/config/security.py` | `api_keys()`, `decrypt_all()` |