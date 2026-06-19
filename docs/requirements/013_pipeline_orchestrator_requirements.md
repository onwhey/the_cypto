# 013 Pipeline Orchestrator / TradingCycle Requirements

> 建议路径：`docs/requirements/pipeline_orchestrator.md`  
> 阶段：013  
> 模块建议：`apps.pipeline_orchestrator`  
> 状态：需求定稿草案  
> 上游依赖：001-012 已有或待实现阶段  
> 核心结论：013 负责按 UTC 4 小时周期编排一轮自动交易流程，创建 TradingCycle，写开始/结束审计，串联关键记录，防止静默失败；013 只编排，不重写业务模块，不解锁订单，不主动跑 012。

---

## 1. 阶段目标

013 的目标是把前面已经完成或即将完成的业务模块串成一轮完整自动交易流程。

013 不新增交易策略。

013 不新增下单逻辑。

013 不新增账户同步逻辑。

013 只负责：

```text
什么时候开始一轮交易周期；
按什么顺序调用已有业务模块；
每一步成功、跳过、阻断、失败、不确定时如何处理；
每轮开始和结束如何审计；
每轮产生了哪些关键记录；
如何防止中途崩溃后静默失败；
如何避免同一个 UTC 周期重复下单。
```

一句话：

```text
013 是运行编排层，不是业务计算层。
```

---

## 2. 013 与前面阶段的关系

013 编排已有阶段。

大致链路：

```text
002 数据采集 / 数据质量 / 回补
003 市场快照
004 特征层
005 原子信号 / 策略信号 / 信号质量
006 交易前账户 / 持仓同步、价格快照
007A OrderPlan
007B RiskCheck
007C OrderPlan ActiveLock Finalization
008 ExecutionPreparation
009 OrderSubmission
010 OrderStatusSync
011 FillSync
012 Account Overview / One-Click Sync
```

其中：

```text
007C 不由 013 直接执行解锁；
012 不由 013 默认自动执行；
010 / 011 只有在本轮确实进入订单提交链路后才执行。
```

---

## 3. 013 不做的事情

013 严禁实现以下能力：

```text
1. 不重新采集 K 线。
2. 不重新实现数据质量检查。
3. 不重新生成市场快照。
4. 不重新计算特征。
5. 不重新计算原子信号。
6. 不重新实现策略逻辑。
7. 不重新生成 OrderPlan。
8. 不重新做 RiskCheck。
9. 不重新准备 Binance 请求。
10. 不直接提交 Binance 订单。
11. 不直接查询 Binance 订单状态。
12. 不直接查询 Binance 成交明细。
13. 不直接同步账户总览。
14. 不做账户估值。
15. 不做订单对账。
16. 不释放 OrderPlanActiveLock。
17. 不标记 OrderPlanActiveLock failed。
18. 不根据流程结束兜底解锁。
19. 不做 WebSocket。
20. 不做 UI。
```

013 只能调用各阶段暴露的标准 pipeline stage 入口。

---

## 4. 运行时间

013 按 UTC 4 小时周期运行。

固定运行点：

```text
00:05 UTC
04:05 UTC
08:05 UTC
12:05 UTC
16:05 UTC
20:05 UTC
```

含义：

```text
4 小时 K 线在 00 / 04 / 08 / 12 / 16 / 20 UTC 收盘；
013 延迟 5 分钟运行；
给 Binance K 线刷新、采集和数据质量检查预留缓冲。
```

00:05 UTC 是特殊周期：

```text
00:05 UTC 这一轮既要处理 4h K 线，也要处理 1d 日线。
```

其他周期：

```text
04:05 / 08:05 / 12:05 / 16:05 / 20:05 UTC 主要处理 4h 周期。
```

---

## 5. 周期类型

013 至少区分两类周期：

```text
普通 4h 周期：
  04:05 / 08:05 / 12:05 / 16:05 / 20:05 UTC。

日线边界周期：
  00:05 UTC。
```

日线边界周期必须确认：

```text
4h 最新已收盘 K 线可用；
1d 最新已收盘 K 线可用。
```

如果 00:05 UTC 周期日线数据不可用，应视为数据前置条件不满足，本轮阻断，不得继续生成策略信号和订单。

---

## 6. 防重复运行

同一个 UTC 周期，只允许自动交易流程跑一次。

含义：

```text
同一个 4 小时周期已经开始运行时，不允许再启动第二轮自动交易。
同一个 4 小时周期已经正常结束时，不允许再次作为自动交易周期重复执行。
同一个 4 小时周期已经提交过订单时，不允许因为重复触发再次提交订单。
```

