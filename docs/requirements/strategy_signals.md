# 005A StrategySignal 需求文档

> 当前阶段：005A StrategySignal  
> 上游：AtomicSignalSet / AtomicSignalValue  
> 下游：005B StrategySignalQuality / 006A DecisionSnapshot  
> 核心原则：fail-closed（无法可靠生成时阻断）
> 当前结论：005A 只生成合格的策略级市场判断，不生成交易决策。

---

## 1. 模块定位

StrategySignal 位于 AtomicSignal 之后、StrategySignalQuality / DecisionSnapshot 之前。

```text
MarketSnapshot
→ FeatureSet / FeatureValue
→ AtomicSignalSet / AtomicSignalValue
→ StrategyRouter / StrategySignal
→ StrategySignalQuality
→ DecisionSnapshot
→ Binance Account Sync / PriceSnapshot
→ OrderPlan
→ CandidateOrderIntent
→ RiskCheck
→ ApprovedOrderIntent
→ ExecutionPreparation
→ Execution
```

005A 的职责是：

```text
读取可用 AtomicSignalSet
读取可参与策略的 AtomicSignalValue
通过 StrategyRouteRule 选择 StrategyDefinition
按 StrategyDefinition 执行策略级信号聚合
处理 gatekeeper 可用性与正常中性状态
处理 voter 聚合与方向冲突记录
生成 StrategySignal
保存证据、输入快照、路由快照、聚合快照
保证幂等、可追溯、可审计，对无法可靠生成的场景 fail-closed
```

005A 不负责：

```text
下单
订单意图
交易执行
仓位建议
杠杆
止损
止盈
账户余额检查
当前持仓处理
最大仓位限制
交易频率限制
连续亏损冷却
最终风控否决
DecisionSnapshot 生成
回测收益评估
模拟盘
参数扫描
UI / 后台页面
```

---

## 2. 核心原则

### 2.1 fail-closed 仅用于无法可靠生成

005A 的 `blocked` 仅表示 StrategySignal 无法被可靠生成，不得用来表达正常市场状态。

只有当以下条件全部满足时，才允许生成可下游消费的 StrategySignal：

```text
AtomicSignalSet 可用
存在匹配的 active + enabled StrategyRouteRule
StrategyRouteRule 选中的 StrategyDefinition 存在
StrategyDefinition active + enabled
StrategyDefinition 配置合法
参与策略的 AtomicSignalValue 足够且有效
required atomic signals 完整
required gatekeeper 存在且有效
voter 聚合成功
结果校验通过
```

如果系统、配置或输入不可用导致上述条件无法满足，必须阻断：

```text
status = blocked / failed
is_usable = False
allows_downstream = False
```

不得在输入不合格时硬生成 bullish / bearish / neutral 策略信号。

以下属于正常策略状态，不得使用 blocked 表达：

```text
多空冲突
方向强度不足
趋势不明显
信号偏弱
gatekeeper 条件正常不满足
```

这些情况应生成：

```text
status = created
direction = neutral
is_usable = True
allows_downstream = True
```

这些 neutral 状态不写 AlertEvent，继续交给后续 DecisionSnapshot 判断 NO_TRADE / NO_TARGET_CHANGE / TARGET_POSITION。

这些 neutral 状态必须写入完整证据，不得只写空白中性结果。

至少必须写入：

```text
evidence_items
evidence_text_zh
payload_summary
aggregation_snapshot
conflict_snapshot
```

这些字段必须说明：

```text
为什么结果是 neutral
哪些 AtomicSignalValue 参与了判断
多头证据分别是什么
空头证据分别是什么
最终为什么没有形成 bullish 或 bearish
是否存在多空冲突、弱信号、趋势不明显、gatekeeper 条件正常不满足等正常市场状态
```

### 2.2 策略信号不是交易决策

StrategySignal 的 direction 只代表策略级市场判断，不代表交易动作。

允许输出：

```text
direction
strength
confidence
evidence_text_zh
evidence_items
payload_summary
input_snapshot
routing_snapshot
aggregation_snapshot
conflict_snapshot
```

禁止输出：

```text
entry_price
stop_loss
take_profit
position_size
leverage
candidate_order_intent
approved_order_intent
open_long
open_short
close_position
NO_TRADE
HOLD
```

说明：

