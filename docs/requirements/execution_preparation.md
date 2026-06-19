# 008 ExecutionPreparation 需求说明

> 适用项目：`cypto_robot` 中低频趋势跟踪自动交易系统  
> 建议路径：`docs/requirements/execution_preparation.md`  
> 阶段编号：`008`  
> 阶段名称：`ExecutionPreparation`  
> 中文定位：执行准备 / 执行请求冻结层  
> 上游阶段：`007B RiskCheck / ApprovedOrderIntent`  
> 下游阶段：`009 Execution / OrderSubmission`  
> 当前状态：需求第二版草案  
> 重要边界：本阶段不真实下单，不撤单，不查成交，不接 WebSocket，不重新风控。

---

## 1. 背景与目标

截至 007B，系统已经完成：

```text
OrderPlan
  ↓
CandidateOrderIntent
  ↓
RiskCheck
  ↓
ApprovedOrderIntent
```

`ApprovedOrderIntent` 表示：某个候选订单已经通过风控审批，允许进入执行准备。

但是 `ApprovedOrderIntent` 仍然不是交易所订单，也不是最终可以直接发送给 Binance 的请求。它主要保存：

```text
风控批准后的交易意图
风控审批证据
上游订单链路追溯关系
```

008 ExecutionPreparation 的目标是：

```text
把 ApprovedOrderIntent 冻结成唯一、幂等、可过期、可审计、尚未提交交易所的待执行请求。
```

本阶段产物建议命名为：

```text
PreparedOrderIntent
```

本文后续统一使用 `PreparedOrderIntent` 表达“待执行请求”。

---

## 2. 008 的一句话职责

008 只负责：

```text
承接已通过风控的 ApprovedOrderIntent，
做执行前必要复核，
冻结交易所可提交参数，
生成唯一客户订单号和幂等凭证，
产出待执行请求。
```

008 不负责：

```text
判断是否应该交易
重新计算目标仓位
重新拆单
重新审批风控
真实下单
订单状态同步
成交同步
持仓更新
WebSocket 行情接入
```

---

## 3. 主链路定位

正式链路为：

```text
DecisionSnapshot
  ↓
OrderPlan
  ↓
CandidateOrderIntent
  ↓
RiskCheck
  ↓
ApprovedOrderIntent
  ↓
ExecutionPreparation
  ↓
PreparedOrderIntent
  ↓
Execution / OrderSubmission
  ↓
ExchangeOrder / ExecutionAttempt
  ↓
OrderStatusSync / FillSync / Tracking
```

各阶段职责：

```text
CandidateOrderIntent：
  表达候选订单意图，不可执行。

ApprovedOrderIntent：
  表达风控批准后的订单意图，允许进入执行准备，但不可直接提交交易所。

PreparedOrderIntent：
  表达已经完成执行准备、参数冻结、编号生成、尚未提交交易所的待执行请求。

Execution / OrderSubmission：
  真正调用 Binance 下单接口，记录提交结果。

OrderStatusSync / FillSync：
  查询订单状态、成交、部分成交、撤单、拒单、持仓变化。
```

---

## 4. P0 范围

008 P0 必须做到：

```text
1. 只消费 ApprovedOrderIntent。
2. 校验 ApprovedOrderIntent 是否处于可准备状态。
3. 校验上游链路身份一致性。
4. 校验当前价格事实是否新鲜。
5. 校验当前 mark_price 与风控审批参考 mark_price 是否偏离过大。
6. 校验账户事实是否仍然可消费。
7. 校验当前持仓方向和数量与风控审批时一致。
8. 校验 reduce-only 语义在当前持仓事实下仍成立。
9. 校验交易规则仍可消费。
10. 冻结交易所可提交请求参数。
11. 生成唯一 client_order_id。
12. 生成唯一 idempotency_key。
13. 创建 PreparedOrderIntent。
14. 保证同一个 ApprovedOrderIntent 只能对应一份 PreparedOrderIntent。
15. 重复运行时返回同一份 PreparedOrderIntent。
16. PreparedOrderIntent 必须有过期时间。
17. 成功、阻断、失败、过期、幂等重放都必须写审计事件。
18. blocked / failed / expired 必须推进相关父级状态，不能留下错误 active 链路。
```

---

## 5. P0 非范围

008 P0 禁止做：

```text
1. 不真实下单。
2. 不调用 Binance 下单接口。
3. 不撤单。
4. 不查订单。
5. 不查成交。
6. 不同步订单状态。
7. 不更新真实持仓。
8. 不接 Binance WebSocket。
9. 不管理 WebSocket 连接、订阅、重连、心跳。
10. 不修改杠杆。
11. 不修改保证金模式。
12. 不修改持仓模式。
13. 不重新计算目标仓位。
14. 不重新生成 CandidateOrderIntent。
15. 不重新生成 ApprovedOrderIntent。
16. 不重新做完整 RiskCheck。
17. 不修改订单方向。
18. 不修改订单数量。
19. 不自动缩单。
20. 不自动拆单。
21. 不做 Backtest。
22. 不做 UI / Admin。
23. 不支持 dry_run。
```

