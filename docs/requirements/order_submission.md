# 009 Order Submission Requirements

> 建议落库路径：`docs/requirements/order_submission.md`  
> 阶段编号：009  
> 模块建议名称：`order_submission`  
> 中文名称：订单提交层  
> 上游阶段：008 ExecutionPreparation  
> 下游阶段：010 OrderStatusSync / FillSync / Reconcile，后续单独定义  
> 当前文档定位：009 正式需求，不是实现计划，不是代码指令。

---

## 1. 背景

008 ExecutionPreparation 已将 `ApprovedOrderIntent` 冻结为 `PreparedOrderIntent`。

`PreparedOrderIntent` 的语义是：

```text
已通过风控、
已完成执行准备、
已冻结订单参数、
已生成 client_order_id / idempotency_key、
尚未提交交易所的订单请求。
```

009 Order Submission 是系统第一次可能调用 Binance 真实下单接口的阶段。

009 的核心职责不是重新判断策略，也不是重新风控，而是：

```text
只消费 008 冻结好的 PreparedOrderIntent；
只提交一次；
记录提交尝试和交易所响应；
不确定时绝不重复下单。
```

---

## 2. 阶段目标

009 负责读取 008 生成的 `prepared` 状态 `PreparedOrderIntent`，在提交前做最后轻量保护，然后根据 `raw_payload.endpoint_family` 与运行时 active domain 选择 Binance USDⓈ-M Futures 或 COIN-M Futures 下单接口，并把提交尝试和交易所响应落库。

009 必须回答以下问题：

```text
1. 这份 PreparedOrderIntent 是否已经发起过提交？
2. 本次是否真正调用了 Binance 下单接口？
3. 当前运行时 active domain 是 USDS-M 还是 COIN-M？
4. PreparedOrderIntent 所属 domain 是否等于 active domain？
5. 调用的是 fapi 还是 dapi？
6. Binance 是否明确 accepted？
7. Binance 是否明确 rejected？
8. 本地是否无法判断交易所是否收到，即 unknown？
```

009 不负责回答以下问题：

```text
1. 订单是否最终成交。
2. 订单成交了多少。
3. 是否 partial fill / full fill。
4. 手续费是多少。
5. Binance 当前订单状态是否已变化。
6. 是否需要撤单。
7. 是否需要重新同步成交。
8. 是否需要更新真实持仓。
```

这些问题必须放到后续 `OrderStatusSync`、`FillSync`、`Reconcile` 或异常处理阶段。

---

## 3. P0 范围

### 3.1 系统能力层面

009 P0 必须同时具备两条执行提交能力：

```text
1. Binance USDⓈ-M Futures / USDS-M / quantity / fapi。
2. Binance COIN-M Futures / COIN-M / contracts / dapi。
```

两条能力同等重要，均属于 P0。

禁止：

```text
只实现 U 本位。
把 COIN-M / contracts 视为 unsupported。
只写 fapi，不写 dapi。
只写 quantity，不写 contracts。
```

### 3.2 运行时链路层面

虽然 009 必须具备两条提交能力，但系统运行时只能选择一个 active Binance account domain / market type。

active domain 由 env 配置决定：

```text
BINANCE_ACTIVE_MARKET_TYPE=usds_m_futures
```

或：

```text
BINANCE_ACTIVE_MARKET_TYPE=coin_m_futures
```

P0 合法值：

```text
usds_m_futures
coin_m_futures
```

P0 中：

```text
active_account_domain == active_market_type == BINANCE_ACTIVE_MARKET_TYPE
```

009 不允许 U 本位和币本位两条正式交易链路并行运行。

正确口径：

```text
系统能力：USDS-M 与 COIN-M 都支持。
运行实例：只激活一个 active domain。
提交动作：只允许提交属于当前 active domain 的 PreparedOrderIntent。
```

错误口径：

```text
错误 1：009 只做 USDS-M。
错误 2：009 同时运行 USDS-M 与 COIN-M 两条正式下单链路。
错误 3：COIN-M PreparedOrderIntent 在 active=USDS-M 时仍可提交。
错误 4：USDS-M PreparedOrderIntent 在 active=COIN-M 时仍可提交。
```

### 3.3 009 P0 支持能力

```text
1. MARKET order。
2. One-Way Mode。
3. positionSide = BOTH。
4. 使用 PreparedOrderIntent.client_order_id 作为 Binance newClientOrderId。
5. 使用 PreparedOrderIntent.raw_payload 作为提交参数来源。
6. 单次真实提交尝试。
7. accepted / rejected / unknown / failed_before_submit / blocked_before_submit 分类。
8. OrderSubmissionAttempt 落库。
9. AlertEvent 审计。
10. fail-closed。
```

### 3.4 009 P0 明确不做

```text
1. LIMIT / STOP / TAKE_PROFIT / TRAILING_STOP。
2. batch order。
3. 自动重试真实下单。
4. 自动补单。
5. 自动缩单。
6. 撤单。
7. 订单状态轮询。
8. 成交同步。
9. 手续费同步。
10. 真实持仓更新。
11. WebSocket user data stream。
12. 多 active domain 并行运行。
13. 多账户并行。
14. 多交易所。
15. Backtest。
16. UI / Admin。
17. dry-run 交易仿真。
18. 调整 leverage。
19. 调整 margin type。
20. 调整 position mode。
21. 使用 client_order_id 查询订单。
```

