# 014 Runtime Guard / 运行保护巡检 Requirements

> 建议路径：`docs/requirements/runtime_guard.md`  
> 阶段：014  
> 模块建议：`apps.runtime_guard`  
> 状态：需求定稿草案  
> 上游依赖：001 Alert / 007C ActiveLock / 009 OrderSubmission / 010 OrderStatusSync / 011 FillSync / 013 TradingCycle  
> 核心结论：014 是运行保护巡检模块，只发现异常、记录异常、写巡检异常 Alert、标记人工处理；不自动修复交易，不下单，不撤单，不解锁，不重跑交易周期。

---

## 1. 阶段目标

014 的目标是：

```text
定期巡检系统运行状态，
发现交易周期漏跑、卡住、订单状态不确定、成交同步异常、订单锁长期 active、账户快照过期、Alert 投递异常等问题，
并写入巡检问题记录和 AlertEvent。
```

014 不是交易主流程。

014 不参与订单链路。

014 不做自动修复。

一句话：

```text
013 负责跑交易流程；
014 负责盯着系统有没有卡住、漏跑、静默失败或长期未处理异常。
```

---

## 2. 014 与 013 的关系

013 是自动交易周期编排：

```text
按 00:05 / 04:05 / 08:05 / 12:05 / 16:05 / 20:05 UTC 运行交易周期；
创建 TradingCycle；
写开始和结束审计；
串联关键 ID；
执行交易主链路。
```

014 是巡检保护：

```text
检查 013 是否按时运行；
检查 013 是否卡在 running；
检查订单链路是否存在长期 unknown；
检查订单锁是否长期 active；
检查 Alert 系统是否异常。
```

014 不替代 013。

014 不补跑 013。

014 不重跑交易周期。

014 不把异常周期改成成功。

---

## 3. 014 不做的事情

014 严禁做：

```text
1. 不下单。
2. 不补单。
3. 不撤单。
4. 不重下单。
5. 不重跑 TradingCycle。
6. 不自动补跑漏掉的交易周期。
7. 不释放 OrderPlanActiveLock。
8. 不标记 OrderPlanActiveLock failed。
9. 不修改 009 订单提交结果。
10. 不修改 010 订单状态同步结果。
11. 不修改 011 成交同步结果。
12. 不更新持仓。
13. 不根据账户快照倒推订单结果。
14. 不做订单对账。
15. 不主动跑 012 Account Overview。
16. 不修改交易允许状态。
17. 不把 unknown 自动改成 success。
18. 不把 not_found 自动解释为订单安全失败。
```

014 只能：

```text
记录巡检问题；
写 Runtime Guard 巡检异常 Alert；
标记需要人工处理；
维护自己的 RuntimeGuardIssue 状态。
```

---

## 4. 运行频率

014 默认每 10 分钟运行一次。

建议调度：

```text
每 10 分钟巡检一次。
```

014 不需要卡 4 小时 K 线周期。

014 不是交易执行任务，因此不需要等待 K 线刷新。

---

## 5. 巡检范围

014 P0 巡检以下问题：

```text
1. TradingCycle 漏跑。
2. TradingCycle 长时间 running。
3. OrderPlanActiveLock 长时间 active。
4. 009 OrderSubmission unknown 长时间未处理。
5. 010 OrderStatusSync unknown / not_found 长时间未处理。
6. 011 FillSync unknown / synced_empty 长时间未处理。
7. 006 账户 / 持仓快照过期。
8. AlertEvent 投递异常。
```

014 可以后续扩展其他巡检项，但 P0 先保持以上范围。

---

## 6. 阈值规则

### 6.1 TradingCycle 漏跑阈值

013 正常应在以下时间运行：

```text
00:05 UTC
04:05 UTC
08:05 UTC
12:05 UTC
16:05 UTC
20:05 UTC
```

如果某个计划周期过了 15 分钟仍然没有对应 TradingCycle，014 记录漏跑。

