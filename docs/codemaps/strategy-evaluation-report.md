# Hummingbot 策略与执行器全面评估报告

> 日期：2026-04-25
> 评估范围：9 个 V1 策略 + 8 个 V2 执行器 + 7 个 V2 控制器
> 评分标准：7 维 1-5 分，综合 = 平均分

---

## 一、评估方法论

### 评分标准（每维 1-5 分）

| 维度 | 1 分 | 3 分 | 5 分 |
|------|------|------|------|
| 盈利确定性 | 依赖极端行情，日常无收益 | 特定行情下有稳定收益 | 收益逻辑自洽，多行情可盈利 |
| 风险可控性 | 无止损/单腿暴露/无重试上限 | 有部分风控但存在盲区 | 三重屏障+自动平仓+重试上限 |
| 适用行情广度 | 仅单一行情有效 | 2 种行情有效 | 牛/熊/震荡均适用 |
| 资金效率 | 大量资金闲置 | 部分资金运作 | 资金持续运作 |
| 实操门槛 | 需深度调参+多交易所配置 | 需基本参数调整 | 开箱即用 |
| 组合灵活性 | 独立运行，不可组合 | 可与部分策略组合 | 可与多种策略组合 |
| 代码成熟度 | 实验性/已知缺陷 | 基本可用，边界情况待完善 | 生产级/久经考验 |

### 组合判定标准

| 判定 | 含义 |
|------|------|
| 协同 | 可同时运行，增强效果 |
| 独立 | 互不影响，可并行但无增益 |
| 条件协同 | 需满足特定前提才能组合 |
| 冲突 | 资源竞争或逻辑矛盾，不可同时运行 |

### 边界约束

1. **V1 策略无三重屏障**：所有 V1 策略的风险可控性评分上限为 3
2. **V1/V2 混用**：V1 策略与 V2 控制器架构不兼容，标注"架构冲突"
3. **流动性假设**：市价单执行假设对手盘充足，低流动性交易对实际滑点可能远超预期
4. **REST 延迟约束**：跨所策略双边 REST 下单 100-400ms

---

## 二、V1 策略评估

### 1. Pure Market Making

**动作行为**：
在单一交易所的单一交易对上持续挂出买单和卖单，以中间价为基准偏移 bid_spread/ask_spread。每个 tick 经历完整提案管线：创建基础报价 → 应用订单层数/价格带/ping-pong 修饰 → 应用价格优化/手续费修饰 → 应用库存偏斜修饰 → 应用预算约束 → 取消旧单 → 执行新单。订单按 order_refresh_time 周期刷新，超过 max_order_age 强制取消，低于 minimum_spread 自动撤单。

**流程图**：
```
c_tick() → 创建基础提案 → 层数/价格带/ping-pong修饰
  → 价格优化/手续费修饰 → 库存偏斜修饰 → 预算约束
  → 取消超龄订单 → 取消低于最小价差订单 → 判断是否需创建
  ├─ 需创建 → 执行提案 → 等待下一周期
  └─ 无需创建 → 等待下一周期
```

**盈利逻辑**：赚取 bid-ask spread，当买卖单均成交时锁定价差利润；库存偏斜通过调整单量引导库存回归目标
**最大亏损场景**：单边行情下库存持续积累（只成交买单或卖单），价格大幅反向波动导致库存浮亏；无止损机制，亏损可无限扩大
**不适用场景**：单边趋势行情（持续上涨时卖单被吃完只留买单，持续下跌反之）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 震荡行情中 spread 收益稳定，但单边行情下库存偏移导致方向性亏损 |
| 风险可控性 | 2 | 有价格带/最小价差/库存偏斜，但无止损止盈、无重试上限、单腿暴露风险显著 |
| 适用行情广度 | 2 | 仅震荡行情有效，趋势行情库存累积导致亏损 |
| 资金效率 | 3 | 双侧挂单持续运作，但库存偏移时一侧资金闲置 |
| 实操门槛 | 3 | 需调整 spread/order_amount/refresh_time 等基本参数，配置项较多但文档完善 |
| 组合灵活性 | 3 | 可与套利策略组合，但自身库存状态影响组合资金分配 |
| 代码成熟度 | 5 | 最早期策略，Cython 加速，功能完备（悬挂单/价格带/库存偏斜/订单覆盖），久经生产考验 |
| **综合** | **3.0** | |

---

### 2. Avellaneda Market Making

**动作行为**：
基于 Avellaneda-Stoikov 做市理论，实时计算 reservation_price（中间价 - q×gamma×vol×time_left_fraction）和 optimal_spread（gamma×vol×T + 2×ln(1+gamma/kappa)/gamma），以预订价格为中心对称挂单。库存偏斜内嵌于预订价格公式中，eta 参数通过指数衰减动态缩量逆向订单。波动率和订单簿流动性通过滑动窗口实时估计。

**流程图**：
```
c_tick() → 采集市场变量(波动率/交易强度) → 算法就绪?
  ├─ 未就绪 → 等待缓冲区填满
  └─ 已就绪 → 计算预订价格和最优价差 → 创建基础提案
    → eta变换(逆向订单缩量) → 价格优化/手续费修饰
    → 预算约束 → 取消旧单 → 执行新单
```

**盈利逻辑**：理论最优 spread 收益 + 库存偏斜通过预订价格偏移自动引导库存回归，波动率自适应使 spread 在高波动时扩大保护收益
**最大亏损场景**：波动率估计滞后导致 spread 过窄，在剧烈行情下被穿仓；gamma 设置过低导致库存风险敞口过大
**不适用场景**：低流动性市场（交易强度 kappa 估计不准导致 spread 异常）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 4 | 收益逻辑有理论保证，库存偏斜内嵌于定价公式而非外部修饰，多行情下表现优于 PMM |
| 风险可控性 | 2 | 有 min_spread 硬下限和 eta 缩量，但无止损/止盈 |
| 适用行情广度 | 3 | 震荡和温和趋势均适用（预订价格自动偏移），但剧烈趋势仍可能穿仓 |
| 资金效率 | 3 | 双侧挂单，eta 变换优化逆向单量 |
| 实操门槛 | 2 | 需理解 gamma/eta 等理论参数，波动率缓冲区需预热，调试门槛高于 PMM |
| 组合灵活性 | 3 | 可与套利策略组合，但参数敏感度更高 |
| 代码成熟度 | 4 | Cython 加速，支持执行时间框架/多层级/悬挂单，但波动率估计在极端情况下不稳定 |
| **综合** | **3.0** | |

---

### 3. Perpetual Market Making

**动作行为**：
在永续合约上做市，分两个阶段运行：无仓位时与 PMM 相同挂买卖单开仓；有仓位时进入仓位管理阶段，同时挂止盈单（entry_price × (1 + profit_taking_spread)）和止损单（entry_price × (1 - stop_loss_spread)），止损单加入 slippage_buffer 并按 time_between_stop_loss_orders 定期续期。支持 Hedge/OneWay 两种仓位模式。

**流程图**：
```
tick() → 无仓位?
  ├─ 是 → 创建基础提案 → 层数/价格带修饰 → 价格优化 → 预算约束 → 过滤taker → 开仓执行
  └─ 否 → manage_positions()
    → 止盈提案(entry_price ± profit_taking_spread) → 止盈执行(CLOSE)
    → 止损提案(entry_price ± stop_loss_spread ± slippage_buffer) → 止损续期? → 止损执行(CLOSE)
```

**盈利逻辑**：开仓赚 spread，有仓位后通过止盈/止损限价单平仓锁定利润或限制亏损
**最大亏损场景**：止损单为限价单（非市价单），在闪崩/跳空时可能无法成交导致亏损超过 stop_loss_spread
**不适用场景**：极端行情导致限价止损单无法成交的场景（交易所故障/流动性枯竭）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | spread 收益 + 止盈/止损机制，但止盈仅在仓位浮盈时挂单 |
| 风险可控性 | 3 | 有止损/止盈/价格带/最小价差，止损单定期续期，但限价止损单在跳空时不保证成交 |
| 适用行情广度 | 3 | 震荡赚 spread，趋势靠止损保护，三种行情均可运行但效果差异大 |
| 资金效率 | 3 | 永续合约杠杆放大资金效率，但仓位占用保证金 |
| 实操门槛 | 3 | 需配置 leverage/止盈止损 spread/仓位模式，参数较多但有合理默认值 |
| 组合灵活性 | 2 | 仓位模式限制（OneWay 仅单仓位），与其他策略共用同一合约困难 |
| 代码成熟度 | 4 | 止损续期/仓位模式切换/预算约束完备，但已知止损为限价单的缺陷 |
| **综合** | **3.0** | |

