# 任务计划：spread-spike-arb

日期：2026-05-05

## 来源

| 类型 | 引用 | 说明 |
|------|------|------|
| 📄 文档 | [2026-05-05-spread-spike-arb-design.md](../achievements/2026-05-05-spread-spike-arb-design.md) | v3 设计文档（经两轮 spec-review 修正） |
| 📄 参考 | [arbitrage_controller.py](../../controllers/generic/arbitrage_controller.py) | 现有套利 Controller，复用其 ArbitrageExecutor 创建模式 |
| 📄 参考 | [stat_arb.py](../../controllers/generic/stat_arb.py) | 现有统计套利 Controller，复用其 Z-Score 计算和 K 线对齐模式 |

## 配置

| 项目 | 值 |
|------|---|
| TDD | 启用 |
| build & test 门控 | ✅ 开启 |
| simplify/code-review 门控 | ✅ 开启 |
| compact 检查 | ✅ 开启 |
| 执行次数 | 0 |
| 本次范围 | 全量 |

## 任务概览

总任务：7 个 ｜ 批次：2 批 ｜ 并行组：2 组

## 批次 1 / 2

### 前置（TDD）

- [ ] 编写测试：任务 1（Config）、任务 2（信号检测）、任务 3（风控逻辑）

### 编码任务

- [ ] [并行] **任务 1** — 实现 `SpreadSpikeArbConfig` — `controllers/generic/spread_spike_arb.py` — 新功能 S
- [ ] [并行] **任务 2** — 实现 `update_processed_data()` 信号检测 — `controllers/generic/spread_spike_arb.py` — 新功能 M
- [ ] [并行] **任务 3** — 实现风控四件套 + 超时清理 — `controllers/generic/spread_spike_arb.py` — 新功能 M

### 门控

- [ ] build & test（新 Agent）
- [ ] simplify（新 Agent，无则 code-review）
- [ ] compact 检查

## 批次 2 / 2

### 编码任务

- [ ] [串行→1,2,3] **任务 4** — 实现 `determine_executor_actions()` 主决策逻辑 — `controllers/generic/spread_spike_arb.py` — 新功能 M
- [ ] [并行] **任务 5** — 创建控制器 YAML 配置 — `conf/controllers/spread_spike_arb.yml` — 新功能 S
- [ ] [并行] **任务 6** — 更新脚本入口配置 — `conf/scripts/v2_with_controllers.yml` — 新功能 S

### 门控

- [ ] build & test（新 Agent）
- [ ] simplify（新 Agent，无则 code-review）
- [ ] compact 检查

## 收尾任务

- [ ] **循环复检** — 独立审查员 Agent（所有批次完成后自动触发）

## 完成状态

| 批次 | 任务 | 状态 | 备注 |
|-----|------|:---:|------|
| 1 | 任务 1：SpreadSpikeArbConfig | ⬜ | |
| 1 | 任务 2：信号检测 | ⬜ | |
| 1 | 任务 3：风控逻辑 | ⬜ | |
| 2 | 任务 4：主决策逻辑 | ⬜ | |
| 2 | 任务 5：控制器 YAML | ⬜ | |
| 2 | 任务 6：脚本入口配置 | ⬜ | |
| 收尾 | 循环复检 | ⬜ | |

---

## 任务详情

### 任务 1：实现 SpreadSpikeArbConfig

**类型**：新功能 S
**文件**：`controllers/generic/spread_spike_arb.py`
**依赖**：无

实现 `SpreadSpikeArbConfig(ControllerConfigBase)` 类，包含：
- `controller_name = "spread_spike_arb"`
- `connector_pair_a: ConnectorPair`、`connector_pair_b: ConnectorPair`
- 信号检测参数：`min_spread_bps`、`zscore_threshold`、`lookback_periods`、`interval`、`min_std_bps`（均带 Field 范围约束）
- 执行参数：`order_amount`（base asset 数量，带 ge/le 约束）、`min_profitability`、`delay_between_executors`、`max_executors_imbalance`
- 风控参数：`max_loss_per_trade_pct`、`daily_max_loss_pct`、`max_consecutive_losses`、`cooldown_after_loss_seconds`
- 仓位限制：`max_open_positions`
- 执行器超时：`executor_timeout_seconds`
- `update_markets()` 方法：注册两个 ConnectorPair 的交易对
- `get_candles_config()` 方法：声明两个交易所的 K 线需求

