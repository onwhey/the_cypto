# 015 Performance Metrics / Cycle Floating PnL Requirements

> 建议路径：`docs/requirements/performance_metrics.md`  
> 阶段：015  
> 模块建议：`apps.performance_metrics`  
> 状态：需求定稿草案  
> 上游依赖：006 Account / Position Snapshot、007A OrderPlan、009 OrderSubmission、010 OrderStatusSync、011 FillSync、013 TradingCycle  
> 下游使用：017 Admin / Ops Console / Review Workbench  
> 核心结论：015 按 4h TradingCycle 计算周期浮动收益，用于复盘策略目标仓位是否合理；它不是订单已实现盈亏汇总，也不是账户总览估值模块。

---

## 1. 阶段目标

015 的核心目标是：

```text
按每个 4h TradingCycle 计算周期浮动收益，
并整理该周期的策略目标仓位、实际持仓、调仓、订单、成交等复盘上下文，
供 017 首页图表和大模型复盘导出使用。
```

015 关注的是：

```text
这个 4h 周期内，策略层提出或维持的目标仓位是否合理。
```

015 不关注：

```text
某一笔订单平仓时最终实现了多少收益。
```

订单已实现收益仍由 011 记录，只作为订单详情和辅助参考。

---

## 2. 主收益口径

015 的主收益口径是：

```text
周期浮动收益 / Cycle Floating PnL / Mark-to-Market PnL
```

也就是：

```text
一个 4h 周期开始时的持仓状态，
到这个 4h 周期结束时的持仓状态，
在价格变化下产生的持仓表现。
```

015 不把以下内容作为主收益口径：

```text
订单 realized_pnl；
减仓收益；
平仓收益；
手续费后净收益；
账户可用余额变化；
全账户估值变化。
```

---

## 3. 周期划分

015 按 UTC 4h 周期切分收益：

```text
00:00 - 04:00
04:00 - 08:00
08:00 - 12:00
12:00 - 16:00
16:00 - 20:00
20:00 - 00:00
```

013 的实际运行时间是周期边界后 5 分钟：

```text
00:05
04:05
08:05
12:05
16:05
20:05
```

015 必须明确区分：

```text
周期边界 = 4h K 线边界；
编排运行时间 = 周期边界后 5 分钟。
```

---

## 4. 006 快照归属规则

04:05 拉到的 006 快照，归属于：

```text
00:00 - 04:00 周期的期末状态。
```

同一条 04:05 快照也可以作为：

```text
04:00 - 08:00 周期的期初状态。
```

因此：

```text
00:05 快照 + 04:05 快照
用于计算 00:00 - 04:00 周期收益。

04:05 快照 + 08:05 快照
用于计算 04:00 - 08:00 周期收益。
```

015 不能把 04:05 快照误认为只属于 04:00 - 08:00 新周期。

---

## 5. 快照选择规则

015 只能使用：

```text
013 自动交易周期绑定的 006 账户 / 持仓快照。
```

015 禁止使用：

```text
012 手动账户总览快照；
运维手动同步快照；
数据库里最新两条账户快照；
非 TradingCycle 绑定的快照；
非 active trading domain 的快照。
```

015 不能用：

```text
最新账户快照 - 上一条账户快照
```

这种方式计算收益。

正确方式是：

```text
按 TradingCycle 关系找到当前自动周期绑定的 006 快照，
再找到上一个自动 TradingCycle 绑定的 006 快照，
用这两条周期边界快照计算对应 4h 周期收益。
```

---

## 6. 快照缺失处理

如果缺少当前周期自动 006 快照，015 不计算本周期收益。

如果缺少上一个自动周期 006 快照，015 不计算本周期收益。

如果中间存在 012 手动快照，015 忽略该手动快照。

快照不足时，015 记录：

```text
calculation_status = insufficient_snapshot
```

并记录具体原因，例如：

```text
missing_current_cycle_snapshot
missing_previous_cycle_snapshot
snapshot_not_bound_to_trading_cycle
manual_snapshot_not_allowed
```

015 不得用手动快照补算。

---

## 7. 015 读取的数据

015 P0 只读取已经落库的数据。

015 可以读取：

```text
013 TradingCycle；
006 自动周期绑定的账户 / 持仓快照；
007A OrderPlan 的策略目标仓位；
009 OrderSubmission 的订单提交记录；
010 OrderStatusSync 的订单状态记录；
011 FillSync 的成交明细和成交汇总；
001 AlertEvent；
014 RuntimeGuardIssue。
```

