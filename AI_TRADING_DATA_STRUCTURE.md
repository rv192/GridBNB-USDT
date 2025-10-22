# AI自主交易系统数据结构设计与落地规划（适配 GridBNB-USDT）

更新日期: 2025-10-22
版本: 2.1（以“数据源优先”的分阶段规划重构，仅文档，不含代码）

---

目标
- 围绕 AITradingContext 字段，优先把“数据源/成本/频率/缓存/降级”落实，再开展策略与执行。
- 以阶段里程碑交付“数据就绪度”，每一阶段完成后再解锁对应策略能力。

---

一、数据契约概览（保持不变，供参照）

AITradingContext（输入，v2.0）最小骨架（示例）
{
  "version": "2.0",
  "context_id": "uuid",
  "timestamp": 1698000000123,
  "symbol": "BNB/USDT",
  "timeframe": "3m",
  "market_core": {
    "price": {"current": 598.15, "open_24h": 610.0, "high_24h": 620.5, "low_24h": 595.2, "change_percent_24h": -1.18, "quote_volume_24h": 9387156.2},
    "orderbook_top": {"best_bid": [598.10, 125.3], "best_ask": [598.20, 89.7], "spread": 0.10, "spread_pct": 0.017},
    "last_candle": {"tf": "3m", "open": 599.1, "high": 600.2, "low": 597.8, "close": 598.2, "volume": 1523.4, "closed_at": 1698000000000}
  },
  "grid_state": {"base_price": 600.0, "grid_size_pct": 0.02, "threshold_pct": 0.004, "upper_band": 612.0, "lower_band": 588.0, "position_pct": 0.655, "risk_state": "ALLOW_ALL"},
  "account_core": {"balances": {"base_free": 2.35, "quote_free": 1250.5}, "total_assets_quote": 850.25},
  "indicators": {"rsi_14": 45.2, "ema_20": 608.2, "atr": 8.7},
  "constraints": {"min_notional": 10.0, "lot_step_base": 0.001, "price_tick": 0.01},
  "plus_derivatives": {"funding_rate": null, "open_interest": null},
  "plus_sentiment": {"fear_greed": null},
  "plus_macro": {"btc_dominance": null, "usd_index": null},
  "quality": {"stale_flags": [], "missing_optional": [], "source_watermark": {"binance": 1698000000060}}
}

AITradingDecision（输出，v1.0）保留不变，详见上一版文档（action: adjust_grid/place_order/cancel/...）。

---

二、面向“数据源/成本”的分阶段路线图（字段为中心）

说明
- 每一阶段完成后，才解锁下一阶段功能。优先级按“免费→低成本→付费”。
- 每一阶段给出：覆盖字段、来源、成本、刷新频率、缓存TTL、速率预算、验收标准。

阶段 D0（免费｜现货核心｜强依赖，必须先完成）
- 覆盖字段
  - market_core.price: current/open_24h/high_24h/low_24h/change_percent_24h/quote_volume_24h
  - market_core.orderbook_top: best_bid/best_ask/spread/spread_pct（Top 5 档可选）
  - market_core.last_candle: 1m/3m/5m 可配置
  - account_core: balances(total/free)、open_orders（放入 grid_state.open_orders）
  - grid_state: base_price/grid_size_pct/threshold_pct/upper_band/lower_band/position_pct/risk_state/recent_trades
  - indicators: rsi_14, ema_20, atr, bb(upper/middle/lower)
  - constraints: min_notional/lot_step_base/price_tick（来自 markets 元数据）
- 来源/接口
  - ccxt.binance（async）：fetch_ticker、fetch_order_book(limit=5/20)、fetch_ohlcv、fetch_balance、fetch_open_orders、load_markets
  - 计算：RSI/EMA/ATR/BB（基于最近 N 根 OHLCV）
- 成本：免费
- 刷新/TTL/速率预算（单符号建议）
  - ticker: 1-2s；TTL=2s
  - orderbook: 1-2s；TTL=2s（Top5 即可）
  - ohlcv(1m/3m/5m): 收盘触发；TTL=1 根周期
  - balances/open_orders: 10-30s 或成交事件；TTL=15-60s
  - markets: 启动加载，失败时或每1-6小时热更新