008 不是 Execution。  
008 不是 RiskCheck 第二遍。  
008 不是 WebSocket 行情模块。  
008 不是订单生命周期管理模块。

---

## 6. ApprovedOrderIntent 与 PreparedOrderIntent 的区别

### 6.1 ApprovedOrderIntent

`ApprovedOrderIntent` 的语义是：

```text
这笔候选订单已经通过风控，允许进入执行准备。
```

它保留：

```text
风控审批结果
被批准的交易意图
候选订单追溯
账户快照追溯
价格快照追溯
风险证据摘要
```

它不负责：

```text
最终交易所请求冻结
client_order_id 生成
待执行请求过期控制
真实提交前幂等控制
Execution 模块的直接请求格式
```

### 6.2 PreparedOrderIntent

`PreparedOrderIntent` 的语义是：

```text
这笔已批准订单已经被整理成唯一、幂等、可过期、可审计、尚未发送交易所的待执行请求。
```

它负责冻结：

```text
交易所
账户域
市场类型
交易对
买卖方向
持仓方向
订单类型
数量
数量单位
是否 reduce-only
time_in_force 语义
client_order_id
idempotency_key
价格复核结果
账户复核结果
交易规则复核结果
准备完成时间
过期时间
配置快照
```

核心区别：

```text
ApprovedOrderIntent = 风控批准后的交易意图
PreparedOrderIntent = 已冻结的交易所待提交请求
```

如果 008 只是复制 `ApprovedOrderIntent`，则没有价值。  
如果 008 开始真实下单，则越界。

---

## 7. 输入合同

ExecutionPreparation P0 的正式输入是：

```text
approved_order_intent_id
reference_time_utc
trace_id
trigger_source
```

P0 不支持：

```text
dry_run
```

### 7.1 ApprovedOrderIntent 前置条件

`ApprovedOrderIntent` 必须满足：

```text
1. 存在。
2. 状态处于 pending_execution_preparation 或等价可准备状态。
3. 绑定有效 RiskCheckResult。
4. 绑定有效 CandidateOrderIntent。
5. 绑定有效 OrderPlan。
6. 绑定有效 PriceSnapshot。
7. 绑定有效 BinanceSyncRun / Binance Account Sync 快照批次。
8. 交易所、账户域、市场类型、交易对一致。
9. 不属于已经终结的订单链路。
10. 当前 OrderPlanActiveLock 仍属于本订单链路。
```

如果不满足：

```text
ExecutionPreparationResult = BLOCKED 或 FAILED
不生成 prepared 状态的 PreparedOrderIntent
写 AlertEvent
推进父级状态
```

---

## 8. 输出合同

008 成功时必须生成：

```text
PreparedOrderIntent
```

它代表：

```text
可交给下一阶段 Execution / OrderSubmission 的待执行请求。
```

P0 至少应包含：

```text
prepared_order_intent_key
source_approved_order_intent_id
source_risk_check_result_id
source_candidate_order_intent_id
source_order_plan_id
exchange
account_domain
market_type
symbol
position_mode
position_side
side
order_type
quantity
quantity_unit
reduce_only
time_in_force
client_order_id
idempotency_key
price_snapshot_id
price_observed_at_utc
reference_price
latest_price
price_deviation_bps
binance_sync_run_id
account_snapshot_id
position_snapshot_id
symbol_rule_snapshot_id
prepared_at_utc
expires_at_utc
status
reason_code
reason_message
trace_id
trigger_source
config_snapshot
evidence
created_at_utc
updated_at_utc
```

注意：

```text
PreparedOrderIntent 仍然不是交易所订单。
PreparedOrderIntent 不表示 Binance 已经收到订单。
PreparedOrderIntent 不表示订单已成交。
PreparedOrderIntent 不允许被 009 以外的模块提交。
```

---

## 9. 状态语义

### 9.1 ExecutionPreparationResult 状态

P0 结果状态：

```text
PREPARED
BLOCKED
FAILED
EXPIRED
IDEMPOTENT_REPLAY
```

含义：

```text
PREPARED：
  已成功生成 PreparedOrderIntent，可进入 009 Execution。

BLOCKED：
  业务前置条件不满足，不能可靠准备执行请求。

FAILED：
  系统异常或无法恢复的内部错误。

EXPIRED：
  已有 PreparedOrderIntent 超过有效期，不能继续交给 009。

IDEMPOTENT_REPLAY：
  相同 ApprovedOrderIntent 重复调用，返回已有未过期 PreparedOrderIntent。
```

### 9.2 PreparedOrderIntent 状态

P0 状态：

```text
prepared
blocked
failed
expired
submitted
```

008 只允许创建或推进到：

```text
prepared
blocked
failed
expired
```