```text
005A 不判断是否真实交易。
005A 不生成目标仓位、订单方向、订单数量、reduce_only、止损止盈或执行参数。
后续是否进入目标仓位由 006A DecisionSnapshot 判断。
候选订单由 007A OrderPlan 生成。
风控审批由 007B RiskCheck 完成，且必须基于 007A OrderPlan 生成的 CandidateOrderIntent。
执行前准备和真实执行分别由 ExecutionPreparation / Execution 完成。
```

### 2.3 路由配置化，不硬编码

StrategyRouter 不得使用硬编码 if-else 固定映射策略。

禁止：

```python
if regime == "trending":
    return "trend_following"
elif regime == "ranging":
    return "mean_reversion"
```

必须采用：

```text
StrategyDefinition = 策略注册表
StrategyRouteRule = 策略路由表
StrategyRouterService = 读取路由表选择策略
```

005A 第一版可以只配置一条固定默认路由，例如：

```text
router_mode = fixed_default
selected_strategy_code = trend_following_v1
```

但该默认选择必须来自数据库或配置表，不得写死在服务代码中。

---

## 3. 输入边界

### 3.1 允许读取

StrategySignalService / StrategyRouterService 允许读取：

```text
AtomicSignalSet
AtomicSignalValue
AtomicSignalDefinition
StrategyDefinition
StrategyRouteRule
```

可以通过外键或冗余字段引用：

```text
FeatureSet
MarketSnapshot
trace_id
market identity
```

但不得重新计算上游数据。

### 3.2 禁止读取或调用

005A 禁止读取或调用：

```text
Kline4h / Kline1d
Binance REST
WebSocket
MarketSnapshotService
FeatureLayerService
AtomicSignalService
DataQualityService
BackfillService
StrategySignalQualityService
DecisionSnapshotService
Binance Account Sync
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
账户余额
当前持仓
订单簿
成交记录
```

原因：005A 必须保持与回测 / 实盘一致的纯信号生成逻辑。

---

## 4. StrategyDefinition 需求

StrategyDefinition 是策略注册表，负责描述“策略怎么算”。

### 4.1 基础字段

建议字段：

```text
strategy_code
strategy_name
strategy_version
strategy_mode
algorithm_name
algorithm_version
params
params_hash
definition_hash
status
enabled
description
created_at
updated_at
```

### 4.2 生命周期

支持状态：

```text
draft
active
deprecated
retired
disabled
```

生产环境只允许使用：

```text
status = active
enabled = True
```

### 4.3 版本管理规则

StrategyDefinition 必须沿用 FeatureDefinition / AtomicSignalDefinition 的版本管理方式：

```text
参数变化 → 新 strategy_version 或新 definition
算法变化 → 新 algorithm_version
已用于生产的身份字段不得原地修改
不得物理删除已用于生产的定义
废弃策略使用 disabled / deprecated / retired
```

关键身份字段包括但不限于：

```text
strategy_code
strategy_version
algorithm_name
algorithm_version
params_hash
definition_hash
```

### 4.4 params 内容

params 至少支持：

```text
allowed_signal_codes
required_signal_codes
required_gatekeeper_codes
optional_gatekeeper_codes
static_weights
min_voter_count
min_total_strength
direction_threshold
conflict_policy
neutral_policy
```

示例：

```json
{
  "allowed_signal_codes": [
    "sma_4h_20_above_sma_4h_60"
  ],
  "required_signal_codes": [],
  "required_gatekeeper_codes": [],
  "optional_gatekeeper_codes": [],
  "static_weights": {
    "sma_4h_20_above_sma_4h_60": 1.0
  },
  "min_voter_count": 1,
  "min_total_strength": 0.0,
  "direction_threshold": 0.2,
  "conflict_policy": "allow_neutral",
  "neutral_policy": "allow_neutral"
}
```

---

## 5. StrategyRouteRule 需求

StrategyRouteRule 是策略选择器，负责描述“当前该运行哪个策略”。

### 5.1 基础字段

建议字段：

```text
route_rule_code
route_rule_version
router_mode
priority
match_conditions
selected_strategy_code
selected_strategy_version
fallback_strategy_code
fallback_strategy_version
fallback_policy
status
enabled
valid_from
valid_to
params
params_hash
definition_hash
created_at
updated_at
```

### 5.2 生命周期

同样支持：

