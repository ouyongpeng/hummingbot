# Hummingbot 安全审计报告

> 审计日期：2026-04-22
> 项目版本：20260421（master 分支）
> 代码规模：约 359,076 行 Python 代码

---

## 一、审计范围

本次审计覆盖以下方向：

1. API Key 存储与传输安全
2. 私钥管理安全
3. 交易与订单安全
4. 网络通信安全
5. 日志泄漏风险
6. 输入验证与注入防护
7. 依赖安全
8. 主观恶意后门排查

---

## 二、CRITICAL（3 个）— 可能直接导致资金损失

### C-1：Gateway 默认 HTTP 传输私钥

- **文件**：`hummingbot/core/gateway/gateway_http_client.py:634-642`
- **问题**：`add_wallet()` 将区块链私钥通过 HTTP 明文发送到 Gateway 容器。默认配置 `gateway_use_ssl=False`，默认部署中私钥暴露在网络传输层
- **影响**：同网段任何设备可嗅探私钥，直接盗取链上资产
- **修复建议**：将 `gateway_use_ssl` 默认值改为 `True`，或移除非 SSL 模式

### C-2：Aevo WebSocket 认证明文传输 API Secret

- **文件**：`hummingbot/connector/derivative/aevo_perpetual/aevo_perpetual_auth.py:85-92`
- **问题**：Aevo 将 API Secret 以明文值放入 WS payload 发送，而非用 Secret 签名。是所有 connector 中唯一传输 Secret 原文的
- **影响**：TLS 会话被截获或日志记录 payload 时，Secret 即泄露
- **修复建议**：改为签名认证模式，不传输 Secret 原文

### C-3：Decibel API Key 明文存储 Bug

- **文件**：`hummingbot/client/config/client_config_map.py:646-656`
- **问题**：由于 `model_construct` 绕过 Pydantic 校验的技术 Bug，Decibel 的 API Key 被迫以明文 `str` 存储，完全绕过加密保护机制。代码注释承认了这是 Bug
- **影响**：配置文件中 API Key 以明文保存，任何能读取文件的人可获取
- **修复建议**：改用 `SecretStr` + 修复 `model_construct` 问题

---

## 三、HIGH（6 个）— 高概率资金风险

### H-1：Hyperliquid/Derive 将私钥作为 api_secret 明文加载到内存

- **文件**：`hyperliquid_perpetual_auth.py:35`、`derive_perpetual_auth.py:33`
- **问题**：私钥解密后以明文 `eth_account.Account` 存在于内存，进程 dump 即可提取
- **修复建议**：使用硬件钱包签名，避免私钥驻留内存

### H-2：Pacifica 硬编码模拟私钥

- **文件**：`hummingbot/core/rate_oracle/sources/pacifica_perpetual_rate_source.py:58`
- **问题**：base58 编码的 64 字节值，格式与真实 Solana 私钥一致，注释声称 dummy 但可能被误用
- **修复建议**：替换为明显的占位符字符串

### H-3：GRVT 明文传输 API Key

- **文件**：`grvt_perpetual_auth.py:107-110`
- **问题**：API Key 放入 JSON body 发送，而非通过 header
- **修复建议**：改为通过 HTTP header 传输 API Key

### H-4：订单跟踪器竞态条件

- **文件**：`hummingbot/connector/client_order_tracker.py`
- **问题**：`_in_flight_orders` 无锁保护，`process_order_update` 用 `safe_ensure_future` 调度，存在 TOCTOU 窗口
- **修复建议**：为关键操作增加 `asyncio.Lock` 保护

### H-5：余额计算非原子性

- **文件**：`hummingbot/connector/connector_base.pyx:399-413`
- **问题**：多策略并发时可能读到相同可用余额，各自下单导致超额
- **修复建议**：余额读取和下单应在同一临界区内完成

### H-6：密码验证时序攻击

- **文件**：`hummingbot/client/config/config_crypt.py:68-79`
- **问题**：用 `==` 而非 `hmac.compare_digest` 比较解密结果
- **修复建议**：改用 `hmac.compare_digest`

---

## 四、MEDIUM（14 个）— 需关注但不紧急

| 编号 | 问题 | 文件 |
|------|------|------|
| M-1 | 远程日志禁用 SSL 验证 (`verify_ssl=False`) | `log_server_client.py:70` |
| M-2 | 缺乏全局日志脱敏过滤器 | 全局架构 |
| M-3 | Auth 类中 api_key/secret 为明文 str | 各 connector `_auth.py` |
| M-4 | `.gitignore` 未覆盖 `.password_verification` | `.gitignore` |
| M-5 | Gateway SSL 证书路径硬编码，不检查过期/篡改 | `gateway_http_client.py:115-118` |
| M-6 | REST 请求无默认超时 | `rest_assistant.py:105` |
| M-7 | 订单创建缺少余额充足性预检查 | `exchange_py_base.py:391-467` |
| M-8 | 缓存订单 TTL 仅 30 秒，可能丢失成交更新 | `client_order_tracker.py:35` |
| M-9 | YAML 解析使用 ruamel.yaml 默认模式 | `config_helpers.py:351` |
| M-10 | 动态模块导入未做验证 | `config_helpers.py:540+` |
| M-11 | 交易所响应数据缺少严格验证 | `in_flight_order.py:222-251` |
| M-12 | 依赖版本约束过于宽松（web3/bip-utils 无版本限制） | `setup.py` |
| M-13 | 加密方案使用 AES-128-CTR（128 位密钥偏弱） | `config_crypt.py:132` |
| M-14 | 安全配置解密后驻留内存，进程生命周期内不清除 | `security.py:27` |

