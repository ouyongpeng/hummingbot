# Hummingbot 策略全景

> 最后更新：2026-04-22

---

## V1 策略（9 个）

### 做市类

| # | 策略 | 核心行为 | 关键参数 | 风险 |
|---|------|---------|---------|------|
| 1 | **Pure Market Making** | 围绕 mid_price 挂买卖限价单，定时刷新。支持库存偏斜、悬挂订单、多层价差、价格区间 | bid_spread, ask_spread, order_refresh_time, inventory_skew | 单边库存风险 |
| 2 | **Avellaneda Market Making** | Avellaneda-Stoikov 理论算法：动态计算 reservation_price 和 optimal_spread，库存偏离时自动偏移报价 | gamma, eta, inventory_target_base_pct, volatility_buffer_size | 指标估计不准时报价不合理 |
| 3 | **Perpetual Market Making** | 永续合约做市 + 仓位管理：开仓挂限价单，持仓后自动止盈/止损 | leverage, long_profit_taking_spread, stop_loss_spread | 爆仓风险、止损可能不成交 |

### 跨所类

| # | 策略 | 核心行为 | 关键参数 | 风险 |
|---|------|---------|---------|------|
| 4 | **Cross Exchange MM (XEMM)** | Maker 端挂限价单，成交后 Taker 端市价对冲。基于 Taker VWAP + min_profitability 计算挂单价格 | min_profitability, slippage_buffer, adjust_order_enabled | 对冲失败 → 单腿暴露 |
| 5 | **Cross Exchange Mining** | XEMM 变体：3σ 波动率动态调整利润率 + 历史绩效自适应 + 自动余额再平衡 | volatility_buffer_size, min_prof_tol_high/low, rate_curve | 历史数据缺失时无法优化 |
| 6 | **AMM Arbitrage** | CEX 与 AMM/DEX 间价差套利，支持并发/顺序提交，考虑 gas 费 | min_profitability, concurrent_orders_submission, gas_token | AMM 确认延迟 + 滑点 |
| 7 | **Spot-Perpetual Arbitrage** | 四阶段状态机：价差 > 开仓阈值 → 双边下单 → 价差收敛 → 平仓 | min_opening_arbitrage_pct, min_closing_arbitrage_pct, perp_leverage | funding rate 侵蚀 + 单腿风险 |

### 其他

| # | 策略 | 核心行为 | 关键参数 | 风险 |
|---|------|---------|---------|------|
| 8 | **Liquidity Mining** | 多交易对同时做市，波动率驱动动态 spread，与 Parrot 奖励平台集成 | spread, volatility_to_spread_multiplier, max_spread | 多对分散资金 |
| 9 | **Hedge** | 聚合所有市场持仓，在单一对冲市场反向交易。支持按数量/按价值对冲 | hedge_ratio, hedge_leverage, value_mode | 对冲延迟 + 部分对冲仍有敞口 |

---

## V2 执行器（8 个）

| # | 执行器 | 核心行为 | 风控 | 适用场景 |
|---|--------|---------|------|---------|
| 1 | **PositionExecutor** | 开仓 → 三重屏障监控 → 止损/止盈/时间限制/追踪止损 → 平仓 | ✅ 三重屏障 | 做市/方向性交易核心单元 |
| 2 | **OrderExecutor** | 简单下单（市价/限价/限价Maker/追踪限价），支持 LIMIT_CHASER 跟价 | ❌ 无 | 辅助操作、仓位再平衡 |
| 3 | **DCAExecutor** | 分批建仓：按 prices+amounts 逐级下单，全部成交后激活风控 | ✅ 三重屏障 | 逢低买入/逢高卖出 |
| 4 | **GridExecutor** | start_price→end_price 线性网格，每层独立开仓+止盈循环 | ✅ 整体止损+limit_price | 震荡行情网格交易 |
| 5 | **XEMMExecutor** | Maker 限价单 + Taker 市价对冲，盈利性偏离 [min,max] 时撤单重挂 | ❌ 无 | 跨所做市 |
| 6 | **ArbitrageExecutor** | 双边同时市价单套利，失败重试 3 次 | ❌ 无 | 跨所价差套利 |
| 7 | **TWAPExecutor** | total_duration 内每隔 order_interval 下单，金额均分 | ❌ 无 | 大额拆分执行、定投 |
| 8 | **LPExecutor** | DEX 集中流动性仓位管理，价格出范围自动平仓 | ❌ 无止损 | Solana DEX LP 挖矿 |

### PositionExecutor 三重屏障详解

