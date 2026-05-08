# 跨所价差 Z-Score 套利策略设计文档

> 设计日期：2026-04-22
> 项目：Hummingbot 自定义价差策略

---

## 一、需求概述

在两个交易所之间监控同一交易对的价差，统计过去 5 分钟的价差均值与标准差，当价差偏离均值超过 2 个标准差时执行套利交易。

### 核心逻辑

```
价差 = (卖出端 VWAP - 买入端 VWAP) / 买入端 VWAP
Z-Score = (当前价差 - 均值) / 标准差

Z-Score > +2σ → 买入端便宜、卖出端贵 → 买低卖高
Z-Score < -2σ → 反向套利
Z-Score 回归均值 → 平仓
```

---

## 二、技术方案

### 2.1 架构选型

基于 Hummingbot V2 策略框架的 Controller-Executor 模式：

- **Controller**：自定义 `SpreadZScoreController`，负责价差采集、统计计算、信号触发
- **Executor**：使用内置 `ArbitrageExecutor`，负责即时双向执行
- **回测**：使用内置 `BacktestingEngine`，无需额外开发

### 2.2 核心组件

| 组件 | 类名 | 职责 |
|------|------|------|
| 配置 | `SpreadZScoreControllerConfig` | 参数定义（交易所、阈值、冷却期等） |
| 控制器 | `SpreadZScoreController` | 价差采集、Z-Score 计算、信号触发 |
| 执行器 | `ArbitrageExecutor`（内置） | 即时双向市价单执行 |

### 2.3 数据流

```
交易所 OrderBook (WS 实时更新)
    │
    ├── 买入端: get_price_for_volume(True, amount) → VWAP
    └── 卖出端: get_price_for_volume(False, amount) → VWAP
    │
    ▼
SpreadZScoreController.update_processed_data()
    │
    ├── 计算价差 → 追加到 deque 滚动窗口
    ├── 计算均值/标准差 → Z-Score
    ├── 判断 |Z-Score| > 阈值 → 发出 ExecutorAction
    │
    ▼
ArbitrageExecutor.execute()
    │
    ├── 买入端: 市价单
    └── 卖出端: 市价单
```

### 2.4 关键设计决策

| 决策点 | 选择 | 理由 |
|--------|------|------|
| 价差定义 | 对数价差 `ln(P_sell/P_buy)` | 更符合正态分布假设，统计更稳健 |
| 滚动窗口 | 固定时长 300s（基于时间戳过滤） | tick 间隔不恒定，时间过滤比样本数更准确 |
| 执行器 | ArbitrageExecutor | Z-Score 套利需要瞬时执行，非做市 |
| 冷却期 | 固定 60s + Z-Score 回归后解除 | 防止连续触发，同时不遗漏回归机会 |
| 止损 | Z-Score > 4σ 时止损 | 价差非均值回归时的安全网 |

### 2.5 风险控制

| 风险 | 缓解措施 |
|------|---------|
| 价差永久偏移（非均值回归） | Z-Score > 4σ 止损 + `min_profitability` 安全网 |
| 执行延迟（价差收敛） | 冷却期 60s + `min_profitability` 覆盖延迟成本 |
| 手续费侵蚀利润 | `min_profitability` 必须覆盖双边手续费 + 滑点 |
| 波动率剧变（开盘/收盘） | 可选：加 NATR 自适应调整阈值 |
| 交易所异常（充提币暂停） | 监控 OrderBook 连续性，异常时暂停 |

---

## 三、实现文件清单

| 文件 | 路径 | 说明 |
|------|------|------|
| 控制器 | `hummingbot/strategy_v2/controllers/spread_zscore_controller.py` | 核心逻辑 |
| 配置模板 | `conf/controllers/spread_zscore_1.yml` | YAML 配置示例 |
| 注册 | `hummingbot/strategy_v2/controllers/__init__.py` | 导入注册 |

---

## 四、配置参数

```yaml
# conf/controllers/spread_zscore_1.yml
controller_name: spread_zscore
controller_type: custom

# 交易所配置
buying_connector: binance
buying_trading_pair: BTC-USDT
selling_connector: kucoin
selling_trading_pair: BTC-USDT

# 统计参数
lookback_seconds: 300          # 5 分钟滚动窗口
zscore_threshold: 2.0           # 触发交易的 Z-Score 阈值
min_samples: 30                 # 最少样本数
stop_loss_zscore: 4.0           # 止损 Z-Score 阈值

# 交易参数
order_amount: 0.001              # 每次交易数量
min_profitability: 0.003        # 最低利润率（安全网）

# 冷却期
cooldown_seconds: 60            # 两次交易间最小间隔
```

---

## 五、回测计划

1. 使用 `BacktestingEngine` 对 3 个月历史数据回测
2. 评估指标：Sharpe Ratio、Max Drawdown、Win Rate、Profit Factor
3. 参数敏感性分析：`zscore_threshold`（1.5/2.0/2.5）、`lookback_seconds`（120/300/600）
4. 验证 `min_profitability` 是否覆盖双边手续费

---

## 六、参考代码

- V2 Controller 基类：`hummingbot/strategy_v2/controllers/controller_base.py`
- 现有统计套利示例：`controllers/generic/stat_arb.py`（Z-Score + 线性回归）
- PMM Dynamic 示例：`controllers/market_making/pmm_dynamic.py`（NATR spread_multiplier）
- ArbitrageExecutor：`hummingbot/strategy_v2/executors/arbitrage_executor/`
- 回测引擎：`hummingbot/strategy_v2/backtesting/backtesting_engine_base.py`