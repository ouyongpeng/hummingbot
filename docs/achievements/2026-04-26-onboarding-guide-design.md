# Hummingbot 新手探索指南 — 设计文档

> 目标用户：有代码项目经验、了解基础交易概念、首次使用 Hummingbot
> 探索路线：回测 → 纸盘 → 小资金实盘（渐进式）
> 最后更新：2026-04-26

---

## 目录

- [1. 整体架构](#1-整体架构)
- [2. 阶段一：初次配置](#2-阶段一初次配置)
- [3. 阶段二：历史数据回测](#3-阶段二历史数据回测)
- [4. 阶段三：实操策略选择](#4-阶段三实操策略选择)
- [5. 阶段四：复盘](#5-阶段四复盘)
- [6. 错误处理与风控](#6-错误处理与风控)
- [7. 附录](#7-附录)

---

## 1. 整体架构

```
阶段一：初次配置（30 分钟）
  ├─ 1.1 环境安装（Conda + 编译）
  ├─ 1.2 首次启动与密码设置
  ├─ 1.3 连接交易所（API Key 配置）
  └─ 1.4 纸盘模式验证连接

阶段二：历史数据回测（1 小时）
  ├─ 2.1 回测概念速览
  ├─ 2.2 选择第一个控制器（推荐 pmm_simple）
  ├─ 2.3 配置回测参数
  ├─ 2.4 运行回测与解读结果
  └─ 2.5 调参实验

阶段三：实操策略选择（1 小时）
  ├─ 3.1 V1 vs V2 架构对比
  ├─ 3.2 策略分类与适用场景
  ├─ 3.3 纸盘交易实操
  └─ 3.4 小资金实盘注意事项

阶段四：复盘（30 分钟）
  ├─ 4.1 日志文件位置与结构
  ├─ 4.2 交易数据查询（数据库）
  ├─ 4.3 回测结果可视化
  └─ 4.4 关键指标解读

附录
  ├─ A. 常见问题排查
  ├─ B. 策略速查表
  └─ C. 配置文件参考
```

**设计原则**：
- 每个阶段都是可独立完成的闭环，用户可以随时停下
- 先给操作步骤，再解释原理
- 所有命令可直接复制执行
- 深入到源码层面解析关键机制

---

## 2. 阶段一：初次配置

### 2.1 环境安装

**前置条件**：macOS + Conda（Miniconda 或 Anaconda）

```bash
# 克隆仓库
git clone https://github.com/hummingbot/hummingbot.git
cd hummingbot

# 一键安装：创建 Conda 环境 + 编译 Cython + 安装依赖
make install
```

**`make install` 做了什么**（源码：`Makefile`）：
1. 检查 Conda 是否在 PATH 中（未找到则报错退出）
2. `mkdir -p logs` — 创建日志目录
3. `conda env create/update -n hummingbot -f environment.yml` — 创建或更新 Conda 环境
4. macOS 专属：`conda install -n hummingbot -y appnope` — 防止 App Nap 休眠
5. `conda develop .` — 将项目根目录加入 Python 路径
6. `pip install --no-deps -r setup/pip_packages.txt` — 安装 pip 额外依赖
7. `pre-commit install` — 安装 Git 预提交钩子
8. Linux 专属：检查并安装 `build-essential`（C 编译工具链）
9. `python setup.py build_ext --inplace` — 编译 Cython 扩展（`.pyx` → `.so`）

**环境定义**：`setup/environment.yml`，核心依赖：
- Python >= 3.10.12
- aiohttp / websockets（异步网络）
- SQLAlchemy（ORM）
- pandas / numpy / scipy（数据分析）
- web3 / eth-account（以太坊交互）
- prompt_toolkit（CLI）

**验证安装**：
```bash
conda run -n hummingbot python -c "import hummingbot; print('OK')"
```

### 2.2 首次启动

```bash
make run
# 或带参数：
make run ARGS="--config-password yourpassword"
```

**启动流程**（源码：`bin/hummingbot_quickstart.py`）：
1. 解析命令行参数（`--config-file-name`、`--v2`、`--config-password`、`--headless`）
2. `ETHKeyFileSecretManger` 解密/加密配置文件（密码保护）
3. 加载 `conf/conf_client.yml`
4. `create_yml_files_legacy()` 生成缺失的默认配置文件
5. `init_logging("hummingbot_logs.yml", ...)` 初始化日志系统
6. 创建 `HummingbotApplication` 实例，进入 CLI 交互界面

**首次启动会要求设置密码**：此密码用于加密 `conf/` 下的敏感配置文件（API Key 等），遗忘后需删除加密文件重新配置。

### 2.3 连接交易所

**方式一：CLI 交互式配置**
```
# 在 Hummingbot CLI 中输入
connect binance
# 按提示输入 API Key 和 Secret
```

**方式二：手动编辑配置文件**
- 配置路径：`conf/connectors/` 下按交易所名生成 YAML
- 文件内容被密码加密，不建议手动编辑

**API Key 权限要求**：
- 必须开启：现货交易（Spot Trading）、读取账户信息
- **禁止开启**：提币（Withdrawal）权限
- 建议设置 IP 白名单

**推荐先用 Binance 测试网**：
- 永续合约测试网：`https://testnet.binancefuture.com/fapi/`（代码中已集成）
- 现货测试网：请从 Binance 官方文档获取最新测试网地址
- 无需真实资金，功能与正式环境一致

### 2.4 纸盘模式验证连接

**启用纸盘模式**：

纸盘模式通过连接器名称后缀 `_paper_trade` 启用，在控制器配置的 `connector_name` 字段中设置：

```yaml
# 控制器配置（conf/controllers/pmm_simple.yml）
connector_name: binance_paper_trade  # 在交易所名后加 _paper_trade
```

**纸盘模式原理**（源码：`hummingbot/connector/exchange/paper_trade/`）：
- 连接器名后缀 `_paper_trade` 触发 `create_paper_trade_market()` 创建模拟交易所
- 使用模拟余额（可在 `conf_client.yml` 的 `paper_trade.paper_trade_account_balance` 中配置初始余额）
- 订单不发送到交易所，仅在本地 `PaperTradeExchange` 模拟撮合
- 适合验证策略逻辑和配置是否正确

**纸盘余额配置**（`conf/conf_client.yml`）：
```yaml
paper_trade:
  paper_trade_exchanges: [binance]
  paper_trade_account_balance:
    ETH: 10
    USDT: 10000
```

**验证步骤**：
1. 启动 Hummingbot
2. 使用 `start --v2 v2_with_controllers.yml` 启动 V2 策略（控制器中 `connector_name` 设为 `binance_paper_trade`）
3. 观察是否正常生成模拟订单
4. 确认连接器状态为"已连接"，CLI 显示 "Paper Trading Active"

---

## 3. 阶段二：历史数据回测

### 3.1 回测概念速览

**什么是回测**：用历史市场数据模拟策略执行，评估策略在过去的表现。

**为什么先回测**：
- 零资金风险
- 快速验证策略逻辑
- 量化评估策略指标（PnL、回撤、夏普比率）
- 参数敏感性分析

**Hummingbot 回测架构**（V2）：

```
控制器配置 (YAML)
    ↓
BacktestingEngineBase.run_backtesting()
    ↓
BacktestingDataProvider → CandlesFactory → 历史K线数据
    ↓
ExecutorSimulator（4种）→ 模拟执行
    ↓
BacktestingResult → 文本摘要 + Plotly 图表
```

**核心源码**：
- 回测引擎：`hummingbot/strategy_v2/backtesting/backtesting_engine_base.py`
- 数据提供者：`hummingbot/strategy_v2/backtesting/backtesting_data_provider.py`
- 结果封装：`hummingbot/strategy_v2/backtesting/backtesting_result.py`
- 执行器模拟器：`hummingbot/strategy_v2/backtesting/executors_simulator/`

### 3.2 选择第一个控制器：pmm_simple

**推荐理由**：
- 参数少、逻辑直观（固定价差做市）
- 适合理解 Controller-Executor 架构
- 回测结果容易解读

**pmm_simple 源码**：`controllers/market_making/pmm_simple.py`

**配置类**：`PMMSimpleControllerConfig`（继承 `MarketMakingControllerConfigBase`）

**核心参数**：
| 参数 | 说明 | 示例值 |
|------|------|--------|
| `connector_name` | 交易所 | `binance` |
| `trading_pair` | 交易对 | `ETH-USDT` |
| `buy_spreads` | 买入价差列表 | `[0.01]`（1%） |
| `sell_spreads` | 卖出价差列表 | `[0.01]`（1%） |
| `buy_amounts` | 买入订单量列表 | `[0.1]` |
| `sell_amounts` | 卖出订单量列表 | `[0.1]` |
| `interval_seconds` | 刷新间隔 | `60` |

### 3.3 配置回测参数

**控制器配置文件**（`conf/controllers/pmm_simple.yml`）：
```yaml
controller_name: pmm_simple
controller_type: market_making

connector_name: binance
trading_pair: ETH-USDT
buy_spreads: [0.01]
sell_spreads: [0.01]
buy_amounts: [0.1]
sell_amounts: [0.1]
interval_seconds: 60
```

**回测脚本配置**（`conf/scripts/v2_with_controllers.yml`）：
```yaml
script_file_name: v2_with_controllers
controllers:
  - controller_name: pmm_simple
    config_file: pmm_simple.yml
```

**回测运行方式**（源码：`BacktestingEngineBase.run_backtesting()`）：
```python
result = await engine.run_backtesting(
    controller_config=config_instance,
    start=start_timestamp,      # 开始时间（epoch 秒）
    end=end_timestamp,          # 结束时间（epoch 秒）
    backtesting_resolution="1m", # K线分辨率
    trade_cost=0.0002           # 交易成本（手续费率）
)
```

### 3.4 运行回测与解读结果

**运行回测**（独立 Python 脚本）：
```bash
# 使用项目自带的回测脚本
conda run -n hummingbot python scripts/backtest_grid_strike.py --days 3 --chart

# 或运行自定义回测脚本
conda run -n hummingbot python scripts/backtest_bollinger_v2.py --days 1 --chart
```

**回测脚本列表**（`scripts/` 目录）：
| 脚本 | 控制器 | 用法 |
|------|--------|------|
| `backtest_grid_strike.py` | grid_strike | `--days N --chart --output file.html` |
| `backtest_bollinger_v2.py` | bollinger_v2 | 同上 |
| `backtest_pmm_mister.py` | pmm_mister | 同上 |

**自定义回测脚本**：参考 `scripts/backtest_grid_strike.py`，核心流程：
1. 构建控制器配置字典
2. 创建 `BacktestingEngineBase` 实例
3. 调用 `engine.run_backtesting()` 运行回测
4. 通过 `BacktestingResult` 获取结果和图表

**结果解读**（`BacktestingResult`）：

1. **文本摘要**（`get_results_summary()`）：
   - 净 PnL（Net PnL）
   - 最大回撤（Max Drawdown）
   - 夏普比率（Sharpe Ratio）
   - 利润因子（Profit Factor）
   - 准确率（Win Rate）
   - 关闭类型统计（按 TP/SL/TL 分类）

2. **交互图表**（`get_backtesting_figure()`）：
   - 第一行：K线 + 执行器标记（买卖点）
   - 第二行：累计 PnL 曲线
   - 第三行：持仓量时序图

3. **执行器详情**（`executors_df`）：
   - DataFrame 格式，每行一个执行器
   - 包含：开仓时间、平仓时间、方向、入场价、出场价、PnL、关闭类型

### 3.5 调参实验

**关键参数敏感性**：
| 参数 | 影响 | 调参方向 |
|------|------|----------|
| `buy_spreads` / `sell_spreads` | 价差越大，成交越少但单笔利润越高 | 从 1% 开始，逐步缩小 |
| `buy_amounts` / `sell_amounts` | 订单量影响资金利用率和风险暴露 | 从最小值开始 |
| `interval_seconds` | 刷新频率影响订单跟踪精度 | 60s 为默认，高频可降到 15s |
| `trade_cost` | 手续费直接影响净 PnL | 按实际交易所费率设置 |

**调参建议**：
- 每次只调一个参数
- 记录每次回测的关键指标
- 警惕过度拟合：回测表现好不代表实盘好

---

## 4. 阶段三：实操策略选择

### 4.1 V1 vs V2 架构对比

| 维度 | V1 | V2 |
|------|----|----|
| 架构 | 策略一体式 | Controller-Executor 分离 |
| 配置方式 | `conf/strategies/*.yml` | `conf/controllers/*.yml` + `conf/scripts/*.yml` |
| 策略逻辑 | 写在策略类中（`.pyx`） | 写在控制器中（`.py`），配置驱动 |
| 执行器 | 内置于策略 | 独立执行器（8种），可组合 |
| 回测 | 不支持 | 原生支持 |
| 扩展性 | 需写 Cython | 写 Python 控制器即可 |
| 推荐度 | 旧项目兼容 | **新用户推荐** |

**V2 架构核心**：
```
Script（入口）
  → Controller（信号生成 + 订单参数）
    → ExecutorOrchestrator（执行器编排）
      → Executor（订单执行 + 风控）
        → Connector（交易所通信）
```

**源码位置**：
- 控制器基类：`hummingbot/strategy_v2/controllers/controller_base.py`
- 执行器编排：`hummingbot/strategy_v2/executors/executor_orchestrator.py`
- 执行器注册映射：`ExecutorOrchestrator._executor_mapping`

### 4.2 策略分类与适用场景

**做市类**（Market Making）— 赚买卖价差

| 控制器 | 源码 | 难度 | 适用场景 |
|--------|------|------|----------|
| pmm_simple | `controllers/market_making/pmm_simple.py` | 入门 | 固定价差做市，适合流动性好的币对 |
| pmm_dynamic | `controllers/market_making/pmm_dynamic.py` | 进阶 | NATR+MACD 动态调整价差，适合波动市场 |
| dman_maker_v2 | `controllers/market_making/dman_maker_v2.py` | 进阶 | 多指标组合做市 |

**方向性交易类**（Directional Trading）— 赚趋势利润

| 控制器 | 源码 | 难度 | 适用场景 |
|--------|------|------|----------|
| supertrend_v1 | `controllers/directional_trading/supertrend_v1.py` | 入门 | Supertrend 指标信号，适合趋势市场 |
| bollinger_v1 | `controllers/directional_trading/bollinger_v1.py` | 入门 | 布林带突破信号 |
| macd_bb_v1 | `controllers/directional_trading/macd_bb_v1.py` | 进阶 | MACD+布林带组合信号 |
| dman_v3 | `controllers/directional_trading/dman_v3.py` | 进阶 | 多指标组合信号 |

**通用类**（Generic）

| 控制器 | 源码 | 难度 | 适用场景 |
|--------|------|------|----------|
| grid_strike | `controllers/generic/grid_strike.py` | 入门 | 网格交易，适合震荡市场 |
| stat_arb | `controllers/generic/stat_arb.py` | 进阶 | 统计套利（Z-Score+协整） |
| arbitrage_controller | `controllers/generic/arbitrage_controller.py` | 进阶 | 跨所套利 |

**执行器选择**：

| 执行器 | 源码 | 核心行为 | 风控 |
|--------|------|----------|------|
| PositionExecutor | `executors/position_executor/` | 三重屏障开仓-平仓 | TP/SL/TL/TrailingStop |
| OrderExecutor | `executors/order_executor/` | 简单下单 | 无 |
| DCAExecutor | `executors/dca_executor/` | 分批建仓 | 三重屏障 |
| GridExecutor | `executors/grid_executor/` | 网格交易 | 整体止损 |
| XEMMExecutor | `executors/xemm_executor/` | 跨所做市 | 无 |
| ArbitrageExecutor | `executors/arbitrage_executor/` | 双边市价套利 | 无 |
| TWAPExecutor | `executors/twap_executor/` | 时间加权拆分 | 无 |
| LPExecutor | `executors/lp_executor/` | DEX 流动性提供 | 无止损 |

**新手推荐组合**：
1. **pmm_simple + PositionExecutor** — 做市入门
2. **supertrend_v1 + PositionExecutor** — 趋势交易入门
3. **grid_strike + GridExecutor** — 网格交易入门

### 4.3 纸盘交易实操

**步骤**：
1. 配置控制器 YAML（`conf/controllers/`），`connector_name` 设为 `binance_paper_trade`
2. 配置脚本 YAML（`conf/scripts/`）
3. 启动 Hummingbot，使用 `start --v2 v2_with_controllers.yml` 启动 V2 策略
4. 观察模拟订单生成和执行

**纸盘余额配置**（`conf_client.yml`）：
```yaml
paper_trade:
  paper_trade_account_balance:
    ETH: 10
    USDT: 10000
```

**验证要点**：
- 订单是否按预期生成
- 止损/止盈是否触发
- 仓位管理是否正确
- 至少运行 3 天验证稳定性

### 4.4 小资金实盘注意事项

**过渡检查清单**：
- [ ] 回测结果为正 PnL
- [ ] 纸盘运行 3 天以上无异常
- [ ] 理解策略的核心参数和风险点
- [ ] API Key 仅开启交易权限（无提币）
- [ ] 设置了 IP 白名单

**实盘启动**：
1. 将控制器 YAML 中 `connector_name` 从 `binance_paper_trade` 改为 `binance`（去掉 `_paper_trade` 后缀）
2. 确认 API Key 已连接（`connect <exchange>`）
3. **订单量 = 交易所最小下单量**
4. 设置止损（三重屏障的 SL 参数）

**三重屏障风控**（PositionExecutor）：
- **TP（Take Profit）**：止盈价格/百分比
- **SL（Stop Loss）**：止损价格/百分比
- **TL（Time Limit）**：持仓时间限制，超时强制平仓
- **Trailing Stop**：追踪止损，锁定浮动利润

**源码**：`hummingbot/strategy_v2/executors/position_executor/position_executor.py`

---

## 5. 阶段四：复盘

### 5.1 日志文件位置与结构

**日志目录**：`logs/`

| 文件 | 内容 | 格式 |
|------|------|------|
| `logs_hummingbot.log` | 主日志（所有模块） | `HH:MM:SS - 模块名 - 消息` |
| `errors.log` | 错误日志 | 同上 |
| `logs_{strategy_name}.log` | 按策略分文件 | 同上 |

**日志配置**：`conf/hummingbot_logs.yml`

**日志级别层次**：

| logger 名 | 级别 | handlers | 说明 |
|-----------|------|----------|------|
| `hummingbot.strategy` | NETWORK (DEBUG+6) | console + file | 策略日志 |
| `hummingbot.connector` | NETWORK | console + file | 连接器日志 |
| `hummingbot.client` | NETWORK | console + file | 客户端日志 |
| `hummingbot.core.event.event_reporter` | EVENT_LOG (15) | file_only | 事件日志（仅写文件） |

**日志轮转**：按天轮转（`when: "D", interval: 1, backupCount: 7`），保留 7 天。

**结构化日志**（`StructLogger`，源码：`hummingbot/logger/struct_logger.py`）：
- `event_log()` 方法：JSON 格式输出，级别 15
- `metrics_log()` 方法：指标日志，级别 14
- 适合程序化分析

### 5.2 交易数据查询（数据库）

**数据库名称**：`hummingbot_trades.sqlite`（源码：`hummingbot/model/sql_connection_manager.py`）

**配置**（`conf/conf_client.yml`）：
```yaml
db_mode:
  db_engine: sqlite
```

**数据持久化**（源码：`ExecutorOrchestrator`）：
- `store_all_executors()` — 策略停止时保存所有执行器状态
- `store_all_positions()` — 保存所有仓位信息

**查询方式**：
```bash
# 进入 SQLite 命令行
sqlite3 hummingbot_trades.sqlite

# 查看所有表
.tables

# 查询执行器记录
SELECT * FROM Executors ORDER BY timestamp DESC LIMIT 20;

# 查询仓位记录
SELECT * FROM Position ORDER BY timestamp DESC LIMIT 20;
```

**回测数据导出**：
```python
# BacktestingResult.executors_df 返回 DataFrame
result.executors_df.to_csv("backtest_result.csv")
```

### 5.3 回测结果可视化

**Plotly 交互图表**（`BacktestingResult.get_backtesting_figure()`）：
- 第一行：K线 + 执行器标记（绿色=买入，红色=卖出）
- 第二行：累计 PnL 曲线
- 第三行：持仓量时序图

**输出格式**：HTML 文件，可在浏览器中交互查看。

### 5.4 关键指标解读

| 指标 | 含义 | 健康范围 |
|------|------|----------|
| **净 PnL** | 扣除手续费后的总利润 | > 0 |
| **最大回撤** | 从峰值到谷值的最大跌幅 | < 20%（做市）；< 30%（方向性） |
| **夏普比率** | 风险调整后收益 | > 1.0 良好；> 2.0 优秀 |
| **利润因子** | 总盈利 / 总亏损 | > 1.5 |
| **准确率** | 盈利交易占比 | 做市 > 50%；方向性 > 40% 即可 |
| **关闭类型统计** | 按 TP/SL/TL 分类的平仓统计 | SL 占比 < 30% |

**注意事项**：
- 回测指标 ≠ 实盘指标（滑点、延迟、流动性差异）
- 夏普比率在短周期回测中可能虚高
- 准确率不是唯一标准（高盈亏比 + 低准确率也可以盈利）

---

## 6. 错误处理与风控

### 6.1 各阶段常见问题

**阶段一：初次配置**

| 问题 | 原因 | 处理 |
|------|------|------|
| `make install` 失败 | Conda 未安装 / Python 版本不对 | 检查 Conda 是否在 PATH 中，确认 Python >= 3.10 |
| Cython 编译报错 | 缺少 C 编译器 | macOS 安装 Xcode Command Line Tools：`xcode-select --install` |
| 首次启动密码遗忘 | 配置文件加密 | 删除 `conf/` 下加密文件重新配置 |
| API Key 连接失败 | Key 权限不足 / IP 白名单 | 确认 Key 开启现货交易权限，检查 IP 限制 |

**阶段二：回测**

| 问题 | 原因 | 处理 |
|------|------|------|
| 历史数据拉取失败 | 网络问题 / 交易所 API 限制 | 检查网络，降低请求频率 |
| 回测结果为空 | 时间范围内无交易信号 | 扩大时间范围或调整控制器参数 |
| 参数不合法 | YAML 格式错误 / 值越界 | 检查 YAML 缩进，对照控制器配置类定义 |

**阶段三：实操**

| 问题 | 原因 | 处理 |
|------|------|------|
| 纸盘余额不足 | 默认余额有限 | 在 `conf_client.yml` 中调整纸盘余额 |
| 实盘下单被拒 | 余额不足 / 最小下单量 | 检查账户余额，确认订单量 >= 交易所最小值 |
| 策略停止运行 | 触发止损 / 连接断开 | 查看日志定位原因，检查网络连接 |

**阶段四：复盘**

| 问题 | 原因 | 处理 |
|------|------|------|
| 日志文件为空 | 策略未运行 / 日志级别过高 | 确认策略已运行，检查 `hummingbot_logs.yml` 级别 |
| 数据库查询失败 | SQLite 文件锁定 / 路径错误 | 确认 Hummingbot 已停止再查询 |

### 6.2 风控红线

- **实盘必须设置止损**（三重屏障的 SL 参数）
- **首次实盘订单量 = 交易所最小下单量**
- **API Key 只开启必要权限**（现货交易，不开提币）
- **回测结果不代表未来表现**（过度拟合警告）
- **单策略最大资金暴露 < 总资金 10%**

---

## 7. 附录

### A. 常见问题排查

**Q: 如何重置所有配置？**
```bash
rm -rf conf/
make run  # 重新生成默认配置
```

**Q: 如何查看当前连接器状态？**
```
# 在 CLI 中
status
```

**Q: 如何停止策略？**
```
# 在 CLI 中
stop
```

**Q: 如何切换策略？**
```
stop          # 先停止当前策略
config        # 重新配置
start         # 启动新策略
```

### B. 策略速查表

| 目标 | 推荐控制器 | 执行器 | 难度 | 适合市场 |
|------|-----------|--------|------|----------|
| 做市赚价差 | pmm_simple | PositionExecutor | 入门 | 震荡/平稳 |
| 动态价差做市 | pmm_dynamic | PositionExecutor | 进阶 | 波动 |
| 趋势跟踪 | supertrend_v1 | PositionExecutor | 入门 | 趋势 |
| 布林带突破 | bollinger_v1 | PositionExecutor | 入门 | 震荡突破 |
| 网格交易 | grid_strike | GridExecutor | 入门 | 震荡 |
| 统计套利 | stat_arb | PositionExecutor | 进阶 | 配对交易 |
| 跨所套利 | arbitrage_controller | ArbitrageExecutor | 进阶 | 价差存在 |

### C. 配置文件参考

**关键配置路径**（源码：`hummingbot/client/settings.py`）：

| 常量 | 路径 | 说明 |
|------|------|------|
| `CONF_DIR_PATH` | `conf/` | 配置根目录 |
| `STRATEGIES_CONF_DIR_PATH` | `conf/strategies/` | V1 策略配置 |
| `SCRIPT_STRATEGY_CONF_DIR_PATH` | `conf/scripts/` | V2 脚本配置 |
| `CONTROLLERS_CONF_DIR_PATH` | `conf/controllers/` | V2 控制器配置 |
| `CONNECTORS_CONF_DIR_PATH` | `conf/connectors/` | 连接器 API Key |

**控制器模块路径**：
- `controllers.market_making.pmm_simple`
- `controllers.market_making.pmm_dynamic`
- `controllers.directional_trading.supertrend_v1`
- `controllers.directional_trading.bollinger_v1`
- `controllers.generic.grid_strike`
- `controllers.generic.stat_arb`

**控制器配置加载流程**（源码：`get_controller_config_instance_from_yml()`）：
1. 读取 YAML → 获取 `controller_type` 和 `controller_name`
2. 动态导入 `controllers.{controller_type}.{controller_name}`
3. 实例化对应的 `ControllerConfigBase` 子类