`submitted` 只能由 009 Execution / OrderSubmission 推进。

---

## 10. 成功状态推进

当 008 成功时：

```text
PreparedOrderIntent.status = prepared
ApprovedOrderIntent.status = execution_prepared
OrderPlan 保持 active / created / 等待后续执行
OrderPlanActiveLock 保持 active
```

原因：

```text
后续 009 还没有真实下单，订单链路尚未结束。
active lock 必须继续阻止同一账户、市场、交易对生成另一条冲突订单链路。
```

008 成功后不得释放 `OrderPlanActiveLock`。

---

## 11. 阻断状态推进

当 008 因业务条件不满足而阻断时：

```text
PreparedOrderIntent.status = blocked
ApprovedOrderIntent.status = preparation_blocked
OrderPlan.status = preparation_blocked
OrderPlanActiveLock.status = released
```

典型阻断原因：

```text
approved_order_intent_not_found
approved_order_intent_not_ready
source_chain_mismatch
active_lock_missing
active_lock_not_active
active_chain_conflict
stale_price_snapshot
fresh_price_unavailable
price_type_mismatch
price_deviation_exceeded
stale_account_snapshot
account_state_changed
position_state_changed
position_amount_changed
position_side_changed
reduce_only_invalid
symbol_rule_unavailable
symbol_rule_changed
exchange_rule_violation
unsupported_order_type
unsupported_position_mode
prepared_request_conflict
expired_prepared_order_intent
```

阻断含义：

```text
本轮订单链路不能继续。
不是系统异常。
不允许进入 009。
```

---

## 12. 失败状态推进

当 008 出现系统异常时：

```text
PreparedOrderIntent.status = failed
ApprovedOrderIntent.status = preparation_failed
OrderPlan.status = preparation_failed
OrderPlanActiveLock.status = failed
```

失败含义：

```text
系统未能可靠完成执行准备。
不能静默失败。
不能继续进入 009。
必须保留审计证据。
```

FAILED 后：

```text
1. 不自动重试。
2. 不自动重新 prepare。
3. 不删除已生成的 PreparedOrderIntent。
4. 不清理 client_order_id。
5. 是否重新开始由编排层或人工处理决定。
6. 同一个 ApprovedOrderIntent 不得直接再次生成新的 PreparedOrderIntent。
```

---

## 13. 过期机制

PreparedOrderIntent 必须有明确有效期。

P0 默认建议：

```text
PREPARED_ORDER_INTENT_TTL_SECONDS = 30
```

含义：

```text
PreparedOrderIntent 创建后，如果超过 30 秒仍未提交交易所，则不能继续使用。
```

过期后：

```text
PreparedOrderIntent.status = expired
ApprovedOrderIntent.status = preparation_expired
OrderPlan.status = preparation_expired
OrderPlanActiveLock.status = released
写 AlertEvent
```

过期后规则：

```text
1. 同一个 ApprovedOrderIntent 不得再次生成新的 PreparedOrderIntent。
2. 不允许把 expired PreparedOrderIntent 改回 prepared。
3. 009 收到 expired PreparedOrderIntent 必须拒绝提交。
4. 后续是否重新走账户同步、OrderPlan、RiskCheck，由编排层决定。
```

008 不自动重新生成策略。  
008 不自动重新风控。  
008 不自动重新下单。

---

## 14. 价格复核

### 14.1 价格复核目的

008 必须确认：

```text
当前价格事实仍然新鲜；
当前 mark_price 相对 007B 风控审批参考 mark_price 没有过大偏离。
```

原因：

```text
RiskCheck 通过时的价格不等于执行准备时的价格。
如果价格已经明显变化，原风控结论可能不再成立。
```

### 14.2 价格类型

P0 只允许使用：

```text
mark_price
```

禁止混用：

```text
last_price
index_price
K线 close price
盘口价
成交价
```

价格偏离检查定义为：

```text
reference_price = ApprovedOrderIntent.reference_price
reference_price_type = mark_price
latest_price = 最新可消费 PriceSnapshot.mark_price
latest_price_type = mark_price
price_deviation_bps = abs(latest_price - reference_price) / reference_price * 10000
```

如果：

```text
PriceSnapshot.price_type != mark_price
ApprovedOrderIntent.reference_price <= 0
PriceSnapshot.mark_price <= 0
```

则：

```text
ExecutionPreparationResult = BLOCKED
reason_code = price_type_mismatch 或 invalid_price
```

### 14.3 价格来源

P0 使用现有 `PriceSnapshot` 机制。

008 不接 WebSocket。  
008 不直接管理行情连接。  
008 不把 WebSocket tick 写成 PriceSnapshot。

未来如果引入 WebSocket，推荐结构是：

```text
WebSocketPriceFeed
  ↓
LatestPriceState / PriceCache
  ↓
PriceSnapshotService
  ↓
PriceSnapshot
  ↓
ExecutionPreparation
```