示例：

```text
04:05 UTC 应该运行。
04:20 UTC 仍没有对应 TradingCycle。
014 记录 cycle_missing，并写 Runtime Guard 巡检异常 Alert。
```

### 6.2 TradingCycle running 卡住阈值

如果 TradingCycle 保持 running 超过 30 分钟，014 记录卡住。

```text
TradingCycle running 超过 30 分钟：
  记录 cycle_stale_running。
  写 Runtime Guard 巡检异常 Alert。
  标记需要人工处理。
```

P0 不直接修改 TradingCycle 最终状态。

如果未来允许自动标记 stale，必须通过 013 自己的 stale finalization 服务，不允许 014 直接乱改。

### 6.3 OrderPlanActiveLock 长期 active 阈值

如果 OrderPlanActiveLock active 超过 30 分钟，014 记录订单锁长期未释放。

```text
OrderPlanActiveLock active 超过 30 分钟：
  记录 active_lock_stale。
  写 Runtime Guard 巡检异常 Alert。
  标记需要人工处理。
```

014 不解锁。

### 6.4 009 unknown 未处理阈值

如果 009 OrderSubmission unknown 超过 30 分钟仍未解决，014 记录问题。

```text
009 unknown 超过 30 分钟：
  记录 order_submission_unknown_unresolved。
  写 Runtime Guard 巡检异常 Alert。
  标记需要人工处理。
```

014 不重新下单，不撤单，不解锁。

### 6.5 010 unknown / not_found 未处理阈值

如果 010 OrderStatusSync unknown 或 not_found 超过 30 分钟仍未解决，014 记录问题。

```text
010 unknown 超过 30 分钟：
  记录 order_status_unknown_unresolved。

010 not_found 超过 30 分钟且关联 009 accepted / unknown：
  记录 order_status_not_found_unresolved。
```

014 不把 not_found 自动解释成订单不存在或未成交。

### 6.6 011 unknown / synced_empty 未处理阈值

如果 011 FillSync unknown 或 synced_empty 超过 30 分钟仍未解决，014 记录问题。

```text
011 unknown 超过 30 分钟：
  记录 fill_sync_unknown_unresolved。

011 synced_empty 超过 30 分钟：
  记录 fill_sync_empty_unresolved。
```

对于当前市价单链路，synced_empty 需要关注。

014 不更新持仓，不释放锁，不自动对账。

### 6.7 账户快照过期阈值

如果最近一次成功的 006 交易前账户 / 持仓快照超过 4 小时没有更新，014 记录账户快照过期。

```text
成功账户快照超过 4 小时未更新：
  记录 account_snapshot_stale。
  写 Runtime Guard 巡检异常 Alert。
```

014 不主动跑 012。

014 不主动跑 006。

014 只提示人工检查 006 或手动刷新 012。

### 6.8 Alert 投递异常阈值

014 应巡检 Alert 系统自身异常。

至少包括：

```text
AlertEvent 长时间 processing；
AlertEvent failed 数量异常；
AlertEvent 连续投递失败次数过多；
AlertEvent stale processing 长期未恢复。
```

具体阈值可以复用 001 Hermes / Alert 配置，或在 014 plan 中细化。

---

## 7. RuntimeGuardIssue

014 应创建自己的巡检问题记录，建议命名：

```text
RuntimeGuardIssue
```

每个问题至少记录：

```text
issue_type
severity
status
first_seen_at_utc
last_seen_at_utc
resolved_at_utc
related_object_type
related_object_id
related_trace_id
description
evidence
needs_manual_attention
alert_event_id
acknowledged_at_utc
acknowledged_by
resolution_note
```

014 可以另有运行记录，例如：

```text
RuntimeGuardRun
```

记录每次巡检执行：

```text
started_at_utc
finished_at_utc
status
checked_item_count
created_issue_count
updated_issue_count
error_count
trace_id
```

