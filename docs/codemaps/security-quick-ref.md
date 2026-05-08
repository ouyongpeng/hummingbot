# Hummingbot 安全审计报告

> 最后更新：2026-04-22
> 完整报告见：[achievements/2026-04-22-security-audit-report.md](../achievements/2026-04-22-security-audit-report.md)

---

## 速查：CRITICAL 问题（3 个）

| # | 问题 | 文件 | 影响 |
|---|------|------|------|
| C-1 | Gateway 默认 HTTP 传输私钥 | `gateway_http_client.py:634` | 同网段嗅探可盗取链上资产 |
| C-2 | Aevo WS 认证明文传输 API Secret | `aevo_perpetual_auth.py:85` | TLS 截获即泄露 Secret |
| C-3 | Decibel API Key 明文存储 Bug | `client_config_map.py:646` | 配置文件中 Key 无加密 |

## 速查：HIGH 问题（6 个）

| # | 问题 | 文件 |
|---|------|------|
| H-1 | Hyperliquid/Derive 私钥明文加载到内存 | `*_auth.py` |
| H-2 | Pacifica 硬编码模拟私钥 | `pacifica_perpetual_rate_source.py:58` |
| H-3 | GRVT 明文传输 API Key | `grvt_perpetual_auth.py:107` |
| H-4 | 订单跟踪器竞态条件 | `client_order_tracker.py` |
| H-5 | 余额计算非原子性 | `connector_base.pyx:399` |
| H-6 | 密码验证时序攻击 | `config_crypt.py:68` |

## 后门排查结论

**未发现主观盗取密钥的后门行为。** 所有 api_key/secret 仅用于交易所 API 签名，无第三方外传。需警惕的非后门风险点：

| 问题 | 文件 | 说明 |
|------|------|------|
| pickle 反序列化远程数据 | `remote_api_order_book_data_source.py:65` | CoinAlpha 被攻破时可执行任意代码 |
| FoxBit eval() 解析 WS 数据 | `foxbit_utils.py:70` | 中间人注入可执行任意代码 |
| 远程日志禁用 SSL 验证 | `log_server_client.py:70` | 中间人可截获日志 |

## 遥测机制

| 机制 | 目标 | 上报内容 | 可关闭 |
|------|------|---------|--------|
| 匿名交易量指标 | `api.coinalpha.com` | 随机 ID + 交易所名 + 交易量 | 是 |
| 远程日志 | `api.coinalpha.com` | 日志条目 | 是 |
| Parrot 奖励 | `api.hummingbot.io` | 仅 GET 获取活动信息 | 是 |

## 安全验证方法

```bash
# 依赖漏洞扫描
pip install safety pip-audit
safety check -r requirements.txt

# 代码静态分析
bandit -r hummingbot/ -f json -o bandit_report.json
semgrep --config auto hummingbot/

# 密钥泄漏扫描
trufflehog git file://. --only-verified

# 检查 Gateway SSL
grep -r "gateway_use_ssl" conf/ --include="*.yml"
```