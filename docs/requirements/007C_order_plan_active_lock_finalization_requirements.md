# 007C OrderPlan ActiveLock Finalization Requirements

> 建议路径：`docs/requirements/order_plan_active_lock_finalization.md`  
> 阶段：007C  
> 模块归属：`apps.order_plan`  
> 上游：`007A OrderPlan`、`007B RiskCheck`  
> 触发方：`008 ExecutionPreparation`、`009 OrderSubmission`、`010 OrderStatusSync`、`011 FillSync`  
> 非触发方：`012 Account Overview`、`013 Pipeline Orchestrator`  
> 状态：需求定稿草案  
> 核心结论：订单锁属于 007 OrderPlan 体系；007C 负责统一收尾 `OrderPlanActiveLock`，后续模块只调用 007C，不得各自直接修改锁。

---

## 1. 背景

007A 已经实现 `OrderPlanActiveLock`。

它的目的不是表示真实交易所订单状态，而是：

```text
防止同一个 market_type + account_domain + symbol 下，
在上一条订单链路尚未结束时，
重复生成新的 active OrderPlan / CandidateOrderIntent，
从而导致重复推进、重复风控、重复准备、重复下单。
```

当前 007A / 007B 已有事实：

```text
007A：
  创建 OrderPlan / CandidateOrderIntent。
  当 OrderPlan.status = created 时创建或占用 OrderPlanActiveLock。
  如果同一市场身份已有其他 active lock，则阻断新的 OrderPlan。

007B：
  ALLOW：
    风控通过，创建 ApprovedOrderIntent。
    OrderPlanActiveLock 继续保持 active，留给 008 / 009 / 010 / 011 后续链路。

  DENY：
    风控拒绝，本轮订单链路结束。
    当前锁释放。

  BLOCKED：
    风控阻断，本轮订单链路结束。
    当前锁释放。

  FAILED：
    风控系统异常。
    当前锁标记 failed。
```

因此，007C 不是从零开始设计订单锁，而是补齐：

```text
007B ALLOW 之后，
订单进入 008 / 009 / 010 / 011 后，
什么时候可以释放锁；
什么时候必须继续保持锁；
谁负责统一修改锁状态。
```

---

## 2. 007C 阶段定位

007C 的正式名称：

```text
007C OrderPlan ActiveLock Finalization
```

007C 只负责一件事：

```text
根据订单链路后续阶段产生的明确事实，
统一收尾 OrderPlanActiveLock。
```

007C 不负责：

```text
生成 OrderPlan
生成 CandidateOrderIntent
执行 RiskCheck
准备 Binance 请求参数
提交 Binance 订单
查询 Binance 订单状态
查询成交明细
更新持仓
账户总览同步
订单对账
运行编排
```

007C 是 007 OrderPlan 体系的一部分，因为锁本身属于 `OrderPlanActiveLock`。

---

## 3. 代码归属

007C 代码应放在 `apps.order_plan` 内。

推荐新增：

```text
apps/order_plan/lock_finalization.py
```

或放入：

```text
apps/order_plan/services.py
```

但必须有独立服务类，例如：

```text
OrderPlanActiveLockFinalizationService
```

统一锁服务负责：

```text
释放锁
标记锁 failed
保持锁 active
记录锁收尾原因
记录锁收尾来源阶段
记录锁收尾来源对象
写 AlertEvent / 审计
```

其他模块不得直接修改 `OrderPlanActiveLock.status`。

---

## 4. 修改权规则

只有 007C 统一锁服务可以真实修改锁状态。

允许调用 007C 的模块：

```text
007B RiskCheck
008 ExecutionPreparation
009 OrderSubmission
010 OrderStatusSync
011 FillSync
人工管理命令 / 运维命令
```

不允许直接修改锁状态的模块：

```text
008 ExecutionPreparation
009 OrderSubmission
010 OrderStatusSync
011 FillSync
012 Account Overview
013 Pipeline Orchestrator
```

这些模块如果需要推进锁生命周期，必须调用 007C 服务。

特别说明：

```text
013 只记录锁最终状态；
013 不释放锁；
013 不标记锁 failed；
013 不兜底解锁。
```

---

## 5. 007C 与 007B 的关系

007B 当前已经有锁生命周期推进逻辑。

