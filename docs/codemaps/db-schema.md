# 数据库 Schema 文档

> 自动生成自 `hummingbot/model/` 目录下的 SQLAlchemy 模型定义
> ORM: SQLAlchemy | 数据库: SQLite | 基类: `HummingbotBase`

---

## 自定义类型

| 类型 | 底层存储 | 说明 |
|------|---------|------|
| `SqliteDecimal(N)` | `BigInteger` | 将 Decimal 乘以 10^N 后以整数存储，读取时还原。N=6 为默认精度 |

---

## 表结构

### Order

**用途**: 存储交易订单信息

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Text | PK, NOT NULL | 订单 ID |
| config_file_path | Text | NOT NULL | 策略配置文件路径 |
| strategy | Text | NOT NULL | 策略名称 |
| market | Text | NOT NULL | 交易所/市场 |
| symbol | Text | NOT NULL | 交易对（如 BTC-USDT） |
| base_asset | Text | NOT NULL | 基础资产 |
| quote_asset | Text | NOT NULL | 计价资产 |
| creation_timestamp | BigInteger | NOT NULL | 创建时间戳（毫秒） |
| order_type | Text | NOT NULL | 订单类型 |
| amount | SqliteDecimal(6) | NOT NULL | 下单数量 |
| leverage | Integer | NOT NULL, DEFAULT 1 | 杠杆倍数 |
| price | SqliteDecimal(6) | NOT NULL | 下单价格 |
| last_status | Text | NOT NULL | 最近状态 |
| last_update_timestamp | BigInteger | NOT NULL | 最后更新时间戳 |
| exchange_order_id | Text | NULL | 交易所订单 ID |
| position | Text | NULL | 持仓方向 |

**关系**:
- `status` -> OrderStatus (1:N, back_populates="order")
- `trade_fills` -> TradeFill (1:N, back_populates="order")

**索引**:
| 名称 | 字段 |
|------|------|
| o_config_timestamp_index | (config_file_path, creation_timestamp) |
| o_market_trading_pair_timestamp_index | (market, symbol, creation_timestamp) |
| o_market_base_asset_timestamp_index | (market, base_asset, creation_timestamp) |
| o_market_quote_asset_timestamp_index | (market, quote_asset, creation_timestamp) |

---

### OrderStatus

**用途**: 记录订单状态变更历史

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Integer | PK, NOT NULL | 自增主键 |
| order_id | Text | FK -> Order.id, NOT NULL | 关联订单 ID |
| timestamp | BigInteger | NOT NULL | 状态变更时间戳 |
| status | Text | NOT NULL | 状态值 |

**关系**:
- `order` -> Order (N:1, back_populates="status")

**索引**:
| 名称 | 字段 |
|------|------|
| os_order_id_timestamp_index | (order_id, timestamp) |

---

### TradeFill

**用途**: 存储成交记录

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| config_file_path | Text | NOT NULL | 策略配置文件路径 |
| strategy | Text | NOT NULL | 策略名称 |
| market | Text | PK, NOT NULL | 交易所/市场 |
| symbol | Text | NOT NULL | 交易对 |
| base_asset | Text | NOT NULL | 基础资产 |
| quote_asset | Text | NOT NULL | 计价资产 |
| timestamp | BigInteger | NOT NULL | 成交时间戳 |
| order_id | Text | PK, FK -> Order.id, NOT NULL | 关联订单 ID |
| trade_type | Text | NOT NULL | 买/卖方向 |
| order_type | Text | NOT NULL | 订单类型 |
| price | SqliteDecimal(6) | NOT NULL | 成交价格 |
| amount | SqliteDecimal(6) | NOT NULL | 成交数量 |
| leverage | Integer | NOT NULL, DEFAULT 1 | 杠杆倍数 |
| trade_fee | JSON | NOT NULL | 手续费信息（JSON） |
| trade_fee_in_quote | SqliteDecimal(6) | NULL | 以计价资产计的手续费 |
| exchange_trade_id | Text | PK, NOT NULL | 交易所成交 ID |
| position | Text | NULL, DEFAULT NIL | 持仓方向 |

**复合主键**: (market, order_id, exchange_trade_id)

**关系**:
- `order` -> Order (N:1, back_populates="trade_fills")

**索引**:
| 名称 | 字段 |
|------|------|
| tf_config_timestamp_index | (config_file_path, timestamp) |
| tf_market_trading_pair_timestamp_index | (market, symbol, timestamp) |
| tf_market_base_asset_timestamp_index | (market, base_asset, timestamp) |
| tf_market_quote_asset_timestamp_index | (market, quote_asset, timestamp) |

---

### Position

