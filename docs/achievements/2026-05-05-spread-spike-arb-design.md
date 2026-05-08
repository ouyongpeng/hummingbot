# 跨交易所价差毛刺套利策略设计

> 日期：2026-05-05
> 状态：已确认（v2 — 审查修正版）

## 需求概述

在 deploy 一体化部署后的 Hummingbot 中，创建 CEX vs CEX 跨交易所价差毛刺套利策略，使用 V2 Controller 框架。

### 核心特征

| 维度 | 选择 |
|------|------|
| 策略类型 | Generic（继承 ControllerBase） |
| 交易所类型 | CEX vs CEX |
| 执行方式 | 双腿市价即时成交（修正：放弃市价+限价混合方案） |
| 信号检测 | 组合判断（固定阈值 AND Z-Score 统计阈值） |
| 资金规模 | 单笔 500-2000 USDT |

### 风控四件套

1. 单笔止损（事后语义：执行器完成后 net_pnl < -max_loss_per_trade_pct 时暂停后续开仓）
2. 日累计止损（滚动24小时窗口，非日历日重置）
3. 单腿风险控制（ArbitrageExecutor 内建双腿市价+失败重试；Controller 层超时清理未成交执行器）
4. 连续亏损暂停

### v1 → v2 修正说明

审查发现原方案存在根本性架构缺陷，修正如下：

| 问题 | 原方案 | 修正方案 |
|------|--------|---------|
| 执行器选择 | 两个独立 PositionExecutor | 复用 ArbitrageExecutor |
| 执行方式 | 腿A市价 + 腿B限价 | 双腿市价即时成交 |
| 风控基准 | 单腿 Triple Barrier | Controller 层面组合 PnL 风控 |
| 信号逻辑 | 固定阈值 OR Z-Score | 固定阈值 AND Z-Score |

**放弃市价+限价混合方案的原因**：套利利润来源于价差回归，限价腿在价差回归后才可能成交，此时利润空间已消失。市价+限价组合在价差毛刺场景下逻辑不自洽。ArbitrageExecutor 内建双腿原子执行、失败重试、盈利性判断，是更合适的执行器。

**套利场景下"止损"的语义**：价差毛刺套利是"检测→即成即结"模式，不是"开仓→持仓→止损"模式。ArbitrageExecutor 只在净利润为正时才执行双腿成交（`min_profitability` 过滤），这本身就是最根本的事前风控。`max_loss_per_trade_pct` 的含义是事后度量——执行器完成后如果实际亏损超过阈值，暂停后续开仓。不存在"持仓期间主动止损"的场景。

---

## 第一节：架构全景

### 模块划分

```
controllers/
└── generic/
    └── spread_spike_arb.py          # 策略核心代码

conf/
├── controllers/
│   └── spread_spike_arb.yml         # 控制器配置
└── scripts/
    └── v2_with_controllers.yml      # 入口脚本配置（复用现有 v2_with_controllers.py）
```

### 职责边界

| 组件 | 职责 | 不负责 |
|------|------|--------|
| `SpreadSpikeArbController` | 价差计算、毛刺检测、信号生成、风控判断、创建 ArbitrageExecutor | 订单生命周期管理、双腿执行协调 |
| `ArbitrageExecutor`（现有） | 双腿市价原子执行、盈利性判断、失败重试、成交确认 | 信号逻辑、风控 |
| `ExecutorOrchestrator`（现有） | 执行器创建/停止/持久化、仓位汇总 | 策略逻辑 |
| `V2WithControllers`（现有入口脚本） | Controller 生命周期、配置热更新、性能报告 | 无需修改 |

### 核心设计决策

1. **继承 `ControllerBase`（Generic）**——套利逻辑是双腿联动，Controller 负责信号检测和风控，ArbitrageExecutor 负责双腿执行
2. **复用 `ArbitrageExecutor`**——内建双腿原子执行（市价单同步下发）、失败自动重试（最多3次）、盈利性判断（扣除手续费后净利润）、双腿成交确认。无需自建双腿协调逻辑
3. **Controller 层面实现组合 PnL 风控**——ArbitrageExecutor 完成后，Controller 通过 `executors_info` 计算累计 PnL，在 `determine_executor_actions()` 中做风控判断
4. **信号检测用 AND 逻辑**——价差超过固定阈值 且 Z-Score 超过统计阈值，过滤低波动环境下的虚假信号