```text
draft
active
deprecated
retired
disabled
```

生产环境只允许使用：

```text
status = active
enabled = True
```

### 5.3 第一版路由模式

005A 第一版必须支持：

```text
fixed_default
```

含义：

```text
不根据市场环境自动切换策略
通过路由表配置固定选择某个默认策略
```

示例：

```json
{
  "route_rule_code": "default_route_v1",
  "route_rule_version": "v1",
  "router_mode": "fixed_default",
  "priority": 100,
  "match_conditions": {},
  "selected_strategy_code": "trend_following_v1",
  "selected_strategy_version": "v1",
  "fallback_strategy_code": null,
  "fallback_policy": "none",
  "status": "active",
  "enabled": true
}
```

### 5.4 后续预留路由模式

P2 预留但 005A 不实现：

```text
regime_based
context_based
multi_condition
soft_route
```

其中 `regime_based` 后续用于：

```text
market_regime = trending → trend_following
market_regime = ranging → mean_reversion
market_regime = high_volatility → defensive / observe_only
```

但 005A 第一版不得实现自动市场环境切换。

### 5.5 fallback 规则

默认不启用 fallback。

如果 RouteRule 选中的 StrategyDefinition 不存在 / disabled / inactive / retired：

```text
StrategySignal.status = blocked
is_usable = False
allows_downstream = False
blocked_reason = selected_strategy_unavailable
```

不得自动寻找其他策略。

fallback 只有在显式配置时才允许使用，且必须满足：

```text
fallback_strategy active + enabled
fallback_policy 显式允许
routing_snapshot 记录 fallback 原因
```

005A 第一版建议不实现 fallback 执行，只预留字段。

---

## 6. StrategyRouterService 需求

StrategyRouterService 负责根据 StrategyRouteRule 选择 StrategyDefinition。

### 6.1 路由流程

```text
1. 读取 active + enabled StrategyRouteRule
2. 按 priority 排序
3. 过滤当前有效时间窗口 valid_from / valid_to
4. 匹配 match_conditions
5. 选择第一条匹配规则
6. 根据 selected_strategy_code / selected_strategy_version 查找 StrategyDefinition
7. 校验 StrategyDefinition active + enabled
8. 返回 selected StrategyDefinition 和 routing_snapshot
```

### 6.2 阻断场景

以下情况必须阻断：

```text
no_active_route_rule
multiple_route_rules_conflict
selected_strategy_not_found
selected_strategy_disabled
selected_strategy_not_active
selected_strategy_retired
route_rule_invalid
fallback_required_but_unavailable
```

### 6.3 routing_snapshot

每次路由必须记录：

```text
route_rule_id
route_rule_code
route_rule_version
router_mode
matched_conditions
selected_strategy_id
selected_strategy_code
selected_strategy_version
fallback_used
fallback_reason
route_reason
```

即使第一版只是 fixed_default，也必须记录 routing_snapshot。

---

## 7. AtomicSignal role 使用需求

AtomicSignal 层已经负责定义并保存 role。005A 不重新定义 role，只消费和校验。

005A 支持的 role：

```text
voter
gatekeeper
filter
descriptor
scaler
```

### 7.1 voter

voter 参与方向聚合。

要求：

```text
is_valid = True
participates_in_strategy = True
status = created
role = voter
signal_code 在 allowed_signal_codes 中
```

### 7.2 gatekeeper

gatekeeper 用于策略信号前置条件记录。005A 中，gatekeeper 缺失或无效属于无法可靠生成；gatekeeper 条件正常不满足属于正常市场状态。

005A 规则：

```text
required_gatekeeper 缺失 → blocked
required_gatekeeper 无效 → blocked
required_gatekeeper 条件正常不满足 → created neutral
optional_gatekeeper 第一版只记录，不影响结果
```

gatekeeper 不得输出 NO_TRADE / HOLD / 仓位 / 订单。005A 不用 gatekeeper 正常不满足来写 AlertEvent，也不提前承担 DecisionSnapshot 职责。

### 7.3 filter

005A 第一版只记录 filter，不实现复杂策略分支过滤。

P2 可扩展：

```text
filter 过滤特定策略分支
filter 降低部分 voter 权重
filter 与 market_regime 组合
```

### 7.4 descriptor

descriptor 用于描述上下文，例如未来的 market_regime_descriptor。