**用途**: 存储执行器持有的仓位信息

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Text | PK, NOT NULL | 仓位 ID |
| controller_id | Text | NOT NULL | 控制器 ID |
| connector_name | Text | NOT NULL | 连接器名称 |
| side | Text | NOT NULL | 方向（买/卖） |
| trading_pair | Text | NOT NULL | 交易对 |
| timestamp | BigInteger | NOT NULL | 时间戳 |
| volume_traded_quote | SqliteDecimal(6) | NOT NULL | 累计交易量（计价资产） |
| amount | SqliteDecimal(6) | NOT NULL | 当前持仓量 |
| breakeven_price | SqliteDecimal(6) | NOT NULL | 盈亏平衡价 |
| unrealized_pnl_quote | SqliteDecimal(6) | NOT NULL | 未实现盈亏（计价资产） |
| realized_pnl_quote | SqliteDecimal(6) | NOT NULL | 已实现盈亏（计价资产） |
| cum_fees_quote | SqliteDecimal(6) | NOT NULL | 累计手续费（计价资产） |

**索引**:
| 名称 | 字段 |
|------|------|
| p_controller_id_timestamp_index | (controller_id, timestamp) |
| p_connector_name_trading_pair_timestamp_index | (connector_name, trading_pair, timestamp) |

---

### MarketData

**用途**: 存储市场行情快照（中间价、最优买卖价、订单簿）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| timestamp | SqliteDecimal(6) | PK, NOT NULL | 时间戳 |
| exchange | Text | NOT NULL | 交易所名称 |
| trading_pair | Text | NOT NULL | 交易对 |
| mid_price | SqliteDecimal(6) | NOT NULL | 中间价 |
| best_bid | SqliteDecimal(6) | NOT NULL | 最优买价 |
| best_ask | SqliteDecimal(6) | NOT NULL | 最优卖价 |
| order_book | JSON | NULL | 订单簿快照（JSON） |

**索引**:
| 名称 | 字段 |
|------|------|
| timestamp | (timestamp, exchange, trading_pair) |

---

### MarketState

**用途**: 按策略配置和市场存储持久化状态（策略恢复用）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Integer | PK, NOT NULL | 自增主键 |
| config_file_path | Text | NOT NULL | 策略配置文件路径 |
| market | Text | NOT NULL | 市场名称 |
| timestamp | BigInteger | NOT NULL | 保存时间戳 |
| saved_state | JSON | NOT NULL | 序列化状态（JSON） |

**索引**:
| 名称 | 字段 | 唯一 |
|------|------|------|
| ms_config_market_index | (config_file_path, market) | UNIQUE |

---

### Metadata

**用途**: 键值对形式的元数据存储（如数据库版本号）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| key | Text | PK, NOT NULL | 元数据键 |
| value | Text | NOT NULL | 元数据值 |

---

### FundingPayment

**用途**: 存储永续合约资金费率支付记录

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| timestamp | BigInteger | PK, NOT NULL | 时间戳 |
| config_file_path | Text | NOT NULL | 策略配置文件路径 |
| market | Text | NOT NULL | 交易所名称 |
| rate | Float | NOT NULL | 资金费率 |
| symbol | Text | NOT NULL | 交易对 |
| amount | Float | NOT NULL | 资金费用金额 |

**索引**:
| 名称 | 字段 |
|------|------|
| fp_config_timestamp_index | (config_file_path, timestamp) |
| fp_market_trading_pair_timestamp_index | (market, symbol, timestamp) |

---

### InventoryCost

**用途**: 跟踪各交易对的库存成本（基础资产和计价资产的累计量）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Integer | PK, NOT NULL | 自增主键 |
| base_asset | String(45) | NOT NULL | 基础资产符号 |
| quote_asset | String(45) | NOT NULL | 计价资产符号 |
| base_volume | Numeric(48,18) | NOT NULL | 基础资产累计量 |
| quote_volume | Numeric(48,18) | NOT NULL | 计价资产累计量 |

**约束**:
| 名称 | 类型 | 字段 |
|------|------|------|
| - | UniqueConstraint | (base_asset, quote_asset) |

---

### RangePositionUpdate

**用途**: 记录 LP 仓位变更事件（添加/移除/收取手续费），用于 PnL 追踪

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Integer | PK | 自增主键 |
| hb_id | Text | NOT NULL | 订单 ID（如 range-SOL-USDC-...） |
| timestamp | BigInteger | NOT NULL | 事件时间戳 |
| tx_hash | Text | NULL | 交易签名 |
| token_id | Integer | NOT NULL | 遗留字段 |
| trade_fee | JSON | NOT NULL | 手续费信息 |
| config_file_path | Text | NULL | 策略配置文件 |
| market | Text | NULL | 连接器名称（如 meteora/clmm） |
| order_action | Text | NULL | 操作类型（ADD/REMOVE） |
| trading_pair | Text | NULL | 交易对 |
| position_address | Text | NULL | LP 仓位 NFT 地址 |
| lower_price | Float | NULL | 价格区间下限 |
| upper_price | Float | NULL | 价格区间上限 |
| mid_price | Float | NULL | 事件时当前价格 |
| base_amount | Float | NULL | 基础代币数量 |
| quote_amount | Float | NULL | 计价代币数量 |
| base_fee | Float | NULL | 基础代币手续费（REMOVE 时） |
| quote_fee | Float | NULL | 计价代币手续费（REMOVE 时） |
| position_rent | Float | NULL | 创建仓位的 SOL 租金（ADD 时） |
| position_rent_refunded | Float | NULL | 关闭仓位退还的 SOL 租金（REMOVE 时） |
| trade_fee_in_quote | Float | NULL | 以计价资产计的交易手续费 |