007C 实现后，007B 的锁推进也应改为调用 007C 服务，而不是在 007B 内部直接写：

```text
lock.status = released
lock.status = failed
```

007B 场景保持原语义：

```text
RiskCheck ALLOW：
  锁继续 active，不释放。

RiskCheck DENY：
  调用 007C 释放锁。
  原因：risk_denied。

RiskCheck BLOCKED：
  调用 007C 释放锁。
  原因：risk_blocked。

RiskCheck FAILED：
  调用 007C 标记锁 failed。
  原因：risk_failed。
```

007C 不改变 007B 的业务判断，只集中锁状态修改。

---

## 6. 锁状态口径

当前已有锁状态：

```text
active
released
failed
```

007C P0 可以继续使用这三个状态。

含义：

```text
active：
  该 market_type + account_domain + symbol 仍有一条未结束订单链路。
  后续 007A 不得生成新的不同 active OrderPlan。

released：
  该订单链路已经明确结束，允许后续周期重新生成新 OrderPlan。

failed：
  该订单链路发生系统异常或风险未知，需要保留异常事实。
  后续是否允许重新接管，必须通过明确规则或人工处理。
```

如果当前模型缺少收尾原因字段，007C 实现时应补充审计字段或至少把原因写入 AlertEvent / evidence。

建议新增字段：

```text
finalized_reason_code
finalized_by_stage
finalized_object_type
finalized_object_id
finalized_at_utc
finalization_evidence
```

如果不新增字段，也必须通过 AlertEvent 和相关对象证据保证可追溯。

---

## 7. 可以释放锁的场景

### 7.1 007B 风控拒绝

条件：

```text
RiskCheck 明确 DENY。
没有 ApprovedOrderIntent 进入后续执行准备。
```

动作：

```text
007C release lock
reason_code = risk_denied
```

### 7.2 007B 风控阻断

条件：

```text
RiskCheck 明确 BLOCKED。
后续不允许进入 ExecutionPreparation。
```

动作：

```text
007C release lock
reason_code = risk_blocked
```

### 7.3 008 执行准备阻断

条件：

```text
008 明确 BLOCKED。
PreparedOrderIntent 没有生成可提交请求。
Binance 没有收到订单。
```

动作：

```text
007C release lock
reason_code = execution_preparation_blocked
```

### 7.4 008 执行准备失败且未产生可提交请求

条件：

```text
008 FAILED。
没有 PreparedOrderIntent 进入可提交状态。
没有 Binance 请求可能被发送。
```

动作：

```text
007C mark failed 或 release，取决于失败性质。
```

P0 建议：

```text
如果是业务阻断 / 输入事实不满足：
  release

如果是系统异常 / 数据库异常 / 不确定是否产生可提交请求：
  failed
```

### 7.5 009 提交前失败

条件：

```text
009 failed_before_submit。
Binance POST /order 没有发出。
```

动作：

```text
007C release lock
reason_code = order_submission_failed_before_submit
```

### 7.6 009 提交前阻断

条件：

```text
009 blocked_before_submit。
例如交易开关关闭、prepared intent 不可消费、payload 校验失败。
Binance POST /order 没有发出。
```

动作：

```text
007C release lock
reason_code = order_submission_blocked_before_submit
```

### 7.7 009 被交易所明确拒绝

条件：

```text
009 rejected。
Binance 明确返回订单未被接受。
没有真实订单进入交易所订单簿。
```

动作：

```text
007C release lock
reason_code = order_submission_rejected
```

### 7.8 011 成交同步完成

条件：

```text
009 accepted。
010 已经确认订单状态。
011 已完成成交明细同步，并生成订单成交汇总。
```

动作：

```text
007C release lock
reason_code = fill_sync_completed
```

对于当前市价单链路，011 成交同步完成是最稳妥的正常收尾点。

### 7.9 人工确认收尾

条件：

```text
人工确认某条锁已不再需要继续保护。
```

动作：

```text
007C release lock 或 mark failed
reason_code = manual_finalization
```

人工收尾必须写 AlertEvent，并记录 operator / trigger_source / reason。

---

## 8. 不得释放锁的场景

### 8.1 009 accepted

条件：

```text
Binance 已接受订单。
```

动作：