---

### 4. Cross Exchange Market Making (XEMM)

**动作行为**：
在 Maker 交易所挂限价单，当 Maker 单成交后立即在 Taker 交易所对冲。Maker 价格基于 Taker 对冲价减去 min_profitability 计算。每个 tick 检查现有订单是否仍盈利、余额是否充足、价格是否偏移，不满足则取消重挂。Taker 侧用限价单对冲，失败时通过 handle_unfilled_taker_order 触发无限重试。

**流程图**：
```
tick() → main() → process_market_pair()
  → 遍历活跃maker单 → 检查盈利性 → 检查余额 → 检查价格偏移
  → (无需活跃单时) 计算做市价格/大小 → 挂maker单
  Maker成交 → hedge_filled_maker_order() → check_and_hedge_orders()
  → 计算对冲量/价(含滑点缓冲) → 挂taker限价单
  Taker失败/取消/过期 → handle_unfilled_taker_order() → 无限重试
```

**盈利逻辑**：Maker 侧吃 spread + 跨所价差，Taker 侧对冲锁定利润
**最大亏损场景**：Taker 对冲无限重试期间价格大幅偏移；Taker 余额不足导致单腿暴露
**不适用场景**：Taker 交易所流动性极差或网络不稳定

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 理论上每笔对冲锁定利润，但实际依赖 Taker 成交质量 |
| 风险可控性 | 1 | 无止损/止盈、Taker 失败无限重试、单腿暴露风险高 |
| 适用行情广度 | 2 | 跨所价差存在时有效，趋势行情下价差消失 |
| 资金效率 | 2 | 需在两个交易所同时持有资金，Maker 侧资金部分闲置 |
| 实操门槛 | 2 | 需配置双交易所连接器、转换率、min_profitability 等多参数 |
| 组合灵活性 | 2 | 占用两个交易所的资金和连接 |
| 代码成熟度 | 4 | 支持 Gateway/AMM 对冲、转换率、anti-hysteresis，但 Taker 无限重试是已知缺陷 |
| **综合** | **2.3** | |

---

### 5. Cross Exchange Mining

**动作行为**：
XEMM 的增强变体，增加三项自适应能力：(1) 3σ 波动率动态调整 min_profitability；(2) 历史绩效自适应；(3) 自动余额再平衡。目标是在保持 XEMM 基础的同时提升鲁棒性。

**流程图**：
```
tick() → XEMM基础流程 → 波动率计算(3σ) → 动态调整min_profitability
  → 历史绩效评估 → 自适应参数调整
  Maker成交 → 对冲(同XEMM) → 检查余额平衡 → 需再平衡? → 执行跨所转账
```

**盈利逻辑**：同 XEMM（跨所价差利润），波动率自适应在高波动时扩大利润率要求
**最大亏损场景**：同 XEMM 的 Taker 对冲失败风险；自动余额再平衡可能因转账延迟导致资金在途时无法对冲
**不适用场景**：跨所转账不支持或延迟过高的交易对

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 波动率自适应提高高波动时的利润率，但核心盈利逻辑同 XEMM |
| 风险可控性 | 2 | 在 XEMM 基础上增加波动率自适应，但 Taker 无限重试和单腿暴露风险仍在 |
| 适用行情广度 | 2 | 波动率自适应扩展了高波动行情的适用性，但趋势行情下价差消失的根本问题未解 |
| 资金效率 | 3 | 自动余额再平衡减少资金闲置，优于 XEMM |
| 实操门槛 | 1 | 在 XEMM 基础上还需配置波动率窗口/再平衡参数，门槛最高 |
| 组合灵活性 | 2 | 同 XEMM，占用双交易所资源 |
| 代码成熟度 | 3 | XEMM 变体但自适应逻辑增加复杂度，历史绩效模块边界情况待完善 |
| **综合** | **2.3** | |

---

### 6. AMM Arbitrage

**动作行为**：
在两个市场间扫描套利机会，计算两个方向的利润率，过滤利润率 ≥ min_profitability 的提案，应用滑点缓冲和预算约束后执行。支持并发（同时提交两侧）或顺序提交（等首单成交再提交次单）。首单失败直接放弃整个套利提案。

**流程图**：
```
tick() → create_arb_proposals() → 过滤 profit >= min_profitability
  → apply_slippage_buffers() → apply_budget_constraint()
  → execute_arb_proposals()
  ├─ concurrent=True → 同时提交两侧订单 → 等待全部完成
  └─ concurrent=False → 提交首单 → 等待成交 → 首单失败? → 放弃 : 提交次单
```

**盈利逻辑**：跨市场价差 > 手续费 + 滑点 时锁定套利利润
**最大亏损场景**：顺序提交模式下首单成交但次单失败，导致单腿暴露
**不适用场景**：两市场价差持续低于 min_profitability + 手续费（无套利窗口）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 套利逻辑自洽，利润在提交时即计算，但依赖实际成交价 |
| 风险可控性 | 2 | 有 min_profitability/滑点缓冲/预算约束，但顺序提交单腿暴露、无重试 |
| 适用行情广度 | 1 | 仅在价差窗口存在时有效，窗口出现频率不可控 |
| 资金效率 | 2 | 大量时间等待套利窗口，资金闲置 |
| 实操门槛 | 3 | 需配置双市场/min_profitability/滑点缓冲，参数较少 |
| 组合灵活性 | 2 | 占用双市场资金，但套利执行短暂可与其他策略时分复用 |
| 代码成熟度 | 4 | 支持 CEX/DEX/AMM/Gateway 多种连接器、并发/顺序双模式 |
| **综合** | **2.4** | |

---

### 7. Spot-Perpetual Arbitrage

**动作行为**：
在现货和永续合约间执行套利，采用四阶段状态机：Closed → Opening → Opened → Closing → Closed。开仓条件：价差 ≥ min_opening_arbitrage_pct；平仓条件：价差收敛至 min_closing_arbitrage_pct。每次平仓后等待 next_arbitrage_opening_delay 再开下一轮。

**流程图**：
```
tick() → update_strategy_state()
  → Closed? → 过滤 profit >= min_opening_arbitrage_pct → 并发执行开仓 → Opening
  → Opening? → 等待双单成交 → Opened
  → Opened? → 过滤 profit >= min_closing_arbitrage_pct → 并发执行平仓 → Closing
  → Closing? → 等待双单成交+仓位归零 → Closed → 延迟等待
```

**盈利逻辑**：开仓时锁定现货-永续价差，平仓时价差收敛释放利润
**最大亏损场景**：开仓后价差非收敛而是扩大，且永续合约持仓产生资金费率成本
**不适用场景**：现货-永续价差持续不存在或持续扩大的行情

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 价差收敛时利润确定，但资金费率可能侵蚀利润 |
| 风险可控性 | 2 | 有开仓/平仓阈值/滑点缓冲/延迟保护，但无止损、无订单重试、单腿风险 |
| 适用行情广度 | 2 | 仅在价差均值回归时有效 |
| 资金效率 | 2 | 持仓期间资金锁定，且需等待延迟才能开下一轮 |
| 实操门槛 | 3 | 需配置开仓/平仓阈值/杠杆/滑点，参数较少且逻辑清晰 |
| 组合灵活性 | 2 | 持仓占用现货和永续资金，状态机限制同时仅一轮套利 |
| 代码成熟度 | 4 | 状态机设计清晰，仓位模式自动切换，预算约束完备 |
| **综合** | **2.6** | |

---

### 8. Liquidity Mining

**动作行为**：
在单一交易所的多个交易对上同时做市，按波动率自适应调整 spread（volatility × volatility_to_spread_multiplier，取较大者，上限 max_spread）。按交易对均分组合价值分配买卖预算，库存偏斜根据预算余额调整买卖单量。集成 Parrot 奖励平台查询挖矿收益。

**流程图**：
```
tick() → update_mid_prices() → update_volatility()
  → create_base_proposals() [每个交易对]
    → spread = max(基础spread, 波动率×乘子), 上限max_spread
  → apply_inventory_skew() → apply_budget_constraint()
  → cancel_active_orders() [超龄或超容差] → execute_orders_proposal()
  → 成交回调 → 更新买卖预算
```