---

## 8. RuntimeGuardIssue 状态

RuntimeGuardIssue 至少支持：

```text
open
acknowledged
resolved
ignored
```

含义：

```text
open：
  新发现或仍未处理。

acknowledged：
  人工已看到，正在处理或等待处理。

resolved：
  问题已解决。

ignored：
  人工确认该问题无需处理。
```

014 自动巡检可以创建或更新 issue。

人工或运维命令可以把 issue 标记为 acknowledged / resolved / ignored。

---

## 9. Issue 去重规则

014 不得每 10 分钟重复刷屏创建同一个问题。

同一个问题未解决前：

```text
不得重复创建新的 RuntimeGuardIssue；
应更新已有 issue 的 last_seen_at_utc、evidence、出现次数；
可按配置间隔重复提醒，但不能无限刷屏。
```

示例：

```text
同一个 OrderPlanActiveLock 长期 active：
  第一次发现：
    创建 RuntimeGuardIssue。
    写 AlertEvent。

  后续仍然 active：
    更新 last_seen_at_utc。
    不创建重复 issue。
    不持续刷屏。

  超过更长时间仍未处理：
    可以升级严重级别或重复提醒一次。
```

---

## 10. Alert 强制口径

014 发出的所有 Alert 必须明确告知：

```text
这是 Runtime Guard 巡检发现的异常。
```

014 Alert 不得伪装成原业务模块实时异常。

必须区分：

```text
009 自己产生的 order_submission_unknown；
014 后来巡检发现这个 unknown 超过 30 分钟未处理。
```

这两个事件不是一回事。

014 的 Alert 标题必须包含：

```text
[RuntimeGuard] 巡检异常：
```

示例：

```text
[RuntimeGuard] 巡检异常：TradingCycle 漏跑
[RuntimeGuard] 巡检异常：TradingCycle 长时间 running
[RuntimeGuard] 巡检异常：订单锁长期 active
[RuntimeGuard] 巡检异常：009 unknown 未处理
[RuntimeGuard] 巡检异常：010 not_found 未处理
[RuntimeGuard] 巡检异常：011 synced_empty 未处理
[RuntimeGuard] 巡检异常：账户快照过期
[RuntimeGuard] 巡检异常：Alert 投递异常
```

Alert 内容必须包含：

```text
alert_source = runtime_guard
detected_by = 014_runtime_guard
issue_type
severity
detected_at_utc
related_object_type
related_object_id
related_trace_id
needs_manual_attention
```

Alert 内容还必须说明：

```text
这是 014 Runtime Guard 巡检任务发现的异常；
014 不会自动修复、不下单、不撤单、不解锁；
请人工检查关联对象。
```

---

## 11. 严重级别

建议支持：

```text
info
warning
error
critical
```

P0 建议：

```text
cycle_missing：
  error

cycle_stale_running：
  error

active_lock_stale：
  error

order_submission_unknown_unresolved：
  critical

order_status_unknown_unresolved：
  error

order_status_not_found_unresolved：
  error

fill_sync_unknown_unresolved：
  error

fill_sync_empty_unresolved：
  error

account_snapshot_stale：
  warning

alert_dispatch_stale：
  error

alert_dispatch_failed_excessive：
  error
```

严重级别可在 plan 中进一步细化。

---

## 12. 各类问题处理动作

### 12.1 TradingCycle 漏跑

发现：

```text
某个 UTC 周期应运行 013，但超过 15 分钟仍没有 TradingCycle。
```

处理：

```text
创建 / 更新 RuntimeGuardIssue。
写 [RuntimeGuard] 巡检异常 Alert。
标记 needs_manual_attention。
```

禁止：

```text
不自动补跑。
不自动下单。
```

### 12.2 TradingCycle 长时间 running

发现：

```text
TradingCycle running 超过 30 分钟。
```

处理：