```text
不释放锁。
继续等待 010 / 011。
```

原因：

```text
accepted 只说明交易所接受了订单，不代表本地成交事实已经同步完成。
```

### 8.2 009 unknown

条件：

```text
009 无法确认 Binance 是否收到订单。
```

动作：

```text
不释放锁。
锁继续 active 或进入需要人工关注状态。
```

原因：

```text
不能证明订单没有提交或没有成交；
误释放会导致下一轮重复下单风险。
```

### 8.3 010 unknown

条件：

```text
010 查询订单状态失败、超时、限频、网络错误或响应不确定。
```

动作：

```text
不释放锁。
等待恢复查询、人工排查或后续订单诊断模块。
```

### 8.4 010 not_found

条件：

```text
010 单笔订单查询明确返回 not_found。
```

动作：

```text
P0 默认不自动释放锁。
```

原因：

```text
not_found 只能说明这次查询未找到订单；
在 009 曾经出现 accepted 或 unknown 的情况下，
不能直接等价于“绝对没有成交”。
```

如果未来要允许 not_found 解锁，必须单独设计更严格条件，例如：

```text
009 明确 failed_before_submit；
或 client_order_id 从未发送；
或人工确认；
或后续诊断模块多次确认。
```

### 8.5 011 unknown

条件：

```text
011 成交同步失败、超时、限频、响应不确定。
```

动作：

```text
不释放锁。
```

### 8.6 011 synced_empty

条件：

```text
011 查询成功但返回 0 条成交。
```

动作：

```text
P0 不自动释放锁，除非 010 状态明确为终态无成交且需求另行定义。
```

原因：

```text
市价单正常应快速成交；
0 成交需要关注，011 只记录事实，不承担订单对账。
```

---

## 9. 010 的锁边界

010 只查询订单状态。

010 默认不解锁。

010 允许调用 007C 的唯一方向是：

```text
记录“不得解锁”的保护结果；
或在未来明确设计的订单诊断规则下触发人工 / 诊断收尾。
```

P0 010 不得因为以下状态自动解锁：

```text
not_found
unknown
partially_filled
new
accepted
```

010 如果查到订单已取消、过期或明确终态，也不得自行直接改锁，必须通过 007C 服务，并且该规则需要在 010 / 007C plan 中明确。

---

## 10. 011 的锁边界

011 查询成交明细并生成订单成交汇总。

011 是当前市价单链路最主要的正常解锁触发方。

当 011 满足以下条件时，可以调用 007C 释放锁：

```text
成交明细查询成功；
成交明细已幂等落库；
订单成交汇总已生成；
汇总结果可关联 009 提交记录和 010 状态记录；
本次同步结果不是 unknown；
没有发现重复累计或数据完整性异常。
```

如果 011 返回：

```text
unknown
sync_failed
query_failed
integrity_failed
```

不得释放锁。

如果 011 返回：

```text
synced_empty
```

P0 不自动释放锁，必须记录需要人工关注。

---

## 11. 与 012 的关系

012 Account Overview / One-Click Sync 是独立账户总览模块。

012 不得：

```text
释放 OrderPlanActiveLock
标记 OrderPlanActiveLock failed
根据账户余额 / 持仓变化判断某笔订单已经完成
根据账户快照倒推订单对账
```

012 只展示账户 / 持仓原始快照。

订单锁收尾属于 007C，不属于 012。

---

## 12. 与 013 的关系

013 Pipeline Orchestrator 不负责解锁。

013 可以记录：

```text
本轮是否存在 OrderPlanActiveLock
锁开始状态
锁结束状态
锁是否仍 active
锁是否 failed
锁收尾 reason_code
是否需要人工处理
```

013 不得：

```text
直接修改 OrderPlanActiveLock
本轮结束时兜底释放锁
根据 pipeline 成功结束自动释放锁
根据 012 账户总览结果释放锁
```

如果本轮结束时锁仍 active，013 应记录：

```text
cycle finished with active order lock
needs_manual_attention = true
```

但不修改锁。

---

## 13. 幂等与并发

007C 必须幂等。

重复调用同一个收尾动作：

```text
不得重复写入冲突状态；
不得重复覆盖更高风险状态；
不得重复写等价 AlertEvent。
```