- 验收标准
  - N=3 符号、2s 心跳下，分钟内成功率>99%，无超配额；
  - 所有字段均有值或在 quality.missing_optional 标注；
  - 指标计算对固定样本对齐标准结果（单测通过）。

阶段 D1（免费｜交易所增强）
- 覆盖字段（AITradingContext.plus_* 与 market_data 深化）
  - plus_derivatives.funding_rate（USDM合约）
  - plus_derivatives.open_interest（USDM合约）
  - market_core 深化：orderbook 多档（Top 20）、aggregated trades（用于 volume_ratio/anomaly）
  - volume 分析：volume_ratio/volume_price_divergence/anomaly_score（自计算）
- 来源/接口
  - Binance Futures REST（免费）：
    - funding rate: /fapi/v1/fundingRate（或 ccxt 对应原生方法）
    - open interest: /fapi/v1/openInterest
    - 辅助（可选）：/futures/data 全局多空比
  - ccxt.binance：fetch_trades(symbol, limit=)
- 成本：免费
- 刷新/TTL/速率预算
  - funding_rate: 8h；TTL=8h
  - open_interest: 10-60s；TTL=10-60s
  - trades: 2-10s；TTL=5s
  - 多档深度: 2-5s；TTL=3s
- 验收标准
  - 字段完整率>95%；
  - 与第三方对照抽检 funding_rate/OI 误差<1%；
  - 速率预算在 Binance 1200 权重/分钟内可控。

阶段 D2（免费/低成本｜外部增强）
- 覆盖字段
  - plus_sentiment.fear_greed（Alternative.me｜免费/日更）
  - plus_macro.btc_dominance/total_market_cap/total_volume_24h（CoinGecko｜免费/小时）
  - plus_macro.usd_index（第三方宏观源或手动维护，低频）
  - correlation_data（内部基于多符号 OHLC 自计算为主，外部作为参考）
- 来源/接口
  - Alternative.me Fear&Greed（REST，免Key）
  - CoinGecko /global（REST，免Key，速率限制友好）
  - 自计算：基于 D0/D1 OHLC，滚动相关性与分散度
- 成本：免费（CryptoCompare 社交指标可选：免费+Key，或低成本）
- 刷新/TTL/速率预算
  - Fear&Greed: 24h；TTL=24h
  - CoinGecko Global: 1-4h；TTL=1-4h
  - 相关性：30-60m 计算；TTL=30-60m
- 验收标准
  - 外部接口可用性>99%；
  - 缓存命中率>80%；
  - 缺失不阻塞主流程，quality.missing_optional 标注覆盖。

阶段 D3（付费｜机构级增强，可选）
- 覆盖字段
  - onchain_data: exchange_flows/whale_activity/network_metrics（Glassnode/Moralis/Santiment）
  - liquidation_monitor: 市场强平密度/距离（第三方如 Coinglass 等，通常付费）
  - performance_metrics: 夏普/回撤/胜率/盈利因子（自计算，需持久化历史）
- 来源/接口
  - Glassnode（部分免费，深度指标付费）、Moralis（免费额度有限）、Santiment（付费）
  - Liquidation feeds（多为付费或限流严格）
- 成本：$100-$500/月（按套餐）
- 刷新/TTL：1h（链上）、1-5m（强平监控）、日级（绩效）
- 验收标准
  - 成本/收益评估明确；
  - 指标对策略收益与风控的边际贡献有统计显著性。

---

三、字段→来源→成本 清单（Field-to-Source Matrix）

market_core
- price.current/open_24h/high_24h/low_24h/change_percent_24h/quote_volume_24h
  - 来源: ccxt.fetch_ticker（Binance 24h 统计）
  - 成本: 免费；频率: 1-2s；TTL: 2s
- orderbook_top.best_bid/best_ask/spread/spread_pct（Top5/Top20）
  - 来源: ccxt.fetch_order_book(symbol, limit=5/20)
  - 成本: 免费；频率: 1-2s；TTL: 2-3s