005A 第一版可以记录 descriptor 到 input_snapshot / evidence_items，但不根据 descriptor 自动切换策略。

### 7.5 scaler

scaler 不属于 005A。

005A 不处理：

```text
scale_factor
position_size
仓位缩放
杠杆
```

如果出现 scaler role，005A 第一版只允许记录为 ignored_or_reserved，不参与计算。

---

## 8. StrategySignal 聚合算法需求

005A 第一版实现一个简单、可审计的规则算法。

建议算法：

```text
atomic_vote
algorithm_version = v1
```

### 8.1 输入过滤

只使用：

```text
AtomicSignalSet.status = created
AtomicSignalSet.is_usable = True
AtomicSignalSet.allows_downstream = True
AtomicSignalValue.status = created
AtomicSignalValue.is_valid = True
AtomicSignalDefinition.participates_in_strategy = True
signal_code 在 StrategyDefinition.allowed_signal_codes 中
```

### 8.2 静态权重

005A 只使用 static_weight。

```text
effective_strength = strength × confidence × static_weight
```

static_weight 来自 StrategyDefinition.params。

如果 signal_code 没有配置 static_weight：

```text
默认 1.0，或根据配置决定 blocked
```

建议第一版默认 1.0，并在 aggregation_snapshot 中记录。

### 8.3 动态权重

动态权重不在 005A 实现，但必须记录为 P2。

P2 需求：

```text
基于历史表现、市场环境、策略状态动态调整 static_weight
必须支持回测验证、样本外验证、交易成本模型后再启用
动态权重结果必须写入 snapshot
防止 overfitting 和 look-ahead bias
```

### 8.4 方向聚合

计算：

```text
bullish_total = sum(effective_strength where direction = bullish)
bearish_total = sum(effective_strength where direction = bearish)
neutral_count = count(direction = neutral)
```

方向判断：

```text
if bullish_total - bearish_total > direction_threshold:
    direction = bullish
elif bearish_total - bullish_total > direction_threshold:
    direction = bearish
else:
    direction = neutral
```

### 8.5 confidence 计算

建议第一版：

```text
confidence = abs(bullish_total - bearish_total) / (bullish_total + bearish_total + epsilon)
```

结果必须限制在：

```text
0 <= confidence <= 1
```

### 8.6 strength 计算

建议第一版：

```text
strength = max(bullish_total, bearish_total)
```

并限制在：

```text
0 <= strength <= 1
```

如果可能超过 1，需要进行 cap：

```text
strength = min(strength, 1)
```

### 8.7 中文策略聚合算法说明文档要求

StrategySignal implementation 必须维护：

```text
docs/implementation/strategy_signal/
```

每个默认策略聚合算法都必须有对应中文说明文档。

例如：

```text
atomic_vote:v1
```

中文说明至少包括：

```text
基本信息
中文名
用途
输入 AtomicSignalSet / AtomicSignalValue
可参与信号筛选条件
role 处理方式
gatekeeper / filter 处理方式
static_weight 使用方式
direction 计算逻辑
strength 计算逻辑
confidence 计算逻辑
冲突处理逻辑
阻断条件
失败条件
输出字段
交易语义边界
```

规则：

```text
中文说明文档只解释策略聚合算法和边界。
不得把 StrategySignal 解释成最终订单。
不得写真实下单逻辑。
不得写 Binance API 调用逻辑。
不得绕过 DecisionSnapshot、OrderPlan、RiskCheck 或 ExecutionPreparation。
```

如果新增默认 StrategyDefinition.algorithm_name / algorithm_version，必须同步新增或更新对应中文说明文档。

---

## 9. 冲突处理需求

必须记录方向冲突。

冲突字段：

```text
conflict_detected
conflict_reason
bullish_total
bearish_total
neutral_count
bullish_count
bearish_count
voter_count
```

建议规则：

```text
bullish_total > 0 且 bearish_total > 0 → conflict_detected = True
差值不足 threshold → direction = neutral
多空冲突、方向强度不足、趋势不明显 → direction = neutral
```

第一版可以简化：

```text
差值不足 threshold → neutral
005A 不实现多空冲突阻断
005A 不把强度不足作为 blocked
```

neutral 是有效中性判断；blocked 只表示 StrategySignal 无法被可靠生成。

### 9.1 neutral 证据要求

