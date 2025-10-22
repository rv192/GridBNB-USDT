# AI自主交易系统数据结构设计与落地规划（适配 GridBNB-USDT）

更新日期: 2025-10-22  
版本: 2.0（规划与数据结构，仅文档，不含代码）

---

## 一、项目现状与AI接入缺口

- 现状综述
  - 交易层: main.py 启动多币种 GridTrader（trader.py），ExchangeClient（exchange_client.py）封装了币安API（含缓存、时间同步、理财转账、总资产跟踪）。
  - 策略层: AdvancedRiskManager（risk_manager.py）限制仓位，PositionControllerS1（position_controller_s1.py）基于52日高低点调仓。
  - 支撑层: OrderTracker（order_tracker.py）记录历史与节流，TradingMonitor（monitor.py）采集状态。
  - 界面层: web_server.py 提供监控页面和 JSON API（/api/status, /api/symbols, /api/logs）。
  - 运行特征: asyncio 并发、Pydantic 配置、Docker 部署、PushPlus 告警。

- 已有数据能力
  - 实时价格、余额、网格参数（基准价/网格大小/阈值）、交易历史（最近N笔）、S1最高/最低、仓位比例、全资产（现货+理财）。
  - 监控API可返回单币种当前状态，便于前端展示。

- AI接入缺口
  - 缺少统一且可扩展的“AI上下文数据结构”（输入契约）与“AI决策契约”（输出契约）。
  - 缺少衍生品数据（资金费率/持仓量）、订单簿微观结构、宏观与情绪数据。
  - 无标准的数据更新时机/机制（心跳、事件驱动、K线收盘）与低延迟数据分发通道。
  - 无安全护栏后的AI落地路径（风控校验、交易合规、网格/AI冲突调和、回退机制）。

---

## 二、设计目标与原则

- 标准化: 用版本化的 AITradingContext（输入）和 AITradingDecision（输出）贯穿内外部模块。
- 分层可选: 核心字段必须、扩展字段可选，按“数据就绪等级”逐步上线。
- 事件驱动: 以低延迟、幂等等特性将“时机”作为一等公民（心跳+事件）。
- 安全先行: 风控优先、网格策略不被破坏、可回滚、可审计。
- 低耦合: 通过总线/接口适配，AI可以内嵌、也可以外置为服务。

---

## 三、AITradingContext v2.0（输入数据契约）

说明:
- 模块化：core 为必须，plus 为扩展（可逐步接入）。
- 每个字段标注来源与刷新建议（摘要在本节结尾）。

示例骨架（最小可用 + 可扩展位）:

```json
{
  "version": "2.0",
  "context_id": "uuid-或-递增序列",
  "timestamp": 1698000000123,
  "symbol": "BNB/USDT",
  "timeframe": "3m",
  "producer": "ai-data-collector",
  "latency_ms": 35,

  "market_core": {
    "price": {
      "current": 598.15,
      "open_24h": 610.0,
      "high_24h": 620.5,
      "low_24h": 595.2,
      "change_percent_24h": -1.18,
      "quote_volume_24h": 9387156.2
    },
    "orderbook_top": {
      "best_bid": [598.10, 125.3],
      "best_ask": [598.20, 89.7],
      "spread": 0.10,
      "spread_pct": 0.017
    },
    "last_candle": {
      "tf": "3m",
      "open": 599.1,
      "high": 600.2,
      "low": 597.8,
      "close": 598.2,
      "volume": 1523.4,
      "closed_at": 1698000000000
    }
  },

  "grid_state": {
    "base_price": 600.0,
    "grid_size_pct": 0.02,
    "threshold_pct": 0.004,
    "upper_band": 612.0,
    "lower_band": 588.0,
    "target_order_amount_quote": 85.0,
    "position_pct": 0.655,
    "s1_daily_high": 690.0,
    "s1_daily_low": 675.0,
    "risk_state": "ALLOW_ALL",
    "throttling": {
      "cooldown_sec": 5,
      "recent_failures": 0
    },
    "open_orders": [
      {"id": "o1", "side": "buy", "type": "limit", "price": 595.0, "amount": 1.0, "filled": 0.5}
    ],
    "recent_trades": [
      {"ts": 1697999900123, "side": "buy", "price": 597.0, "amount": 0.2, "profit": 0.0}
    ]
  },

  "account_core": {
    "balances": {
      "base_free": 2.35,
      "quote_free": 1250.5,
      "base_total": 2.50,
      "quote_total": 1300.5
    },
    "total_assets_quote": 850.25
  },

  "indicators": {
    "rsi_14": 45.2,
    "ema_20": 608.2,
    "atr": 8.7,
    "bb": {"upper": 625.8, "middle": 610.5, "lower": 595.2}
  },

  "constraints": {
    "min_notional": 10.0,
    "lot_step_base": 0.001,
    "price_tick": 0.01
  },

  "plus_derivatives": {
    "funding_rate": 7.08e-06,
    "funding_rate_percentile": 85,
    "open_interest": 25120.38
  },

  "plus_sentiment": {
    "fear_greed": 35,
    "sentiment_overall": 0.68
  },

  "plus_macro": {
    "btc_dominance": 57.5,
    "usd_index": 105.2
  },

  "quality": {
    "stale_flags": [],
    "missing_optional": [],
    "source_watermark": {"binance": 1698000000060}
  }
}
```