```text
创建 / 更新 RuntimeGuardIssue。
写 [RuntimeGuard] 巡检异常 Alert。
标记 needs_manual_attention。
```

P0 不直接改 TradingCycle 状态。

### 12.3 OrderPlanActiveLock 长期 active

发现：

```text
OrderPlanActiveLock active 超过 30 分钟。
```

处理：

```text
创建 / 更新 RuntimeGuardIssue。
写 [RuntimeGuard] 巡检异常 Alert。
标记 needs_manual_attention。
```

禁止：

```text
不解锁。
不标记 failed。
不允许下一轮绕过锁。
```

### 12.4 009 unknown 未处理

发现：

```text
009 unknown 超过 30 分钟。
```

处理：

```text
创建 / 更新 RuntimeGuardIssue。
写 critical 级别 [RuntimeGuard] 巡检异常 Alert。
标记 needs_manual_attention。
```

禁止：

```text
不重新下单。
不撤单。
不解锁。
```

### 12.5 010 unknown / not_found 未处理

发现：

```text
010 unknown / not_found 超过 30 分钟。
```

处理：

```text
创建 / 更新 RuntimeGuardIssue。
写 [RuntimeGuard] 巡检异常 Alert。
标记 needs_manual_attention。
```

禁止：

```text
不把 not_found 当成安全完成。
不根据 not_found 解锁。
```

### 12.6 011 unknown / synced_empty 未处理

发现：

```text
011 unknown / synced_empty 超过 30 分钟。
```

处理：

```text
创建 / 更新 RuntimeGuardIssue。
写 [RuntimeGuard] 巡检异常 Alert。
标记 needs_manual_attention。
```

禁止：

```text
不更新持仓。
不释放订单锁。
不做对账。
```

### 12.7 账户快照过期

发现：

```text
006 成功账户快照超过 4 小时未更新。
```

处理：

```text
创建 / 更新 RuntimeGuardIssue。
写 [RuntimeGuard] 巡检异常 Alert。
提示人工检查 006 或手动刷新 012。
```

禁止：

```text
不主动触发 012。
不主动触发交易。
```

### 12.8 Alert 投递异常

发现：

```text
AlertEvent 长时间 processing 或失败异常。
```

处理：

```text
创建 / 更新 RuntimeGuardIssue。
写 [RuntimeGuard] 巡检异常 Alert。
```

如果 Alert 系统本身不可用，至少写日志和 RuntimeGuardIssue。

---

## 13. 与 012 的关系

012 Account Overview 是手动账户总览刷新模块。

014 可以发现：

```text
账户快照过期。
```

但 014 不主动执行：

```text
012 sync all
012 one-click sync
```

014 只提示：

```text
请人工刷新 012 或检查 006 交易前账户同步。
```

---

## 14. 与 007C 的关系

007C 负责订单锁收尾。

014 可以发现：

```text
OrderPlanActiveLock 长期 active。
```

但 014 不调用 007C 解锁。

014 只记录：

```text
active_lock_stale
```

并写 Runtime Guard 巡检异常 Alert。

如果需要人工释放或标记失败，必须通过 007C 的人工管理入口完成，不属于 014 自动行为。

---

## 15. 与 009 / 010 / 011 的关系

014 可以发现：

```text
009 unknown 未处理；
010 unknown / not_found 未处理；
011 unknown / synced_empty 未处理。
```

但 014 不修改这些阶段的业务结果。

014 不创建 010 查询。

014 不创建 011 成交同步。

014 不重新提交 009。

014 只记录巡检问题并提醒人工。

---

## 16. 与 001 Alert 的关系

014 依赖 001 AlertEvent 写告警。

但 014 也要巡检 Alert 系统本身。

如果 014 发现 AlertEvent 投递异常，应创建 RuntimeGuardIssue。

如果 Alert 投递系统不可用，014 至少要：

```text
记录 RuntimeGuardIssue；
写系统日志；
保留后续人工排查证据。
```