当 `direction=neutral` 且 `status=created` 时，必须保留完整审计证据。

`payload_summary` 至少包含：

```text
neutral_reason
participating_atomic_signal_codes
bullish_evidence
bearish_evidence
neutral_evidence
why_not_bullish
why_not_bearish
normal_market_state_flags
```

`normal_market_state_flags` 至少用于表达：

```text
conflict_detected
weak_signal
unclear_trend
gatekeeper_condition_not_met
```

`aggregation_snapshot` 必须能解释多头/空头强度、阈值和最终 neutral 结论；`conflict_snapshot` 必须能解释是否存在冲突。`evidence_text_zh` 必须用中文说明为什么没有形成 bullish 或 bearish。

---

## 10. StrategySignal 模型需求

建议字段：

```text
strategy_signal_key
atomic_signal_set
feature_set
market_snapshot
strategy_definition
strategy_route_rule
exchange
market_type
symbol
base_timeframe
higher_timeframe
strategy_code
strategy_version
algorithm_name
algorithm_version
params_hash
definition_hash
route_rule_code
route_rule_version
direction
strength
confidence
status
is_usable
allows_downstream
blocked_reason
error_code
error_message
used_atomic_signal_value_ids
input_snapshot
routing_snapshot
role_group_snapshot
aggregation_snapshot
conflict_snapshot
evidence_items
evidence_text_zh
payload_summary
trace_id
trigger_source
created_at
updated_at
```

### 10.1 direction

允许值：

```text
bullish
bearish
neutral
none
```

语义：

```text
bullish = 策略级偏多市场判断
bearish = 策略级偏空市场判断
neutral = 有效中性判断
none = 无有效策略方向，通常用于 blocked / failed
```

### 10.2 status

允许值：

```text
created
blocked
failed
```

下游只允许消费：

```text
status = created
is_usable = True
allows_downstream = True
```

---

## 11. 幂等需求

StrategySignal 必须幂等。

建议 strategy_signal_key 基于：

```text
atomic_signal_set_id
strategy_definition_id
strategy_code
strategy_version
algorithm_name
algorithm_version
params_hash
route_rule_id
route_rule_version
strategy_signal_schema_version
```

相同输入重复执行：

```text
返回已有 StrategySignal
不得重复写入 StrategySignal
不得重复写 AlertEvent
```

如果已有记录为 failed / blocked，是否允许重算需要明确参数控制。

建议第一版：

```text
默认返回已有记录
--force 参数后续再考虑
```

---

## 12. 失败与阻断处理

### 12.1 blocked

blocked 表示系统、配置、路由或输入不可用，导致 StrategySignal 无法被可靠生成。

blocked 不表示正常市场状态。多空冲突、方向强度不足、趋势不明显、信号偏弱、gatekeeper 条件正常不满足，都必须生成 created + neutral，并保持 allows_downstream=True。

典型场景：

```text
AtomicSignalSet 不可用
没有 active route rule
没有匹配 route rule
选中策略不存在
选中策略 disabled / inactive / retired
StrategyDefinition 配置非法
没有 participating atomic signals
没有有效 voter
required signal 缺失
required gatekeeper 缺失
required gatekeeper 无效
路由不可用
聚合算法不可用
```

### 12.2 failed

failed 表示服务执行异常或结果校验失败。

典型场景：

```text
数据库异常
字段校验失败
算法异常
snapshot 序列化失败
strength / confidence 越界
transaction 回滚
```

### 12.3 写入要求

必须使用：

```text
transaction.atomic
full_clean
唯一约束
```

不得半写。

---

## 13. AlertEvent 需求

需要写 AlertEvent 的场景：

```text
StrategySignal blocked
StrategySignal failed
no_active_route_rule
selected_strategy_unavailable
no_participating_atomic_signals
required_signal_missing
required_gatekeeper_missing
required_gatekeeper_invalid
strategy_signal_validation_failed
```

不写 AlertEvent 的场景：

```text
正常 created
created neutral
多空冲突 / 方向强度不足 / 趋势不明显 / 信号偏弱
gatekeeper 条件正常不满足
dry-run
重复幂等返回已有记录
```

---

## 14. dry-run 需求

StrategySignalService 必须支持 dry-run。

dry-run 时：