**盈利逻辑**：多交易对 spread 收益 + Parrot 挖矿奖励；波动率自适应在高波动时自动加宽 spread
**最大亏损场景**：多交易对同时遭遇单边行情，库存全面偏移
**不适用场景**：所有交易对同时进入趋势行情（多交易对非分散风险而是放大方向性暴露）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | spread 收益 + 挖矿奖励双重收入，但多交易对同时偏移时亏损放大 |
| 风险可控性 | 2 | 有库存偏斜/波动率自适应/max_spread/max_order_age，但无止损、多交易对风险叠加 |
| 适用行情广度 | 2 | 震荡行情赚 spread + 奖励，趋势行情下多交易对库存全面偏移 |
| 资金效率 | 3 | 组合价值均分到多交易对，资金在多对间分配运作 |
| 实操门槛 | 3 | 需配置多交易对/波动率参数/Parrot 集成，但 spread 自适应降低调参需求 |
| 组合灵活性 | 2 | 多交易对占用单一交易所大量资金 |
| 代码成熟度 | 4 | 预算分配/波动率计算/Parrot 集成/空订单簿处理完备 |
| **综合** | **2.7** | |

---

### 9. Hedge

**动作行为**：
按固定间隔（hedge_interval）计算需要对冲的数量/价值，在目标市场执行对冲订单。两种模式：按数量对冲（hedge_by_amount）— 按相同 base asset 汇总各市场持有量；按价值对冲（hedge_by_value）— 按价值汇总所有市场持有量。对冲价格 = 中间价 × (1 ± slippage)，低于 min_trade_size 不执行。支持现货和永续对冲。

**流程图**：
```
tick() → 检查/取消超龄订单 → 到达hedge_interval?
  → hedge() → hedge_by_value() 或 hedge_by_amount()
    → 计算对冲方向和数量/价值 → 计算对冲价格(含滑点)
    → 获取订单候选(预算约束) → place_orders()
  永续模式 → 检查是否有对向仓位可平 → 先平仓再开仓(Hedge模式)
```

**盈利逻辑**：非盈利策略，目标是减少方向性风险敞口；通过 hedge_ratio 控制对冲比例
**最大亏损场景**：对冲延迟期间标的价格大幅波动导致对冲价格偏移；hedge_ratio < 1 时未对冲部分持续暴露
**不适用场景**：作为独立策略无盈利场景，必须与产生敞口的策略配合使用

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 1 | 非盈利策略，对冲本身产生成本，仅减少风险不产生收益 |
| 风险可控性 | 2 | 有滑点缓冲/min_trade_size/hedge_ratio/超龄取消，但无止损、对冲延迟期间敞口暴露 |
| 适用行情广度 | 3 | 所有行情下均可执行对冲，但行情剧烈时对冲质量下降 |
| 资金效率 | 2 | 对冲占用目标市场资金，且对冲本身不产生收益 |
| 实操门槛 | 3 | 需配置对冲市场/hedge_ratio/slippage/interval，逻辑清晰 |
| 组合灵活性 | 5 | 设计为辅助策略，可与所有产生敞口的策略组合使用 |
| 代码成熟度 | 4 | 支持现货/永续双模式、数量/价值双模式、Hedge/OneWay 仓位模式 |
| **综合** | **2.9** | |

---

### V1 策略综合排名

| 排名 | 策略 | 综合分 | 核心优势 | 核心风险 |
|:---:|------|:---:|------|------|
| 1 | Pure MM | 3.0 | 生产级成熟度，功能完备 | 无止损，库存暴露 |
| 1 | Avellaneda MM | 3.0 | 理论最优定价，库存偏斜内嵌 | 参数调试门槛高 |
| 1 | Perpetual MM | 3.0 | 止损止盈机制，杠杆增效 | 限价止损不保证成交 |
| 4 | Hedge | 2.9 | 组合灵活性最高 | 非盈利策略 |
| 5 | Liquidity Mining | 2.7 | 挖矿奖励+波动率自适应 | 多交易对风险叠加 |
| 6 | Spot-Perp Arb | 2.6 | 状态机清晰，价差收敛逻辑确定 | 无止损，资金费率侵蚀 |
| 7 | AMM Arb | 2.4 | 多市场类型支持 | 窗口稀少，单腿暴露 |
| 8 | XEMM | 2.3 | 跨所对冲逻辑自洽 | Taker无限重试，单腿风险 |
| 8 | Cross Ex Mining | 2.3 | XEMM+自适应增强 | 复杂度最高，门槛最高 |

---

## 三、V2 执行器评估

### 1. PositionExecutor

**动作行为**：
开仓后持续监控三重屏障（止损/止盈/时间限制/追踪止损），任一屏障触发即市价平仓。开仓支持 MARKET/LIMIT_MAKER 两种模式，止盈支持 LIMIT_MAKER 限价单以减少滑点。追踪止损在 PnL 达到 activation_price 后激活，回撤超过 trailing_delta 时触发平仓。部分成交时精确匹配平仓量。

**流程图**：
```
NOT_STARTED → place_open_order() → RUNNING
  → control_barriers() 每 tick 检查:
    ├─ net_pnl_pct <= -stop_loss → 市价平仓(STOP_LOSS)
    ├─ trailing_stop 激活且回撤 > delta → 市价平仓(TRAILING_STOP)
    ├─ net_pnl_pct >= take_profit → 限价/市价平仓(TAKE_PROFIT)
    ├─ 超时 → 市价平仓(TIME_LIMIT)
    └─ 无触发 → 继续监控
  → SHUTTING_DOWN → place_close_order() → TERMINATED
```

**盈利逻辑**：由外部 Controller 决定开仓方向和时机，执行器负责风控平仓；take_profit 锁定利润
**最大亏损场景**：开仓后价格瞬间跳空超过 stop_loss，市价平仓实际亏损超过预期
**不适用场景**：无方向性信号的纯执行场景（需 Controller 驱动）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 盈利由 Controller 信号决定，执行器仅负责风控执行 |
| 风险可控性 | 5 | 三重屏障完整（SL/TP/TL/TS），max_retries=10，部分成交精确匹配 |
| 适用行情广度 | 4 | 不依赖行情类型，由 Controller 信号决定适用性 |
| 资金效率 | 3 | 开仓占用资金至平仓，时间限制避免长期占用 |
| 实操门槛 | 4 | 参数直觉化（止损%/止盈%/时间限制），TripleBarrierConfig 统一配置 |
| 组合灵活性 | 5 | V2 架构核心执行单元，可与所有 Controller 组合 |
| 代码成熟度 | 5 | 797 行核心代码，状态机清晰，部分成交/续期/EARLY_STOP 均处理完备 |
| **综合** | **4.1** | |

---

### 2. OrderExecutor

**动作行为**：
简单下单执行器，支持 MARKET/LIMIT/LIMIT_MAKER/LIMIT_CHASER 四种模式。LIMIT_CHASER 模式持续跟踪市场价格：当市价偏离挂单价超过 refresh_threshold 比率时，自动取消旧单并在新价格重新挂单。无内置止损/止盈，依赖外部 Controller 管理生命周期。

**流程图**：
```
NOT_STARTED → place_open_order() → RUNNING
  → LIMIT_CHASER 模式:
    → 每 tick 检查: |current_price - order.price| / current_price > refresh_threshold?
      ├─ 是 → cancel_order() → place_open_order(新价格) → 继续追踪
      └─ 否 → 保持挂单
  → SHUTTING_DOWN → 取消或转 POSITION_HOLD → TERMINATED
```

**盈利逻辑**：无独立盈利逻辑，作为辅助执行单元由 Controller 驱动；LIMIT_CHASER 通过跟价提高限价单成交率
**最大亏损场景**：无止损/止盈，仓位完全暴露于市场波动；Controller 未及时停止时亏损无限
**不适用场景**：需要独立风控的场景（必须配合 Controller 使用）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 2 | 无独立盈利逻辑，完全依赖 Controller |
| 风险可控性 | 2 | 仅余额验证，无止损/止盈/时间限制；LIMIT_CHASER 跟价但不限亏 |
| 适用行情广度 | 4 | 不依赖行情类型，由 Controller 决定 |
| 资金效率 | 3 | LIMIT_CHASER 持续运作，其他模式单次执行 |
| 实操门槛 | 4 | 参数少（order_type/distance/refresh_threshold），配置简单 |
| 组合灵活性 | 5 | 可与所有 Controller 组合，LIMIT_CHASER 是跟价场景的唯一选择 |
| 代码成熟度 | 4 | 403 行，LIMIT_CHASER 逻辑清晰，renew_order() 取消+重挂原子操作 |
| **综合** | **3.4** | |

---

### 3. DCAExecutor

