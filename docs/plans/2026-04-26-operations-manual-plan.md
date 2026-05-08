# 任务计划：Hummingbot 操作手册

日期：2026-04-26

## 来源

| 类型 | 引用 | 说明 |
|------|------|------|
| 📄 文档 | [2026-04-26-operations-manual-design.md](../achievements/2026-04-26-operations-manual-design.md) | 设计文档，定义章节结构、模块深度、内容获取策略 |
| 💬 上下文 | 对话输入 | 从官网 https://hummingbot.org/docs/ 抽取关键知识点，制作全面参考手册；全部 14 章节覆盖；官网+源码补充 |

## 配置

| 项目 | 值 |
|------|---|
| TDD | ❌ 跳过（文档类任务） |
| build & test 门控 | ❌ 关闭 |
| simplify/code-review 门控 | ❌ 关闭 |
| compact 检查 | ✅ 开启 |
| 执行次数 | 0 |
| 本次范围 | 全量 |

## 任务概览

总任务：15 个 ｜ 批次：4 批 ｜ 并行组：4 组

---

## 批次 1 / 4 — 基础框架 + 核心模块（上）

> 先完成概览建立全局视角，再并行处理安装、客户端、策略框架三个核心章节

### 编码任务

- [ ] **编写章节 1：概览** — `docs/hummingbot-operations-manual.md` — 文档 M
  - 抓取 https://hummingbot.org/docs/ 生态全景
  - 抓取 https://hummingbot.org/installation/ 组件总览
  - 绘制组件关系图（Mermaid）
  - 写入文件头部 + 概览章节

- [ ] [并行] **编写章节 2：安装部署** — `docs/hummingbot-operations-manual.md` — 文档 M
  - 抓取 https://hummingbot.org/installation/hummingbot-client/
  - 抓取 https://hummingbot.org/installation/condor/
  - 抓取 https://hummingbot.org/installation/（Gateway 源码安装部分）
  - 补充源码中的环境要求

- [ ] [并行] **编写章节 3：客户端操作** — `docs/hummingbot-operations-manual.md` — 文档 M
  - 抓取 https://hummingbot.org/client/ 全部子页面
  - 从 `hummingbot/client/` 源码提取 CLI 命令列表
  - 整理配置文件说明

- [ ] [并行] **编写章节 4：策略框架** — `docs/hummingbot-operations-manual.md` — 文档 L
  - 抓取 https://hummingbot.org/strategies/v2-strategies/ 架构说明
  - 抓取 https://hummingbot.org/strategies/v2-strategies/executors/ 全部 Executor 类型
  - 抓取 https://hummingbot.org/strategies/v2-strategies/controllers/ 全部 Controller 类型
  - 抓取 https://hummingbot.org/strategies/scripts/ 脚本说明
  - 抓取 https://hummingbot.org/strategies/v2-strategies/data/ MarketDataProvider
  - 抓取 https://hummingbot.org/strategies/v1-strategies/ 全部 V1 策略
  - 从 `hummingbot/strategy_v2/` 和 `controllers/` 源码补充 Config 参数

### 门控

- [ ] compact 检查

---

## 批次 2 / 4 — 核心模块（下）

> 连接器、Gateway、API 三个模块并行

### 编码任务

- [ ] [并行] **编写章节 5：连接器** — `docs/hummingbot-operations-manual.md` — 文档 L
  - 抓取 https://hummingbot.org/exchanges/ CLOB 连接器列表
  - 抓取 https://hummingbot.org/gateway/connectors/ AMM DEX 连接器列表
  - 从 `hummingbot/connector/` 源码补充连接器配置参数

- [ ] [并行] **编写章节 6：Gateway** — `docs/hummingbot-operations-manual.md` — 文档 M
  - 抓取 https://hummingbot.org/gateway/ 架构与安装
  - 抓取 Gateway 配置子页面
  - 整理 Router/AMM/CLMM 三种连接器类型