008 只关心价格事实是否可信，不关心价格事实来自 REST、账户同步、手工 fixture 还是未来 WebSocket。

### 14.4 价格新鲜度

P0 默认建议：

```text
EXECUTION_PREPARATION_PRICE_MAX_AGE_SECONDS = 30
```

检查：

```text
PriceSnapshot 存在
PriceSnapshot 可消费
PriceSnapshot.price_type = mark_price
symbol 匹配
market_type 匹配
account_domain 匹配
mark_price > 0
reference_time_utc - PriceSnapshot.as_of_utc <= 30 秒
```

如果价格快照超过 30 秒：

```text
ExecutionPreparationResult = BLOCKED
reason_code = stale_price_snapshot
不生成 prepared 状态的 PreparedOrderIntent
写 AlertEvent
```

### 14.5 价格偏离

P0 默认建议：

```text
EXECUTION_PREPARATION_MAX_PRICE_DEVIATION_BPS = 50
```

即：

```text
0.5%
```

检查：

```text
latest mark_price 与 ApprovedOrderIntent.reference_price 偏离 <= 0.5%
```

如果偏离超过阈值：

```text
ExecutionPreparationResult = BLOCKED
reason_code = price_deviation_exceeded
不生成 prepared 状态的 PreparedOrderIntent
写 AlertEvent
```

说明：

```text
价格偏离过大时，不允许 008 直接继续。
不允许 008 自动修改数量或订单方向。
应由上游重新生成 / 重新风控。
```

---

## 15. 账户事实复核

008 不重新做完整 RiskCheck，但必须做执行前账户事实复核。

### 15.1 账户快照来源

P0 使用：

```text
006B BinanceSyncRun
006B BinanceAccountSnapshot
006B BinancePositionSnapshot
006B BinanceSymbolRuleSnapshot
```

不新增交易规则缓存服务。  
不在 008 直接请求 Binance。  
不在 008 调用 REST / WebSocket。

### 15.2 当前账户快照要求

008 必须读取当前可消费的最新账户同步批次。

当前账户同步批次必须满足：

```text
1. 存在。
2. 未过期。
3. 同步状态可消费。
4. exchange 匹配。
5. account_domain 匹配。
6. market_type 匹配。
7. symbol 对应的 position snapshot 存在。
8. symbol 对应的 symbol rule snapshot 存在。
```

如果不满足：

```text
ExecutionPreparationResult = BLOCKED
reason_code = stale_account_snapshot 或 account_snapshot_unavailable
```

### 15.3 与风控审批时账户事实比较

如果当前账户同步批次与 RiskCheck 使用的是同一批：

```text
仍需确认未过期；
仍需确认关键字段存在；
可视为账户事实未变化。
```

如果当前账户同步批次晚于 RiskCheck 使用的批次：

必须比较关键字段：

```text
position_mode
margin_mode
observed_exchange_leverage
normalized_position_side
position_amount
symbol_rule_snapshot_hash / rule_hash
```

任一关键字段变化：

```text
ExecutionPreparationResult = BLOCKED
reason_code = account_state_changed / position_state_changed / symbol_rule_changed
```

P0 不允许：

```text
余额减少 5% 内继续
持仓变化 5% 内继续
持仓变化 20% 内继续
```

只要关键字段发生实质变化，就阻断。

---

## 16. 持仓方向与数量一致性

008 必须确认：

```text
执行准备时的真实持仓方向与 RiskCheck 审批时使用的持仓方向一致。
执行准备时的真实持仓数量与 RiskCheck 审批时使用的持仓数量一致。
```

P0 采用 fail-closed 精确一致规则。

### 16.1 方向一致

以下任一情况必须 BLOCKED：

```text
审批时无仓，执行准备时变成有仓
审批时有仓，执行准备时变成无仓
审批时多仓，执行准备时变成空仓
审批时空仓，执行准备时变成多仓
审批时 position_side 与执行准备时 position_side 不一致
```

### 16.2 数量一致

以下情况必须 BLOCKED：

```text
审批时 position_amount 与执行准备时 position_amount 不一致
审批时持仓绝对值与执行准备时持仓绝对值不一致
方向一致但数量变化
```

允许：

```text
Decimal 格式等价
例如 0.010000000000 == 0.01
```

不允许：

```text
变化 5% 以内继续
变化 20% 以内继续
自动缩单
自动补单
自动改方向
自动重新拆解
```

### 16.3 变化后的处理

如果持仓方向或数量发生实质变化：

```text
ExecutionPreparationResult = BLOCKED
reason_code = position_side_changed 或 position_amount_changed
不生成 prepared 状态的 PreparedOrderIntent
释放当前 active lock
写 AlertEvent
```

后续应该重新走：

```text
账户同步
OrderPlan
RiskCheck
ExecutionPreparation
```

008 不负责自动重走上游。

---

## 17. reduce-only 校验

008 必须复核 `reduce_only` 语义仍然成立。

如果：

```text
reduce_only = true
```