如果重复触发：

```text
已有本周期正在运行：
  新触发不启动新的交易流程，记录重复触发。

已有本周期已经完成：
  新触发不重新执行，不重复下单，记录已完成。

已有本周期处于未知或失败状态：
  不自动重跑下单链路。
  只能进入人工诊断或后续专门恢复命令。
```

人工重跑不得伪装成同一个自动交易周期继续下单。

目的：

```text
避免同一个 4h 周期重复生成订单、重复风控、重复准备、重复提交 Binance。
```

---

## 7. TradingCycle 记录

013 必须创建自己的周期记录，建议命名：

```text
TradingCycle
```

TradingCycle 表示一轮自动交易周期。

每一轮 013 开始时，必须先创建 TradingCycle。

TradingCycle 至少记录：

```text
本轮所属 UTC 周期
周期类型
开始时间
结束时间
当前状态
最终结果
结束原因
最后完成阶段
最后停止阶段
是否产生交易信号
是否产生订单计划
是否提交 Binance
是否进入 010
是否进入 011
是否出现 unknown
是否需要人工关注
trace_id
```

TradingCycle 还应尽量记录关键主线对象 ID。

建议记录：

```text
market_snapshot_id
feature_set_id 或 feature_batch_id
atomic_signal_set_id 或 atomic_batch_id
strategy_signal_id
strategy_signal_quality_result_id
account_sync_run_id
price_snapshot_id
order_plan_id
risk_check_id
approved_order_intent_id
execution_preparation_id
prepared_order_intent_id
order_submission_id
order_status_snapshot_id
fill_summary_id
active_lock_id
active_lock_final_status
```

注意：

```text
特征、原子信号可能一轮产生多条记录；
TradingCycle 不强行保存所有细碎子记录 ID；
只保存能串联主流程的关键 ID 或 batch ID。
```

如果原业务模型已经可以通过外键串联，不要求 TradingCycle 重复保存所有 ID。

---

## 8. 开始审计

013 每轮开始时必须写 AlertEvent / 审计事件。

事件含义：

```text
trading_cycle_started
```

目的：

```text
防止系统中途崩溃后完全无记录；
只要本轮启动过，就必须有开始痕迹。
```

开始事件至少包含：

```text
TradingCycle ID
UTC 周期
周期类型
触发来源
trace_id
开始时间
```

如果后续系统中途崩溃，数据库中至少存在：

```text
TradingCycle = running
trading_cycle_started AlertEvent
```

后续可被 stale cycle 扫描发现。

---

## 9. 结束审计

013 每轮结束时必须写最终结果审计。

无论本轮是否下单，都必须有结束结果。

可能结果包括：

```text
completed_no_trade
completed_no_order
completed_order_submitted
completed_order_synced
blocked
unknown
failed
partial_completed
```

结束事件建议包括：

```text
trading_cycle_completed_no_trade
trading_cycle_completed_no_order
trading_cycle_completed_order_submitted
trading_cycle_completed_order_synced
trading_cycle_blocked
trading_cycle_unknown
trading_cycle_failed
trading_cycle_partial_completed
```

结束事件至少包含：

```text
TradingCycle ID
最终状态
结束原因
最后完成阶段
最后停止阶段
是否下单
订单方向 / 数量摘要
009 提交记录 ID
010 状态记录 ID
011 成交汇总 ID
active lock 最终状态
是否需要人工关注
trace_id
结束时间
```

---

## 10. 最终状态口径

### 10.1 running

本轮已经开始，但尚未结束。

如果长期停留在 running，说明可能发生了系统中断，需要 stale cycle 处理。

### 10.2 completed_no_trade

策略没有给出交易操作。

示例：

```text
市场快照、特征、原子、策略都正常完成；
策略最终结论是不交易；
本轮不进入 007 / 008 / 009。
```

### 10.3 completed_no_order

有策略信号，但订单计划阶段判断无需下单。

示例：

```text
目标仓位与当前仓位差额太小；
低于 rebalance band；
低于最小调整额；
本轮正常不下单。
```

### 10.4 completed_order_submitted

本轮已经提交 Binance，但 010 / 011 没有完整完成。

该状态只适用于需求明确允许的场景。

如果 009 / 010 / 011 出现不确定状态，优先进入 unknown。

### 10.5 completed_order_synced

本轮订单主线完成。

基本要求：

```text
009 提交记录存在；
010 状态查询完成；
011 成交同步完成；
订单成交汇总生成；
007C 已在合适时机完成锁收尾或明确保持保护。
```