---

## 五、LOW（5 个）

| 编号 | 问题 | 文件 |
|------|------|------|
| L-1 | 订单金额/价格缺少负值验证 | `exchange_py_base.py` |
| L-2 | 余额限制可被绕过（默认空字典） | `connector_base.pyx:361-379` |
| L-3 | WebSocket 无消息完整性验证 | `ws_connection.py` |
| L-4 | API 密钥作为明文实例属性存储 | 各 `binance_exchange.py` 等 |
| L-5 | REST 响应内容类型未严格验证 | `data_types.py:114-130` |

---

## 六、主观恶意后门排查

### 排查结论：未发现主观盗取用户密钥的后门行为

| 排查方向 | 结论 |
|---------|------|
| 网络外传 | 所有 api_key/secret/private_key 仅发送到对应交易所 API，无第三方外传 |
| 隐蔽通道 | 无 DNS 隧道、无隐蔽 WS、无文件上传通道 |
| 混淆/编码 | 所有 base64/hex 编码仅用于本地签名计算，未外传 |
| 依赖投毒 | 全部依赖来自 PyPI 官方源，无拼写混淆，无 GitHub URL 安装 |
| 动态代码加载 | `importlib` 仅加载项目自身模块，无远程代码加载 |
| 遥测/数据收集 | 仅上报匿名交易量指标（可关闭），不含密钥/钱包地址 |
| 第三方域名 | 所有外部域名均为合法交易所/区块链节点/公开数据服务 |

### 需警惕的非后门风险点

| 问题 | 文件 | 说明 |
|------|------|------|
| pickle 反序列化远程数据 | `remote_api_order_book_data_source.py:65` | CoinAlpha 服务器被攻破时可执行任意代码（该类当前未被引用） |
| FoxBit 使用 eval() 解析 WS 数据 | `foxbit_utils.py:70` | 中间人注入恶意数据可执行任意代码，应改用 json.loads() |
| 远程日志禁用 SSL 验证 | `log_server_client.py:70` | 中间人可截获/篡改日志传输 |

---

## 七、遥测机制详情

Hummingbot 存在以下数据收集行为，均**可关闭**且**不含密钥**：

| 机制 | 目标域名 | 上报内容 | 可关闭 |
|------|---------|---------|--------|
| 匿名交易量指标 | `api.coinalpha.com/reporting-proxy-v2/client_metrics` | source、instance_id（随机）、exchange 名称、version、交易量 | 是（`AnonymizedMetricsDisabledMode`） |
| 远程日志 | `api.coinalpha.com/reporting-proxy-v2/` | 日志条目（错误/网络消息标记 `do_not_send: True`） | 是 |
| Parrot 奖励 | `api.hummingbot.io/bounty/` | 仅 GET 请求获取活动信息 | 是 |

---

## 八、修复优先级建议

### 立即修复（实盘前必须）

1. 将 `gateway_use_ssl` 默认值改为 `True`
2. 修复 Decibel API Key 明文存储 Bug
3. Aevo WS 认证改为签名模式

### 尽快修复

4. 为 `ClientOrderTracker` 关键操作加 `asyncio.Lock`
5. 在 `_create_order` 中增加余额充足性预检查
6. 密码验证改用 `hmac.compare_digest`

### 计划中

7. 添加全局日志脱敏过滤器
8. 依赖版本锁定（特别是 `web3`、`bip-utils`、`cryptography`）
9. 加密方案升级 AES-128 → AES-256
10. 移除 `eval()` 和 `pickle.loads()` 使用

---

## 九、安全性验证方法

### 1. 验证 API Key 是否泄漏

```bash
# 检查配置文件是否加密
grep -r "api_key" conf/connectors/ --include="*.yml" | head -5

# 检查 git 历史是否意外提交了密钥
git log -p --all -S "api_key" -- "*.yml" | head -50

# 检查 .gitignore 覆盖
git status conf/.password_verification
```

### 2. 验证网络传输安全

```bash
# 检查 Gateway 是否使用 SSL
grep -r "gateway_use_ssl" conf/ --include="*.yml"

# 抓包验证 Gateway 通信是否加密
tcpdump -i any -w gateway.pcap port 15888
```

### 3. 自动化安全扫描

```bash
# 依赖漏洞扫描
pip install safety pip-audit
safety check -r requirements.txt
pip-audit -r requirements.txt

# 代码静态分析
pip install bandit semgrep
bandit -r hummingbot/ -f json -o bandit_report.json
semgrep --config auto hummingbot/

# 密钥泄漏扫描
pip install trufflehog
trufflehog git file://. --only-verified
```

### 4. 运行时验证

- 启动后检查日志输出中是否包含 API Key 原文
- 使用只读 API Key 测试，确认不会执行交易操作
- 设置 `balance_asset_limit` 限制每资产最大使用量
- 在隔离网络环境中运行，确认 Gateway 通信走 HTTPS
