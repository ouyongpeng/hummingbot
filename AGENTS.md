# AGENTS.md — Hummingbot

> AI 协作文档，描述项目结构、技术栈与开发规范。
> 由 `update-agents` 技能生成，更新请运行 `/update-agents`。
> **最后更新**：2026-04-22

---

## 目录

- [1. 开发规范](#1-开发规范)
- [2. 项目概览](#2-项目概览)
- [3. 架构说明](#3-架构说明)
- [4. 代码模块索引](#4-代码模块索引)
- [5. 数据库规范](#5-数据库规范)
- [6. API 设计规范](#6-api-设计规范)
- [7. 文档索引](#7-文档索引)
- [8. 开发命令](#8-开发命令)
- [9. 安全规范](#9-安全规范)
- [10. 快速上手](#10-快速上手)

---

## 1. 开发规范
<!-- maintained-by: update-agents -->

<!-- [TEMPLATE] -->
> DDD + TDD，先文档后代码，80% 覆盖率卡点。新功能/修改须通过单元、集成、接口三类测试，未通过不得合并。
<!-- [/TEMPLATE] -->

<!-- [DYNAMIC:update-agents] -->
_由 `/update-agents` 自动维护，请勿手动编辑_
<!-- [/DYNAMIC:update-agents] -->

<!-- [INDEX] -->
- [完整开发规范](docs/standards/dev-workflow.md)
<!-- [/INDEX] -->

---

## 2. 项目概览

**项目名称**：Hummingbot
**类型**：加密货币做市与交易机器人框架
**语言 / 框架**：Python 3.10+ / Cython / asyncio / SQLAlchemy / Web3

### 核心功能

| 功能 | 说明 |
|------|------|
| [MANUAL_FEATURE_1] | [MANUAL_FEATURE_1_DESC] |
| [MANUAL_FEATURE_2] | [MANUAL_FEATURE_2_DESC] |

---

## 3. 架构说明

### 技术栈

- **语言**：Python 3.10+，核心路径用 Cython 加速
- **异步框架**：asyncio + aiohttp
- **数据库**：SQLite（SQLAlchemy ORM）
- **区块链交互**：Web3.py / eth-account / solders（Solana）/ injective-py / xrpl-py
- **数据分析**：pandas / numpy / scipy / TA-Lib / pandas-ta
- **CLI**：prompt_toolkit
- **配置**：YAML（ruamel.yaml / PyYaml）
- **容器化**：Docker / Docker Compose
- **包管理**：Conda + pip

### 项目结构

```
hummingbot/
├── bin/                    # 启动脚本
├── conf/                   # 配置文件
├── controllers/            # 策略控制器
├── hummingbot/             # 核心包
│   ├── client/             # 客户端 / CLI 界面
│   ├── connector/          # 交易所与链连接器
│   │   ├── derivative/     # 衍生品连接器
│   │   ├── exchange/       # 现货交易所连接器
│   │   ├── gateway/        # Gateway 网关连接器
│   │   └── utilities/      # 连接器工具
│   ├── core/               # 核心基础设施
│   │   ├── api_throttler/  # API 限流
│   │   ├── data_type/      # 数据类型定义
│   │   ├── event/          # 事件系统
│   │   ├── gateway/        # Gateway 通信
│   │   ├── rate_oracle/    # 价格预言机
│   │   └── web_assistant/  # HTTP/WebSocket 助手
│   ├── data_feed/          # 数据源
│   ├── logger/             # 日志系统
│   ├── model/              # 数据模型（SQLAlchemy）
│   ├── notifier/           # 通知系统
│   ├── remote_iface/       # 远程接口
│   ├── strategy/           # V1 策略（纯做市/跨所套利/AMM 套利等）
│   ├── strategy_v2/        # V2 策略框架
│   │   ├── backtesting/    # 回测引擎
│   │   ├── controllers/    # 策略控制器
│   │   ├── executors/      # 执行器
│   │   └── models/         # 策略模型
│   ├── templates/          # 配置模板
│   └── user/               # 用户配置管理
├── logs/                   # 运行日志
├── scripts/                # 辅助脚本
├── setup/                  # 环境配置
└── test/                   # 测试
```

### 服务架构

单体应用架构，基于事件驱动模型：
- CLI 客户端 → 策略引擎 → 连接器层 → 交易所/链上 API
- 事件总线（EventReporter）贯穿各层，实现松耦合通信

> 数据流：用户配置 → 策略实例 → 连接器 → 交易所 API → 订单管理 → 事件通知

### 数据存储

**数据库类型**：SQLite
**ORM / 驱动**：SQLAlchemy
**迁移工具**：无自动迁移（模型定义即 Schema）

> ⚠️ `[MANUAL_SECTION]`：请补充核心表/实体说明，详细规范见第 5 章「数据库规范」。

---

## 4. 代码模块索引
<!-- maintained-by: update-module-docs -->

<!-- [TEMPLATE] -->
> 路由/服务/模型/中间件/工具函数索引，供 AI 快速定位代码位置与职责边界。
<!-- [/TEMPLATE] -->

<!-- [DYNAMIC:update-module-docs] -->
_由 `/update-module-docs` 自动维护，请勿手动编辑_
<!-- [/DYNAMIC:update-module-docs] -->

<!-- [INDEX] -->
- [模块索引详情](docs/codemaps/module-index.md)
<!-- [/INDEX] -->

---

## 5. 数据库规范
<!-- maintained-by: update-db-docs -->

<!-- [TEMPLATE] -->
> snake_case 命名，必备字段(id/created_at/updated_at/enabled)，软删除用 enabled=0，禁止物理 DELETE，所有 SQL 参数化。
<!-- [/TEMPLATE] -->

<!-- [DYNAMIC:update-db-docs] -->
_由 `/update-db-docs` 自动维护，请勿手动编辑_
<!-- [/DYNAMIC:update-db-docs] -->

<!-- [INDEX] -->
- [数据库规范](docs/standards/db-conventions.md)
- [数据库 Schema](docs/codemaps/db-schema.md)
<!-- [/INDEX] -->

---

## 6. API 设计规范
<!-- maintained-by: update-api-docs -->

<!-- [TEMPLATE] -->
> RESTful，统一响应 {code,message,data}，错误码 40001-50099，分页用 page/pageSize，所有接口参数化防注入。
<!-- [/TEMPLATE] -->

<!-- [DYNAMIC:update-api-docs] -->
_由 `/update-api-docs` 自动维护，请勿手动编辑_
<!-- [/DYNAMIC:update-api-docs] -->

<!-- [INDEX] -->
- [API 设计规范](docs/standards/api-conventions.md)
- [API 端点列表](docs/codemaps/api-endpoints.md)
<!-- [/INDEX] -->

---

## 7. 文档索引

| 文档 | 路径 | 说明 |
|------|------|------|
| README | README.md | 项目介绍与快速开始 |
| 环境配置 | setup/environment.yml | Conda 环境定义 |
| Dockerfile | Dockerfile | Docker 构建配置 |
| 开发规范 | docs/standards/dev-workflow.md | DDD+TDD 工作流 |
| 安全规范 | docs/standards/security-checklist.md | 安全检查清单 |

---

## 8. 开发命令

```bash
# 安装依赖（Conda 环境）
make install

# 开发（启动 Hummingbot）
make run

# 构建 Docker 镜像
make build

# 运行测试
make test                    # 运行所有测试（coverage + pytest）
make run_coverage            # 测试 + 覆盖率报告

# 数据库
# 无自动迁移命令，Schema 由 SQLAlchemy 模型定义

# Docker 部署
make setup                   # 配置 Docker Compose（可选 Gateway）
make deploy                  # 启动 Docker 容器
make down                    # 停止容器
```

**环境变量**（复制 `.env.example` 并填写）：

```
# Hummingbot 无 .env 文件，配置通过 conf/ 目录下的 YAML 文件管理
# 关键配置文件：
#   conf/conf_global.yml         — 全局配置
#   conf/strategy_*.yml          — 策略配置
#   conf/connectors/*.yml        — 连接器配置
```

---

## 9. 安全规范
<!-- maintained-by: update-agents -->

<!-- [TEMPLATE] -->
> 提交前检查：无硬编码密钥、参数化 SQL、XSS 防护、限流。发现 CRITICAL/HIGH 问题立即停止并修复。
<!-- [/TEMPLATE] -->

<!-- [DYNAMIC:update-agents] -->
_由 `/update-agents` 自动维护，请勿手动编辑_
<!-- [/DYNAMIC:update-agents] -->

<!-- [INDEX] -->
- [安全规范清单](docs/standards/security-checklist.md)
<!-- [/INDEX] -->

---

## 10. 快速上手

```bash
# 1. 安装依赖（需要 Conda）
make install

# 2. 编译 Cython 扩展
python setup.py build_ext --inplace

# 3. 启动 Hummingbot
make run

# 或使用 Docker
make setup    # 选择是否启用 Gateway
make deploy   # 启动容器
```

## 重要实现细节

<!-- 手动补充项目特有的关键信息，例如: -->
<!-- - 认证/鉴权机制（交易所 API Key 管理） -->
<!-- - 第三方服务集成（Gateway、dYdX、Injective 等） -->
<!-- - 策略框架 V1 vs V2 的差异与迁移路径 -->
<!-- - Cython 编译对开发流程的影响 -->
<!-- - 部署流程和注意事项 -->
<!-- - 已知性能瓶颈或技术债务 -->
<!-- - 环境差异（开发/测试/生产的关键配置区别） -->