则必须满足：

```text
1. 当前存在可被减少的同向持仓。
2. 当前持仓数量 > 0。
3. 请求数量 <= 当前可减少持仓数量。
4. side 必须是减少该持仓的方向。
5. position_side 必须与当前持仓方向匹配。
```

如果不满足：

```text
ExecutionPreparationResult = BLOCKED
reason_code = reduce_only_invalid
```

如果：

```text
reduce_only = false
```

仍必须满足：

```text
当前持仓方向和数量与 RiskCheck 审批时一致。
```

008 不允许因为当前持仓变化而把订单语义从：

```text
开仓变成加仓
平仓变成反手
反手变成单边开仓
```

---

## 18. 交易规则校验

### 18.1 交易规则来源

P0 交易规则来源：

```text
006B BinanceSymbolRuleSnapshot
```

来源要求：

```text
1. 必须与当前可消费 BinanceSyncRun 绑定。
2. symbol 匹配。
3. market_type 匹配。
4. account_domain 匹配。
5. 规则快照未过期或随账户同步批次可消费。
6. 与 RiskCheck 使用的规则快照相比，关键规则未变化。
```

如果缺失、过期或关键规则变化：

```text
ExecutionPreparationResult = BLOCKED
reason_code = symbol_rule_unavailable 或 symbol_rule_changed
```

### 18.2 校验内容

P0 必须校验：

```text
quantity > 0
quantity_unit 与上游一致，且属于 008 P0 合法组合
quantity 精度符合 quantity_step
quantity >= min_quantity
quantity <= max_quantity
notional >= min_notional
notional <= max_notional
```

如果不满足：

```text
ExecutionPreparationResult = BLOCKED
reason_code = exchange_rule_violation
```

008 不允许：

```text
为了满足交易规则而自动缩小数量
为了满足交易规则而自动放大数量
为了满足最小名义价值而补单
```

---

## 19. 数量单位

008 不重新定义数量单位。

必须沿用上游 `CandidateOrderIntent / ApprovedOrderIntent` 的数量单位。

008 P0 已支持的合法组合：

```text
USDS-M Futures + USDS-M account_domain + quantity
COIN-M Futures + COIN-M account_domain + contracts
```

endpoint_family 映射：

```text
USDS-M Futures / quantity -> endpoint_family = fapi
COIN-M Futures / contracts -> endpoint_family = dapi
```

COIN-M / contracts 是 008 P0 正常支持路径，不得被视为 unsupported。

COIN-M contracts 语义：

```text
contracts 表示 COIN-M 合约张数。
008 不把 contracts 转成 base asset。
008 不把 contracts 转成 quote asset。
008 不重新计算下单数量。
ApprovedOrderIntent.requested_size 作为合约张数冻结到 PreparedOrderIntent.quantity。
```

非法组合必须 BLOCKED：

```text
USDS-M / contracts
COIN-M / quantity
market_type 与 account_domain 不一致
未知 market_type
未知 account_domain
未知 quantity_unit
```

非法组合处理：

```text
ExecutionPreparationResult = BLOCKED
reason_code = unsupported_quantity_unit
```

---

## 20. 交易所请求参数冻结

008 必须把已批准订单意图冻结成交易所可提交参数。

P0 至少冻结：

```text
exchange
account_domain
market_type
symbol
side
position_side
order_type
quantity
quantity_unit
reduce_only
time_in_force
client_order_id
idempotency_key
```

### 20.1 订单类型

P0 只支持：

```text
MARKET
```

暂不支持：

```text
LIMIT
STOP_MARKET
TAKE_PROFIT_MARKET
TRAILING_STOP_MARKET
```

如果配置或输入要求非 P0 支持类型：

```text
ExecutionPreparationResult = BLOCKED
reason_code = unsupported_order_type
```

### 20.2 MARKET 的 time_in_force

P0 只支持 MARKET。  
MARKET 单不发送 `timeInForce`。

PreparedOrderIntent 中：

```text
time_in_force = not_applicable 或 null
```

禁止：

```text
为 MARKET 强行设置 IOC
为 MARKET 强行设置 GTC
为 MARKET 强行设置 FOK
```

`time_in_force` 的真实提交参数由 009 根据 `order_type` 处理。

### 20.3 Binance 参数映射

008 可以准备 Binance 参数，但不得提交。

P0 参数语义至少包括：

```text
endpoint_family
market_type
account_domain
symbol
side
positionSide
type
quantity
quantity_unit
reduceOnly
newClientOrderId
```

endpoint_family 映射：

```text
USDS-M Futures / quantity -> endpoint_family = fapi
COIN-M Futures / contracts -> endpoint_family = dapi
```

One-Way Mode：

```text
positionSide = BOTH
```

如果当前为 Hedge Mode 且 P0 未支持：

```text
ExecutionPreparationResult = BLOCKED
reason_code = hedge_mode_not_supported
```

---

## 21. client_order_id