- last_candle(tf=1m/3m/5m)
  - 来源: ccxt.fetch_ohlcv(symbol, tf, limit=1000)
  - 成本: 免费；触发: 收盘；TTL: 1 根周期

grid_state（内部）
- base_price/grid_size_pct/threshold_pct/upper_band/lower_band
  - 来源: GridTrader 内部计算
  - 频率: 事件/心跳（3-5s）
- position_pct/risk_state/throttling/recent_trades/open_orders
  - 来源: RiskManager/OrderTracker/GridTrader；open_orders 来自 ccxt.fetch_open_orders
  - 频率: 事件/心跳（3-10s）

account_core
- balances(total/free)
  - 来源: ccxt.fetch_balance（合并理财余额，若启用）
  - 频率: 10-30s 或成交后；TTL: 15-60s
- total_assets_quote
  - 来源: ExchangeClient.calculate_total_account_value（已有）；频率同上

indicators（自计算）
- rsi_14/ema_20/atr/bb
  - 来源: OHLCV（D0）
  - 频率: 收盘或30-60s；TTL: 1 根周期

constraints
- min_notional/lot_step_base/price_tick
  - 来源: ccxt.load_markets() 中的精度/限额
  - 频率: 启动+每1-6h 或失败回退时刷新

plus_derivatives（D1）
- funding_rate
  - 来源: Binance Futures /fapi/v1/fundingRate（或 ccxt 原生）；免费；8h
- open_interest
  - 来源: Binance Futures /fapi/v1/openInterest；免费；10-60s

plus_sentiment（D2）
- fear_greed
  - 来源: Alternative.me；免费；24h
- social（可选）
  - 来源: CryptoCompare；免费/低成本；1h

plus_macro（D2）
- btc_dominance/total_market_cap/total_volume_24h
  - 来源: CoinGecko /global；免费；1-4h
- usd_index
  - 来源: 外部宏观源（需选型）；免费/低成本；日级

onchain_data（D3，可选）
- exchange_flows/whale_activity/network_metrics
  - 来源: Glassnode/Moralis/Santiment；付费；1h

liquidation_monitor（D3，可选）
- liquidation_distances/min_distance 等
  - 来源: 市场数据供应商（多为付费）；1-5m

performance_metrics（D3，自计算）
- sharpe/sortino/calmar/max_drawdown/win_rate/profit_factor
  - 来源: 交易历史与账户净值序列；日级/小时级

---

四、数据采集与分发机制（时机/频率）

- 心跳（2-5s/符号，D0/D1 核心）
  - 拉取 ticker、orderbook_top、必要余额/订单、更新 grid_state；
  - 生成“轻量上下文”交付 AI（或缓存供拉取）。
- 事件（即时）
  - 订单成交/失败、风控变化、网格重算、S1 触发、spread/volume 异常；
  - 生成“事件上下文”。
- 收盘（1m/3m/5m）
  - 计算指标、更新 last_candle；
  - 推送“收盘上下文”。
- 批处理（D1/D2/D3）
  - funding_rate（8h）、open_interest（10-60s）、CoinGecko（1-4h）、Fear&Greed（24h）、链上（1h）。

缓存与降级
- L1 内存（秒级 TTL）、L2 本地（SQLite/文件，分钟/小时级）、L3 永久（历史）。
- 缺失/过期字段加入 quality.missing_optional/stale_flags；AI 侧需能容错。

速率与配额（Binance 1200 权重/分钟参考）
- 预算示例（每符号）
  - ticker 30/min、orderbook 30/min、trades 6-30/min、ohlcv 1/min、balance+orders 2-6/min → 合计<<1200。

---

五、按阶段的实施任务（只列规划，不写代码）

D0 任务（现货核心，免费）
- 接口对齐：fetch_ticker/order_book/ohlcv/balance/open_orders/load_markets。
- 指标计算器：RSI/EMA/ATR/BB（基于缓存 OHLCV）。
- 心跳与事件：定义 2-5s 心跳、事件触发点与上下文构建。
- 缓存：内存缓存 + 本地文件（最近 N=500-1000 根K线）。
- 验收：三符号稳定运行 24h，无明显超配额/抖动。