---

## 4. Active Domain 强隔离要求

### 4.1 运行时 active domain

009 启动和提交时必须读取：

```text
settings.BINANCE_ACTIVE_MARKET_TYPE
```

并派生：

```text
active_market_type_at_runtime = settings.BINANCE_ACTIVE_MARKET_TYPE
active_account_domain_at_runtime = settings.BINANCE_ACTIVE_MARKET_TYPE
```

P0 只允许：

```text
usds_m_futures
coin_m_futures
```

如果配置不存在或不合法：

```text
不得调用 Binance。
OrderSubmissionAttempt.status = failed_before_submit 或 blocked_before_submit。
reason_code = invalid_active_market_type。
写 AlertEvent。
```

### 4.2 active domain 与 PreparedOrderIntent 必须一致

调用 Binance 前必须校验：

```text
PreparedOrderIntent.market_type == active_market_type_at_runtime
PreparedOrderIntent.account_domain == active_account_domain_at_runtime
```

如果不一致：

```text
不得调用 Binance。
OrderSubmissionAttempt.status = failed_before_submit 或 blocked_before_submit。
reason_code = inactive_market_domain 或 market_domain_mismatch。
写 AlertEvent。
```

这不是“不支持另一个市场”。

这是：

```text
另一个市场当前不是 active domain，因此本次运行不得消费。
```

### 4.3 合法组合矩阵

#### USDS-M active

当：

```text
BINANCE_ACTIVE_MARKET_TYPE=usds_m_futures
```

合法组合必须是：

```text
PreparedOrderIntent.market_type = usds_m_futures
PreparedOrderIntent.account_domain = usds_m_futures
PreparedOrderIntent.quantity_unit = quantity
raw_payload.endpoint_family = fapi
adapter = FapiOrderAdapter
```

禁止：

```text
coin_m_futures PreparedOrderIntent
quantity_unit = contracts
endpoint_family = dapi
DapiOrderAdapter
COIN-M API key / secret
COIN-M base_url
```

#### COIN-M active

当：

```text
BINANCE_ACTIVE_MARKET_TYPE=coin_m_futures
```

合法组合必须是：

```text
PreparedOrderIntent.market_type = coin_m_futures
PreparedOrderIntent.account_domain = coin_m_futures
PreparedOrderIntent.quantity_unit = contracts
raw_payload.endpoint_family = dapi
adapter = DapiOrderAdapter
```

禁止：

```text
usds_m_futures PreparedOrderIntent
quantity_unit = quantity
endpoint_family = fapi
FapiOrderAdapter
USDS-M API key / secret
USDS-M base_url
```

### 4.4 非 active domain 的处理

非 active domain 的 `PreparedOrderIntent` 即使状态是 `prepared`、未过期、参数完整，也不得提交。

必须：

```text
不调用 Binance。
不调用任何 adapter。
创建或更新 OrderSubmissionAttempt。
写 AlertEvent。
保留审计证据。
```

---

## 5. 双市场域提交要求

### 5.1 USDⓈ-M Futures 路径

允许组合：

```text
active_market_type_at_runtime = usds_m_futures
market_type = usds_m_futures
account_domain = usds_m_futures
quantity_unit = quantity
raw_payload.endpoint_family = fapi
```

提交要求：

```text
1. 使用 FapiOrderAdapter。
2. 调用 Binance USDⓈ-M Futures new order endpoint。
3. 使用 raw_payload.quantity 作为 Binance quantity。
4. 使用 PreparedOrderIntent.client_order_id 作为 newClientOrderId。
5. MARKET order 不发送 timeInForce。
6. 不重新计算 quantity。
7. 不重新生成 client_order_id。
8. 不读取 COIN-M API key / secret / base_url。
```

### 5.2 COIN-M Futures 路径

允许组合：

```text
active_market_type_at_runtime = coin_m_futures
market_type = coin_m_futures
account_domain = coin_m_futures
quantity_unit = contracts
raw_payload.endpoint_family = dapi
```

提交要求：

```text
1. 使用 DapiOrderAdapter。
2. 调用 Binance COIN-M Futures new order endpoint。
3. 使用 raw_payload.quantity 作为 Binance quantity。
4. raw_payload.quantity 的语义是 contracts 合约张数。
5. 使用 PreparedOrderIntent.client_order_id 作为 newClientOrderId。
6. MARKET order 不发送 timeInForce。
7. 不把 contracts 转成 base asset。
8. 不把 contracts 转成 quote asset。
9. 不重新计算 contracts。
10. 不重新生成 client_order_id。
11. 不读取 USDS-M API key / secret / base_url。
```

### 5.3 错误组合 fail-closed

以下组合必须在提交前阻断，不得调用 Binance：

```text
1. active=usds_m_futures，但 PreparedOrderIntent.market_type=coin_m_futures。
2. active=coin_m_futures，但 PreparedOrderIntent.market_type=usds_m_futures。
3. active=usds_m_futures，但 endpoint_family=dapi。
4. active=coin_m_futures，但 endpoint_family=fapi。
5. USDS-M / contracts。
6. COIN-M / quantity。
7. endpoint_family=fapi 但 account_domain/market_type 是 COIN-M。
8. endpoint_family=dapi 但 account_domain/market_type 是 USDS-M。
9. 未知 market_type。
10. 未知 account_domain。
11. 未知 quantity_unit。
12. 未知 endpoint_family。
13. market_type / account_domain / quantity_unit / endpoint_family / active domain 互相不一致。
```

