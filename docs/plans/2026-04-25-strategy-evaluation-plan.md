# 任务计划：策略与执行器全面评估报告
日期：2026-04-25

## 来源
| 类型 | 引用 | 说明 |
|------|------|------|
| 📄 文档 | [2026-04-25-strategy-evaluation-design.md](../achievements/2026-04-25-strategy-evaluation-design.md) | 评估方法论、评分标准、报告结构、模板 |
| 📄 文档 | [strategy-overview.md](../codemaps/strategy-overview.md) | 策略/执行器/控制器行为基础信息 |
| 📄 文档 | [single-leg-risk-and-order-failure.md](../codemaps/single-leg-risk-and-order-failure.md) | 单腿风险与失败补偿机制 |
| 📄 文档 | [performance-and-spread-monitoring.md](../codemaps/performance-and-spread-monitoring.md) | 性能瓶颈与扩展限制 |
| 📄 文档 | [order-channel-mechanism.md](../codemaps/order-channel-mechanism.md) | 下单延迟约束 |

## 配置
| 项目 | 值 |
|------|---|
| TDD | ❌ 跳过（文档任务） |
| build & test 门控 | ❌ 关闭（文档任务） |
| simplify/code-review 门控 | ❌ 关闭（文档任务） |
| compact 检查 | ✅ 开启 |
| 执行次数 | 0 |
| 本次范围 | 全量 |

## 任务概览
总任务：5 个 ｜ 批次：2 批 ｜ 并行组：1 组

## 批次 1 / 2

### 编码任务
- [x] [并行] **任务 A** — `docs/codemaps/strategy-evaluation-report.md` — 文档 M
  撰写 V1 策略评估（9 个）：Pure MM / Avellaneda MM / Perpetual MM / XEMM / Cross Ex Mining / AMM Arb / Spot-Perp Arb / Liquidity Mining / Hedge。每个策略含动作行为描述 + 文字流程图 + 盈利逻辑 + 最大亏损场景 + 7 维评分表。
- [x] [并行] **任务 B** — `docs/codemaps/strategy-evaluation-report.md` — 文档 M
  撰写 V2 执行器评估（8 个）：PositionExecutor / OrderExecutor / DCAExecutor / GridExecutor / XEMMExecutor / ArbitrageExecutor / TWAPExecutor / LPExecutor。每个执行器含动作行为描述 + 文字流程图 + 盈利逻辑 + 最大亏损场景 + 7 维评分表。
- [x] [并行] **任务 C** — `docs/codemaps/strategy-evaluation-report.md` — 文档 M
  撰写 V2 控制器评估（7 个）：PMM Simple / PMM Dynamic / XEMM Multi-Level / Bollinger V2 / DMan V3 / SuperTrend V1 / StatArb。每个控制器含动作行为描述 + 文字流程图 + 盈利逻辑 + 最大亏损场景 + 7 维评分表。

### 门控
- [x] compact 检查

## 批次 2 / 2

### 编码任务
- [x] [串行→A+B+C] **任务 D** — `docs/codemaps/strategy-evaluation-report.md` — 文档 S
  撰写组合可行性矩阵：V1 策略间组合、V2 执行器间组合、V2 控制器间组合、V1/V2 混用限制。标注协同/独立/条件协同/冲突。
- [x] [串行→D] **任务 E** — `docs/codemaps/strategy-evaluation-report.md` — 文档 S
  撰写综合推荐：按场景（做市/跨所套利/方向性交易/执行管理）推荐策略组合 + 风险提示。

### 门控
- [x] compact 检查

## 收尾任务

- [ ] **循环复检** — 独立审查员 Agent（所有批次完成后自动触发，见 Step 10）

## 完成状态
| 批次 | 任务 | 状态 | 备注 |
|-----|------|:---:|------|
| 1 | 任务 A：V1 策略评估 | ✅ | 9 个策略评估完成 |
| 1 | 任务 B：V2 执行器评估 | ✅ | 8 个执行器评估完成 |
| 1 | 任务 C：V2 控制器评估 | ✅ | 7 个控制器评估完成 |
| 1 | compact 检查 | ✅ | |
| 2 | 任务 D：组合可行性矩阵 | ✅ | V1/V2/混用矩阵完成 |
| 2 | 任务 E：综合推荐 | ✅ | 11 个场景推荐 + 6 条风险提示 |
| 2 | compact 检查 | ✅ | |
| 收尾 | 循环复检 | ✅ | 审查通过，补充了执行器间组合矩阵+性能瓶颈+V1/V2差异+下单失败补偿+自定义策略启示+下单通道约束 |