015 P0 不直接请求 Binance。

015 P0 不主动刷新账户。

015 P0 不查订单状态。

015 P0 不查成交明细。

015 P0 不跑 012。

---

## 8. 006 的作用

006 在 015 中的作用是：

```text
提供周期边界的账户 / 持仓状态；
提供 active trading domain；
提供持仓数量、持仓方向、标记价格、未实现盈亏等计算所需上下文。
```

006 不是账户总览估值模块。

015 不直接把 006 的 available_balance 当作收益。

015 不直接把 wallet_balance 当作周期收益。

015 不直接把账户字段随便相减作为收益。

015 必须基于持仓、价格和周期边界进行收益计算。

---

## 9. 011 的作用

011 的 realized_pnl 是订单级已实现盈亏。

015 可以保留 011 的 realized_pnl 作为辅助字段。

011 realized_pnl 不能作为 015 的主周期收益。

011 realized_pnl 主要用于：

```text
订单详情页；
订单成交复盘；
解释某个周期是否发生减仓 / 平仓；
辅助大模型复盘。
```

015 的主收益仍然是周期浮动收益。

---

## 10. 手续费处理

015 P0 保留手续费字段，但手续费不作为策略浮动收益主口径。

手续费用于解释执行成本，例如：

```text
本周期发生了多少手续费；
最近 N 个周期手续费是否过高；
调仓频率是否导致成本过高。
```

017 可以同时展示：

```text
周期浮动收益；
订单 realized_pnl；
手续费；
订单净收益。
```

但 015 不把手续费强行归因到策略层目标仓位是否合理。

---

## 11. 资金费处理

015 P0 暂不处理资金费。

015 P0 不同步 funding fee。

015 P0 不拆分资金费影响。

后续如果需要，可以单独设计 funding fee / income sync 模块。

---

## 12. 周期无下单也要计算

即使某个 TradingCycle 没有下单，只要存在连续两个自动周期 006 快照，015 也要计算该周期浮动收益。

原因是：

```text
策略层每 4h 都提出或维持一个理想仓位；
不下单也代表策略认为当前仓位无需变化；
所以该周期持仓表现仍然属于策略判断的一部分。
```

015 不能因为没有订单就跳过收益计算。

---

## 13. 策略目标仓位复盘

015 必须记录每个周期的策略目标仓位。

015 必须记录每个周期的实际持仓状态。

015 必须记录策略目标仓位是否发生变化。

015 必须记录本周期是否发生调仓。

015 的复盘重点是：

```text
策略层提出的理想仓位是否合理。
```

而不是：

```text
这次实际调仓多少是否赚钱。
```

---

## 14. 多账户域处理

015 不自己决定 U 本位或币本位。

015 根据 006 自动周期快照绑定的 active trading domain 处理数据。

如果 006 当前自动周期是 USDS-M，015 处理 USDS-M 数据。

如果未来 006 当前自动周期是 COIN-M，015 处理 COIN-M 数据。

012 手动同步出的非 active domain 数据，不参与 015 交易收益统计。

015 P0 不把币本位强制折算成 USDT。

---

## 15. 输出记录

015 应为每个 TradingCycle 生成最多一条周期表现记录。

建议模型名：

```text
TradingCyclePerformance
```

该记录用于：

```text
017 首页收益图表；
017 TradingCycle 详情页；
017 大模型复盘导出；
014 后续巡检收益计算是否缺失。
```

---

## 16. 建议记录字段

TradingCyclePerformance 建议记录以下业务信息：

```text
trading_cycle_id
cycle_start_utc
cycle_end_utc
market_type
symbol
account_domain
start_snapshot_id
end_snapshot_id
target_position_direction
target_position_quantity
target_position_notional
actual_position_direction_start
actual_position_quantity_start
actual_position_direction_end
actual_position_quantity_end
mark_price_start
mark_price_end
cycle_floating_pnl
cycle_floating_pnl_pct
has_order_plan
has_order_submission
has_fill
order_realized_pnl
order_commission
order_net_realized_pnl
calculation_status
reason_code
trace_id
created_at_utc
updated_at_utc
```

字段名可在 plan 中根据现有模型实际字段调整。

需求层面要求记录这些业务含义，不要求完全固定字段名。

---

## 17. 计算状态