---

## 第二节：核心组件与接口

### Config 类：`SpreadSpikeArbConfig`

继承 `ControllerConfigBase`，`controller_type = "generic"`：

```python
class SpreadSpikeArbConfig(ControllerConfigBase):
    controller_name: str = "spread_spike_arb"
    # 交易所配置（使用 ConnectorPair，与框架惯例一致）
    connector_pair_a: ConnectorPair     # 交易所A（如 binance, BTC-USDT）
    connector_pair_b: ConnectorPair     # 交易所B（如 bybit, BTC-USDT）
    # 信号检测
    min_spread_bps: float = Field(default=50, ge=10, le=500)      # 最小价差阈值（基点）
    zscore_threshold: float = Field(default=2.0, ge=0.5, le=5.0)  # Z-Score 触发阈值
    lookback_periods: int = Field(default=100, ge=20, le=1000)    # 统计窗口长度
    interval: str = "1m"              # K 线时间粒度
    min_std_bps: float = Field(default=5, ge=1, le=50)            # 最小标准差（基点），低于此值不产生信号
    # 执行参数
    order_amount: Decimal = Field(ge=Decimal("1"), le=Decimal("10000"))  # 单笔金额（base asset 数量）
    # 注意：order_amount 是 base asset 数量（如 1 SOL），不是 quote 金额。
    # 用户应根据当前价格换算：如 SOL=$150，order_amount=1 对应 $150。
    min_profitability: Decimal = Decimal("0.003")  # ArbitrageExecutor 最小盈利性（扣除手续费后）
    delay_between_executors: int = Field(default=10, ge=1, le=300)  # 两次执行器创建最小间隔（秒）
    max_executors_imbalance: int = Field(default=1, ge=1, le=10)    # 买卖方向最大不平衡量
    # 风控参数
    max_loss_per_trade_pct: float = Field(default=0.003, ge=0.001, le=0.05)   # 单笔最大亏损 0.3%
    daily_max_loss_pct: float = Field(default=0.02, ge=0.005, le=0.1)         # 日累计最大亏损 2%
    max_consecutive_losses: int = Field(default=3, ge=1, le=10)                # 连续亏损暂停阈值
    cooldown_after_loss_seconds: float = Field(default=60, ge=10, le=600)      # 亏损后冷却时间
    # 仓位限制
    max_open_positions: int = Field(default=1, ge=1, le=5)  # 最大并发套利数量
    # 执行器超时
    executor_timeout_seconds: float = Field(default=300, ge=60, le=3600)  # 执行器最大等待时间（秒），超时未成交则 StopExecutorAction
```

### Controller 类：`SpreadSpikeArbController`

继承 `ControllerBase`，核心方法：

| 方法 | 职责 |
|------|------|
| `update_processed_data()` | 计算实时价差、滚动均值/标准差、Z-Score、信号，写入 `processed_data` |
| `determine_executor_actions()` | 检查风控 → 检测毛刺信号 → 创建 ArbitrageExecutor |
| `get_candles_config()` | 声明 K 线数据需求（两个交易所的 candles） |
| `_check_risk_controls()` | 检查四项风控条件，返回是否允许开仓 |
| `_update_risk_state()` | 从 executors_info 更新累计 PnL、连续亏损等风控状态（注意：必须通过 `executor_info.config.buying_market` / `executor_info.config.selling_market` 访问 ArbitrageExecutor 的属性，而非 ExecutorInfo.trading_pair / connector_name） |
| `_check_executor_timeout()` | 检查活跃执行器是否超时未成交，超时则发 StopExecutorAction |

### 执行流程

