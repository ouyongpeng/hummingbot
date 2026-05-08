# 任务计划：跨所价差 Z-Score 套利策略
日期：2026-04-22

## 来源
| 类型 | 引用 | 说明 |
|------|------|------|
| 📄 文档 | [spread-zscore-strategy-design.md](../achievements/2026-04-22-spread-zscore-strategy-design.md) | 策略设计文档（架构、参数、数据流） |
| 📄 代码 | `controllers/generic/stat_arb.py` | 最接近的参考实现（Z-Score + 协整套利） |
| 📄 代码 | `hummingbot/strategy_v2/controllers/controller_base.py` | ControllerBase 接口定义 |
| 📄 代码 | `hummingbot/strategy_v2/executors/arbitrage_executor/data_types.py` | ArbitrageExecutorConfig 定义 |

## 配置
| 项目 | 值 |
|------|---|
| TDD | ✅ 启用 |
| build & test 门控 | ✅ 开启 |
| simplify/code-review 门控 | ✅ 开启 |
| compact 检查 | ✅ 开启 |
| 执行次数 | 0 |
| 本次范围 | 全量 |

## 任务概览
总任务：5 个 ｜ 批次：2 批 ｜ 并行组：1 组

## 批次 1 / 2

### 前置（TDD）
- [ ] 编写测试：任务 A（控制器核心逻辑）

### 编码任务
- [ ] [串行] **任务 A** — `controllers/generic/spread_zscore.py` — 新功能 M
  - 实现 `SpreadZScoreControllerConfig`（继承 ControllerConfigBase）
  - 实现 `SpreadZScoreController`（继承 ControllerBase）
  - 覆盖 `update_processed_data()`：价差采集 + Z-Score 计算
  - 覆盖 `determine_executor_actions()`：信号触发 + ArbitrageExecutor 创建
  - 覆盖 `update_markets()`：注册两个交易所交易对
  - 覆盖 `to_format_status()`：状态显示
  - 覆盖 `get_candles_config()`：声明 K 线数据需求

### 门控
- [ ] build & test（新 Agent）
- [ ] simplify（新 Agent，无则 code-review）
- [ ] compact 检查

## 批次 2 / 2

### 编码任务
- [ ] [并行] **任务 B** — `conf/controllers/spread_zscore_1.yml` — 新功能 S
  - 创建 YAML 配置模板文件
  - 包含所有配置参数及注释说明
- [ ] [并行] **任务 C** — `test/hummingbot/strategy_v2/controllers/test_spread_zscore_controller.py` — 新功能 M
  - 集成测试：控制器初始化、价差计算、Z-Score 触发、冷却期、止损
  - 使用 mock 连接器和 OrderBook
- [ ] [并行] **任务 D** — `docs/achievements/2026-04-22-spread-zscore-strategy-design.md` — 文档 S
  - 更新设计文档：补充实际实现路径、配置示例、运行说明

### 门控
- [ ] build & test（新 Agent）
- [ ] simplify（新 Agent，无则 code-review）
- [ ] compact 检查

## 收尾任务

- [ ] **循环复检** — 独立审查员 Agent（所有批次完成后自动触发）

## 完成状态
| 批次 | 任务 | 状态 | 备注 |
|-----|------|:---:|------|
| 1 | 任务 A：实现控制器 | ⬜ | |
| 1 | build & test | ⬜ | |
| 1 | simplify | ⬜ | |
| 1 | compact | ⬜ | |
| 2 | 任务 B：YAML 配置 | ⬜ | |
| 2 | 任务 C：集成测试 | ⬜ | |
| 2 | 任务 D：更新文档 | ⬜ | |
| 2 | build & test | ⬜ | |
| 2 | simplify | ⬜ | |
| 2 | compact | ⬜ | |
| 收尾 | 循环复检 | ⬜ | |