```
开仓成交 → 屏障检查（每 tick）
  ├── 止损触发 → 市价平仓（net_pnl_pct <= -stop_loss）
  ├── 追踪止损触发 → 市价平仓（PnL 达到 activation_price 后回撤 > trailing_delta）
  ├── 止盈触发 → 限价/市价平仓（net_pnl_pct >= take_profit）
  ├── 时间限制触发 → 市价平仓（超时）
  └── 部分成交 → amount_to_close 精确匹配平仓量
```

### OrderExecutor LIMIT_CHASER 模式

```python
# 每秒检查市场价格偏移
if current_price - order.price > current_price * refresh_threshold:
    cancel_order()       # 撤单
    place_open_order()   # 在新价格重挂
```

这是 V2 中最直接的"挂单跟价"实现。

---

## V2 控制器（用户级）

### 做市类

| # | 控制器 | 核心行为 | 关键参数 |
|---|--------|---------|---------|
| 1 | **PMM Simple** | 固定价差做市：mid_price ± buy/sell_spreads，每档 PositionExecutor | buy_spreads, sell_spreads, executor_refresh_time |
| 2 | **PMM Dynamic** | NATR 动态价差 + MACD 偏移参考价。buy_spreads 以 NATR 为单位，N=2 即近似 mean-2σ | natr_length, macd_fast/slow/signal, buy_spreads=[1,2,4] |

### 跨所类

| # | 控制器 | 核心行为 | 关键参数 |
|---|--------|---------|---------|
| 3 | **XEMM Multiple Levels** | 多档位 XEMM：每档独立 XEMMExecutor，max_executors_imbalance 控制买卖不平衡 | buy_levels, sell_levels, min/max_profitability |

### 方向性类

| # | 控制器 | 核心行为 | 关键参数 |
|---|--------|---------|---------|
| 4 | **Stat Arb** | Z-Score + 线性回归协整：sklearn LinearRegression 计算 hedge_ratio，Z-Score > 阈值触发双边 PositionExecutor | entry_threshold, lookback_period, pos_hedge_ratio |
| 5 | **Bollinger V1/V2** | Bollinger Band %B 生成信号：价格触及上轨 → 做空，下轨 → 做多 | bb_length, bb_std |
| 6 | **DMAN V3** | 多指标组合：EMA 趋势 + RSI + MACD + Bollinger 综合评分 | ema_length, rsi_length, macd_params |
| 7 | **Supertrend V1** | Supertrend 指标信号：趋势翻转时触发 | supertrend_length, supertrend_multiplier |

### 执行类

| # | 控制器 | 核心行为 | 关键参数 |
|---|--------|---------|---------|
| 8 | **DCA** | 逢低分批买入：价格下跌时逐级加仓 | amounts_quote, prices, stop_loss |

---

## V1 vs V2 关键差异

| 维度 | V1 | V2 |
|------|----|----|
| 架构 | 单体策略类 | Controller-Executor 分离 |
| 风控 | 策略内自行实现 | 三重屏障统一框架 |
| 多档位 | order_levels 参数 | 每个 level 独立 Executor |
| 统计指标 | 仅 Avellaneda 用波动率 | NATR/MACD/Z-Score/回归 |
| 回测 | 无 | BacktestingEngine + ExecutorSimulator |
| 热更新 | 部分参数 | is_updatable 参数全部支持 |
| 仓位聚合 | 无 | ExecutorOrchestrator PositionHold |

---

## 策略行为分类树

```
做市类（赚取价差）
├── Pure MM          ─ 固定价差，单交易所
├── Avellaneda MM    ─ 算法价差，库存自适应
├── Perpetual MM     ─ 永续合约做市 + 止盈止损
├── PMM Simple       ─ V2 固定价差做市
├── PMM Dynamic      ─ V2 NATR+MACD 动态价差
└── Liquidity Mining ─ 多交易对做市 + 挖矿奖励

跨所类（利用价差）
├── XEMM             ─ Maker挂单 + Taker对冲
├── XEMM Multi-Level ─ V2 多档位跨所做市
├── AMM Arb          ─ CEX-DEX 价差套利
├── Spot-Perp Arb    ─ 现货-永续基差套利
├── ArbitrageExecutor─ V2 双边市价套利
└── Cross Ex Mining  ─ 波动率自适应跨所做市

方向性类（信号驱动）
├── Bollinger V1/V2  ─ 布林带信号
├── DMAN V3          ─ 多指标组合信号
├── Supertrend V1    ─ 超级趋势信号
└── Stat Arb         ─ Z-Score+协整信号

执行类（订单管理）
├── DCA              ─ 分批建仓
├── Grid             ─ 网格交易
├── TWAP             ─ 时间加权拆分
├── LP               ─ DEX 流动性提供
└── Hedge            ─ 多市场持仓对冲
```