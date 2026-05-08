# 任务计划：Hummingbot 新手指南设计文档修正

日期：2026-04-26

## 来源

| 类型 | 引用 | 说明 |
|------|------|------|
| 📄 文档 | [新手指南设计文档](../achievements/2026-04-26-onboarding-guide-design.md) | 需修正的设计文档，含 5 处技术细节错误 |

## 配置

| 项目 | 值 |
|------|---|
| TDD | 跳过（文档修正任务） |
| build & test 门控 | ❌ 关闭 |
| simplify/code-review 门控 | ❌ 关闭 |
| compact 检查 | ✅ 开启 |
| 执行次数 | 0 |
| 本次范围 | 全量 |

## 任务概览

总任务：5 个 ｜ 批次：1 批 ｜ 并行组：1 组（全部可并行）

## 错误清单

通过源码验证，设计文档存在以下 5 处技术细节错误：

| # | 位置 | 文档描述 | 实际情况 | 严重度 |
|---|------|----------|----------|--------|
| 1 | §2.1 Makefile install 步骤 | 5 步：conda env create/update、mkdir logs、pip install、pre-commit install、build_ext | 实际 8 步：增加 conda PATH 检查、Darwin 安装 appnope、conda develop .、Linux 检查 build-essential | 中 |
| 2 | §2.4 纸盘模式启用方式 | `paper_trade_enabled: true`（conf_client.yml） | 不存在此字段。纸盘模式通过连接器名称后缀 `_paper_trade` 启用（如 `binance_paper_trade`），在控制器 YAML 的 `connector_name` 中设置 | **高** |
| 3 | §3.4 回测运行命令 | `start --script v2_with_controllers --backtest` | CLI `start` 命令不支持 `--script` 和 `--backtest` 参数。回测通过独立 Python 脚本运行：`conda run -n hummingbot python scripts/backtest_xxx.py` | **高** |
| 4 | §5.2 数据库文件名 | `trades.sqlite` | 实际为 `hummingbot_trades.sqlite`（源码：`sql_connection_manager.py:63`） | 低 |
| 5 | §2.3 Binance 测试网 URL | `https://testnet.binance.vision/` | 代码库中未引用此 URL。永续合约测试网为 `testnet.binancefuture.com`；现货测试网 URL 需从 Binance 官方文档确认 | 中 |

## 批次 1 / 1

### 编码任务

- [x] [并行] **修正 Makefile install 步骤描述** — `docs/achievements/2026-04-26-onboarding-guide-design.md` — 文档 S
- [x] [并行] **修正纸盘模式启用方式** — `docs/achievements/2026-04-26-onboarding-guide-design.md` — 文档 S
- [x] [并行] **修正回测运行命令** — `docs/achievements/2026-04-26-onboarding-guide-design.md` — 文档 S
- [x] [并行] **修正数据库文件名** — `docs/achievements/2026-04-26-onboarding-guide-design.md` — 文档 S
- [x] [并行] **修正 Binance 测试网 URL** — `docs/achievements/2026-04-26-onboarding-guide-design.md` — 文档 S

### 门控

- [ ] compact 检查

## 收尾任务

- [ ] **循环复检** — 独立审查员 Agent（所有批次完成后自动触发）

## 完成状态

| 批次 | 任务 | 状态 | 备注 |
|-----|------|:---:|------|
| 1 | 修正 Makefile install 步骤描述 | ✅ | 5步→9步，补充 conda检查/appnope/conda develop/build-essential |
| 1 | 修正纸盘模式启用方式 | ✅ | paper_trade_enabled→connector_name后缀_paper_trade，修正§2.4+§4.3+§4.4 |
| 1 | 修正回测运行命令 | ✅ | start --script/--backtest→独立Python脚本 |
| 1 | 修正数据库文件名 | ✅ | trades.sqlite→hummingbot_trades.sqlite，修正§5.2两处 |
| 1 | 修正 Binance 测试网 URL | ✅ | testnet.binance.vision→testnet.binancefuture.com(永续) |
| 收尾 | 循环复检 | ✅ | 发现3处遗漏并已修正：§4.3字段名、§3.4控制器名映射、§5.2表名 |