字段来源与刷新建议（摘要）:
- market_core
  - 来源: ExchangeClient.fetch_ticker / fetch_order_book / fetch_ohlcv（新增3m配置）。
  - 刷新: 价格/盘口 1-2秒；K线在收盘事件触发。
- grid_state
  - 来源: GridTrader 内部状态、OrderTracker（历史）、PositionControllerS1（高低点）、RiskManager（risk_state）。
  - 刷新: 心跳3-5秒 + 事件触发（下单/成交/风控变化/网格重算）。
- account_core
  - 来源: ExchangeClient.fetch_balance（结合理财转账信息）。
  - 刷新: 余额变动/订单成交后立即，心跳降频10-30秒。
- indicators
  - 来源: 自计算（RSI/EMA/ATR/BB），输入 last_n_ohlcv（数据收集器缓存）。
  - 刷新: K线收盘或每30-60秒重算。
- constraints
  - 来源: ccxt markets 元数据（启动时加载，必要时热更新）。
  - 刷新: 稀疏更新（小时级或发现下单失败时刷新）。
- plus_derivatives / plus_sentiment / plus_macro
  - 来源: 币安合约API、Alternative.me、CoinGecko 等（可选）。
  - 刷新: 资金费率 8h；OI 10-60秒；情绪/宏观 小时/日级。
- quality
  - 来源: 数据采集器自评估，标注缺失/过期。

数据就绪等级:
- L0 Core: market_core、grid_state、account_core、indicators、constraints。
- L1 Exchange Plus: orderbook 多档深度、更多TF K线、流动性估计。
- L2 External Plus: derivatives、sentiment、macro、on-chain（可选）。

---

## 四、AITradingDecision v1.0（AI输出决策契约）

说明:
- 分“建议模式”和“执行模式”。
- 强制带有效期、风险信息与保护条件，所有指令经过风控与交易对约束校验。

示例（调整网格）:

```json
{
  "version": "1.0",
  "decision_id": "uuid",
  "timestamp": 1698000000456,
  "symbol": "BNB/USDT",
  "mode": "advisory",
  "ttl_ms": 30000,
  "action": "adjust_grid",
  "adjust_grid": {
    "new_grid_size_pct": 0.025,
    "new_base_price": 599.2,
    "new_threshold_pct": 0.0035
  },
  "risk": {
    "expected_risk_usd": 120.0,
    "confidence": 0.72,
    "stop_loss": 588.0,
    "take_profit": 612.0
  },
  "guards": {
    "require_risk_state_any_of": ["ALLOW_ALL", "ALLOW_SELL_ONLY"],
    "require_price_between": [590.0, 605.0]
  },
  "telemetry": {"model": "transformer-v4", "reason": "波动率下降+RSI中性，收紧网格"},
  "trace_parent": "trace-id"
}
```

place_order 分支（示例）:

```json
{
  "action": "place_order",
  "place_order": {
    "side": "buy",
    "type": "limit",
    "quote_amount": 85.0,
    "price": 595.0,
    "post_only": true,
    "time_in_force": "GTC",
    "reduce_only": false
  }
}
```

约束与落地要点:
- 所有变更由 RiskManager 二次校验（仓位上限、最小底仓、连续失败保护、冷却时间）。
- 交易约束：min_notional、lot_step_base、price_tick、余额、滑点/价差保护。
- 网格优先：默认 AI 先“调整网格参数”，再考虑“直接下单”。
- 有效期与幂等：超时丢弃；decision_id 去重。

---

## 五、数据更新机制与触发时机

- 心跳驱动（每3-5秒/符号）
  - 刷新 market_core 与 grid_state 轻量快照，推送或缓存供AI拉取。
- 事件驱动（即时）
  - 订单成交/失败、风控变更、网格重算、S1触发、spread/volume 异常。
- K线收盘驱动（按1m/3m/5m等配置）
  - 重算 indicators 并推送“收盘上下文”。
- 外部数据批驱动
  - 资金费率、持仓量、情绪、宏观按各自周期拉取，更新缓存并触发“增强上下文”。
- 传输方式
  - 内嵌函数回调（generate_decision(context)）。
  - 本地HTTP回调（POST /ai/decision 或 /ai/context）。
  - Pub/Sub（Redis频道 ai:context:{symbol} / ai:decision）。
- 幂等与时序
  - context_id/decision_id 唯一，Watermark/序列号保证只处理最新上下文。

---

## 六、与现有代码的落地集成（不写代码，列改造点）