阻断后必须记录 `OrderSubmissionAttempt`，并写 AlertEvent。

---

## 6. 核心原则

### 6.1 不重复下单优先

009 最大风险不是交易所拒单，而是本地进入 `unknown` 后重复提交。

典型场景：

```text
1. 请求已经发出。
2. Binance 可能已经收到。
3. 本地网络超时、连接断开或进程崩溃。
4. 本地不知道订单是否已经存在。
```

此时系统不得生成新的 `client_order_id`，不得盲目再次提交同一交易意图。

### 6.2 008 冻结参数不可被 009 修改

009 不得重新计算或修改：

```text
1. side。
2. quantity。
3. quantity_unit。
4. reduce_only。
5. order_type。
6. position_side。
7. client_order_id。
8. idempotency_key。
9. raw_payload。
10. endpoint_family。
11. market_type。
12. account_domain。
```

如果发现参数不合法，009 应阻断或标记 `failed_before_submit`，而不是自动修正。

### 6.3 本地 idempotency_key 与交易所 client_order_id 分工明确

```text
PreparedOrderIntent.client_order_id -> Binance newClientOrderId
PreparedOrderIntent.idempotency_key -> 本地幂等键 / 审计键 / attempt 映射键
```

`idempotency_key` 不应作为未知参数发送给 Binance。

### 6.4 unknown 必须保守处理

进入 `unknown` 后：

```text
1. 不释放 OrderPlanActiveLock。
2. 不允许新的订单链路绕过 active lock。
3. 不允许自动生成新订单。
4. 不允许自动重试真实提交。
5. 不把 unknown 当作 rejected。
6. 不把 unknown 当作 submitted。
7. 必须等待后续 OrderStatusSync / Reconcile / 人工处理确认。
```

---

## 7. 输入与输出

### 7.1 Service 输入

009 service 输入建议为：

```text
prepared_order_intent_id
reference_time_utc
trace_id
trigger_source
```

要求：

```text
1. reference_time_utc 必须是 UTC aware datetime。
2. trace_id 可沿用 PreparedOrderIntent.trace_id。
3. trigger_source 必须记录触发来源，例如 management_command / scheduler / manual。
```

### 7.2 上游读取对象

009 必须读取：

```text
1. PreparedOrderIntent。
2. ExecutionPreparationResult。
3. OrderPlanActiveLock。
4. PriceSnapshot 或最新可消费价格事实，用于轻量提交前保护。
5. settings/env。
```

009 可以通过外键追溯：

```text
1. ApprovedOrderIntent。
2. OrderPlan。
3. CandidateOrderIntent。
```

但不得重新执行：

```text
1. OrderPlan 计算。
2. RiskCheck 审批。
3. ExecutionPreparation 冻结。
```

### 7.3 输出对象

009 至少应新增一个提交尝试对象：

```text
OrderSubmissionAttempt
```

P0 不强制拆分为多个表。若后续需要更细粒度审计，可以在后续阶段拆分：

```text
OrderSubmissionAttempt
ExchangeOrderSubmission
ExecutionAttempt
```

---

## 8. OrderSubmissionAttempt 模型需求

`OrderSubmissionAttempt` 至少应记录以下字段：

```text
id

prepared_order_intent_id
approved_order_intent_id
order_plan_id
execution_preparation_result_id

exchange
account_domain
market_type
active_market_type_at_runtime
active_account_domain_at_runtime
endpoint_family

symbol
side
position_side
order_type
quantity
quantity_unit
reduce_only

client_order_id
idempotency_key

request_payload
request_payload_hash
request_payload_schema_version

submitted_at_utc
finished_at_utc

status
exchange_order_id
exchange_client_order_id
exchange_status
exchange_response
exchange_response_hash

http_status_code
binance_error_code
binance_error_message
exception_class
error_code
error_message
reason_code
rate_limit_snapshot

trace_id
trigger_source
created_at_utc
updated_at_utc
```

字段要求：

```text
1. active_market_type_at_runtime 必须记录提交时 env 激活的 market_type。
2. active_account_domain_at_runtime 必须记录提交时 env 激活的 account_domain。
3. endpoint_family 必须记录 fapi 或 dapi。
4. quantity_unit 必须记录 quantity 或 contracts。
5. request_payload 不得包含 API secret。
6. request_payload 不得包含 signature。
7. request_payload 不得包含完整敏感 header。
8. exchange_response 可保存 Binance 返回的非敏感响应摘要。
9. client_order_id 必须与 PreparedOrderIntent.client_order_id 一致。
10. idempotency_key 必须与 PreparedOrderIntent.idempotency_key 一致。
11. trace_id / trigger_source 必须沿链路保留。
```

唯一性建议：

```text
1. prepared_order_intent_id 在 P0 中应具有唯一有效提交 attempt。
2. client_order_id 应在本地唯一。
3. idempotency_key 应在本地唯一。
```

如果实现需要允许历史多 attempt，必须确保同一 `PreparedOrderIntent` 不会出现多个可真实提交的 active attempt。

---

