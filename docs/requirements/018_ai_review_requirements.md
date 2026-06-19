# 018 AI Review / 大模型复盘模块 Requirements

> 建议路径：`docs/requirements/ai_review.md`  
> 阶段：018  
> 模块建议：`apps.ai_review`  
> 状态：需求定稿草案  
> 上游依赖：004 Feature、005 Atomic / Strategy、006 Account Snapshot、007 OrderPlan / RiskCheck、008 ExecutionPreparation、009 OrderSubmission、010 OrderStatusSync、011 FillSync、013 TradingCycle、014 RuntimeGuard、015 Performance Metrics、017 Web Console  
> 下游使用：017 Admin / Ops Console / Review Workbench  
> 核心结论：018 负责调用大模型进行交易复盘，并把复盘报告、问题发现、优化建议保存到数据库；大模型只提供分析和建议，不得自动改策略、不得自动下单、不得自动修改生产参数。

---

## 1. 阶段定位

018 的正式定位是：

```text
AI Review / 大模型复盘模块
```

018 是一个离线复盘分析模块。

018 不是交易主流程。

018 不是自动策略优化器。

018 不是自动调参器。

018 不是自动交易执行器。

018 的核心目标是：

```text
把最近一段 TradingCycle 的完整数据打包给大模型分析，
让大模型输出复盘报告、问题发现和优化建议，
并把这些结果保存到数据库，
供 017 后台查看、筛选、人工评估和后续策略版本改进。
```

---

## 2. 017 与 018 的分工

017 是页面和操作入口。

018 是大模型复盘引擎。

关系如下：

```text
017：
  选择复盘范围；
  发起复盘请求；
  查看复盘报告；
  查看大模型建议；
  标记建议状态；
  管理人工采纳流程。

018：
  组装复盘数据包；
  调用大模型；
  保存大模型返回；
  拆分报告、发现、建议；
  记录 prompt、模型、输入摘要、输出结果；
  管理复盘请求状态。
```

一句话：

```text
017 负责看和操作；
018 负责分析和落库。
```

---

## 3. 018 的输入范围

018 P0 支持按 TradingCycle 范围发起复盘。

默认支持：

```text
最近 20 个 TradingCycle；
最近 50 个 TradingCycle；
最近 100 个 TradingCycle；
自定义时间范围；
指定 TradingCycle ID 列表。
```

复盘范围必须固定下来并记录到数据库。

复盘报告生成后，不应因为后续数据变化而改变历史报告输入范围。

---

## 4. 018 读取的数据

018 可以读取以下已落库数据：

```text
013 TradingCycle；
003 MarketSnapshot；
004 FeatureResult；
005 AtomicSignalResult；
005 StrategySignal；
005 StrategySignalQualityResult；
006 Account / Position Snapshot；
007A OrderPlan；
007B RiskCheck；
008 ExecutionPreparation；
009 OrderSubmission；
010 OrderStatusSync；
011 FillSync / FillSummary；
014 RuntimeGuardIssue；
015 TradingCyclePerformance；
001 AlertEvent。
```

018 不直接请求 Binance。

018 不重新计算特征。

018 不重新计算原子信号。

018 不重新计算策略信号。

018 不重新计算 015 周期浮动收益。

018 只读取已有结果并整理成复盘数据包。

---

## 5. Review Package / 复盘数据包

018 必须生成结构化复盘数据包。

复盘数据包用于发送给大模型，也可以供 017 下载查看。

复盘数据包至少包含：

```text
复盘范围；
每个 TradingCycle 的时间和状态；
市场快照摘要；
关键特征值；
原子信号；
策略信号；
策略目标仓位；
实际持仓；
OrderPlan；
RiskCheck；
ExecutionPreparation；
OrderSubmission；
OrderStatusSync；
FillSync；
015 周期浮动收益；
blocked / unknown / failed 原因；
AlertEvent 摘要；
RuntimeGuardIssue 摘要。
```

复盘数据包必须适合大模型阅读。

---

## 6. 敏感信息过滤

018 发送给大模型前必须过滤敏感信息。

禁止发送：

