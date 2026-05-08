# Hummingbot 单腿暴露与下单失败补偿机制

> 最后更新：2026-04-22

---

## 一、单腿暴露风险等级

| 执行器 | 风险 | 重试次数 | 止损 | 丢失订单恢复 |
|--------|------|---------|------|------------|
| ArbitrageExecutor | **高** | 3 | 无 | 无 |
| XEMMExecutor | 中高 | 10 | 无 | 无 |
| PositionExecutor | 低 | 10 | 有（三重屏障） | 有 |
| V1 XEMM | 中 | 无限 | 无 | 无 |

### ArbitrageExecutor — 风险最高

```
买入端成交 → 卖出端失败 → 重试卖出端（最多3次）
                              ↓ 超过3次
                         CloseType.FAILED → 执行器终止
                              ↓
                    买入端持仓完全暴露，无人管理
```

**核心缺陷**：一端成交另一端失败后，直接标记 FAILED 并终止。无止损、无时间限制、无 POSITION_HOLD。

### XEMMExecutor — Taker 端失败立即重试

```python
# xemm_executor.py:291-299
def process_order_failed_event(self, _, market, event):
    if ...:  # Taker 端失败
        self.place_taker_order()  # 立即重下 Taker 单
        self._current_retries += 1
```

重试 10 次（比 ArbitrageExecutor 的 3 次更多），但超过上限后同样以 FAILED 关闭。

### V1 XEMM — 最完善但无上限

Taker 端取消/失败/过期三种场景均触发重试，且有 `_ongoing_hedging` 互斥机制防止新 Maker 单。但无重试上限。

### ExecutorOrchestrator — 只追踪不平仓

`PositionHold` 对象记录已成交订单的净持仓和盈亏，但**不自动创建平仓执行器**。

---

## 二、下单失败检测与补偿

### 失败检测链路

```
下单请求失败 ──→ _on_order_failure ──→ OrderState.FAILED ──→ MarketOrderFailureEvent ──→ 策略层
订单查询不到 ──→ process_order_not_found ──→ 计数器++ ──→ 超过3次 ──→ FAILED (lost order)
交易所拒绝 ──→ 常量映射 (REJECTED/EXPIRED) ──→ FAILED
```

### CEX：无自动重试

下单失败后直接标记 FAILED，不自动重试。唯一例外：时间同步器相关 IOError 最多重试 2 次。

### Gateway：有完善重试

```python
max_retries = 10
# 不重试：INSUFFICIENT_BALANCE / SLIPPAGE_EXCEEDED / INVALID_PARAMS
# 重试：TRANSACTION_TIMEOUT / status == 0 (pending)
```

### 余额恢复

| 模式 | 恢复方式 | 时效 |
|------|---------|------|
| `real_time=True` | WS 推送余额更新 | 即时 |
| `real_time=False` | 快照差值公式自动修正 | 1-15 秒 |

订单标记 FAILED 后，`in_flight_asset_balances()` 自动排除，锁定的余额**立即释放**。

### 策略层补偿对比

| 策略 | Taker 失败 | Maker 失败 | 重试 | 取消对端 |
|------|-----------|-----------|------|---------|
| V1 XEMM | 重新对冲 | 无特殊处理 | 无限 | 仅余额不足/不再盈利时 |
| V2 ArbitrageExecutor | 重试同端 | 重试同端 | 3 次 | 否 |
| V2 XEMMExecutor | 立即重下 Taker 单 | 重置 Maker 单 | 10 次 | 否 |

---

## 三、对自定义策略的启示

1. **检测单腿**：检查 ArbitrageExecutor 的 `close_type == FAILED`，判断哪端成交
2. **主动平仓**：创建反向 PositionExecutor（带止损/止盈/时间限制）管理暴露仓位
3. **冷却期**：单腿暴露未解决前不发新信号
4. **重试策略**：Taker 端失败先重试 3-5 次，超过后转为 PositionExecutor 管理
5. **止损硬编码**：单腿暴露仓位止损设保守值（如 0.5%）