D1 任务（交易所增强，免费）
- 接 Futures REST：funding_rate/open_interest；
- trades/深度增强：volume_ratio、异常检测；
- 速率与缓存：OI 10-60s，trades 2-10s，多档深度 2-5s；
- 验收：字段对齐、对照抽检误差<1%。

D2 任务（外部增强，免费/低成本）
- 接 Alternative.me、CoinGecko；
- 建立小时级批任务与缓存，429 退避与重试；
- 相关性自计算（多符号 OHLCV）。

D3 任务（付费增强，可选）
- 供应商选型与成本评估（Glassnode/Moralis/Santiment/Coinglass）。
- 最小闭环：链上流量/强平监控对策略的边际收益评估。

放权节奏
- 仅在 D0 100% 达标后，开放“调整网格参数”的 AI 决策；
- D1 达标后，开放“仓位/阈值动态调节与成交量驱动”的AI增强；
- D2/D3 达标后，再考虑“情绪/宏观/链上”的权重引入。

---

六、验收清单（Data Readiness Gate）

- 完整性：阶段字段覆盖率≥既定范围，缺失字段明确标注且不阻塞流程。
- 质量：采集失败率<1%，延迟<1s（D0/D1），缓存命中>80%。
- 配额：所有外部 API 调用均低于限额 70%，保留弹性。
- 可观测：为每个数据源暴露延迟/错误率/最后更新时间指标。
- 可回退：外部数据缺失时，AI 仍能基于 D0/D1 核心数据运行。

---

七、附：策略与决策（简述）

- 本文以“数据源优先”为纲，AITradingDecision 与执行护栏保持上一版设计：
  - 风控优先（RiskManager）、交易约束校验（min_notional/step/tick/余额）、网格共存。
  - 触发时机：心跳/事件/收盘/批处理。
  - 幂等与TTL：context_id/decision_id 去重与超时丢弃。

---

八、最小样例（留存，与上一版一致）

最小上下文（L0）
{
  "version": "2.0",
  "context_id": "ctx-001",
  "timestamp": 1698000000123,
  "symbol": "BNB/USDT",
  "timeframe": "3m",
  "market_core": {
    "price": {"current": 598.15, "change_percent_24h": -1.18},
    "orderbook_top": {"best_bid": [598.10, 125.3], "best_ask": [598.20, 89.7], "spread": 0.10, "spread_pct": 0.017},
    "last_candle": {"tf": "3m", "open": 599.1, "high": 600.2, "low": 597.8, "close": 598.2, "volume": 1523.4, "closed_at": 1698000000000}
  },
  "grid_state": {"base_price": 600.0, "grid_size_pct": 0.02, "threshold_pct": 0.004, "upper_band": 612.0, "lower_band": 588.0, "position_pct": 0.655, "risk_state": "ALLOW_ALL"},
  "account_core": {"balances": {"base_free": 2.35, "quote_free": 1250.5}, "total_assets_quote": 850.25},
  "indicators": {"rsi_14": 45.2, "ema_20": 608.2, "atr": 8.7},
  "constraints": {"min_notional": 10.0, "lot_step_base": 0.001, "price_tick": 0.01}
}

最小决策（建议模式）
{
  "version": "1.0",
  "decision_id": "dec-001",
  "timestamp": 1698000000456,
  "symbol": "BNB/USDT",
  "mode": "advisory",
  "ttl_ms": 30000,
  "action": "adjust_grid",
  "adjust_grid": {"new_grid_size_pct": 0.025},
  "risk": {"expected_risk_usd": 0, "confidence": 0.65},
  "guards": {"require_risk_state_any_of": ["ALLOW_ALL"]},
  "telemetry": {"model": "rule-baseline", "reason": "带宽收缩，收紧网格提高成交频率"}
}

---

备注
- 本文档为“数据结构与数据源落地规划”，不包含代码改动。
- 实施时需把“字段→来源→成本→频率→缓存/降级→验收”作为每次迭代的唯一准入标准。