- [ ] [并行] **编写章节 7：Hummingbot API** — `docs/hummingbot-operations-manual.md` — 文档 S
  - 抓取 https://hummingbot.org/hummingbot-api/ 概览
  - 抓取 https://hummingbot.org/hummingbot-api/installation/ 安装说明

### 门控

- [ ] compact 检查

---

## 批次 3 / 4 — 扩展模块

> MCP、Condor、Dashboard、Quants Lab 四个模块并行

### 编码任务

- [ ] [并行] **编写章节 8：MCP & Skills** — `docs/hummingbot-operations-manual.md` — 文档 S
  - 抓取 https://hummingbot.org/mcp/ 概览与使用方法

- [ ] [并行] **编写章节 9：Condor** — `docs/hummingbot-operations-manual.md` — 文档 S
  - 抓取 https://hummingbot.org/condor/ 概览与使用方法

- [ ] [并行] **编写章节 10：Dashboard** — `docs/hummingbot-operations-manual.md` — 文档 S
  - 抓取 https://hummingbot.org/dashboard/ 概览

- [ ] [并行] **编写章节 11：Quants Lab** — `docs/hummingbot-operations-manual.md` — 文档 S
  - 抓取 https://hummingbot.org/quants-lab/ 概览与使用方法

### 门控

- [ ] compact 检查

---

## 批次 4 / 4 — 收尾章节

> 高级配置、FAQ、术语表并行，最终整合

### 编码任务

- [ ] [并行] **编写章节 12：高级配置** — `docs/hummingbot-operations-manual.md` — 文档 M
  - 抓取客户端高级配置子页面（Kill Switch、余额限制、外部数据库、费率预言机等）
  - 从源码补充配置参数

- [ ] [并行] **编写章节 13：FAQ & 排障** — `docs/hummingbot-operations-manual.md` — 文档 S
  - 抓取 https://hummingbot.org/faq/
  - 抓取 https://hummingbot.org/troubleshooting/

- [ ] [并行] **编写章节 14：术语表** — `docs/hummingbot-operations-manual.md` — 文档 S
  - 抓取 https://hummingbot.org/glossary/

- [ ] **最终整合与校验** — `docs/hummingbot-operations-manual.md` — 文档 M
  - 合并所有章节到单文件
  - 添加版本标注和文档抓取日期
  - 校验内部链接一致性
  - 标记 `[待补充]` 缺失内容

### 门控

- [ ] compact 检查

---

## 收尾任务

- [ ] **循环复检** — 独立审查员 Agent（所有批次完成后自动触发）

---

## 完成状态

| 批次 | 任务 | 状态 | 备注 |
|-----|------|:---:|------|
| 1 | 编写章节 1：概览 | ✅ | 已完成 |
| 1 | 编写章节 2：安装部署 | ✅ | 已完成 |
| 1 | 编写章节 3：客户端操作 | ✅ | 已完成 |
| 1 | 编写章节 4：策略框架 | ✅ | 已完成 |
| 2 | 编写章节 5：连接器 | ✅ | 已完成 |
| 2 | 编写章节 6：Gateway | ✅ | 已完成 |
| 2 | 编写章节 7：Hummingbot API | ✅ | 已完成 |
| 3 | 编写章节 8：MCP & Skills | ✅ | 已完成（含 [待补充] 占位） |
| 3 | 编写章节 9：Condor | ✅ | 已完成 |
| 3 | 编写章节 10：Dashboard | ✅ | 已完成 |
| 3 | 编写章节 11：Quants Lab | ✅ | 已完成 |
| 4 | 编写章节 12：高级配置 | ✅ | 已完成（详细版 12.1-12.8） |
| 4 | 编写章节 13：FAQ & 排障 | ✅ | 已完成 |
| 4 | 编写章节 14：术语表 | ✅ | 已完成 |
| 4 | 最终整合与校验 | ✅ | 已完成 |
| 收尾 | 循环复检 | ✅ | 已完成，补充了 MCP/FAQ/术语表内容 |