015 至少支持以下计算状态：

```text
calculated
insufficient_snapshot
skipped
failed
```

含义：

```text
calculated：
  本周期收益已成功计算。

insufficient_snapshot：
  缺少必要的自动周期快照，不能计算。

skipped：
  本周期不适合计算，例如非交易周期或缺少必要上游结果。

failed：
  计算过程发生代码异常或不可预期错误。
```

---

## 18. 原因码

015 应记录不能计算或跳过的原因。

建议原因码包括：

```text
missing_current_cycle_snapshot
missing_previous_cycle_snapshot
snapshot_not_bound_to_trading_cycle
manual_snapshot_not_allowed
missing_position_data
missing_mark_price
unsupported_account_domain
upstream_cycle_not_finalized
calculation_exception
```

原因码可在 plan 中继续细化。

---

## 19. 幂等要求

015 必须幂等。

同一个 TradingCycle 重复运行 015，不得重复创建多条周期表现记录。

如果已有记录，015 应更新同一条记录，或按明确规则跳过。

每个 TradingCycle 最多一条有效 TradingCyclePerformance。

---

## 20. 对交易主流程的影响

015 不影响交易主流程。

015 计算失败不能阻断下一轮交易。

015 计算失败不能修改 006 账户快照。

015 计算失败不能修改 009 订单提交结果。

015 计算失败不能修改 010 订单状态结果。

015 计算失败不能修改 011 成交同步结果。

015 计算失败不能释放订单锁。

015 计算失败不能触发下单。

---

## 21. 与 013 的关系

013 负责创建 TradingCycle 并绑定每轮关键对象。

015 依赖 013 提供的 TradingCycle 和其绑定的 006 快照。

如果 013 没有正确绑定 006 快照，015 不得按时间乱找快照补算。

015 应标记 insufficient_snapshot 或 missing binding。

---

## 22. 与 014 的关系

014 可以巡检 015 是否长期缺失或计算失败。

014 发现 015 异常时，只写 RuntimeGuard 巡检异常 Alert。

014 不替 015 计算收益。

014 不修复 015 结果。

---

## 23. 与 017 的关系

017 首页收益图表默认使用 015 的 cycle_floating_pnl。

017 TradingCycle 详情页显示：

```text
周期浮动收益；
策略目标仓位；
实际持仓；
订单提交；
订单状态；
成交明细；
订单 realized_pnl；
手续费；
Alert；
RuntimeGuardIssue。
```

017 大模型复盘导出必须包含 015 的周期表现记录。

---

## 24. P0 不做范围

015 P0 不做：

```text
完整账户净值曲线；
资金费拆分；
跨账户域统一估值；
币本位强制 USDT 折算；
VaR / 夏普比率 / 最大回撤等复杂指标；
长期组合绩效分析；
手动快照补算；
订单级已实现收益作为周期主收益；
自动复盘结论生成。
```

这些可以后续扩展。

---

## 25. 测试要求

015 至少需要覆盖以下测试：

```text
1. 00:05 和 04:05 自动周期快照可计算 00:00-04:00 周期浮动收益。
2. 04:05 快照可作为上一周期 ending snapshot 和下一周期 starting snapshot。
3. 012 手动快照夹在两个自动快照之间时，015 必须忽略手动快照。
4. 缺少当前周期自动快照时，状态为 insufficient_snapshot。
5. 缺少上一个周期自动快照时，状态为 insufficient_snapshot。
6. 无下单但有持仓时，也要计算周期浮动收益。
7. 有订单 realized_pnl 时，只作为辅助字段保留，不能替代 cycle_floating_pnl。
8. 手续费字段保留，但不影响策略浮动收益主口径。
9. 同一个 TradingCycle 重复计算不得生成重复记录。
10. 计算失败不得修改 006 / 009 / 010 / 011 / 013 上游结果。
11. 非 active domain 或 012 非交易快照不得参与收益计算。
12. 资金费 P0 不处理。
```

---

## 26. 最终结论

015 的最终定位是：

```text
周期浮动收益统计 + 策略复盘上下文。
```

015 用于回答：

```text
每个 4h 周期里，策略层提出或维持的目标仓位表现如何？
```

015 不用于回答：

```text
某次平仓最终赚了多少钱？
```

一句话：

```text
015 按 4h TradingCycle 计算策略持仓表现，不按订单平仓收益计算周期收益。
```