```text
Binance API key；
Binance secret；
签名；
request header；
认证 token；
数据库密码；
Redis 密码；
环境变量；
服务器路径中的敏感信息；
完整原始请求中包含的认证内容。
```

如果需要保留交易所响应，必须先脱敏。

大模型输入中可以保留：

```text
order_id；
client_order_id；
symbol；
side；
quantity；
price；
status；
error_code；
error_message；
trace_id；
业务字段；
复盘所需数值。
```

---

## 7. 大模型调用

018 负责调用大模型服务。

大模型调用必须记录：

```text
model_provider；
model_name；
prompt_version；
input_package_hash；
request_status；
started_at_utc；
finished_at_utc；
error_code；
error_message；
trace_id。
```

018 P0 不要求多模型对比。

018 P0 可以先支持一个配置好的大模型 provider。

018 不在数据库中保存大模型 API secret。

大模型 API key / secret 仍应通过 `.env` 或安全配置提供，不由后台页面管理。

---

## 8. Prompt 版本管理

018 必须记录 prompt 版本。

原因是：

```text
不同 prompt 版本会导致不同复盘结论；
历史报告必须能追溯当时使用的分析模板。
```

每次复盘必须记录：

```text
prompt_name；
prompt_version；
prompt_hash；
prompt_summary。
```

如果 prompt 更新，旧报告不应被覆盖。

---

## 9. 大模型输出内容

大模型输出应包含：

```text
总体复盘结论；
收益表现总结；
策略目标仓位评价；
特征层问题；
原子信号问题；
策略信号问题；
风控问题；
执行问题；
订单 / 成交问题；
异常 / blocked / unknown 影响；
重点周期列表；
建议人工复查的周期；
参数或策略优化建议；
风险提示；
置信度或优先级。
```

018 应保存完整原始输出。

018 还应尽量拆分结构化结果，方便 017 展示和筛选。

---

## 10. 建议输出分类

AIReviewFinding 建议支持以下类型：

```text
feature_issue；
atomic_signal_issue；
strategy_issue；
risk_check_issue；
execution_issue；
order_issue；
fill_issue；
runtime_issue；
pnl_issue；
data_quality_issue；
manual_review_required；
other。
```

AIReviewSuggestion 建议支持以下类型：

```text
adjust_feature；
adjust_atomic_signal；
adjust_strategy_weight；
adjust_strategy_filter；
adjust_risk_rule；
adjust_rebalance_rule；
retire_definition；
create_new_version；
investigate_cycle；
investigate_order；
no_action；
other。
```

---

## 11. 建议状态流转

大模型建议不能直接执行。

建议必须进入人工状态流转。

AIReviewSuggestion 至少支持：

```text
pending_review；
accepted；
rejected；
converted_to_task；
implemented；
ignored。
```

含义：

```text
pending_review：
  大模型刚提出，等待人工评估。

accepted：
  人工认可这个建议，但尚未实施。

rejected：
  人工明确拒绝。

converted_to_task：
  已转为人工开发 / 策略调整任务。

implemented：
  已通过人工方式落地，例如创建了新策略版本。

ignored：
  人工确认暂不处理。
```

---

## 12. 如何利用大模型反馈优化策略

正确流程是：

```text
大模型提出建议
  ↓
018 保存建议
  ↓
017 展示建议
  ↓
人工审核
  ↓
如认可，人工创建新的特征 / 原子 / 策略版本
  ↓
人工启用新版本
  ↓
后续通过 TradingCycle 和 015 收益继续观察
```

禁止流程是：

```text
大模型提出建议
  ↓
系统自动修改生产参数
  ↓
系统自动启用策略
  ↓
系统自动交易
```

018 不允许自动修改任何生产策略。

---

## 13. 与特征 / 原子 / 策略版本管理的关系

018 可以引用：

```text
FeatureDefinition；
AtomicSignalDefinition；
StrategyDefinition；
StrategyRouteRule。
```

018 可以建议：

```text
某个定义需要调整；
某个定义可能误导；
某个策略版本应考虑 retire；
某个参数应考虑降低或提高；
某个过滤条件应考虑新增。
```

但 018 不能直接修改这些定义。

所有修改必须通过 017 的策略管理页面或后续人工开发流程，并遵守：