**动作行为**：
分批建仓执行器，按 prices + amounts_quote 列表逐级下单。TAKER 模式：市价单在价格触及各级 activation_bounds 时逐级触发；MAKER 模式：限价单同时挂出，但止损需所有开仓单完成后才激活。全部成交后激活三重屏障管理整体仓位。支持 trailing_stop 和动态止损/止盈。

**流程图**：
```
NOT_STARTED → RUNNING
  → control_open_order_process():
    → TAKER模式: 价格触及 activation_bounds[level]? → 市价下单 → 下一级
    → MAKER模式: 同时挂出所有限价单 → 等待逐级成交
  → 全部成交 → control_barriers():
    ├─ stop_loss → 整体市价平仓(STOP_LOSS)
    ├─ trailing_stop → 追踪止损平仓(TRAILING_STOP)
    ├─ take_profit → 止盈平仓(TAKE_PROFIT)
    └─ time_limit → 超时平仓(TIME_LIMIT)
  → SHUTTING_DOWN → TERMINATED
```

**盈利逻辑**：分批建仓降低平均成本（逢低买入/逢高卖出），价格回归时整体盈利
**最大亏损场景**：多级全部成交后价格继续反向运动，累计持仓亏损超过 stop_loss；MAKER 模式下止损延迟激活期间亏损扩大
**不适用场景**：强趋势行情（DCA 逆势加仓，趋势中累积大量仓位）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 均值回归行情中分批建仓降低成本，盈利确定性高于单次入场 |
| 风险可控性 | 4 | 三重屏障 + activation_bounds 逐级激活 + max_retries=15；MAKER 模式止损延迟是已知限制 |
| 适用行情广度 | 2 | 仅均值回归/震荡行情有效，趋势中逆势加仓风险大 |
| 资金效率 | 3 | 分级建仓渐进投入，未成交级别资金闲置 |
| 实操门槛 | 3 | 需配置 prices/amounts_quote/activation_bounds 多级参数，但逻辑直觉化 |
| 组合灵活性 | 4 | 可与方向性 Controller 组合（DMan V3 内置使用） |
| 代码成熟度 | 4 | 546 行，TAKER/MAKER 双模式完备，activation_bounds 逻辑清晰 |
| **综合** | **3.4** | |

---

### 4. GridExecutor

**动作行为**：
在 start_price → end_price 间线性生成多级 GridLevel，每级独立循环：挂开仓限价单 → 成交后挂平仓限价单（take_profit = step 或自定义）→ 平仓完成 → 重置为未激活 → 重新挂开仓单。整体受 TripleBarrierConfig 保护（止损/时间限制/追踪止损），limit_price 作为价格边界触发整体平仓。

**流程图**：
```
NOT_STARTED → RUNNING
  → update_grid_levels(): 生成/更新网格层级
  → 每级 GridLevel 独立循环:
    → NOT_ACTIVE → 挂开仓限价单 → OPEN_ORDER_PLACED
    → 成交 → OPEN_ORDER_FILLED → 挂平仓限价单 → CLOSE_ORDER_PLACED
    → 平仓成交 → COMPLETE → reset_level → NOT_ACTIVE (循环)
  → control_triple_barrier():
    ├─ stop_loss → 整体市价平仓(STOP_LOSS)
    ├─ limit_price 触发 → POSITION_HOLD 或 STOP_LOSS
    ├─ time_limit → 超时平仓(TIME_LIMIT)
    └─ trailing_stop → 追踪止损平仓(TRAILING_STOP)
  → SHUTTING_DOWN → TERMINATED
```

**盈利逻辑**：震荡行情中价格在网格区间内反复穿越各级，每次穿越完成一次开仓→平仓循环赚取 step 价差
**最大亏损场景**：价格突破网格区间（超过 limit_price），所有开仓级同时持仓且方向一致，整体止损金额巨大
**不适用场景**：单边趋势行情（价格穿越网格后不再回归，所有级同向持仓）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 4 | 震荡行情中每次穿越赚取 step 价差，收益逻辑清晰且稳定 |
| 风险可控性 | 4 | 三重屏障 + limit_price 边界 + max_open_orders 限制同时持仓级数 + coerce_tp_to_step |
| 适用行情广度 | 2 | 仅震荡行情有效，趋势行情中网格被单边穿越 |
| 资金效率 | 4 | 多级并行运作，资金在网格区间内持续流转 |
| 实操门槛 | 3 | 需配置 start_price/end_price/total_amount_quote，网格参数直觉化 |
| 组合灵活性 | 3 | 可与方向性 Controller 组合，但网格区间固定限制了灵活性 |
| 代码成熟度 | 4 | 942 行最复杂执行器，GridLevel 子状态机清晰，reset_level 循环完备 |
| **综合** | **3.4** | |

---

### 5. XEMMExecutor

**动作行为**：
Maker 端挂限价单，成交后 Taker 端市价对冲。盈利性偏离 [min_profitability, max_profitability] 时取消 Maker 单重挂。支持跨 quote 资产换算（USDT/USDC）和可互换代币识别（WETH/ETH）。Taker 失败后自动重下，最多重试 10 次。

**流程图**：
```
NOT_STARTED → RUNNING
  → update_prices_and_tx_costs() → control_maker_order():
    → 无活跃maker单 → create_maker_order(基于target_profitability计算价格)
    → 有活跃maker单 → 盈利性在[min,max]内? → 保持 : 取消重挂
  → Maker成交 → place_taker_order() → SHUTTING_DOWN
  → Taker失败 → 重下(最多10次) → 超限 → FAILED
  → SHUTTING_DOWN → 等待双方订单完成 → TERMINATED
```

**盈利逻辑**：跨所价差利润 = taker_price × (1 - target_profitability) - maker_price - tx_cost
**最大亏损场景**：Taker 对冲失败 10 次后标记 FAILED，Maker 端持仓完全暴露无人管理
**不适用场景**：Taker 交易所流动性极差或频繁故障

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 跨所价差利润逻辑自洽，但依赖 Taker 成交质量 |
| 风险可控性 | 2 | min/max_profitability 控制盈利窗口，但无止损/止盈/时间限制，单腿暴露风险 |
| 适用行情广度 | 2 | 仅跨所价差存在时有效 |
| 资金效率 | 2 | 双交易所资金占用，大量时间在盈利窗口外未成交 |
| 实操门槛 | 3 | 需配置双交易所/min/max_profitability，参数较少 |
| 组合灵活性 | 3 | 可与 XEMM Multiple Levels Controller 组合，但占用双交易所资源 |
| 代码成熟度 | 4 | 371 行，跨 quote 换算和可互换代币识别逻辑完备 |
| **综合** | **2.7** | |

---

### 6. ArbitrageExecutor

**动作行为**：
同时两个市场下市价单套利（买入低价市场 + 卖出高价市场）。盈利性 = (sell_price - buy_price) / buy_price - tx_cost。支持不同 quote 资产换算（rate_oracle）和 AMM gas 费纳入成本。失败后重试同方向单，最多 3 次。

**流程图**：
```
NOT_STARTED → RUNNING
  → update_trade_pnl_pct() + update_tx_cost() → 计算 _current_profitability
  → profitability > min_profitability?
    ├─ 是 → execute_arbitrage():
    │     → place_buy_arbitrage_order() + place_sell_arbitrage_order()
    │     → SHUTTING_DOWN → 等待双方完成 → COMPLETED
    └─ 否 → 继续监控
  → 任一端失败 → 重试同端(最多3次) → 超限 → FAILED
```

**盈利逻辑**：双边价差利润 = (sell_price - buy_price) / buy_price - tx_cost
**最大亏损场景**：一端成交另一端失败 3 次后 FAILED，成交端持仓完全暴露，无止损管理
**不适用场景**：两市场价差持续低于 min_profitability + tx_cost

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 套利逻辑自洽，利润在执行时即确定 |
| 风险可控性 | 1 | 无止损/止盈/时间限制，重试仅 3 次（最低），单腿暴露风险最高 |
| 适用行情广度 | 1 | 仅价差窗口存在时有效 |
| 资金效率 | 2 | 双市场资金占用，窗口稀少时大量闲置 |
| 实操门槛 | 3 | 参数少（buying/selling_market/order_amount/min_profitability） |
| 组合灵活性 | 3 | 可与 Controller 组合，但占用双市场资源 |
| 代码成熟度 | 4 | 357 行，rate_oracle 换算和 gas 费纳入完备 |
| **综合** | **2.4** | |

---

### 7. TWAPExecutor