008 必须生成唯一客户订单号。

要求：

```text
1. 同一个 ApprovedOrderIntent 只能生成一个稳定 client_order_id。
2. 重复运行必须返回同一个 client_order_id。
3. 不同 ApprovedOrderIntent 不得生成相同 client_order_id。
4. client_order_id 必须满足 Binance 长度和字符约束。
5. client_order_id 必须能追溯到系统内部对象。
6. client_order_id 一旦生成，不因 FAILED / EXPIRED 被清理或复用。
```

如果发生冲突：

```text
ExecutionPreparationResult = FAILED
reason_code = client_order_id_conflict
不允许进入 009
写 AlertEvent
```

---

## 22. 幂等性

008 必须幂等。

### 22.1 唯一约束

PreparedOrderIntent 必须具备数据库唯一约束：

```text
source_approved_order_intent_id unique
prepared_order_intent_key unique
client_order_id unique
idempotency_key unique
```

同一个 ApprovedOrderIntent 最多只能创建一份 PreparedOrderIntent。

### 22.2 并发保护

P0 使用：

```text
数据库事务
select_for_update
唯一约束
IntegrityError 后读取已有对象
```

不引入：

```text
分布式锁
Redis 锁
外部锁服务
```

### 22.3 幂等规则

```text
相同 ApprovedOrderIntent 重复运行：
  返回已有 PreparedOrderIntent。

同一 ApprovedOrderIntent 并发运行：
  最终只能创建一份 PreparedOrderIntent。

同一账户、市场、交易对存在另一条 active 订单链路：
  BLOCKED。

已有 PreparedOrderIntent 为 prepared 且未过期：
  返回已有对象，结果为 IDEMPOTENT_REPLAY。

已有 PreparedOrderIntent 已过期：
  标记 expired，返回 EXPIRED，不创建新对象。

已有 PreparedOrderIntent 已 submitted：
  不允许重新准备，不允许覆盖。
```

### 22.4 幂等 key 组成

idempotency_key 至少绑定：

```text
approved_order_intent_id
risk_check_result_id
candidate_order_intent_id
order_plan_id
symbol
market_type
account_domain
side
position_side
quantity
quantity_unit
reduce_only
order_type
```

实现必须防止：

```text
任务重试造成重复准备
并发请求造成重复准备
异常恢复后生成第二份待执行请求
```

---

## 23. active chain guard

008 必须检查同一：

```text
exchange
account_domain
market_type
symbol
```

是否存在其他未完成订单链路。

P0 使用：

```text
OrderPlanActiveLock
```

规则：

```text
如果 OrderPlanActiveLock 不存在：
  BLOCKED。

如果 OrderPlanActiveLock 不是 active：
  BLOCKED。

如果 OrderPlanActiveLock.active_order_plan_id 等于当前 ApprovedOrderIntent 所属 OrderPlan：
  允许继续。

如果 OrderPlanActiveLock 属于另一条 OrderPlan：
  BLOCKED。

如果当前 OrderPlan 已终结：
  BLOCKED。
```

原因：

```text
008 不能被自己的 007A / 007B active lock 挡住；
但必须阻止另一条不同订单链路并行进入执行准备。
```

---

## 24. trace_id 与 trigger_source

PreparedOrderIntent 必须保留全链路追踪信息。

规则：

```text
PreparedOrderIntent.trace_id 必须优先继承 ApprovedOrderIntent.trace_id。
如果调用方显式传入 trace_id，必须与上游 trace_id 兼容或作为 child_trace_id 记录。
trigger_source 必须继承或显式记录触发来源。
009 Execution 必须继续沿用同一个 trace_id。
AlertEvent 必须使用同一 trace_id。
```

不允许：

```text
008 生成全新的孤立 trace_id 导致链路断裂。
009 丢失 008 的 trace_id。
```

---

## 25. reference_time_utc

`reference_time_utc` 是本次 ExecutionPreparation 判断使用的统一时间点。

用途：

```text
判断价格快照是否过期
判断账户快照是否过期
计算 PreparedOrderIntent.expires_at_utc
保证测试可复现
```

要求：

```text
1. 必须是 UTC aware datetime。
2. 生产运行默认由 service 内部取当前 UTC。
3. 测试和管理命令可以显式传入。
4. 不得早于上游关键事实时间。
5. 如果 reference_time_utc 早于 PriceSnapshot.as_of_utc 或账户快照 as_of_utc，BLOCKED。
```

008 不负责 Binance server time 校验。  
交易所时间戳与 recvWindow 问题由 009 Binance 下单客户端处理。

---

## 26. AlertEvent / 审计要求

008 所有正式结果都必须写 AlertEvent 或等价审计记录。  
P0 使用项目现有 `AlertEvent`。

### 26.1 event_type

至少包括：