```text
参数变化创建新版本；
旧版本 retire；
新版本 enable；
保留审计。
```

---

## 14. 018 不做的事情

018 P0 不做：

```text
自动修改策略参数；
自动创建策略版本；
自动启用策略；
自动禁用策略；
自动下单；
自动撤单；
自动补单；
自动释放订单锁；
自动修改历史 TradingCycle；
自动修改 009 / 010 / 011 结果；
自动修改 015 收益结果；
自动触发 013 交易周期；
自动触发 012 账户同步；
自动调用 020 修改交易开关。
```

018 只输出分析、发现、建议和人工待办。

---

## 15. 数据库模型建议

018 建议包含以下模型：

```text
AIReviewRequest；
AIReviewPackage；
AIReviewReport；
AIReviewFinding；
AIReviewSuggestion。
```

实际模型名可以在 plan 中调整，但需求层面应覆盖这些业务对象。

---

## 16. AIReviewRequest

AIReviewRequest 表示一次复盘请求。

建议记录：

```text
id；
review_range_type；
cycle_count；
cycle_start_utc；
cycle_end_utc；
selected_cycle_ids；
status；
requested_by；
requested_at_utc；
started_at_utc；
finished_at_utc；
model_provider；
model_name；
prompt_name；
prompt_version；
input_package_hash；
error_code；
error_message；
trace_id。
```

状态至少包括：

```text
pending；
building_package；
calling_model；
completed；
failed；
cancelled。
```

---

## 17. AIReviewPackage

AIReviewPackage 表示发送给大模型的数据包。

建议记录：

```text
id；
review_request_id；
package_format；
package_hash；
cycle_count；
data_summary；
sanitized；
created_at_utc；
payload；
payload_storage_path；
trace_id。
```

如果 payload 很大，可以后续考虑文件存储。

P0 可以先存数据库，但必须控制大小。

---

## 18. AIReviewReport

AIReviewReport 表示大模型完整复盘报告。

建议记录：

```text
id；
review_request_id；
title；
summary；
full_report；
model_provider；
model_name；
prompt_version；
input_package_hash；
output_hash；
created_at_utc；
trace_id。
```

报告必须和 request 绑定。

历史报告不得被新报告覆盖。

---

## 19. AIReviewFinding

AIReviewFinding 表示大模型发现的具体问题。

建议记录：

```text
id；
review_report_id；
finding_type；
severity；
title；
description；
related_cycle_id；
related_feature_code；
related_atomic_signal_code；
related_strategy_code；
related_order_id；
evidence；
confidence；
needs_manual_attention；
created_at_utc。
```

Finding 用于后台筛选和重点复查。

---

## 20. AIReviewSuggestion

AIReviewSuggestion 表示大模型提出的优化建议。

建议记录：

```text
id；
review_report_id；
suggestion_type；
priority；
title；
description；
target_object_type；
target_object_id；
suggested_action；
rationale；
expected_impact；
risk_note；
status；
reviewed_by；
reviewed_at_utc；
decision_note；
created_at_utc。
```

Suggestion 必须支持人工状态流转。

---

## 21. 后台页面需求

017 需要增加 AI Review 页面。

页面至少包含：

```text
发起复盘；
选择最近 20 / 50 / 100 个周期；
查看复盘请求列表；
查看复盘报告详情；
查看 Finding 列表；
查看 Suggestion 列表；
按类型 / 优先级 / 状态筛选；
标记建议 accepted / rejected / ignored；
把建议转成人工任务；
下载复盘数据包；
下载复盘报告。
```

017 展示大模型报告时，必须明确标识：

```text
这是大模型分析结果；
不是系统自动执行结论；
需要人工审核。
```

---

## 22. 复盘触发方式

018 P0 支持人工触发。

触发入口来自 017。

018 P0 不做自动定时复盘。

后续可以扩展：

```text
每周自动复盘；
每 100 个 TradingCycle 自动复盘；
发生严重亏损后自动复盘；
RuntimeGuard 发现异常后建议复盘。
```

但 P0 先不做自动触发。

---

## 23. 权限要求

发起 AI Review 需要权限。

查看 AI Review 报告需要权限。