### 10.6 blocked

本轮因业务前置条件不满足而停止。

示例：

```text
数据质量失败；
市场快照阻断；
账户快照失败；
风控阻断；
执行准备阻断；
交易开关关闭；
```

blocked 是正常业务结果，不是系统异常。

### 10.7 unknown

本轮出现外部状态不确定。

示例：

```text
009 提交结果 unknown；
010 查询订单状态 unknown；
011 成交同步 unknown；
订单是否被 Binance 接收或成交无法确认。
```

unknown 必须保守处理，不得自动重复下单。

### 10.8 failed

本轮出现未预期系统异常。

示例：

```text
代码异常；
数据库异常；
未捕获异常；
调用参数错误导致集成失败；
```

failed 表示编排或系统层异常，不等同于业务阻断。

### 10.9 partial_completed

主流程完成，但非主链路附属动作失败。

P0 中 012 不由 013 自动触发，因此 partial_completed 暂不因为 012 产生。

后续如果新增非阻断性辅助阶段，可以使用 partial_completed。

---

## 11. 业务阻断不是编排失败

013 必须区分：

```text
业务阻断
系统异常
```

业务阻断包括：

```text
数据质量不通过；
策略不交易；
风控拒绝；
执行准备阻断；
订单计划无需调仓。
```

业务阻断应让 TradingCycle 正常收尾为：

```text
blocked
completed_no_trade
completed_no_order
```

不得因为某个业务阶段返回 blocked / skipped / no_trade，就让 013 编排进程异常退出。

真正的系统异常才记录为：

```text
failed
```

---

## 12. 各阶段标准 pipeline 入口

013 不直接调用 management command。

013 不直接调用 shell 命令。

013 不依赖 command 输出。

每个会被 013 调用的模块，必须暴露标准 pipeline stage 入口。

建议每个模块提供：

```text
run_stage(context)
```

该入口统一返回：

```text
PipelineStageResult
```

字段建议：

```text
stage_code
status
reason_code
message
should_continue
primary_object_id
related_object_ids
raw_result_summary
needs_manual_attention
```

但 013 需求文件不强制具体字段名，plan 中再定。

要求：

```text
业务模块自己负责把本模块原始 service 返回值转换成 pipeline 标准结果；
013 不写一堆针对每个模块的特殊翻译逻辑；
013 只消费标准 pipeline stage 结果。
```

这意味着：

```text
旧模块 001-009：
  可以保留原 service 返回值；
  但需要新增或补充 pipeline stage 标准入口。

新模块 010 以后：
  从一开始就提供 pipeline stage 标准入口。
```

---

## 13. 阶段继续 / 停止业务规则

013 的业务规则只分三类：

```text
继续下一步
正常结束本轮
停止本轮并记录原因
```

### 13.1 数据质量失败

停止本轮，记录数据质量阻断。

不得继续生成市场快照、策略信号或订单。

### 13.2 市场快照失败或阻断

停止本轮，记录市场快照失败或阻断。

### 13.3 特征层或原子层失败

停止本轮，记录信号前置计算失败。

### 13.4 策略没有操作

正常结束本轮。

记录：

```text
策略没有给出操作，本轮不下单。
```

不得进入订单计划、执行准备、订单提交。

### 13.5 策略质量不通过

停止本轮，记录策略信号质量阻断。

不得进入订单计划。

### 13.6 006 账户 / 持仓同步失败

停止本轮，记录交易前账户快照失败。

不得生成或推进订单。

### 13.7 007A 订单计划判断无需下单

正常结束本轮。

记录：

```text
本轮无需调仓 / 调仓金额太小 / 未达到最小调整条件。
```

不得进入 008 / 009。

### 13.8 007B 风控拒绝或阻断

停止本轮，记录风控拒绝或阻断。

锁生命周期由 007C 处理，不由 013 直接处理。

### 13.9 008 执行准备阻断

停止本轮。

不得进入 009。

锁生命周期由 007C 处理，不由 013 直接处理。

### 13.10 009 提交前失败或提交前阻断

停止本轮。

如果 Binance 没收到订单，锁生命周期由 007C 处理。

013 不解锁。

### 13.11 009 明确被交易所拒绝

停止订单链路。

锁生命周期由 007C 处理。

013 记录本轮没有成功进入交易所订单。

### 13.12 009 accepted

继续进入 010。

### 13.13 009 unknown

继续进入 010，尝试确认 Binance 是否有这笔订单。

如果 010 仍无法确认，TradingCycle 进入 unknown。

不得重复下单。

不得自动解锁。