- 新增数据与契约模型
  - ai_schema.py：Pydantic 模型（AITradingContext、AITradingDecision）。
  - ai_data_collector.py：聚合 ExchangeClient、GridTrader、OrderTracker、RiskManager、PositionControllerS1；计算指标；构建 constraints。
  - ai_bus.py：事件发布/订阅（内存队列，后续可接 Redis），幂等与优先级（事件>收盘>心跳）。

- Trader 侧的AI挂载点
  - 在主循环与关键事件处生成上下文（心跳/事件/K线），经 ai_bus 分发并收集决策。
  - AI 决策网关：risk_manager.check + 交易约束校验 + 网格冲突调和。
  - 决策执行器：将 adjust_grid/ place_order / cancel / rebalance / pause 映射到现有能力，严格单位与步长对齐。

- Web与可观测性
  - 只读调试接口：/api/ai/context?symbol，/api/ai/last-decision。
  - 全链路日志：记录 context_id / decision_id 与执行结果。

- 配置与开关
  - Settings 增加开关与参数（启用AI、心跳频率、timeframe、外部服务地址、Redis）。
  - 外部数据API Key 可选，默认禁用。

---

## 七、需要新增的数据与优化（按优先级）

- 必选（阶段1）
  - 盘口 Top 1-5 档（best_bid/ask + spread）。
  - K线 last_candle（默认3m，可配）。
  - 交易所约束（最小名义、步长、tick）。
  - 基础技术指标（RSI/EMA/ATR/BB）。
  - 网格状态细化（阈值、上下轨、节流状态、risk_state）。

- 建议（阶段2）
  - 资金费率、持仓量（合约）。
  - 成交量分析（volume_ratio、价量背离）。
  - 情绪与宏观（恐慌贪婪、BTC主导率、USD指数）。
  - 订单簿流动性估计（冲击成本、累积深度）。

- 进阶（阶段3）
  - 社交/链上。
  - 相关性与对冲建议（组合层）。
  - 策略绩效面板（实时夏普、回撤、胜率、利润因子）。

---

## 八、安全与护栏

- 风控优先：ALLOW_ALL/ALLOW_SELL_ONLY/ALLOW_BUY_ONLY、连续失败熔断、冷却时间、最小底仓、最大仓位比例。
- 交易约束：min_notional、lot_step、price_tick、余额、滑点/价差保护。
- 网格与AI共存：优先调整网格；直接下单需白名单与上限；冲突订单先处理。
- 审计与回滚：全链路日志、TTL与幂等、异常回退到纯网格模式。

---

## 九、测试与验证

- 单元测试：契约模型校验、交易约束/风控校验、指标计算正确性。
- 集成测试：事件→上下文→AI模拟→决策→风控→执行闭环；边界场景（余额不足、跳变、数据缺失）。
- 回测/前测：重放 data/trader_state_*.json 与 trade_history.json；对照“网格+AI叠加”与“纯网格”。

---

## 十、性能与成本

- 缓存分层：内存（秒级实时）、本地文件/SQLite（分钟/小时）、可选Redis（共享）。
- API配额：请求去重、指数退避、外部数据集中批任务与长TTL缓存。
- 延迟目标：端到端 < 1 秒（事件路径优先）。

---

## 十一、策略扩展建议（与网格互补）

- 波动率收缩-扩张（ATR/BB 带宽）→ 动态 grid_size_pct。
- 趋势过滤（EMA/ADX）→ 决定“仅卖/仅买/全开”。
- 盘口微观（spread/不对称）→ 动态阈值与下单偏移。
- 资金费率/OI 偏差 → 调整仓位上限或建议减仓。
- 情绪极端 → 网格宽度扩张/收缩。

---

## 十二、最小可行样例

最小上下文（L0）:

```json
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
```

最小决策（建议模式）:

```json
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
```

---

## 十三、实施路线图（不写代码的任务拆解）

- 第1周：冻结 L0 字段词典与单位，确定心跳/事件/K线触发模型与缓存策略。
- 第2周：内嵌AI占位（仅返回最小决策），在心跳与关键事件处生成 L0 上下文；仅支持 adjust_grid。
- 第3-4周：扩展到 place_order/cancel/rebalance；上线 /api/ai/context 调试接口与延迟/异常监控。
- 第5-6周：接入 L1/L2（资金费率、OI、情绪、宏观），支持外置AI服务（HTTP/PubSub）、幂等与回退。

---

## 十四、注意事项与风险

- 网格策略可持续运行优先，AI为叠加器而非破坏者。
- 明确“建议模式”与“执行模式”，分阶段放权。
- 时间同步与时序一致性（沿用 ExchangeClient 时间同步并透出时间戳）。
- 外部数据缺失不阻塞AI：quality.missing_optional 标注降级。

---

本文档为规划与数据结构说明，不包含任何代码改动。与实现时请严格对齐契约字段与时机设计，使 AI 决策路径可控、可审计、可回滚。