**动作行为**：
在 total_duration 内每隔 order_interval 下单，金额均分（动态调整：剩余金额/剩余单数）。MAKER 模式使用 limit_order_buffer 偏移价格 + order_resubmission_time 超时刷新。全部成交后 COMPLETED。

**流程图**：
```
NOT_STARTED → RUNNING
  → evaluate_create_order(): 到达下一时间槽?
    → 计算单笔金额 = 剩余金额 / 剩余单数 → create_order()
  → MAKER模式: evaluate_refresh_orders(): 超时未成交? → 取消重挂
  → 全部成交 → SHUTTING_DOWN → COMPLETED
  → 订单失败 → 重置时间槽重试(最多15次) → 超限 → FAILED
```

**盈利逻辑**：非盈利执行器，目标是拆分大额订单减少市场冲击；MAKER 模式可赚取 maker rebate
**最大亏损场景**：执行期间价格大幅波动，MAKER 模式限价单可能部分未成交导致执行不完整
**不适用场景**：需要快速全部成交的场景（TWAP 故意延迟执行）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 2 | 非盈利执行器，MAKER 模式可赚 rebate 但非主要目标 |
| 风险可控性 | 2 | 仅余额验证和最小下单量，无止损/止盈；执行期间价格偏移风险 |
| 适用行情广度 | 4 | 不依赖行情类型，所有行情下均可执行拆单 |
| 资金效率 | 3 | 渐进投入，MAKER 模式资金利用率低于 TAKER |
| 实操门槛 | 5 | 参数直觉化（total_duration/order_interval），开箱即用 |
| 组合灵活性 | 5 | 纯执行工具，可与任何策略组合用于拆单 |
| 代码成熟度 | 4 | 287 行，时间槽管理和动态金额调整逻辑清晰 |
| **综合** | **3.6** | |

---

### 8. LPExecutor

**动作行为**：
通过 Gateway 管理 DEX 集中流动性（CLMM）仓位。创建仓位（add_liquidity）→ 监控价格位置（IN_RANGE/OUT_OF_RANGE）→ 价格出范围时根据 upper/lower_limit_price 决定平仓（close_position）→ 可选执行 close-out swap 将净 base 换回原始 quote。非事件驱动，直接 await Gateway 调用。

**流程图**：
```
NOT_ACTIVE → OPENING → _create_position() → IN_RANGE
  → 价格在范围内 → 静默监控(赚取手续费)
  → 价格出范围 → OUT_OF_RANGE
    → 超出 limit_price? → CLOSING → _close_position()
      → keep_position=False → SWAPPING → _execute_closeout_swap()
      → COMPLETE
    → 未超 limit_price → 继续监控
  → Gateway 调用失败 → FAILED
```

**盈利逻辑**：DEX LP 手续费收入（交易量 × fee_tier），价格在范围内时持续赚取
**最大亏损场景**：无常损失（IL）— 价格大幅单边移动导致 LP 仓位价值低于单纯持有；close-out swap 滑点
**不适用场景**：价格剧烈波动的交易对（IL 超过手续费收入）

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | LP 手续费收入稳定但受 IL 侵蚀，净收益取决于波动率 |
| 风险可控性 | 3 | upper/lower_limit_price 价格限位 + max_retries=10；但无常损失无法止损，仅能平仓退出 |
| 适用行情广度 | 2 | 仅窄幅震荡行情有效（价格在范围内赚取手续费） |
| 资金效率 | 3 | 资金在范围内持续赚取手续费，出范围后需平仓重配 |
| 实操门槛 | 2 | 需配置 Gateway/链/DEX/价格范围/limit_price，链上操作门槛高 |
| 组合灵活性 | 2 | 独立于 CEX 策略运行，但 Gateway 资源独占 |
| 代码成熟度 | 3 | 1109 行最复杂，但依赖 Gateway 稳定性；链上交易确认延迟可能导致状态不同步 |
| **综合** | **2.6** | |

---

### V2 执行器综合排名

| 排名 | 执行器 | 综合分 | 核心优势 | 核心风险 |
|:---:|--------|:---:|------|------|
| 1 | PositionExecutor | 4.1 | 三重屏障完整，V2 架构核心 | 跳空时实际亏损超预期 |
| 2 | TWAPExecutor | 3.6 | 开箱即用，组合灵活性高 | 非盈利，执行期间价格偏移 |
| 3 | OrderExecutor | 3.4 | LIMIT_CHASER 跟价，组合灵活 | 无风控，完全依赖 Controller |
| 3 | DCAExecutor | 3.4 | 分批建仓+三重屏障 | 趋势中逆势加仓风险 |
| 3 | GridExecutor | 3.4 | 震荡收益稳定，资金效率高 | 趋势中网格被单边穿越 |
| 6 | XEMMExecutor | 2.7 | 跨所价差逻辑自洽 | 单腿暴露，无三重屏障 |
| 7 | LPExecutor | 2.6 | DEX LP 手续费收入 | 无常损失无法止损 |
| 8 | ArbitrageExecutor | 2.4 | 套利逻辑自洽 | 单腿风险最高，重试仅3次 |

---

## 四、V2 控制器评估

### 1. PMM Simple

**动作行为**：
围绕中间价以固定偏移挂买卖限价单，每个价位创建一个 PositionExecutor，由三重屏障管理。到期未成交的执行器自动刷新，止损后进入冷却期。现货模式下自动进行底仓再平衡。

**流程图**：
```
mid_price → 计算各档位价格(buy: mid×(1-spread), sell: mid×(1+spread))
    → 检查活跃执行器 + 冷却期止损执行器
    → 缺失档位 → 创建 PositionExecutor(含 TripleBarrierConfig)
    → 已到期未成交 → StopExecutor + 重建
    → 现货底仓偏差 > threshold → 创建再平衡市价单
```

**盈利逻辑**：做市价差收入 — 买单在低价成交、卖单在高价成交，赚取买卖价差；每档独立由三重屏障保护
**最大亏损场景**：单边行情中所有买入档位全部成交，价格持续下跌触发各档 stop_loss，累计亏损 = 档位数 × 单档止损额
**不适用场景**：剧烈单边趋势行情 — 价差持续被击穿，做市仓位反复止损

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 4 | 价差收入逻辑自洽，震荡行情中收益稳定 |
| 风险可控性 | 4 | 三重屏障完整+冷却期+max_executors_per_side；现货自动再平衡 |
| 适用行情广度 | 3 | 震荡行情最优，趋势行情中固定价差持续被单边击穿 |
| 资金效率 | 3 | 多档位并行运作，远档位在窄幅行情中长期闲置 |
| 实操门槛 | 5 | 参数直觉化，继承基类默认值即可运行，开箱即用 |
| 组合灵活性 | 4 | 可与方向性策略叠加，可与 DCA 执行器配合 |
| 代码成熟度 | 4 | 代码极简（32 行），逻辑完全委托基类 |
| **综合** | **3.9** | |

---

### 2. PMM Dynamic

**动作行为**：
在 PMM Simple 基础上引入 NATR（归一化平均真实波幅）和 MACD 动态调整。NATR 作为价差乘数（波动大时拉宽价差），MACD histogram 偏移参考价格（趋势向上时参考价上移）。价差以 NATR 为单位定义（如 buy_spreads=[1,2,4] 表示距参考价 1/2/4 倍波动率）。

**流程图**：
```
K线数据 → NATR(波动率) + MACD(趋势)
    → price_multiplier = (0.5×macd_signal + 0.5×macdh_signal) × NATR/2
    → reference_price = close × (1 + price_multiplier)
    → spread_multiplier = NATR
    → 各档实际价差 = 配置价差 × NATR
    → 创建 PositionExecutor(含 TripleBarrierConfig)
```

**盈利逻辑**：动态做市 — MACD 趋势偏移使报价顺趋势方向倾斜；NATR 价差自适应使高波动时赚更宽价差
**最大亏损场景**：MACD 趋势判断失误（假突破），参考价偏移方向错误导致逆势挂单密集成交
**不适用场景**：快速 V 型反转 — MACD 滞后性导致偏移方向刚确立即反转

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 4 | 动态价差+趋势偏移提高震荡和温和趋势行情的收益 |
| 风险可控性 | 4 | 继承基类三重屏障+冷却期；NATR 自适应在高波动时拉宽价差 |
| 适用行情广度 | 4 | 震荡+温和趋势均适用，优于 PMM Simple |
| 资金效率 | 3 | 远档位在低波动时闲置；波动率自适应缓解了此问题 |
| 实操门槛 | 3 | 需理解 NATR/MACD 含义和价差以波动率为单位的定义 |
| 组合灵活性 | 4 | 与 PMM Simple 相同的组合能力，动态特性使其与方向策略互补性更好 |
| 代码成熟度 | 4 | 指标计算使用 pandas_ta 标准库；macd_signal 标准化处理缺乏调优依据 |
| **综合** | **3.7** | |