## 9. 状态机需求

### 9.1 PreparedOrderIntent 状态处理

008 现有 `PreparedOrderIntent` 已有状态中，`submitted` 语义属于 009 推进。

009 P0 推荐：

```text
PreparedOrderIntent.status 只表达主链路粗状态。
OrderSubmissionAttempt.status 表达提交尝试细状态。
```

推荐映射：

```text
Binance accepted:
  OrderSubmissionAttempt.status = accepted
  PreparedOrderIntent.status = submitted

Binance rejected:
  OrderSubmissionAttempt.status = rejected
  PreparedOrderIntent.status = failed 或 blocked
  reason_code = submission_rejected

submission unknown:
  OrderSubmissionAttempt.status = unknown
  PreparedOrderIntent.status = failed 或 blocked
  reason_code = submission_unknown

failed_before_submit:
  OrderSubmissionAttempt.status = failed_before_submit
  PreparedOrderIntent.status = failed 或保持 prepared，具体由 plan 固定

blocked_before_submit:
  OrderSubmissionAttempt.status = blocked_before_submit
  PreparedOrderIntent.status = blocked 或保持 prepared，具体由 plan 固定
```

要求：

```text
1. 不得出现 Binance rejected 但本地没有任何显式记录。
2. 不得出现 unknown 但本地没有任何显式记录。
3. 如果不扩展 PreparedOrderIntent.status，必须通过 OrderSubmissionAttempt.status + reason_code 完整表达。
4. 具体映射必须在 009 plan 和 implementation 中固定，不得让实现自由选择。
```

### 9.2 OrderSubmissionAttempt 状态

`OrderSubmissionAttempt.status` P0 必须支持：

```text
created
blocked_before_submit
submitting
accepted
rejected
unknown
failed_before_submit
```

语义：

```text
created:
  attempt 已创建，尚未提交。

blocked_before_submit:
  业务前置条件阻断，未调用 Binance。

submitting:
  已抢占提交权，准备调用或正在调用 Binance。

accepted:
  Binance 明确接收订单。

rejected:
  Binance 明确拒绝订单，且可判断未接受订单。

unknown:
  本地无法判断 Binance 是否收到。

failed_before_submit:
  本地异常或配置错误，且确认未调用 Binance。
```

### 9.3 不处理交易所完整订单状态

009 不应把 Binance 的完整订单生命周期作为主状态机处理。

以下状态只允许记录为 `exchange_status`，不得在 009 P0 中驱动成交逻辑：

```text
NEW
PARTIALLY_FILLED
FILLED
CANCELED
REJECTED
EXPIRED
EXPIRED_IN_MATCH
```

`PARTIALLY_FILLED`、`FILLED`、`CANCELED` 的后续处理必须属于后续阶段。

---

## 10. 提交前校验

009 在调用 Binance 前必须做 fail-closed 校验。

### 10.1 PreparedOrderIntent 校验

必须校验：

```text
1. PreparedOrderIntent 存在。
2. PreparedOrderIntent.status == prepared。
3. reference_time_utc <= PreparedOrderIntent.expires_at_utc。
4. client_order_id 非空。
5. idempotency_key 非空。
6. raw_payload 非空且 schema 可识别。
7. raw_payload.endpoint_family in [fapi, dapi]。
8. order_type == MARKET。
9. market_type / account_domain / quantity_unit / endpoint_family 组合合法。
10. market_type / account_domain 必须等于 active domain。
```

不得在 009 中硬编码单一 symbol。symbol 必须来自 `PreparedOrderIntent.raw_payload`，并且必须在配置允许列表中。

建议配置：

```text
ORDER_SUBMISSION_ALLOWED_SYMBOLS
```

### 10.2 Active lock 校验

必须校验：

```text
1. OrderPlanActiveLock 存在。
2. OrderPlanActiveLock 仍 active。
3. OrderPlanActiveLock 属于当前 order_plan / chain。
4. OrderPlanActiveLock.market_type == PreparedOrderIntent.market_type。
5. OrderPlanActiveLock.account_domain == PreparedOrderIntent.account_domain。
6. 未被 preparation blocked / failed / expired 释放。
```

如果 active lock 不满足，009 不得调用 Binance。

### 10.3 Kill switch 校验

009 必须支持全局交易提交开关。

建议配置：

```text
ORDER_SUBMISSION_ENABLED=false
ORDER_SUBMISSION_REAL_TRADING_ENABLED=false
```

当关闭时：

```text
1. 不调用 Binance。
2. 记录 OrderSubmissionAttempt.status = blocked_before_submit 或 failed_before_submit。
3. 写 AlertEvent。
4. 不释放 active lock，除非 plan 明确该阻断是终结性阻断。
```

默认必须关闭真实提交，避免误触发真实交易。

### 10.4 Binance 参数校验

P0 MARKET order 必须校验：

```text
1. symbol 必填。
2. side 必填，且只能 BUY / SELL。
3. type 必填，且只能 MARKET。
4. quantity 必填。
5. quantity_unit 与 endpoint_family 匹配。
6. positionSide = BOTH。
7. One-Way Mode 下 positionSide 可发送 BOTH，或按实现约定省略，但本地语义必须是 BOTH。
8. timeInForce 不得发送。
9. price 不得发送。
10. stopPrice 不得发送。
11. newClientOrderId 必须等于 PreparedOrderIntent.client_order_id。
12. reduceOnly 按 PreparedOrderIntent.reduce_only 发送，但 Hedge Mode 不支持时必须 fail-closed。
```