```text
execution_preparation_prepared
execution_preparation_blocked
execution_preparation_failed
execution_preparation_expired
execution_preparation_idempotent_replay
execution_preparation_active_chain_conflict
execution_preparation_stale_price
execution_preparation_price_deviation_exceeded
execution_preparation_stale_account
execution_preparation_account_state_changed
execution_preparation_position_changed
execution_preparation_reduce_only_invalid
execution_preparation_client_order_id_conflict
execution_preparation_exchange_rule_violation
```

### 26.2 最低审计字段

AlertEvent / evidence 至少应能追溯：

```text
trace_id
trigger_source
event_type
severity
reason_code
approved_order_intent_id
prepared_order_intent_id
order_plan_id
candidate_order_intent_id
risk_check_result_id
symbol
account_domain
market_type
price_snapshot_id
binance_sync_run_id
account_snapshot_id
position_snapshot_id
symbol_rule_snapshot_id
reference_price
latest_price
price_deviation_bps
client_order_id
idempotency_key
config_snapshot
```

要求：

```text
成功也写 AlertEvent。
失败也写 AlertEvent。
阻断也写 AlertEvent。
过期也写 AlertEvent。
幂等重放也写 AlertEvent 或审计日志。
不直接调用 Hermes。
不静默吞异常。
```

---

## 27. 配置项

P0 配置建议：

```text
EXECUTION_PREPARATION_PRICE_MAX_AGE_SECONDS = 30
EXECUTION_PREPARATION_MAX_PRICE_DEVIATION_BPS = 50
PREPARED_ORDER_INTENT_TTL_SECONDS = 30
EXECUTION_PREPARATION_SUPPORTED_ORDER_TYPES = MARKET
EXECUTION_PREPARATION_SUPPORTED_POSITION_MODE = one_way
```

配置要求：

```text
1. 必须通过 settings / env 读取。
2. 不得在 service 内硬编码不可覆盖值。
3. 测试中允许 override_settings 覆盖。
4. P0 不支持运行时热加载。
5. 每次 prepare 必须把实际使用配置写入 config_snapshot / evidence。
```

不支持热加载的原因：

```text
同一条执行链路必须能审计当时使用的阈值。
运行时热加载会增加审计复杂度，不属于 008 P0。
```

---

## 28. 管理命令 / 服务入口

008 P0 至少提供 service 入口：

```text
prepare_execution(approved_order_intent_id, reference_time_utc, trace_id, trigger_source)
```

可选管理命令：

```text
python manage.py prepare_execution --approved-order-intent-id <id>
```

命令职责：

```text
解析参数
调用 service
输出结果摘要
不直接实现业务逻辑
不调用 Binance
```

---

## 29. dry_run

008 P0 不实现 dry_run。

原因：

```text
dry_run 会引入额外状态语义：
是否创建审计
是否生成 client_order_id
是否参与幂等
是否推进父级状态
是否写 AlertEvent
是否允许后续 Execution
```

为避免 P0 复杂化：

```text
dry_run = P1
```

如果未来实现 P1 dry_run，必须单独定义：

```text
返回结构
校验结果
参数预览
是否写审计
是否生成临时 client_order_id
是否参与幂等
是否持久化
```

---

## 30. 008 与 009 的责任边界

008 成功后：

```text
PreparedOrderIntent.status = prepared
OrderPlanActiveLock 保持 active
```

009 负责：

```text
读取 PreparedOrderIntent
确认未过期
做真正提交前最后轻量检查
调用 Binance 下单接口
记录提交结果
推进 submitted / rejected / unknown 等提交状态
```

009 或后续订单终态模块负责：

```text
在明确终态后释放 OrderPlanActiveLock
```

如果 009 提交状态为 unknown：

```text
不得释放 OrderPlanActiveLock
不得让新订单链路进入
必须等待订单查询、状态同步或人工处理
```

008 不负责 009 失败后的锁释放。  
008 只负责自己阶段的 `blocked / failed / expired` 锁处理。

---

## 31. 测试要求

P0 测试必须覆盖：

### 31.1 成功路径

```text
ApprovedOrderIntent pending_execution_preparation
价格快照新鲜
价格类型为 mark_price
价格偏离 <= 0.5%
账户事实新鲜
持仓方向一致
持仓数量一致
reduce-only 语义成立
交易规则满足
生成 PreparedOrderIntent
PreparedOrderIntent.status = prepared
ApprovedOrderIntent.status = execution_prepared
client_order_id 存在且唯一
idempotency_key 存在且唯一
OrderPlanActiveLock 保持 active
写 AlertEvent
```

### 31.2 幂等路径

```text
同一个 ApprovedOrderIntent 重复 prepare
返回同一个 PreparedOrderIntent
不创建第二条
client_order_id 不变
idempotency_key 不变
结果为 IDEMPOTENT_REPLAY
写审计
```

### 31.3 并发路径

```text
同一个 ApprovedOrderIntent 并发 prepare
最终只能有一条 PreparedOrderIntent
另一条返回已有对象或被安全阻断
唯一约束不被破坏
```