---

### 3. XEMM Multiple Levels

**动作行为**：
跨交易所做市 — 在 Maker 交易所挂限价单，成交后在 Taker 交易所对冲。支持多档位（每档独立目标盈利），max_executors_imbalance 控制买卖不平衡防止单腿暴露。

**流程图**：
```
Maker MidPrice → 计算各档位目标盈利 → 检查活跃执行器
    → 买单侧: 无活跃 & imbalance < max_imbalance → XEMMExecutor(maker买入, taker卖出)
    → 卖单侧: 无活跃 & imbalance > -max_imbalance → XEMMExecutor(maker卖出, taker买入)
    → 已停止且已成交的执行器 → 计算 imbalance → 限制对侧新建
```

**盈利逻辑**：跨所价差套利 — Maker 成交后 Taker 市价对冲锁定价差利润
**最大亏损场景**：单腿风险 — Maker 单成交但 Taker 对冲失败，持有一侧裸头寸；max_executors_imbalance 仅限新建但不对已有单腿提供止损
**不适用场景**：两所价差极小或手续费吃掉全部利润

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 跨所套利逻辑清晰，但盈利空间受限于两所价差减去双倍手续费 |
| 风险可控性 | 2 | max_executors_imbalance 仅控制新建，不对单腿暴露提供止损 |
| 适用行情广度 | 2 | 仅在两所存在持续价差时有效 |
| 资金效率 | 2 | 资金分配在两侧交易所各50%，大量时间未成交 |
| 实操门槛 | 2 | 需配置两个交易所账户、理解盈利窗口参数 |
| 组合灵活性 | 2 | 独立运行，需要两个交易所同时连接 |
| 代码成熟度 | 3 | 逻辑完整，有 gas token 自动获取和 AMM 支持 |
| **综合** | **2.3** | |

---

### 4. Bollinger V2

**动作行为**：
基于布林带 %B 指标的方向性交易。%B < 0（价格跌破下轨）→ 做多信号；%B > 1（价格突破上轨）→ 做空信号。信号触发后创建 PositionExecutor，由三重屏障管理。

**流程图**：
```
K线数据 → Bollinger Band(upper/middle/lower) + %B
    → %B < bb_long_threshold(0) → signal=1(做多)
    → %B > bb_short_threshold(1) → signal=-1(做空)
    → 其余 → signal=0(无操作)
    → can_create_executor? → PositionExecutor(含 TripleBarrierConfig)
```

**盈利逻辑**：均值回归 — 价格偏离布林带轨道后预期回归中轨，在极端偏离时入场
**最大亏损场景**：趋势突破行情中价格持续沿轨道外侧运行，均值回归失败，连续止损
**不适用场景**：强趋势行情 — 布林带开口扩大伴随持续突破，均值回归失效

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 均值回归在震荡/弱趋势行情有效，强趋势中反复逆势亏损 |
| 风险可控性 | 3 | 继承基类三重屏障+冷却期+max_executors_per_side；无趋势过滤 |
| 适用行情广度 | 2 | 仅均值回归行情有效 |
| 资金效率 | 3 | 信号触发时资金运作，无信号时闲置 |
| 实操门槛 | 4 | 布林带是广为人知的指标，参数含义直觉化 |
| 组合灵活性 | 3 | 可与做市策略组合，但与其它方向策略信号可能冲突 |
| 代码成熟度 | 4 | 代码清晰，使用 TA-Lib 双重验证 |
| **综合** | **3.1** | |

---

### 5. DMan V3

**动作行为**：
基于布林带 %B 信号触发的 DCA 网格执行策略。信号触发后，DCAExecutor 以多级价差逐步建仓，动态模式下以布林带宽度（BBB）乘数作为实际价差。支持 trailing_stop、dynamic_target（止盈止损随波动率缩放）、activation_bounds（逐级激活条件）。

**流程图**：
```
K线数据 → Bollinger Band → %B
    → %B < bb_long_threshold → signal=1(做多)
    → %B > bb_short_threshold → signal=-1(做空)
    → 计算价差: dca_spreads × spread_multiplier(BB_width/200 if dynamic)
    → 计算各级价格和金额
    → DCAExecutorConfig(prices, amounts_quote, stop_loss, trailing_stop, activation_bounds)
```

**盈利逻辑**：均值回归 + DCA 分批建仓 — 在价格偏离布林带时分批入场，越偏离仓位越重，价格回归时整体盈利
**最大亏损场景**：极端单边趋势 + 高杠杆 — 多级 DCA 全部成交后价格继续反向运动，累计持仓亏损超过 stop_loss
**不适用场景**：高杠杆 + 强趋势 — DCA 多级建仓在趋势中累积大量逆势仓位

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | DCA 分批建仓在均值回归行情中盈利确定性高于单次入场 |
| 风险可控性 | 4 | DCAExecutor 含三重屏障 + activation_bounds + dynamic_target |
| 适用行情广度 | 2 | 信号逻辑与 Bollinger V2 相同，仅均值回归有效 |
| 资金效率 | 3 | DCA 分级建仓使资金渐进投入 |
| 实操门槛 | 2 | 参数多且互相关联，需深入理解 DCA 机制 |
| 组合灵活性 | 3 | DCAExecutor 限制了与做市策略的直接组合 |
| 代码成熟度 | 4 | 代码结构清晰，dynamic_target/spread_multiplier 逻辑完整 |
| **综合** | **3.0** | |

---

### 6. SuperTrend V1

**动作行为**：
基于 SuperTrend 指标的方向性交易。SuperTrend direction 翻转（-1→1 做多，1→-1 做空）且价格距 SuperTrend 线距离小于 percentage_threshold 时产生信号。创建 PositionExecutor，由三重屏障管理。

**流程图**：
```
K线数据 → SuperTrend(value, direction) + percentage_distance
    → direction=1 & percentage_distance < threshold → signal=1(做多)
    → direction=-1 & percentage_distance < threshold → signal=-1(做空)
    → can_create_executor? → PositionExecutor(含 TripleBarrierConfig)
```

**盈利逻辑**：趋势跟踪 — SuperTrend 翻转标志趋势转向，在翻转初期入场，跟随趋势获利
**最大亏损场景**：震荡行情中 SuperTrend 频繁翻转（假突破），反复止损
**不适用场景**：横盘震荡 — SuperTrend 在缺乏趋势时产生大量假信号

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 趋势行情中盈利确定性高；震荡行情中假信号频繁 |
| 风险可控性 | 3 | 继承基类三重屏障+冷却期+percentage_threshold 入场过滤 |
| 适用行情广度 | 2 | 仅趋势行情有效 |
| 资金效率 | 3 | 信号触发时资金运作，无信号时闲置 |
| 实操门槛 | 4 | SuperTrend 参数直觉化，默认参数即可用于多数场景 |
| 组合灵活性 | 3 | 趋势策略可与做市策略组合 |
| 代码成熟度 | 4 | 代码简洁（89 行），percentage_distance 过滤是好设计 |
| **综合** | **3.3** | |

---

### 7. StatArb

**动作行为**：
统计套利 — 对两个协整资产进行配对交易。使用 sklearn LinearRegression 在累积收益率上计算对冲比率（beta），计算价差 Z-Score。Z-Score > entry_threshold → 做多价差（买dominant/卖hedge）；Z-Score < -entry_threshold → 做空价差。双边各创建 PositionExecutor 以限价单报价入场。全局止盈止损（tp_global/sl_global）监控配对整体 PnL。

**流程图**：
```
两资产K线 → 累积收益率 → LinearRegression → alpha, beta(hedge_ratio)
    → spread_pct = (hedge_cum - predicted) / predicted × 100
    → Z-Score = (spread - mean) / std
    → Z-Score > entry_threshold → signal=1(买dominant/卖hedge)
    → Z-Score < -entry_threshold → signal=-1(卖dominant/买hedge)
    → 检查全局 PnL: tp_global/sl_global → 全部平仓
    → 检查仓位偏差: imbalance > max_position_deviation → 过滤偏差侧
    → 限价单报价: entry_price = best_price × (1 ± quoter_spread)
    → 创建 PositionExecutor(dominant) + PositionExecutor(hedge)
```