### 10.5 交易规则复核

009 必须基于当前可消费的交易规则快照做提交前复核。

USDS-M 路径：

```text
1. 使用 USDS-M 的 BinanceSymbolRuleSnapshot。
2. 校验 MARKET_LOT_SIZE.minQty。
3. 校验 MARKET_LOT_SIZE.maxQty。
4. 校验 MARKET_LOT_SIZE.stepSize。
5. 校验 MIN_NOTIONAL。
6. MARKET order 的 notional 可基于当前可消费 mark_price * quantity 估算。
```

COIN-M 路径：

```text
1. 使用 COIN-M 的 BinanceSymbolRuleSnapshot。
2. 校验 contracts 数量满足 COIN-M 合约交易规则。
3. 不把 contracts 当作 base asset quantity。
4. 不把 contracts 当作 quote notional。
5. 如需 notional / 合约面值校验，必须依据 COIN-M symbol rule 中的 contract_size、合约规格和交易所规则。
```

共同要求：

```text
1. 校验失败不得自动调整 quantity/contracts。
2. 校验失败不得自动缩单。
3. 校验失败不得重新生成 PreparedOrderIntent。
4. 校验失败必须 blocked_before_submit 或 failed_before_submit，并写 AlertEvent。
```

### 10.6 价格保护

009 可以做轻量提交前 price guard，但不得替代 008 风控。

建议校验：

```text
1. 当前 PriceSnapshot 未超过最大年龄。
2. 当前价格相对 PreparedOrderIntent.reference_price 未超过配置阈值。
3. 当前价格来源 as_of_utc <= reference_time_utc。
4. PriceSnapshot 的 market_type / account_domain 必须与 PreparedOrderIntent 一致。
5. PriceSnapshot 的 market_type / account_domain 必须与 active domain 一致。
```

校验失败应阻断提交。

---

## 11. Binance REST Client / Adapter 需求

009 需要新增或复用 Binance order client，但真实下单必须通过 active domain 和 endpoint_family 双重分流。

P0 至少需要两个 adapter：

```text
FapiOrderAdapter
DapiOrderAdapter
```

选择规则：

```text
active=usds_m_futures + endpoint_family=fapi -> FapiOrderAdapter
active=coin_m_futures + endpoint_family=dapi -> DapiOrderAdapter
其他组合 -> failed_before_submit / blocked_before_submit
```

### 11.1 Endpoint 要求

USDS-M 路径：

```text
FapiOrderAdapter -> Binance USDⓈ-M Futures new order endpoint
```

COIN-M 路径：

```text
DapiOrderAdapter -> Binance COIN-M Futures new order endpoint
```

009 不得实现：

```text
1. cancel order。
2. query order。
3. query trade list。
4. batch order。
5. user data stream。
```

这些属于后续阶段。

### 11.2 认证与签名

Binance `TRADE` endpoint 必须使用 API key 和签名。

要求：

```text
1. API key / secret 必须来自 settings/env。
2. USDS-M 与 COIN-M 的 API key / secret 配置必须明确。
3. 根据 active domain 选择对应 key / secret / base_url。
4. secret 不得写入日志。
5. secret 不得写入数据库。
6. signature 不得写入 request_payload。
7. X-MBX-APIKEY 不得完整写入日志。
8. timestamp 必须使用 Binance server time 或可靠时间偏移机制。
9. recvWindow 必须来自配置。
```

### 11.3 网络超时

必须配置：

```text
1. connect timeout。
2. read timeout。
3. recvWindow。
```

网络超时分类：

```text
1. 如果请求未发出前失败：failed_before_submit。
2. 如果请求已发出但未收到确定响应：unknown。
3. 如果无法判断请求是否发出：unknown。
```

---

## 12. Binance 响应分类

009 必须明确把提交结果分类。

### 12.1 accepted

满足以下条件可视为 accepted：

```text
1. HTTP 2xx。
2. Binance 返回可识别 orderId 或 clientOrderId。
3. 返回 clientOrderId 与 PreparedOrderIntent.client_order_id 一致。
4. 响应未包含明确错误码。
```

accepted 后：

```text
1. OrderSubmissionAttempt.status = accepted。
2. PreparedOrderIntent.status = submitted。
3. 记录 exchange_order_id / exchange_status。
4. 写 AlertEvent。
5. 不释放 OrderPlanActiveLock。
```

不释放 active lock 的原因：

```text
订单已进入交易所，但最终成交状态未知，必须等待后续状态同步。
```

### 12.2 rejected

满足以下条件可视为 rejected：

```text
1. Binance 返回明确业务错误。
2. 可以确认交易所未接受该订单。
3. 错误不是网络超时或 execution status unknown。
```

rejected 后：

```text
1. OrderSubmissionAttempt.status = rejected。
2. PreparedOrderIntent 按 plan 中固定映射推进。
3. 记录 binance_error_code / binance_error_message。
4. 写 AlertEvent。
5. active lock 是否释放必须在 plan 中明确。
```

P0 建议：