### 13.14 010 查询成功

继续进入 011。

### 13.15 010 unknown

停止订单后续同步，记录 unknown。

等待人工排查或后续单独订单诊断模块。

不得自动解锁。

### 13.16 010 not_found

P0 默认记录 unknown 或需人工关注。

不直接进入自动解锁。

不得把 not_found 等价成“订单一定没有成交”。

### 13.17 011 成交同步完成

订单主线结束。

007C 可在 011 明确完成后释放锁。

013 记录成交汇总 ID 和锁最终状态。

### 13.18 011 synced_empty

记录需要人工关注。

P0 不把 synced_empty 当作正常完成。

不得自动解锁。

### 13.19 011 unknown

记录 unknown。

不得自动解锁。

---

## 14. 012 调用规则

013 P0 不主动调用 012。

原因：

```text
订单之前已经有 006 账户 / 持仓同步；
012 是账户总览一键同步 / 手动刷新模块；
012 不属于订单主线。
```

013 不做：

```text
有订单后自动跑 012；
无订单后自动跑 012；
根据 012 结果改变本轮订单状态；
根据 012 结果释放订单锁。
```

012 的触发来源：

```text
用户手动刷新；
UI 一键同步；
运维命令；
后续独立调度。
```

---

## 15. 007C 调用边界

013 不直接调用 007C 做解锁。

订单锁收尾由订单链路阶段在拿到明确事实后触发：

```text
007B：
  风控 DENY / BLOCKED / FAILED 后调用 007C。

008：
  执行准备明确阻断且没有提交 Binance 时调用 007C。

009：
  提交前失败、提交前阻断、交易所明确拒绝时调用 007C。

011：
  成交同步完成后调用 007C。
```

010 默认不触发解锁。

013 只记录：

```text
本轮是否有 active lock；
锁开始状态；
锁结束状态；
锁是否仍 active；
锁是否 failed；
锁最终原因；
是否需要人工关注。
```

013 不得：

```text
兜底解锁；
因本轮结束自动解锁；
因 010 not_found 自动解锁；
因 011 synced_empty 自动解锁；
因 012 账户变化自动解锁。
```

---

## 16. Stale running cycle 处理

013 必须能发现长期停留在 running 的旧周期。

场景：

```text
TradingCycle 已创建；
cycle_started Alert 已写；
系统中途崩溃；
没有 cycle_finished / cycle_failed / cycle_blocked 记录。
```

下一次 013 启动前，应扫描 stale running cycle。

处理方式：

```text
把超时 running 周期标记为 failed 或 stale_interrupted；
写 AlertEvent；
记录最后已知阶段；
不得自动重跑下单链路；
不得自动解锁订单锁。
```

目的：

```text
防止静默失败。
```

---

## 17. 异常处理

013 必须有全局保护。

基本要求：

```text
每轮开始后必须尽量保证有最终状态；
业务 blocked / skipped / unknown 不得导致编排异常退出；
未预期异常必须被记录为 failed；
结束时必须尽量写 finished_at；
AlertEvent 写入失败不得造成二次崩溃。
```

013 调用 stage 时：

```text
stage 返回业务阻断：
  TradingCycle 正常收尾为 blocked。

stage 返回正常跳过：
  TradingCycle 正常收尾为 completed_no_trade 或 completed_no_order。

stage 返回 unknown：
  TradingCycle 收尾为 unknown 或按后续规则进入确认阶段。

stage 抛出未预期异常：
  TradingCycle 收尾为 failed。
```

---

## 18. Alert / 审计要求

013 至少写以下审计事件：

```text
trading_cycle_started
trading_cycle_completed_no_trade
trading_cycle_completed_no_order
trading_cycle_completed_order_submitted
trading_cycle_completed_order_synced
trading_cycle_blocked
trading_cycle_unknown
trading_cycle_failed
trading_cycle_stale_interrupted
duplicate_trading_cycle_triggered
```

订单相关成功与失败都应写审计。

本轮如果发生真实下单或调仓，应在结束事件中说明：

```text
方向
symbol
market_type
account_domain
目标数量
提交数量
009 submission id
成交汇总 id
```

---

## 19. 关键 ID 串联原则

013 应尽量串联关键主线 ID，但不强行记录所有细碎子记录。

原则：

```text
能让 UI / 审计 / 排错从 TradingCycle 找到本轮关键路径即可。
```

必须重点记录：

```text
策略信号 ID
订单计划 ID
风控结果 ID
执行准备 ID
订单提交 ID
订单状态记录 ID
成交汇总 ID
active lock ID
```