**索引**:
| 名称 | 字段 |
|------|------|
| rpu_timestamp_index | (hb_id, timestamp) |
| rpu_config_file_index | (config_file_path, timestamp) |
| rpu_position_index | (position_address) |

---

### RangePositionCollectedFees

**用途**: 记录 LP 手续费收取情况

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Integer | PK, NOT NULL | 自增主键 |
| config_file_path | Text | NOT NULL | 策略配置文件路径 |
| strategy | Text | NOT NULL | 策略名称 |
| token_id | Integer | NOT NULL | LP 仓位 Token ID |
| token_0 | Text | NOT NULL | Token 0 符号 |
| token_1 | Text | NOT NULL | Token 1 符号 |
| claimed_fee_0 | Float | NOT NULL | 已收取的 Token 0 手续费 |
| claimed_fee_1 | Float | NOT NULL | 已收取的 Token 1 手续费 |

**索引**:
| 名称 | 字段 |
|------|------|
| rpf_id_index | (token_id, config_file_path) |

---

### Controllers

**用途**: 存储控制器配置快照

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Text | NULL | 控制器标识 |
| controller_id | Integer | PK, AUTOINCREMENT | 自增主键 |
| timestamp | Float | NOT NULL | 创建时间戳 |
| type | Text | NOT NULL | 控制器类型 |
| config | JSON | NOT NULL | 控制器配置（JSON） |

**索引**:
| 名称 | 字段 |
|------|------|
| c_type | (type) |

---

### Executors

**用途**: 存储执行器运行状态和盈亏数据

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | Text | PK | 执行器 ID |
| timestamp | Float | NOT NULL | 创建时间戳 |
| type | Text | NOT NULL | 执行器类型 |
| close_type | Integer | NULL | 关闭类型（枚举映射） |
| close_timestamp | BigInteger | NULL | 关闭时间戳 |
| status | Integer | NOT NULL | 运行状态（枚举映射） |
| config | JSON | NOT NULL | 执行器配置（JSON） |
| net_pnl_pct | Float | NOT NULL | 净盈亏百分比 |
| net_pnl_quote | Float | NOT NULL | 净盈亏（计价资产） |
| cum_fees_quote | Float | NOT NULL | 累计手续费（计价资产） |
| filled_amount_quote | Float | NOT NULL | 已成交金额（计价资产） |
| is_active | Boolean | NOT NULL | 是否活跃 |
| is_trading | Boolean | NOT NULL | 是否交易中 |
| custom_info | JSON | NOT NULL | 自定义信息（JSON） |
| controller_id | Text | NULL | 关联控制器 ID |

**索引**:
| 名称 | 字段 |
|------|------|
| ex_type | (type) |
| ex_type_timestamp | (type, timestamp) |
| ex_timestamp | (timestamp) |
| ex_close_timestamp | (close_timestamp) |
| ex_status | (status) |
| ex_type_status | (type, status) |

---

## 实体关系图 (ER)

```
Order (1) ──────< (N) OrderStatus
  │                    FK: order_id -> Order.id
  │
  └──────< (N) TradeFill
               FK: order_id -> Order.id
               复合 PK: (market, order_id, exchange_trade_id)

Controllers (1) ──< (N) Executors
                    逻辑关联: controller_id (无 FK 约束)

Controllers (1) ──< (N) Position
                    逻辑关联: controller_id (无 FK 约束)

独立表（无外键关系）:
  - MarketData
  - MarketState
  - Metadata
  - FundingPayment
  - InventoryCost
  - RangePositionUpdate
  - RangePositionCollectedFees
```

---

## 数据库配置

| 项 | 值 |
|----|-----|
| 默认数据库文件 | `hummingbot_trades.sqlite` |
| 存储路径 | Hummingbot 数据目录 |
| 版本管理 | Metadata 表 key=`local_db_version`，当前版本=`20230516` |
| 连接管理 | `SQLConnectionManager`（单例模式） |
| 事务管理 | `TransactionBase`（自动 commit/rollback） |
| 迁移工具 | `hummingbot/model/db_migration/migrator.py` |