```text
1. 对参数错误、规则错误、余额不足、权限错误等终结性拒绝，可释放或 failed active lock。
2. 对限流、系统繁忙、无法确认是否接收的错误，不得直接释放 active lock。
3. 对 unknown 绝不释放 active lock。
```

### 12.3 unknown

以下情况必须进入 unknown：

```text
1. 请求发出后 read timeout。
2. 连接中断且无法确认 Binance 是否处理。
3. HTTP 503 且 Binance 返回 execution status unknown 语义。
4. 进程在提交期间异常中断，恢复后发现 attempt 停留在 submitting。
5. 无法判断交易所是否收到订单的其他情况。
```

unknown 后：

```text
1. OrderSubmissionAttempt.status = unknown。
2. PreparedOrderIntent 按 plan 中固定映射推进。
3. 不释放 OrderPlanActiveLock。
4. 不允许自动重试生成新订单。
5. 写 critical/high AlertEvent。
6. 等待后续 OrderStatusSync / Reconcile 使用 client_order_id 查询确认。
```

### 12.4 failed_before_submit

以下情况属于 failed_before_submit：

```text
1. PreparedOrderIntent 不存在。
2. PreparedOrderIntent 状态不允许提交。
3. PreparedOrderIntent 已过期。
4. active lock 不存在或不匹配。
5. active domain 非法。
6. active domain 与 PreparedOrderIntent 不一致。
7. kill switch 关闭。
8. settings 缺失。
9. API key / secret 缺失。
10. 本地参数校验失败。
11. 交易规则复核失败。
12. endpoint_family / account_domain / market_type / quantity_unit 组合不合法。
13. 请求构造失败，且确认未调用 Binance。
```

failed_before_submit 后：

```text
1. 不调用 Binance。
2. OrderSubmissionAttempt.status = failed_before_submit 或 blocked_before_submit。
3. PreparedOrderIntent 按 plan 中固定映射推进，或保持 prepared 并记录阻断结果。
4. 写 AlertEvent。
```

---

## 13. 幂等与并发

009 必须防止并发重复提交。

### 13.1 锁要求

实现必须使用事务和行锁。

建议：

```text
1. select_for_update 锁定 PreparedOrderIntent。
2. select_for_update 锁定相关 OrderPlanActiveLock。
3. 创建或读取 OrderSubmissionAttempt 时也必须在同一事务中完成幂等判断。
```

### 13.2 幂等规则

同一 `PreparedOrderIntent`：

```text
1. 若已存在 accepted attempt，不得再次提交。
2. 若已存在 unknown attempt，不得再次提交。
3. 若已存在 submitting attempt，必须判断是否 stale。
4. 若 submitting 已 stale，不得直接再次调用 Binance，应先转 unknown 或进入人工/后续 reconcile。
5. 若 failed_before_submit 且确认从未调用 Binance，可允许再次触发，但必须复用同一 client_order_id。
```

### 13.3 client_order_id 规则

```text
1. 009 不得生成新的 client_order_id。
2. 009 必须使用 PreparedOrderIntent.client_order_id。
3. Binance 参数名必须为 newClientOrderId。
4. 若 client_order_id 不符合 Binance 格式要求，必须 failed_before_submit。
```

---

## 14. 费率限制与熔断

009 P0 不做复杂调度器，但必须遵守交易所限制。

要求：

```text
1. 读取并记录 Binance 响应中的订单速率 header。
2. 对 HTTP 429 进入 rate_limited 分类，不得快速重试。
3. 对 HTTP 418 进入 banned / critical 分类，并写高严重级别 AlertEvent。
4. 本地必须支持订单提交频率保护配置。
5. 超过本地配置频率时，必须 blocked_before_submit，不调用 Binance。
```

P0 不建议自动退避重试下单，因为重试会放大 unknown 重复提交风险。

---

## 15. 配置需求

009 至少需要以下 settings/env：

```text
BINANCE_ACTIVE_MARKET_TYPE

ORDER_SUBMISSION_ENABLED
ORDER_SUBMISSION_REAL_TRADING_ENABLED

ORDER_SUBMISSION_USDS_M_BASE_URL
ORDER_SUBMISSION_COIN_M_BASE_URL
ORDER_SUBMISSION_USDS_M_TESTNET_ENABLED
ORDER_SUBMISSION_COIN_M_TESTNET_ENABLED

ORDER_SUBMISSION_CONNECT_TIMEOUT_SECONDS
ORDER_SUBMISSION_READ_TIMEOUT_SECONDS
ORDER_SUBMISSION_RECV_WINDOW_MS
ORDER_SUBMISSION_MAX_PRICE_AGE_SECONDS
ORDER_SUBMISSION_MAX_PRICE_DEVIATION_BPS
ORDER_SUBMISSION_ALLOWED_SYMBOLS
ORDER_SUBMISSION_SUPPORTED_ORDER_TYPES
ORDER_SUBMISSION_SUPPORTED_POSITION_MODE
ORDER_SUBMISSION_LOCAL_RATE_LIMIT_PER_MINUTE
ORDER_SUBMISSION_STALE_SUBMITTING_SECONDS

BINANCE_USDS_M_TRADE_API_KEY
BINANCE_USDS_M_TRADE_API_SECRET
BINANCE_COIN_M_TRADE_API_KEY
BINANCE_COIN_M_TRADE_API_SECRET
```

要求：