并发规则：

```text
007C 修改锁时必须使用数据库行锁。
同一 lock 同一时间只能有一个收尾动作生效。
```

状态推进原则：

```text
active -> released
active -> failed
released -> released 允许幂等返回
failed -> failed 允许幂等返回
failed 不得被普通自动流程改回 released
released 不得被普通自动流程改回 active
```

只有 007A 在创建新 OrderPlan 时，可以在明确规则下重新接管已 released / failed 的锁行。

---

## 14. Alert / 审计

007C 每次真实改变锁状态，必须写 AlertEvent。

建议事件类型：

```text
order_plan_active_lock_released
order_plan_active_lock_failed
order_plan_active_lock_kept_active
order_plan_active_lock_manual_finalized
```

严重级别建议：

```text
released：
  info

failed：
  error

kept_active because unknown：
  warning

manual_finalized：
  warning
```

审计内容至少包含：

```text
lock_id
lock_key
order_plan_id
order_plan_key
market_type
account_domain
symbol
old_status
new_status
reason_code
finalized_by_stage
finalized_object_type
finalized_object_id
trace_id
trigger_source
```

---

## 15. 007C 不做的事情

007C 不允许做：

```text
生成订单计划
审批风险
准备订单
提交 Binance 订单
查询 Binance 订单
查询成交明细
撤单
补单
重下单
更新持仓
账户同步
订单对账
Pipeline 编排
WebSocket 实时跟踪
```

007C 只做锁生命周期收尾。

---

## 16. 测试要求

### 16.1 007B DENY 释放锁

期望：

```text
RiskCheck DENY 后调用 007C。
OrderPlanActiveLock 从 active 变为 released。
写 AlertEvent。
重复调用幂等。
```

### 16.2 007B BLOCKED 释放锁

期望：

```text
RiskCheck BLOCKED 后调用 007C。
锁 released。
不释放其他 OrderPlan 的锁。
```

### 16.3 007B FAILED 标记 failed

期望：

```text
RiskCheck FAILED 后调用 007C。
锁 failed。
不得被普通自动流程改成 released。
```

### 16.4 008 阻断释放锁

期望：

```text
008 BLOCKED 且没有可提交 PreparedOrderIntent。
调用 007C 后锁 released。
```

### 16.5 009 提交前失败释放锁

期望：

```text
009 failed_before_submit。
确认 Binance POST /order 未发出。
调用 007C 后锁 released。
```

### 16.6 009 rejected 释放锁

期望：

```text
009 rejected。
Binance 明确拒绝订单。
调用 007C 后锁 released。
```

### 16.7 009 accepted 不释放锁

期望：

```text
009 accepted。
锁仍 active。
等待 010 / 011。
```

### 16.8 009 unknown 不释放锁

期望：

```text
009 unknown。
锁仍 active 或保持风险保护状态。
写 warning。
```

### 16.9 010 unknown / not_found 不自动释放锁

期望：

```text
010 unknown 不释放锁。
010 not_found P0 不自动释放锁。
```

### 16.10 011 synced 释放锁

期望：

```text
011 成交明细同步成功。
成交汇总生成成功。
调用 007C 后锁 released。
```

### 16.11 011 synced_empty 不自动释放锁

期望：

```text
011 查询成功但 0 成交。
P0 不释放锁。
写 warning 或保留人工关注标记。
```

### 16.12 012 不影响锁

期望：

```text
012 一键同步账户总览成功或失败，都不修改 OrderPlanActiveLock。
```

### 16.13 013 不修改锁

期望：

```text
013 编排结束时不直接修改锁。
如果锁仍 active，只记录本轮结果和人工关注信息。
```

### 16.14 并发收尾

期望：

```text
两个进程同时尝试收尾同一个 lock。
只有一个真实状态变更生效。
另一个返回幂等结果或冲突保护结果。
```

---

## 17. 最终结论

007C 的职责是：

```text
把 007A 创建、007B 已部分推进的 OrderPlanActiveLock，
在 008 / 009 / 010 / 011 后续订单事实出现后，
用统一服务进行安全收尾。
```

一句话：

```text
锁属于 007；
解锁也属于 007C；
008 / 009 / 010 / 011 只调用 007C；
012 / 013 不解锁。
```