**盈利逻辑**：配对交易的均值回归 — 两个协整资产的价差偏离历史均值后预期回归，价差收敛时获利
**最大亏损场景**：协整关系断裂 — 两资产间的历史相关性失效，价差持续扩大不回归
**不适用场景**：低流动性/高波动小市值配对 — 单腿成交困难，对冲失败导致裸头寸

| 维度 | 评分 | 说明 |
|------|:---:|------|
| 盈利确定性 | 3 | 配对套利逻辑严谨，市场中性设计理论上降低方向风险 |
| 风险可控性 | 4 | tp_global/sl_global + max_position_deviation + max_orders 限制敞口 |
| 适用行情广度 | 3 | 牛/熊/震荡均可运行（市场中性）；但需两资产持续协整 |
| 资金效率 | 3 | 两腿各占约50%资金 |
| 实操门槛 | 1 | 需选择协整配对（需统计预分析）、配置对冲比率、理解 Z-Score 机制 |
| 组合灵活性 | 2 | 双腿占用两个交易对，难以与其他策略共享 |
| 代码成熟度 | 3 | 逻辑完整但代码较长（477行），LinearRegression 空数据时可能无返回值 |
| **综合** | **2.7** | |

---

### V2 控制器综合排名

| 排名 | 控制器 | 综合分 | 核心优势 | 核心风险 |
|:---:|--------|:---:|------|------|
| 1 | PMM Simple | 3.9 | 开箱即用，三重屏障完整 | 固定价差，趋势中反复止损 |
| 2 | PMM Dynamic | 3.7 | NATR/MACD 自适应 | MACD 假突破导致逆势挂单 |
| 3 | SuperTrend V1 | 3.3 | 趋势跟踪+入场过滤 | 震荡中假信号频繁 |
| 4 | Bollinger V2 | 3.1 | 均值回归，参数直觉化 | 强趋势中均值回归失效 |
| 5 | DMan V3 | 3.0 | DCA 分批建仓+动态止损 | 参数多且互相关联 |
| 6 | StatArb | 2.7 | 市场中性，风控层次丰富 | 实操门槛极高，协整可能失效 |
| 7 | XEMM Multi-Level | 2.3 | 跨所多档位做市 | 单腿风险，XEMMExecutor 无三重屏障 |

---

## 五、组合可行性矩阵

### V1 策略间组合

```
          PMM  Avell  PerpMM  XEMM  CrossMin  AMMArb  SpotPerp  LiqMin  Hedge
PMM        -    冲突    冲突    独立    独立     独立     独立     冲突   条件协同
Avell      冲突   -     冲突    独立    独立     独立     独立     冲突   条件协同
PerpMM     冲突  冲突    -     独立    独立     独立     独立     冲突   条件协同
XEMM       独立  独立   独立    -     冲突     冲突     冲突     独立   条件协同
CrossMin   独立  独立   独立   冲突    -       冲突     冲突     独立   条件协同
AMMArb     独立  独立   独立   冲突   冲突      -       独立     独立   条件协同
SpotPerp   独立  独立   独立   冲突   冲突     独立      -       独立   条件协同
LiqMin     冲突  冲突   冲突   独立    独立     独立     独立      -     条件协同
Hedge      条件  条件   条件   条件    条件     条件     条件     条件    -
```

**关键冲突说明**：
- **PMM/Avell/PerpMM/LiqMin 互斥**：同一交易所同一交易对不能同时运行多个做市策略（订单管理冲突）
- **XEMM/CrossMin/AMMArb/SpotPerp 互斥**：跨所策略占用双交易所资源，同时运行导致资金竞争
- **Hedge 条件协同**：需与产生敞口的策略配合，且对冲市场不能与源策略市场冲突

### V2 控制器间组合

```
          PMMS  PMMD  XEMMML  Boll  DMan  SuperT  StatArb
PMMS       -    冲突    独立    协同   独立   协同     独立
PMMD      冲突   -     独立    协同   独立   协同     独立
XEMMML    独立  独立    -     独立   独立   独立     独立
Boll      协同  协同   独立    -     冲突   冲突     独立
DMan      独立  独立   独立   冲突    -     冲突     独立
SuperT    协同  协同   独立   冲突   冲突    -       独立
StatArb   独立  独立   独立   独立   独立   独立      -
```

**关键协同说明**：
- **PMM + Bollinger/SuperTrend 协同**：做市提供流动性 + 方向信号偏移报价，增强收益
- **Bollinger/DMan/SuperTrend 互斥**：同属方向性策略，信号可能冲突导致同时开多空仓
- **XEMMML/StatArb 独立**：跨所策略与其他策略在不同交易对上独立运行

### V2 执行器间组合

```
          PosEx  OrdEx  DCAEx  GridEx  XEMMEx  ArbEx  TWAPEx  LPEx
PosEx       -    协同   协同    独立    独立    独立    独立    独立
OrdEx      协同    -    独立    独立    独立    独立    独立    独立
DCAEx      协同   独立    -     独立    独立    独立    独立    独立
GridEx     独立   独立   独立    -      冲突    冲突    独立    独立
XEMMEx     独立   独立   独立   冲突     -      冲突    独立    独立
ArbEx      独立   独立   独立   冲突    冲突     -      独立    独立
TWAPEx     独立   独立   独立   独立    独立    独立     -      独立
LPEx       独立   独立   独立   独立    独立    独立    独立     -
```

**关键说明**：
- **PositionExecutor + OrderExecutor/DCAExecutor 协同**：PositionExecutor 做主仓位管理，OrderExecutor 做辅助操作（如限价跟单），DCAExecutor 做分批建仓后由 PositionExecutor 接管风控
- **Grid/XEMM/Arbitrage 互斥**：均占用双市场资源或产生方向性仓位，同时运行导致资金竞争和仓位冲突
- **TWAPExecutor/LPExecutor 独立**：TWAP 是纯执行工具，LP 是链上操作，与其他执行器无资源冲突

### V1/V2 混用限制

| 组合 | 判定 | 原因 |
|------|------|------|
| V1 策略 + V2 控制器 | **架构冲突** | V1 和 V2 使用不同的订单管理和事件系统，不可同时运行 |
| V1 策略 + V2 执行器 | **架构冲突** | 执行器由 V2 Controller 驱动，V1 策略无法直接使用 V2 执行器 |
| 多个 V2 控制器 | **条件协同** | 需在不同交易对上运行；同一交易对仅能运行一个 Controller |

---

## 六、综合推荐

### 按场景推荐策略组合

| 场景 | 推荐组合 | 理由 | 风险提示 |
|------|---------|------|---------|
| **震荡做市** | PMM Dynamic + PositionExecutor | NATR 自适应价差 + 三重屏障，震荡行情最优 | MACD 假突破时逆势挂单 |
| **保守做市** | PMM Simple + PositionExecutor | 开箱即用，参数直觉化，风控完整 | 固定价差在波动变化时不自适应 |
| **永续合约做市** | Perpetual MM (V1) | 止损止盈+杠杆增效，永续合约专用 | 限价止损不保证成交 |
| **跨所套利** | XEMM Multi-Level + XEMMExecutor | 多档位跨所做市，盈利窗口可控 | **单腿风险高**，Taker 失败无止损 |
| **统计套利** | StatArb + PositionExecutor | 市场中性，Z-Score 信号+全局止盈止损 | 实操门槛极高，协整可能失效 |
| **趋势跟踪** | SuperTrend V1 + PositionExecutor | 趋势翻转入场+三重屏障保护 | 震荡中假信号频繁 |
| **均值回归** | Bollinger V2 + PositionExecutor | 布林带偏离入场+三重屏障保护 | 强趋势中均值回归失效 |
| **逢低建仓** | DMan V3 + DCAExecutor | 多级 DCA 分批建仓+动态止损 | 趋势中逆势加仓风险大 |
| **网格交易** | GridExecutor (独立) | 震荡行情中持续赚取 step 价差 | 趋势中网格被单边穿越 |
| **大额拆单** | TWAPExecutor (辅助) | 时间加权拆分减少冲击 | 非盈利执行器，仅用于执行 |
| **风险对冲** | Hedge (V1) + 任意做市策略 | 自动对冲做市产生的库存暴露 | 对冲本身产生成本 |

### 全局风险提示