```text
1. 默认 ORDER_SUBMISSION_ENABLED 应为 false，避免误触发真实交易。
2. 默认 ORDER_SUBMISSION_REAL_TRADING_ENABLED 应为 false。
3. 测试环境必须可替换 Binance client / adapter。
4. 所有阈值不得硬编码在 service 中。
5. API secret 不得出现在 .env.example 的真实值中。
6. U 本位与币本位 base_url / API key / secret 必须配置清晰，不能隐式混用。
7. active domain 决定使用哪一组 base_url / API key / secret。
```

---

## 16. AlertEvent 与日志

009 必须写 AlertEvent，不得直接调用 Hermes 投递。

必须写 AlertEvent 的场景：

```text
1. accepted。
2. rejected。
3. unknown。
4. failed_before_submit。
5. blocked_before_submit。
6. stale submitting 恢复为 unknown。
7. kill switch 阻断。
8. real trading disabled。
9. rate limit / banned。
10. active domain mismatch。
11. endpoint_family / account_domain / market_type / quantity_unit 不一致。
```

AlertEvent 必须包含：

```text
source_module = order_submission
trace_id
trigger_source
prepared_order_intent_id
order_submission_attempt_id
order_plan_id
client_order_id
active_market_type_at_runtime
active_account_domain_at_runtime
endpoint_family
market_type
account_domain
quantity_unit
status
reason_code / error_code
```

日志要求：

```text
1. 使用结构化日志。
2. 必须包含 trace_id。
3. 必须包含 prepared_order_intent_id。
4. 必须包含 order_submission_attempt_id。
5. 必须包含 client_order_id。
6. 必须包含 active_market_type_at_runtime。
7. 必须包含 endpoint_family。
8. 不得包含 API secret。
9. 不得包含 signature。
```

---

## 17. 异常恢复

009 必须考虑进程崩溃或中断。

### 17.1 submitting stale

如果存在长时间停留在 `submitting` 的 `OrderSubmissionAttempt`：

```text
1. 不得假设失败。
2. 不得自动重新提交。
3. 应转为 unknown，或保留 submitting 并由恢复命令转 unknown。
4. 写 AlertEvent。
```

### 17.2 后续恢复边界

009 可以记录 unknown，但不负责确认 unknown 最终结果。

后续应由 010 或独立 reconcile 模块：

```text
1. 使用 client_order_id / origClientOrderId 查询 Binance。
2. 按 endpoint_family 选择 fapi 或 dapi 查询接口。
3. 确认订单是否存在。
4. 同步订单状态。
5. 同步成交。
6. 更新 active lock。
```

---

## 18. Management command

009 可提供 management command：

```text
submit_order
```

参数建议：

```text
--prepared-order-intent-id
--reference-time-utc
--trace-id
--trigger-source
```

要求：

```text
1. command 只调用 service。
2. command 不直接写复杂业务逻辑。
3. command 输出提交结果摘要。
4. command 不打印敏感参数。
5. command 不允许绕过 kill switch，除非 plan 明确提供受控 override 且默认关闭。
```

---

## 19. 测试要求

009 必须至少覆盖以下测试。

### 19.1 USDS-M active 成功提交

```text
1. BINANCE_ACTIVE_MARKET_TYPE=usds_m_futures。
2. PreparedOrderIntent.market_type=usds_m_futures。
3. PreparedOrderIntent.account_domain=usds_m_futures。
4. quantity_unit=quantity。
5. endpoint_family=fapi。
6. prepared -> submitted。
7. 创建 OrderSubmissionAttempt accepted。
8. 使用 PreparedOrderIntent.client_order_id。
9. 使用 FapiOrderAdapter。
10. 不调用 DapiOrderAdapter。
11. MARKET payload 不包含 timeInForce。
12. 写 AlertEvent。
13. 不释放 OrderPlanActiveLock。
```

### 19.2 COIN-M active 成功提交

```text
1. BINANCE_ACTIVE_MARKET_TYPE=coin_m_futures。
2. PreparedOrderIntent.market_type=coin_m_futures。
3. PreparedOrderIntent.account_domain=coin_m_futures。
4. quantity_unit=contracts。
5. endpoint_family=dapi。
6. prepared -> submitted。
7. 创建 OrderSubmissionAttempt accepted。
8. 使用 PreparedOrderIntent.client_order_id。
9. 使用 DapiOrderAdapter。
10. 不调用 FapiOrderAdapter。
11. raw_payload.quantity 作为 contracts 张数提交。
12. 不把 contracts 转成 base asset。
13. 不把 contracts 转成 quote asset。
14. MARKET payload 不包含 timeInForce。
15. 写 AlertEvent。
16. 不释放 OrderPlanActiveLock。
```

### 19.3 active domain mismatch

```text
1. env=usds_m_futures，但 PreparedOrderIntent=coin_m_futures -> 不提交。
2. env=coin_m_futures，但 PreparedOrderIntent=usds_m_futures -> 不提交。
3. env=usds_m_futures，但 endpoint_family=dapi -> 不提交。
4. env=coin_m_futures，但 endpoint_family=fapi -> 不提交。
5. env=usds_m_futures，但 quantity_unit=contracts -> 不提交。
6. env=coin_m_futures，但 quantity_unit=quantity -> 不提交。
```

验证：