修改 AIReviewSuggestion 状态需要更高权限。

下载复盘包需要权限。

普通只读用户可以查看报告，但不能标记建议状态。

---

## 24. 操作审计

018 相关人工操作必须写审计。

包括：

```text
发起复盘；
取消复盘；
查看敏感复盘包；
下载复盘包；
标记建议 accepted；
标记建议 rejected；
标记建议 ignored；
转人工任务；
关联策略版本变更。
```

审计至少记录：

```text
操作人；
操作时间；
操作类型；
操作对象；
修改前状态；
修改后状态；
trace_id。
```

---

## 25. 成本和大小控制

018 必须控制大模型输入大小。

如果复盘范围太大，应提示：

```text
数据量过大；
建议缩小周期范围；
或使用摘要模式。
```

018 P0 默认支持最近 100 个周期，不建议一次性提交无限历史。

018 应记录 input token / output token 或近似大小，方便后续成本评估。

如果 provider 返回 token usage，应落库保存。

---

## 26. 错误处理

如果大模型调用失败，AIReviewRequest 应标记 failed。

失败时应记录：

```text
error_code；
error_message；
provider_error；
retryable；
trace_id。
```

失败不得影响交易主流程。

失败不得修改 015 收益结果。

失败不得修改策略定义。

---

## 27. 幂等要求

同一 ReviewRequest 不得重复生成多份互相冲突的报告。

如果调用超时或失败，重试必须清晰记录。

同一请求如果重新调用大模型，应保留历史 attempt 或明确覆盖规则。

P0 建议：

```text
同一 request 只保存一个 completed report；
失败后允许人工重新发起新的 request。
```

---

## 28. 与 014 的关系

014 可以巡检 AI Review 长期 failed 或 stuck。

例如：

```text
AIReviewRequest 长时间 calling_model；
AIReviewRequest 多次 failed；
AIReviewPackage 生成失败。
```

014 只写 RuntimeGuard 巡检异常 Alert。

014 不替 018 调用大模型。

---

## 29. 与 020 的关系

020 可以提供是否允许发起 AI Review 的运行时开关。

P0 可以先不做该开关。

即使未来增加开关，020 也只控制是否允许发起复盘，不影响历史报告查看。

018 不允许通过大模型建议自动调用 020 修改交易开关。

---

## 30. P0 不做范围

018 P0 不做：

```text
自动调参；
自动策略优化；
自动创建策略版本；
自动启用 / 禁用策略；
自动下单；
自动定时复盘；
多模型投票；
复杂向量数据库；
长期知识库；
RAG 检索系统；
自动回测；
自动验证建议收益；
大模型直接写代码；
大模型直接提交 Git。
```

这些可以后续扩展。

---

## 31. 测试要求

018 至少覆盖以下测试：

```text
1. 能按最近 20 / 50 / 100 个 TradingCycle 生成 ReviewRequest。
2. 能生成脱敏 ReviewPackage。
3. ReviewPackage 不包含 API key / secret / token。
4. 能调用 mock 大模型并保存 AIReviewReport。
5. 能拆分并保存 AIReviewFinding。
6. 能拆分并保存 AIReviewSuggestion。
7. Suggestion 初始状态为 pending_review。
8. 人工可以标记 Suggestion accepted / rejected / ignored。
9. 大模型建议不能自动修改 FeatureDefinition / AtomicSignalDefinition / StrategyDefinition。
10. 大模型建议不能自动下单。
11. 大模型调用失败时 Request 标记 failed。
12. AI Review 失败不影响交易主流程。
13. 未授权用户不能发起复盘。
14. 下载 ReviewPackage 需要权限。
15. Prompt 版本必须落库。
```

---

## 32. 最终结论

018 的最终定位是：

```text
大模型复盘分析模块。
```

018 负责：

```text
组装复盘数据；
调用大模型；
保存复盘报告；
保存问题发现；
保存优化建议；
支持后台查看和人工评估。
```

018 不负责：

```text
自动改策略；
自动调参；
自动交易；
自动修复；
自动修改历史数据。
```

一句话：

```text
018 让大模型参与复盘和提出建议，但所有策略优化必须经过人工审核和版本管理。
```