```
检测到毛刺信号（spread_bps >= min_spread_bps AND abs(z_score) >= zscore_threshold）
  → 检查风控（四项全部通过）
  → 检查并发限制（活跃 ArbitrageExecutor < max_open_positions）
  → 检查创建间隔（距上次创建 > delay_between_executors）
  → 创建 ArbitrageExecutorConfig：
      buying_market = 便宜的那一端
      selling_market = 贵的那一端
      min_profitability = 扣除双边手续费后的最低盈利
  → ArbitrageExecutor 内部：
      持续监控盈利性 → 达标后双腿市价同步下单 → 检查成交 → 失败重试
  → Controller 后续：
      从 executors_info 获取 ArbitrageExecutor 结果
      更新 _daily_pnl / _consecutive_losses
      触发风控条件时暂停开新仓
```

---

## 第三节：数据流与状态管理

### 数据流

```
MarketDataProvider
  ├─ get_candles_df(connector_pair_a, interval, lookback_periods) → candles_a
  ├─ get_candles_df(connector_pair_b, interval, lookback_periods) → candles_b
  └─ get_price_by_exchanger(connector_pair_a.connector_name, trading_pair) → price_a
  └─ get_price_by_exchanger(connector_pair_b.connector_name, trading_pair) → price_b
        │
        ▼
update_processed_data()  [每个 control_loop 周期执行]
  ├─ 按 timestamp 对齐 candles_a 和 candles_b（CEX 场景优先 inner join on timestamp，缺失数据 fallback 到 merge_asof with tolerance=interval/2）
  ├─ 计算历史价差序列：spread = (price_a - price_b) / mid_price
  ├─ rolling_mean / rolling_std（从对齐后的历史序列计算）
  ├─ 如果 rolling_std < min_std_bps / 10000 → signal = 0（标准差过小，不产生信号）
  ├─ z_score = (current_spread - rolling_mean) / rolling_std
  ├─ signal 判断：
  │     if spread_bps >= min_spread_bps AND z_score >= zscore_threshold → signal = 1（A贵B便宜）
  │     elif spread_bps >= min_spread_bps AND z_score <= -zscore_threshold → signal = -1（A便宜B贵）
  │     else → signal = 0
  └─ 写入 self.processed_data = {
       "price_a", "price_b", "spread", "spread_bps",
       "rolling_mean", "rolling_std", "z_score", "signal",
       "daily_pnl", "consecutive_losses", "cooldown_until",
       "active_executors_count"
     }
        │
        ▼
determine_executor_actions()
  ├─ 读取 processed_data
  ├─ _update_risk_state()（从 executors_info 更新风控状态，通过 config.buying_market/selling_market 访问属性）
  ├─ _check_risk_controls() → allowed: bool
  ├─ _check_executor_timeout() → [StopExecutorAction, ...]（超时未成交的执行器）
  ├─ if signal != 0 and allowed and active_count < max_open_positions:
  │     → 创建 ArbitrageExecutorConfig
  │     → CreateExecutorAction(executor_config, controller_id)
  └─ else: return []
        │
        ▼
ExecutorOrchestrator.execute_action()
  → 创建 ArbitrageExecutor 实例
  → ArbitrageExecutor 内部：监控盈利性 → 双腿市价下单 → 成交确认
```

### 状态管理

| 状态 | 存储位置 | 说明 |
|------|---------|------|
| 当前价差/信号 | `self.processed_data` | 每周期刷新 |
| 活跃执行器信息 | `self.executors_info` | 框架自动维护 |
| 日累计 PnL | `self._daily_pnl: float` + `self._pnl_history: List[Tuple[float, Decimal]]` | 滚动24小时窗口（非日历日重置，避免竞态），每次检查时过滤掉 24h 前的记录 |
| 连续亏损次数 | `self._consecutive_losses: int` | 盈利后重置为 0 |
| 冷却截止时间 | `self._cooldown_until: float` | 亏损后设置，到期后恢复 |
| 统计窗口 | K 线数据 | 通过 `get_candles_df()` 获取，无需自行维护 deque |

---