```text
不调用 FapiOrderAdapter。
不调用 DapiOrderAdapter。
写 OrderSubmissionAttempt failed_before_submit / blocked_before_submit。
写 AlertEvent。
```

### 19.4 Binance 明确拒绝

```text
1. fapi Binance 返回业务错误 -> rejected。
2. dapi Binance 返回业务错误 -> rejected。
3. OrderSubmissionAttempt.status = rejected。
4. PreparedOrderIntent 按 plan 固定映射推进。
5. 记录 binance_error_code。
6. 写 AlertEvent。
```

### 19.5 网络超时 unknown

```text
1. fapi 请求发出后 read timeout -> unknown。
2. dapi 请求发出后 read timeout -> unknown。
3. OrderSubmissionAttempt.status = unknown。
4. PreparedOrderIntent 按 plan 固定映射推进。
5. 不释放 OrderPlanActiveLock。
6. 再次触发不得重新提交。
7. 写 AlertEvent。
```

### 19.6 提交前失败

```text
1. expired PreparedOrderIntent 不提交。
2. non-prepared PreparedOrderIntent 不提交。
3. active lock 不匹配不提交。
4. kill switch 关闭不提交。
5. real trading disabled 不提交。
6. 参数非法不提交。
7. 交易规则不满足不提交。
8. endpoint_family / account_domain / market_type / quantity_unit 不一致不提交。
9. active domain 与 PreparedOrderIntent 不一致不提交。
```

### 19.7 并发幂等

```text
1. 两个并发请求同一个 PreparedOrderIntent。
2. 最多只有一个 attempt 可以进入 submitting。
3. 不产生两个 Binance client 调用。
4. client_order_id 不变。
5. accepted 后再次触发不重新提交。
6. unknown 后再次触发不重新提交。
```

### 19.8 敏感信息保护

```text
1. request_payload 不包含 secret。
2. request_payload 不包含 signature。
3. logs 不包含 secret。
4. AlertEvent 不包含 secret。
5. .env.example 不包含真实 secret。
```

---

## 20. 验收标准

009 验收前必须确认：

```text
1. 正式需求文件路径为 docs/requirements/order_submission.md。
2. 只消费 PreparedOrderIntent。
3. 拒绝 expired / non-prepared。
4. 使用 PreparedOrderIntent.client_order_id。
5. 不重新生成 client_order_id。
6. 不重新生成 raw_payload。
7. 不重新做 OrderPlan。
8. 不重新做 RiskCheck。
9. 不修改订单方向和数量。
10. 不修改 quantity_unit。
11. 不修改 endpoint_family。
12. 系统能力支持 USDS-M / quantity / fapi。
13. 系统能力支持 COIN-M / contracts / dapi。
14. 运行时只能有一个 active domain。
15. 非 active domain PreparedOrderIntent 不会被提交。
16. USDS-M active 时只允许 fapi / quantity。
17. COIN-M active 时只允许 dapi / contracts。
18. accepted / rejected / unknown / failed_before_submit 分类清楚。
19. unknown 不释放 active lock。
20. unknown 后再次触发不会重复下单。
21. accepted 后不释放 active lock。
22. Binance response / error code 已落库。
23. AlertEvent 已覆盖 accepted / rejected / unknown / failed_before_submit。
24. 无 WebSocket。
25. 无撤单。
26. 无查成交。
27. 无订单状态轮询。
28. 无真实持仓更新。
29. 无敏感信息落库。
30. 测试覆盖 fapi 网络超时 -> unknown。
31. 测试覆盖 dapi 网络超时 -> unknown。
32. 测试覆盖 active domain mismatch。
```

常规命令：

```powershell
.\.venv\Scripts\python.exe manage.py makemigrations --check --dry-run
.\.venv\Scripts\python.exe migrate --plan
.\.venv\Scripts\python.exe manage.py check
.\.venv\Scripts\python.exe manage.py test tests.order_submission --verbosity 1
.\.venv\Scripts\python.exe manage.py test
.\.venv\Scripts\pytest.exe
git diff --check
```

---

## 21. 后续阶段预留

009 不做但必须为后续预留字段或接口：

```text
1. 使用 client_order_id 查询订单。
2. 按 endpoint_family 选择 fapi / dapi 查询订单。
3. 同步 Binance order status。
4. 同步 partial fill / full fill。
5. 同步 commission / realized pnl。
6. 异常 reconcile。
7. 撤单。
8. WebSocket user data stream。
9. 多 symbol。
10. LIMIT / STOP / TAKE_PROFIT。
```

---

## 22. 最终结论

009 Order Submission 是一个窄模块。

它的主要动作只有一个：

```text
把 008 冻结好的 PreparedOrderIntent 安全提交给 Binance。
```

它必须同时具备：

```text
1. USDS-M Futures / quantity / fapi。
2. COIN-M Futures / contracts / dapi。
```

但运行时只能启用：

```text
一个 active market_type / account_domain。
```

它的主要风险是：

```text
unknown 状态下重复提交，造成重复开仓或重复平仓。
```

因此 009 的实现必须围绕：

```text
active domain 二选一
幂等
锁
client_order_id
endpoint_family
quantity_unit
提交 attempt 落库
accepted / rejected / unknown 分类
AlertEvent
fail-closed
```

不要在 009 中实现成交同步、撤单、订单状态轮询或 WebSocket。