---

## 17. 幂等与并发

014 必须幂等。

重复运行同一次巡检不得重复创建同一 issue。

并发运行时：

```text
同一个 issue_key 同一时间只能创建一个 open issue。
```

建议每类问题定义稳定 issue_key，例如：

```text
issue_type + related_object_type + related_object_id
```

对于周期漏跑：

```text
issue_type + scheduled_for_utc + market_type + symbol
```

实际字段在 plan 中细化。

---

## 18. 测试要求

### 18.1 巡检频率配置

期望：

```text
014 可按 10 分钟调度执行。
```

### 18.2 TradingCycle 漏跑

期望：

```text
计划周期超过 15 分钟仍无 TradingCycle。
创建 cycle_missing issue。
写 [RuntimeGuard] 巡检异常 Alert。
不补跑。
```

### 18.3 TradingCycle stale running

期望：

```text
TradingCycle running 超过 30 分钟。
创建 cycle_stale_running issue。
写 [RuntimeGuard] 巡检异常 Alert。
不直接修改 TradingCycle 最终状态。
```

### 18.4 ActiveLock stale

期望：

```text
OrderPlanActiveLock active 超过 30 分钟。
创建 active_lock_stale issue。
写 [RuntimeGuard] 巡检异常 Alert。
不解锁。
```

### 18.5 009 unknown unresolved

期望：

```text
009 unknown 超过 30 分钟。
创建 order_submission_unknown_unresolved issue。
写 critical [RuntimeGuard] 巡检异常 Alert。
不重新下单。
```

### 18.6 010 unknown / not_found unresolved

期望：

```text
010 unknown / not_found 超过 30 分钟。
创建 RuntimeGuardIssue。
不把 not_found 当成安全完成。
不解锁。
```

### 18.7 011 unknown / synced_empty unresolved

期望：

```text
011 unknown / synced_empty 超过 30 分钟。
创建 RuntimeGuardIssue。
不更新持仓。
不释放锁。
```

### 18.8 账户快照过期

期望：

```text
006 成功账户快照超过 4 小时未更新。
创建 account_snapshot_stale issue。
写 [RuntimeGuard] 巡检异常 Alert。
不主动跑 012。
```

### 18.9 Alert 投递异常

期望：

```text
AlertEvent 长时间 processing 或 failed 异常。
创建 alert_dispatch issue。
```

### 18.10 告警文案必须明确巡检异常

期望：

```text
014 发出的 Alert 标题必须包含 [RuntimeGuard] 巡检异常。
Alert 内容必须说明这是 014 巡检发现，不是原业务模块实时异常。
```

### 18.11 Issue 去重

期望：

```text
同一问题重复巡检时，不重复创建 open issue。
更新 last_seen_at_utc。
不刷屏。
```

### 18.12 人工状态流转

期望：

```text
RuntimeGuardIssue 可从 open 标记为 acknowledged / resolved / ignored。
```

### 18.13 不修改业务状态

期望：

```text
014 不修改 TradingCycle 最终状态。
014 不修改 OrderPlanActiveLock。
014 不修改 009 / 010 / 011 结果。
014 不提交订单。
014 不撤单。
```

---

## 19. 最终结论

014 的职责是：

```text
每 10 分钟巡检系统运行状态；
发现 TradingCycle 漏跑、卡住；
发现订单锁长期 active；
发现 009 / 010 / 011 unknown 或 empty 长期未处理；
发现账户快照过期；
发现 Alert 投递异常；
创建 RuntimeGuardIssue；
写明确带有 [RuntimeGuard] 巡检异常 的 Alert；
标记需要人工处理；
避免重复告警刷屏。
```

014 不负责：

```text
自动修复；
补跑交易周期；
下单；
撤单；
补单；
解锁；
对账；
改业务结果；
主动跑 012。
```

一句话：

```text
014 是巡检报警器，不是自动修理工。
```