```text
不写 StrategySignal
不写 AlertEvent
不修改数据库
返回 routing_snapshot
返回 payload_summary
返回 aggregation_snapshot
返回 evidence_text_zh
返回 blocked / failed 摘要
```

---

## 15. seed 需求

005A 需要提供默认 seed。

### 15.1 默认 StrategyDefinition

第一版可以 seed 一个默认策略：

```text
strategy_code = trend_following_v1
strategy_version = v1
algorithm_name = atomic_vote
algorithm_version = v1
status = active
enabled = True
```

注意：该策略是否真正能生成 created，仍取决于是否存在可参与策略的 AtomicSignalValue。

如果当前原子信号仍全部是观察模式：

```text
StrategySignal 应 blocked
blocked_reason = no_participating_atomic_signals
```

### 15.2 默认 StrategyRouteRule

第一版 seed 一条 fixed_default 路由：

```text
route_rule_code = default_route_v1
route_rule_version = v1
router_mode = fixed_default
selected_strategy_code = trend_following_v1
status = active
enabled = True
```

---

## 16. management command 需求

建议提供命令：

```bash
python manage.py generate_strategy_signal --atomic-signal-set-id <id>
```

支持参数：

```text
--atomic-signal-set-id
--dry-run
--trace-id
--trigger-source
```

可选后续参数：

```text
--route-rule-code
--strategy-code
--force
```

第一版不建议启用 `--force`。

---

## 17. 测试需求

至少覆盖：

```text
AtomicSignalSet 不可用 → blocked
没有 active route rule → blocked
route rule 选中策略不存在 → blocked
route rule 选中策略 disabled → blocked
不自动 fallback 到其他策略
没有 participating atomic signals → blocked
没有有效 voter → blocked
required signal 缺失 → blocked
required gatekeeper 缺失 → blocked
required gatekeeper 无效 → blocked
required gatekeeper 条件正常不满足 → created neutral
有效 bullish voter → created bullish
有效 bearish voter → created bearish
bullish / bearish 差值不足 threshold → created neutral
多空冲突 → created neutral
invalid AtomicSignalValue 被忽略
不在 allowed_signal_codes 的信号被忽略
role 不匹配 → blocked / failed
strength / confidence 越界 → failed
重复执行幂等返回已有记录
dry-run 不写库、不写 AlertEvent
blocked / failed 写 AlertEvent
正常 created 不写 AlertEvent
```

---

## 18. P2 后续需求记录

以下需求必须记录，但不进入 005A 实现范围：

```text
动态权重 dynamic_weight
市场环境自动路由 regime_based router
多策略模式自动切换
soft routing
filter 复杂分支过滤
descriptor 影响策略选择或权重
fallback 策略执行
参数热更新
回测收益评估
paper trading
参数扫描
策略表现统计
scaler 仓位缩放
持仓状态融合
StrategySignalQuality
DecisionSnapshot
Binance Account Sync
PriceSnapshot
OrderPlan
CandidateOrderIntent
RiskCheck
ApprovedOrderIntent
ExecutionPreparation
Execution
UI / 后台页面
```

---

## 19. 005A 验收标准

005A 完成后必须满足：

```text
StrategyDefinition 可注册策略
StrategyRouteRule 可配置默认策略选择
StrategyRouterService 不硬编码具体策略
StrategySignalService 可执行选中的策略定义
没有合格策略输入时默认 blocked
选中策略不可用时 blocked，不自动找其他策略
voter 聚合可生成 bullish / bearish / neutral
required gatekeeper 缺失或无效时 blocked；gatekeeper 条件正常不满足时 created neutral
结果可追溯到 AtomicSignalSet / FeatureSet / MarketSnapshot
routing_snapshot / payload_summary / aggregation_snapshot / evidence_text_zh 完整
幂等写入
失败不半写
blocked / failed 有 AlertEvent
测试覆盖主要失败路径和成功路径
```

---

## 20. 最终边界声明

005A 的最终目标是：

```text
建立配置化策略注册与路由架构，
基于可用 AtomicSignal 生成可审计、可追溯、无法可靠生成时阻断的 StrategySignal。
```

005A 不是：

```text
风控模块
交易决策模块
仓位管理模块
订单模块
回测收益模块
策略优化模块
UI 模块
```

一句话：

```text
StrategySignal 只回答“当前是否形成合格的策略级市场判断”，不回答“是否应该真实交易”。
```