**参考**：`ArbitrageControllerConfig`（connector_pair 模式）、`StatArbConfig`（candles 配置模式）

### 任务 2：实现 update_processed_data() 信号检测

**类型**：新功能 M
**文件**：`controllers/generic/spread_spike_arb.py`
**依赖**：任务 1（Config 定义）

实现 `SpreadSpikeArbController(ControllerBase)` 的 `update_processed_data()` 方法：
- 从 `MarketDataProvider` 获取两个交易所的 K 线数据
- 按 timestamp inner join 对齐（缺失数据 fallback merge_asof with tolerance=interval/2）
- 计算历史价差序列 `spread = (price_a - price_b) / mid_price`
- 计算 `rolling_mean` / `rolling_std`
- `min_std_bps` 过滤：`rolling_std < min_std_bps / 10000` → signal = 0
- 计算 `z_score = (current_spread - rolling_mean) / rolling_std`
- AND 信号逻辑：`spread_bps >= min_spread_bps AND abs(z_score) >= zscore_threshold`
- K 线数据不足时 signal = 0
- 价格数据中断时 signal = 0
- 写入 `self.processed_data`

**参考**：`StatArb.get_spread_and_z_score()`（Z-Score 计算模式）

### 任务 3：实现风控四件套 + 超时清理

**类型**：新功能 M
**文件**：`controllers/generic/spread_spike_arb.py`
**依赖**：任务 1（Config 定义）

实现以下方法：
- `_update_risk_state()`：从 `executors_info` 更新风控状态
  - 通过 `executor_info.config.buying_market` / `executor_info.config.selling_market` 访问属性（非 ExecutorInfo.trading_pair/connector_name）
  - 维护 `_pnl_history: List[Tuple[float, Decimal]]`（时间戳, PnL），滚动24小时窗口
  - 更新 `_daily_pnl`、`_consecutive_losses`（盈利后重置为 0）
  - "亏损"定义：COMPLETED 且 net_pnl < 0，或 FAILED
- `_check_risk_controls()` → bool：四项风控检查
  1. 单笔止损：最近一笔亏损 > max_loss_per_trade_pct → False
  2. 日累计止损：滚动24h PnL < -daily_max_loss_pct * total_amount_quote → False
  3. 连续亏损暂停：_consecutive_losses >= max_consecutive_losses → False
  4. 冷却期：当前时间 < _cooldown_until → False
- `_check_executor_timeout()` → List[StopExecutorAction]：检查活跃执行器创建后超过 executor_timeout_seconds 仍未进入 SHUTTING_DOWN

### 任务 4：实现 determine_executor_actions() 主决策逻辑

**类型**：新功能 M
**文件**：`controllers/generic/spread_spike_arb.py`
**依赖**：任务 1、2、3

实现 `determine_executor_actions()` 方法：
- 调用 `_update_risk_state()`
- 调用 `_check_risk_controls()` → allowed
- 调用 `_check_executor_timeout()` → timeout_actions
- 检查并发限制：活跃 ArbitrageExecutor 数量 < max_open_positions
- 检查创建间隔：距上次创建 > delay_between_executors
- 检查不平衡：参考 ArbitrageController.update_arbitrage_stats() 模式
- 信号 != 0 且所有条件通过 → 创建 `ArbitrageExecutorConfig`：
  - `buying_market` = 便宜端 ConnectorPair
  - `selling_market` = 贵端 ConnectorPair
  - `order_amount` = config.order_amount
  - `min_profitability` = config.min_profitability
- 返回 `[CreateExecutorAction, ...timeout_actions]`

### 任务 5：创建控制器 YAML 配置

**类型**：新功能 S
**文件**：`conf/controllers/spread_spike_arb.yml`
**依赖**：任务 1（Config 字段定义）

创建 YAML 配置文件，包含：
- `id`（必填）
- `controller_name: spread_spike_arb`
- `controller_type: generic`
- `connector_pair_a` / `connector_pair_b`（嵌套 dict）
- 所有信号检测、执行、风控参数的默认值

### 任务 6：更新脚本入口配置

**类型**：新功能 S
**文件**：`conf/scripts/v2_with_controllers.yml`
**依赖**：任务 5

在现有 `v2_with_controllers.yml` 中添加 spread_spike_arb 控制器引用：
- `controller_name: spread_spike_arb`
- `config_file: spread_spike_arb.yml`