对于一轮会产生多条记录的层：

```text
feature
atomic signal
issues
alerts
```

优先记录 batch ID / set ID / summary ID。

不要求 TradingCycle 主表塞入所有子 ID。

---

## 20. 手动触发

013 可以支持人工触发一轮检查，但必须区分：

```text
自动交易周期
人工诊断运行
```

人工诊断运行不得伪装成自动交易周期重复下单。

P0 自动交易周期防重复优先。

如果未来需要人工重跑，必须单独设计：

```text
只读诊断
不下单重算
人工确认后执行
```

不在 013 P0 默认实现。

---

## 21. 与 010 / 011 的关系

010 / 011 是订单提交后的事实同步阶段。

013 只在本轮存在 009 提交记录时进入 010。

013 只在 010 成功同步订单状态后进入 011。

如果本轮没有 009 提交记录：

```text
不调用 010。
不调用 011。
```

如果 009 unknown：

```text
仍应进入 010 尝试确认订单状态。
```

如果 010 unknown / not_found：

```text
P0 不进入 011。
记录 unknown。
等待人工或后续订单诊断。
```

---

## 22. 测试要求

### 22.1 正常无交易周期

期望：

```text
创建 TradingCycle。
写 started Alert。
策略无操作。
TradingCycle = completed_no_trade。
写 completed_no_trade Alert。
不进入 007 / 008 / 009。
```

### 22.2 订单计划无需调仓

期望：

```text
策略有信号。
007A 判断无需调仓。
TradingCycle = completed_no_order。
不进入 008 / 009。
```

### 22.3 数据质量阻断

期望：

```text
数据质量失败。
TradingCycle = blocked。
写 blocked Alert。
后续阶段不执行。
```

### 22.4 006 账户快照失败

期望：

```text
006 失败。
TradingCycle = blocked。
不得进入订单计划或订单提交。
```

### 22.5 008 阻断

期望：

```text
008 阻断。
TradingCycle = blocked。
不进入 009。
013 不直接解锁。
```

### 22.6 009 accepted 后进入 010

期望：

```text
009 accepted。
013 继续进入 010。
```

### 22.7 009 unknown 后进入 010

期望：

```text
009 unknown。
013 继续进入 010 尝试确认。
不得重复下单。
不得自动解锁。
```

### 22.8 010 unknown

期望：

```text
010 unknown。
TradingCycle = unknown。
不进入 011。
不自动解锁。
```

### 22.9 011 synced

期望：

```text
011 成交同步完成。
TradingCycle = completed_order_synced。
记录 fill_summary_id。
记录 active lock 最终状态。
```

### 22.10 011 synced_empty

期望：

```text
011 返回 0 成交。
TradingCycle = unknown 或 needs_manual_attention。
不自动解锁。
```

### 22.11 012 不自动执行

期望：

```text
TradingCycle 正常运行时，不自动调用 012。
无论有没有订单，都不因 013 自动刷新账户总览。
```

### 22.12 防重复

期望：

```text
同一个 UTC 周期已有 running 记录时，重复触发不启动第二轮。
同一个 UTC 周期已有完成记录时，重复触发不重复下单。
```

### 22.13 stale running cycle

期望：

```text
旧 TradingCycle 长时间 running。
下一轮启动前标记 stale_interrupted / failed。
写 Alert。
不重跑下单链路。
不解锁。
```

### 22.14 编排异常

期望：

```text
某 stage 抛未预期异常。
TradingCycle = failed。
写 failed Alert。
记录 exception 信息。
```

### 22.15 业务阻断不是 failed

期望：

```text
数据质量 blocked / 风控 blocked / 执行准备 blocked。
TradingCycle = blocked。
不是 failed。
```

---

## 23. 最终结论

013 的职责是：

```text
按 UTC 4 小时周期，在 00:05 / 04:05 / 08:05 / 12:05 / 16:05 / 20:05 运行；
每轮创建 TradingCycle；
开始时写 started Alert；
按标准 pipeline stage 入口调用业务模块；
区分继续、正常结束、阻断、unknown、failed；
记录关键主线 ID；
结束时写最终结果 Alert；
发现 stale running cycle，防止静默失败；
防止同一个周期重复下单。
```

013 不负责：

```text
业务计算；
直接下单；
直接查单；
直接查成交；
账户总览自动刷新；
订单锁解锁；
订单对账；
WebSocket；
UI。
```

一句话：

```text
013 是自动交易周期的总调度和审计收口；
不是订单锁收尾、不是账户总览、不是业务模块重写。
```