## 第四节：错误处理策略

### 错误分类与处理

| 错误场景 | 检测方式 | 处理策略 |
|---------|---------|---------|
| **单腿成交失败** | ArbitrageExecutor 内部 `process_order_failed_event` | 自动重试失败腿（最多3次），重试耗尽标记 FAILED |
| **余额不足** | ArbitrageExecutor 内部 `validate_sufficient_balance` | 标记 `CloseType.INSUFFICIENT_BALANCE`，终止执行器 |
| **盈利性不足** | ArbitrageExecutor 内部 `_current_profitability < min_profitability` | 持续等待，不执行双腿 |
| **价格数据中断** | `MarketDataProvider` 返回 None 或空 DataFrame | `update_processed_data()` 中检测，signal 置 0 |
| **K 线数据不足** | candles 长度 < `lookback_periods` | signal 置 0，不产生信号 |
| **标准差过小** | `rolling_std < min_std_bps / 10000` | signal 置 0，避免 Z-Score 爆炸 |
| **日累计亏损超限** | 滚动24小时 PnL 超过 `daily_max_loss_pct * total_amount_quote` | `_check_risk_controls()` 返回 False |
| **连续亏损暂停** | `_consecutive_losses >= max_consecutive_losses` | 进入冷却期 |
| **执行器超时未成交** | 创建后超过 `executor_timeout_seconds` 仍未进入 SHUTTING_DOWN | `_check_executor_timeout()` 发 StopExecutorAction |
| **交易所连接断开** | Connector 状态异常 | 依赖 `StrategyV2Base` 的连接器状态检查，自动暂停 |

### ArbitrageExecutor 的单腿风险处理

ArbitrageExecutor 的设计天然降低了单腿风险：
- **双腿市价同步下发**——市价单几乎确保即时成交，大幅减少单腿暴露时间
- **失败自动重试**——一条腿下单失败时，自动重试该腿（最多3次），另一腿已发出的订单保持
- **重试耗尽标记 FAILED**——Controller 检测到 FAILED 状态后更新 `_consecutive_losses` 和 `_daily_pnl`，暂停后续开仓

极端情况（市价单部分成交或交易所拒绝）下，ArbitrageExecutor 会标记 `CloseType.FAILED`，Controller 在下一周期检测到后更新风控状态。建议监控 FAILED 状态执行器，必要时人工介入处理裸露头寸。

---

## 第五节：测试策略

### 单元测试

| 测试目标 | 测试内容 | 工具 |
|---------|---------|------|
| `SpreadSpikeArbConfig` | 配置校验（必填字段、默认值、类型约束、Field 范围约束） | pytest |
| `update_processed_data()` | 价差计算正确性、Z-Score 计算、AND 信号逻辑、min_std_bps 过滤、K 线对齐 | pytest + mock MarketDataProvider |
| `_check_risk_controls()` | 四项风控的触发/恢复条件 | pytest |
| `determine_executor_actions()` | 信号+风控+并发+间隔的综合判断 | pytest |
| `_update_risk_state()` | 滚动24小时 PnL 计算、连续亏损统计 | pytest |
| `_check_executor_timeout()` | 执行器超时检测和 StopExecutorAction 生成 | pytest |
### 集成测试

| 测试目标 | 测试内容 | 工具 |
|---------|---------|------|
| Controller + ArbitrageExecutor | 信号触发 → 执行器创建 → 双腿成交 → PnL 更新 | pytest + mock connectors |
| 配置加载 | YAML → Config → Controller 实例化 | pytest |
| 风控联动 | 连续亏损后暂停、日亏损超限后停止开仓 | pytest + mock PnL |

### 回测

| 测试目标 | 测试内容 | 工具 |
|---------|---------|------|
| 历史价差毛刺回测 | 使用历史数据验证信号准确性和盈利能力 | V2 BacktestingEngine + ArbitrageExecutorSimulator（现有） |

> ArbitrageExecutor 已有对应的 `ArbitrageExecutorSimulator`，回测可复用现有基础设施，无需自建模拟器。