1. **单腿暴露是最大风险**：ArbitrageExecutor（重试仅 3 次）和 XEMMExecutor（无三重屏障）的单腿暴露无自动止损管理。建议检测 `close_type == FAILED` 后创建反向 PositionExecutor 管理暴露仓位。
2. **V1 策略无统一止损**：所有 V1 策略的风险可控性上限为 3，建议优先使用 V2 控制器+执行器组合。
3. **REST 下单延迟**：跨所策略双边 REST 下单 100-400ms，高频套利可能错过窗口。
4. **流动性假设**：市价单执行假设对手盘充足，低流动性交易对实际滑点可能远超预期。
5. **协整关系非永恒**：StatArb 的协整关系可能因基本面变化而断裂，需定期重新验证。
6. **所有 V2 控制器无独立单元测试**：边界情况（数据不足、指标计算异常）处理不完善。

---

## 七、补充分析

### V1 vs V2 关键差异

| 维度 | V1 策略 | V2 控制器+执行器 |
|------|---------|----------------|
| 架构 | 单体策略类 | Controller-Executor 分离 |
| 风控 | 策略内自行实现，无统一框架 | 三重屏障统一框架（stop_loss/take_profit/time_limit/trailing_stop） |
| 多档位 | order_levels 参数 | 每个 level 独立 Executor |
| 统计指标 | 仅 Avellaneda 用波动率 | NATR/MACD/Z-Score/回归 |
| **回测** | **无** | **BacktestingEngine + ExecutorSimulator** |
| **热更新** | **部分参数** | **is_updatable 参数全部支持** |
| **仓位聚合** | **无** | **ExecutorOrchestrator PositionHold** |
| 下单通道 | REST POST | REST POST（同 V1） |

**回测能力的影响**：V2 的 BacktestingEngine 允许在不冒真金白银的情况下验证策略参数，直接降低了实操门槛。V1 策略只能用历史数据手动回测或直接实盘试错。

**热更新的影响**：V2 的 is_updatable 参数支持运行时修改，无需重启策略。V1 策略修改参数需停止→重新配置→启动，在快速行情变化中响应滞后。

**仓位聚合的影响**：V2 的 ExecutorOrchestrator 通过 PositionHold 对象追踪所有执行器的净持仓和盈亏，提供全局视角。V1 策略各自独立管理仓位，无法跨策略聚合。

### 下单失败补偿机制

| 场景 | CEX 连接器 | Gateway 连接器 |
|------|-----------|---------------|
| 下单失败 | 标记 FAILED，**无自动重试** | max_retries=10，自动重试 |
| 不重试场景 | — | INSUFFICIENT_BALANCE / SLIPPAGE_EXCEEDED / INVALID_PARAMS |
| 重试场景 | 仅时间同步 IOError 最多 2 次 | TRANSACTION_TIMEOUT / status==0 (pending) |
| 余额恢复 | real_time=True: WS 推送（即时）；real_time=False: 快照差值公式（1-15秒） | 链上余额查询 |

**关键差异**：CEX 连接器下单失败后直接标记 FAILED 不重试，而 Gateway 有完善的分类重试机制。这意味着跨所策略（XEMM/Arbitrage）的 CEX 端失败后依赖策略层重试（ArbitrageExecutor 3 次、XEMMExecutor 10 次），而非连接器层自动重试。

### 对自定义策略的 5 条启示

1. **检测单腿**：检查 ArbitrageExecutor 的 `close_type == FAILED`，判断哪端成交
2. **主动平仓**：创建反向 PositionExecutor（带止损/止盈/时间限制）管理暴露仓位
3. **冷却期**：单腿暴露未解决前不发新信号
4. **重试策略**：Taker 端失败先重试 3-5 次，超过后转为 PositionExecutor 管理
5. **止损硬编码**：单腿暴露仓位止损设保守值（如 0.5%）

### 性能瓶颈与扩展限制

| 瓶颈 | 影响 | 量化（50 对 × 5 所） |
|------|------|---------------------|
| WS 消息串行处理 | 所有交易对 diff 消息同一条协程解析 | 5000 msg/s，50-750ms/s |
| OrderBook 初始化串行化 | 逐个获取 REST 快照，每对间隔 1 秒 | 50 秒初始化 |
| API 限流器锁竞争 | 全局 asyncio.Lock，高并发请求串行化 | O(T×L) 复杂度 |
| 策略 tick 串行 | Clock 串行调用所有 child_iterator.tick() | 150 策略 × 0.5-2ms = 75-300ms |
| SQLite 同步写入 | session.commit() 在主事件循环中同步执行 | 高频时阻塞数百毫秒 |

**对策略选择的实际影响**：
- 跨所策略（XEMM/Arbitrage）的可监控交易对数受 WS 限制（Binance 5 连接/IP，每连接约 100 对），5 所 × 50 对接近极限
- 做市策略的多档位执行器受 tick 串行影响，150 个执行器 × 0.5-2ms 占用 7.5%-30% 的 1 秒窗口
- 建议单实例控制在 20 个交易对以内，超过则使用多实例部署 + MQTT 消息队列共享

### 下单通道约束

**核心结论**：所有 CEX 连接器通过 REST API 下单，WebSocket 仅用于接收数据。

| 场景 | 延迟 |
|------|------|
| 单交易所内 | 50-100ms |
| 跨所双边 REST | 100-400ms |
| WS 推送信号 → REST 下单 | 信号 ~1ms + 下单 50-200ms |

**优化方向**：部分交易所支持 `cancelReplace`（改单 API），将撤单+重挂合并为一次请求，延迟降至 50-100ms。当前 Hummingbot 未实现此优化。

---

## 附录：评分明细汇总

### V1 策略

| 策略 | 盈利 | 风控 | 行情 | 资金 | 门槛 | 组合 | 成熟 | **综合** |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Pure MM | 3 | 2 | 2 | 3 | 3 | 3 | 5 | **3.0** |
| Avellaneda MM | 4 | 2 | 3 | 3 | 2 | 3 | 4 | **3.0** |
| Perpetual MM | 3 | 3 | 3 | 3 | 3 | 2 | 4 | **3.0** |
| XEMM | 3 | 1 | 2 | 2 | 2 | 2 | 4 | **2.3** |
| Cross Ex Mining | 3 | 2 | 2 | 3 | 1 | 2 | 3 | **2.3** |
| AMM Arb | 3 | 2 | 1 | 2 | 3 | 2 | 4 | **2.4** |
| Spot-Perp Arb | 3 | 2 | 2 | 2 | 3 | 2 | 4 | **2.6** |
| Liquidity Mining | 3 | 2 | 2 | 3 | 3 | 2 | 4 | **2.7** |
| Hedge | 1 | 2 | 3 | 2 | 3 | 5 | 4 | **2.9** |

### V2 执行器

| 执行器 | 盈利 | 风控 | 行情 | 资金 | 门槛 | 组合 | 成熟 | **综合** |
|--------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| PositionExecutor | 3 | 5 | 4 | 3 | 4 | 5 | 5 | **4.1** |
| OrderExecutor | 2 | 2 | 4 | 3 | 4 | 5 | 4 | **3.4** |
| DCAExecutor | 3 | 4 | 2 | 3 | 3 | 4 | 4 | **3.4** |
| GridExecutor | 4 | 4 | 2 | 4 | 3 | 3 | 4 | **3.4** |
| XEMMExecutor | 3 | 2 | 2 | 2 | 3 | 3 | 4 | **2.7** |
| ArbitrageExecutor | 3 | 1 | 1 | 2 | 3 | 3 | 4 | **2.4** |
| TWAPExecutor | 2 | 2 | 4 | 3 | 5 | 5 | 4 | **3.6** |
| LPExecutor | 3 | 3 | 2 | 3 | 2 | 2 | 3 | **2.6** |

### V2 控制器

| 控制器 | 盈利 | 风控 | 行情 | 资金 | 门槛 | 组合 | 成熟 | **综合** |
|--------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| PMM Simple | 4 | 4 | 3 | 3 | 5 | 4 | 4 | **3.9** |
| PMM Dynamic | 4 | 4 | 4 | 3 | 3 | 4 | 4 | **3.7** |
| XEMM Multi-Level | 3 | 2 | 2 | 2 | 2 | 2 | 3 | **2.3** |
| Bollinger V2 | 3 | 3 | 2 | 3 | 4 | 3 | 4 | **3.1** |
| DMan V3 | 3 | 4 | 2 | 3 | 2 | 3 | 4 | **3.0** |
| SuperTrend V1 | 3 | 3 | 2 | 3 | 4 | 3 | 4 | **3.3** |
| StatArb | 3 | 4 | 3 | 3 | 1 | 2 | 3 | **2.7** |