### 31.4 价格阻断

```text
PriceSnapshot 缺失 → BLOCKED
PriceSnapshot price_type 不是 mark_price → BLOCKED
PriceSnapshot symbol 不匹配 → BLOCKED
PriceSnapshot market_type 不匹配 → BLOCKED
PriceSnapshot account_domain 不匹配 → BLOCKED
PriceSnapshot mark_price <= 0 → BLOCKED
PriceSnapshot 超过 30 秒 → BLOCKED
ApprovedOrderIntent.reference_price <= 0 → BLOCKED
价格偏离 > 0.5% → BLOCKED
```

### 31.5 账户阻断

```text
账户快照缺失 → BLOCKED
账户快照过期 → BLOCKED
账户域不一致 → BLOCKED
市场类型不一致 → BLOCKED
持仓模式变化 → BLOCKED
保证金模式变化 → BLOCKED
observed_exchange_leverage 变化 → BLOCKED
持仓方向变化 → BLOCKED
持仓数量变化 → BLOCKED
交易规则快照变化 → BLOCKED
```

### 31.6 reduce-only 阻断

```text
reduce_only=true 但当前无对应持仓 → BLOCKED
reduce_only=true 但请求数量超过可减少持仓 → BLOCKED
reduce_only=true 但 side 不是减少方向 → BLOCKED
reduce_only=true 但 position_side 不匹配 → BLOCKED
```

### 31.7 交易规则阻断

```text
数量 <= 0 → BLOCKED
数量单位不支持 → BLOCKED
数量精度不符合 quantity_step → BLOCKED
数量低于 min_quantity → BLOCKED
名义价值低于 min_notional → BLOCKED
数量超过 max_quantity → BLOCKED
名义价值超过 max_notional → BLOCKED
unsupported order type → BLOCKED
hedge mode unsupported → BLOCKED
MARKET 单 time_in_force 被错误设置为 GTC/IOC/FOK → BLOCKED 或测试失败
```

### 31.8 状态推进

```text
PREPARED 后父级状态正确，active lock 保持 active
BLOCKED 后父级状态终结并释放 active lock
FAILED 后父级状态失败并保留审计
EXPIRED 后父级状态过期并释放 active lock
IDEMPOTENT_REPLAY 不重复推进错误状态
```

### 31.9 过期路径

```text
PreparedOrderIntent 超过 TTL → EXPIRED
Expired 后 ApprovedOrderIntent 不可再次 prepare
Expired PreparedOrderIntent 不允许进入 009
不允许 expired 改回 prepared
```

### 31.10 禁止越界

```text
不调用 Binance 下单接口
不调用 Binance REST / WebSocket
不撤单
不查订单
不查成交
不修改杠杆
不修改保证金模式
不修改持仓模式
不重新生成 CandidateOrderIntent
不重新生成 ApprovedOrderIntent
不修改订单方向和数量
不自动缩单
不自动拆单
不实现 dry_run
```

---

## 32. 008 验收标准

008 requirements / plan / implementation 通过的最低标准：

```text
1. 明确 008 是 ExecutionPreparation，不是 Execution。
2. 明确 PreparedOrderIntent 与 ApprovedOrderIntent 的实质区别。
3. 明确不真实下单。
4. 明确不接 WebSocket。
5. 明确价格复核只用 mark_price。
6. 明确价格偏离参考价格来自 ApprovedOrderIntent.reference_price。
7. 明确持仓方向和数量采用 P0 精确一致规则。
8. 明确 reduce-only 复核规则。
9. 明确交易规则来源为 006B BinanceSymbolRuleSnapshot。
10. 明确 MARKET 单不发送 time_in_force。
11. 明确 client_order_id 和 idempotency_key 唯一。
12. 明确 PreparedOrderIntent 与 ApprovedOrderIntent 一对一。
13. 明确并发保护使用数据库事务、行锁和唯一约束。
14. 明确 active chain 判断使用 OrderPlanActiveLock。
15. 明确 PreparedOrderIntent 过期后 ApprovedOrderIntent 不可复用。
16. 明确 trace_id / trigger_source 继承。
17. 明确 AlertEvent / 审计要求。
18. 明确 008 成功后 active lock 保持 active，由 009 或后续终态模块释放。
19. 明确配置不热加载，但记录配置快照。
20. 明确测试覆盖所有 P0 边界。
```

---

## 33. 当前阶段最终结论

008 应该做，而且应该保持窄边界。

正确定位：

```text
风控批准后的订单意图
  ↓
执行前复核
  ↓
冻结交易所待提交请求
  ↓
等待 009 真实提交 Binance
```

008 的价值不是产生新的交易逻辑，而是为真实下单前建立：

```text
确定性
幂等性
唯一性
可过期性
可审计性
执行边界
```

如果 008 只是复制 ApprovedOrderIntent 字段，则没有价值。  
如果 008 开始真实下单，则越界。  
008 必须只做执行请求冻结。
