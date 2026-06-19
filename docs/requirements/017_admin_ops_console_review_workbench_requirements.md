# 017 Admin / Ops Console / Review Workbench Requirements

> 建议路径：`docs/requirements/admin_ops_console.md`  
> 阶段：017  
> 模块建议：后端 `apps.ops_console` / 前端独立 `frontend` 或 `web_console`  
> 状态：需求定稿草案  
> 上游依赖：001 Alert、004 Feature、005 Atomic / Strategy、006 Account Snapshot、007A/007B/007C、008、009、010、011、012、013、014、015  
> 核心结论：017 是独立 Web Console，不使用 Django Admin 作为主后台；它负责看系统、管异常、人工补查、查看收益、追踪周期、管理策略、导出复盘，不负责自动交易决策和自动修复交易。

---

## 1. 阶段定位

017 的正式定位是：

```text
Admin / Ops Console / Review Workbench
```

017 是一个独立 Web Console，用于查看和管理整个自动交易系统。

017 不是 Django Admin。

017 不是传统数据库后台。

017 不是交易执行器。

017 是：

```text
可视化运维控制台 + 交易周期详情页 + 收益图表 + 人工补查入口 + 策略管理入口 + 大模型复盘工作台。
```

---

## 2. 技术方向

017 前端建议使用：

```text
Next.js + shadcn/ui
```

图表建议使用：

```text
Recharts 或 ECharts
```

后端仍然使用：

```text
Django + 现有业务 service + API
```

017 前端通过 API 调用 Django 后端。

017 前端不得直接访问数据库。

017 前端不得直接修改业务表。

所有人工操作必须调用后端 service。

---

## 3. 不使用 Django Admin 作为主后台

017 不使用 Django Admin 作为主后台需求。

017 不要求基于 Unfold、Jazzmin 或其他 Django Admin 主题实现。

017 可以完全独立于 Django Admin。

如果开发过程中临时保留 Django Admin，只能作为开发便利，不属于 017 正式产品需求。

正式后台应是独立 Web Console。

---

## 4. 总体页面结构

017 P0 至少包含以下页面：

```text
1. Dashboard 首页
2. TradingCycle 列表页
3. TradingCycle 详情页
4. Order 订单详情页
5. Account Overview 账户总览页
6. Runtime Guard 巡检异常页
7. Alert 告警记录页
8. Strategy Registry 策略 / 特征 / 原子管理页
9. AI Review Export 大模型复盘导出页
10. Ops Actions 人工运维操作入口
```

这些页面可以按一个统一左侧导航组织。

---

## 5. Dashboard 首页

Dashboard 首页用于快速查看系统当前状态和最近收益表现。

首页至少展示：

```text
最近 N 个 4h 周期浮动收益曲线；
累计浮动收益曲线；
最近 TradingCycle 状态；
最近是否有下单；
最近是否有 unknown；
当前是否存在 active lock；
当前是否存在 RuntimeGuard open issue；
最近 Alert；
账户快照更新时间；
最近一次 012 账户总览同步时间；
最近一次 015 收益计算状态。
```

收益图表默认读取 015 的：

```text
cycle_floating_pnl
```

首页不得使用 011 realized_pnl 作为周期收益主图。

011 realized_pnl 只能作为订单级辅助指标展示。

---

## 6. Dashboard 收益展示口径

首页收益展示必须区分：

```text
周期浮动收益；
订单已实现收益；
手续费；
订单净收益。
```

其中：

```text
周期浮动收益 = 015 主收益口径；
订单已实现收益 = 011 realized_pnl；
手续费 = 011 commission；
订单净收益 = realized_pnl - commission。
```

首页主图使用周期浮动收益。

订单详情页使用订单已实现收益。

不得把平仓 realized_pnl 误展示为 4h 周期收益。

---

## 7. TradingCycle 列表页

TradingCycle 列表页用于查看最近交易周期。

列表至少展示：

```text
周期开始时间；
周期结束时间；
运行时间；
最终状态；
是否产生策略信号；
是否生成 OrderPlan；
是否通过 RiskCheck；
是否提交订单；
是否成交；
015 周期浮动收益；
是否 blocked；
是否 unknown；
是否 failed；
是否需要人工处理；
trace_id。
```

列表支持筛选：

```text
最近 20 / 50 / 100 个周期；
只看有订单周期；
只看无订单周期；
只看 blocked；
只看 unknown；
只看 failed；
只看收益已计算周期；
只看收益缺失周期；
只看存在 RuntimeGuardIssue 的周期。
```

---

## 8. TradingCycle 详情页

TradingCycle 详情页必须展示某个周期的完整业务链路。

详情页至少展开：

```text
TradingCycle 基本信息；
数据质量结果；
市场快照；
特征结果；
原子信号；
策略信号；
策略质量结果；
006 账户 / 持仓快照；
007A OrderPlan；
007B RiskCheck；
008 ExecutionPreparation；
009 OrderSubmission；
010 OrderStatusSync；
011 FillSync / FillSummary；
015 Performance Metrics；
007C ActiveLock Finalization；
AlertEvent；
RuntimeGuardIssue。
```

详情页的核心目标是回答：

```text
这一轮为什么下单？
这一轮为什么不下单？
这一轮为什么 blocked？
这一轮为什么 unknown？
这一轮策略目标仓位是否合理？
这一轮收益表现如何？
```

---

## 9. TradingCycle 链路追踪要求

013 必须提供足够的关键对象 ID，供 017 展示完整链路。

017 不应通过复杂猜测来拼链路。

017 应优先基于：

```text
TradingCycle 绑定的关键 ID；
各阶段 trace_id；
各阶段外键关系；
订单 client_order_id；
Binance order_id。
```

如果链路缺失，017 应明确显示：

```text
链路缺失；
缺失阶段；
缺失原因；
是否需要人工处理。
```

不得静默隐藏缺失链路。

---

## 10. Order 订单详情页

Order 页面围绕 009 OrderSubmission 展开。

订单详情至少展示：

```text
009 订单提交记录；
Binance orderId；
clientOrderId；
symbol；
side；
positionSide；
market_type；
quantity；
order_type；
submit_status；
submit_response；
error_code；
error_message；
010 订单状态同步记录；
011 成交明细；
011 成交汇总；
realized_pnl；
commission；
net_realized_pnl；
关联 TradingCycle；
关联 OrderPlan；
关联 RiskCheck；
关联 ActiveLock；
关联 Alert；
关联 RuntimeGuardIssue。
```

订单详情页的收益口径是订单级收益，不是 015 周期收益。

订单详情页可以展示：

```text
011 realized_pnl；
011 commission；
011 net_realized_pnl。
```

但不得把这些字段称为周期收益。

---

## 11. 010 / 011 手动补查入口

017 必须提供安全人工补查入口：

```text
手动同步某个 009 的 010 订单状态；
手动同步某个 009 的 011 成交明细；
查看补查结果；
查看补查失败原因。
```

手动补查必须调用 010 / 011 后端 service。

手动补查不得直接修改订单结果表。

手动补查不得绕过 010 / 011 的幂等逻辑。

手动补查不得自动释放订单锁，除非通过 007C 明确服务入口完成。

---

## 12. Account Overview 账户总览页

Account Overview 页面读取 012 的账户总览数据。

页面至少展示：

```text
U 本位账户快照；
币本位账户快照；
资产余额；
持仓；
保证金相关字段；
同步状态；
最近同步时间；
错误信息；
trace_id。
```

页面提供：

```text
手动触发 012 账户总览一键同步。
```

012 页面只用于账户总览。

012 页面不参与交易主流程。

012 页面不参与 015 周期收益计算。

012 手动快照不得作为 015 收益计算快照。

---

## 13. Runtime Guard 巡检异常页

Runtime Guard 页面展示 014 发现的巡检异常。

至少展示：

```text
TradingCycle 漏跑；
TradingCycle running 卡住；
OrderPlanActiveLock 长期 active；
009 unknown 未处理；
010 unknown / not_found 未处理；
011 unknown / synced_empty 未处理；
账户快照过期；
Alert 投递异常；
015 收益计算长期缺失或失败。
```

页面支持处理 RuntimeGuardIssue 状态：

```text
acknowledged；
resolved；
ignored。
```

017 不直接修复交易异常。

017 只通过后端 service 更新 RuntimeGuardIssue 自身状态。

---

## 14. Alert 告警记录页

Alert 页面展示系统 AlertEvent。

至少展示：

```text
交易周期开始 / 结束 Alert；
订单提交 Alert；
订单状态同步 Alert；
成交同步 Alert；
007C 锁收尾 Alert；
014 RuntimeGuard 巡检异常 Alert；
Alert 投递失败记录。
```

014 的告警必须明确显示：

```text
[RuntimeGuard] 巡检异常
```

017 不得把 RuntimeGuard 巡检异常显示成原业务模块实时异常。

017 应区分：

```text
业务模块实时 Alert；
014 巡检发现的 Alert。
```

---

## 15. Strategy Registry 管理页

017 应提供策略体系管理页面。

管理范围包括：

```text
FeatureDefinition；
AtomicSignalDefinition；
StrategyDefinition；
StrategyRouteRule；
策略启用 / 禁用状态；
策略生命周期状态；
参数 hash；
definition hash；
版本信息。
```

支持操作：

```text
查看；
启用；
禁用；
retire；
创建新版本；
查看参数；
查看 hash；
查看依赖关系。
```

---

## 16. 特征 / 原子 / 策略修改边界

017 不允许随便覆盖生产中的策略定义。

禁止直接就地修改：

```text
algorithm_name；
algorithm_version；
生产中的 params；
definition_hash；
params_hash。
```

参数变化应走新版本。

老版本应 retire。

新版本启用应保留审计。

017 的策略管理必须遵守项目已有版本管理原则。

---

## 17. AI Review Export 大模型复盘导出

017 必须支持选择最近 N 个 TradingCycle 打包导出。

默认支持：

```text
最近 20 个周期；
最近 50 个周期；
最近 100 个周期；
自定义时间范围。
```

导出内容至少包含：

```text
TradingCycle 基本信息；
市场快照摘要；
特征结果；
原子信号；
策略信号；
策略目标仓位；
实际持仓；
007A OrderPlan；
007B RiskCheck；
008 ExecutionPreparation；
009 OrderSubmission；
010 OrderStatusSync；
011 FillSync；
015 周期浮动收益；
Alert；
RuntimeGuardIssue；
blocked / unknown / failed 原因。
```

导出目标是：

```text
给大模型复盘策略目标仓位是否合理；
分析特征、原子、策略、风控、执行、收益之间的关系。
```

---

## 18. AI Review Export 格式

017 P0 至少支持一种结构化导出格式。

建议支持：

```text
JSON；
Markdown；
JSON + Markdown summary。
```

导出文件必须便于大模型阅读。

导出中不得包含：

```text
API key；
secret；
签名；
请求 header；
认证 token；
数据库密码；
环境变量敏感信息。
```

---

## 19. Ops Actions 人工运维操作

017 至少提供以下人工操作：

```text
手动同步某个 009 的 010 状态；
手动同步某个 009 的 011 成交；
手动触发 012 账户总览一键同步；
标记 RuntimeGuardIssue acknowledged；
标记 RuntimeGuardIssue resolved；
标记 RuntimeGuardIssue ignored；
查看当前 active lock；
通过 007C 人工入口处理锁；
查看当前系统是否允许交易；
导出最近 N 个周期复盘数据。
```

所有人工操作都必须调用后端 service。

页面按钮不得直接写数据库。

---

## 20. 危险操作确认

以下操作必须有二次确认：

```text
手动补查 010；
手动补查 011；
手动触发 012 全账户同步；
处理 active lock；
修改策略启用状态；
retire 策略 / 特征 / 原子定义；
创建新版本；
标记 RuntimeGuardIssue ignored。
```

二次确认内容必须说明：

```text
操作对象；
操作影响；
是否会调用 Binance；
是否会修改业务数据；
是否会影响交易主流程。
```

---

## 21. 操作审计

017 的人工操作必须写审计记录。

审计记录至少包含：

```text
操作人；
操作时间；
操作类型；
操作对象；
操作前状态；
操作后状态；
操作结果；
失败原因；
trace_id。
```

审计记录不得被普通页面操作删除。

---

## 22. 权限与登录

017 必须有登录认证。

017 不允许匿名访问。

017 至少区分：

```text
只读查看权限；
运维操作权限；
策略管理权限；
超级管理员权限。
```

只读用户不能执行补查、同步、修改策略、处理锁等操作。

策略管理权限不能自动获得订单补查权限。

运维操作权限不能直接修改策略版本。

---

## 23. API 安全

017 前端通过 API 调用后端。

API 必须做权限校验。

API 不得返回敏感信息。

API 不得返回：

```text
Binance API key；
Binance secret；
签名参数；
认证 token；
环境变量；
数据库连接信息；
完整 request header。
```

API 对危险操作必须要求后端校验权限。

不能只依赖前端隐藏按钮。

---

## 24. 017 与交易主流程边界

017 不是自动交易流程的一部分。

017 不自动下单。

017 不自动撤单。

017 不自动补单。

017 不自动释放订单锁。

017 不自动修改 TradingCycle 最终状态。

017 不自动把 unknown 改成 success。

017 不绕过 007C / 010 / 011 / 012 / 014 / 015 的服务。

017 所有操作必须通过后端安全 service。

---

## 25. 与 015 的关系

017 首页收益图表使用 015 的周期浮动收益。

017 TradingCycle 详情页展示 015 周期表现记录。

017 AI 复盘导出包含 015 周期浮动收益。

017 不自己计算周期收益。

017 不用 011 realized_pnl 替代 015 cycle_floating_pnl。

017 不使用 012 手动快照计算收益。

---

## 26. 与 013 的关系

017 TradingCycle 页面读取 013 TradingCycle。

017 不创建 TradingCycle。

017 不重跑 TradingCycle。

017 不修改 TradingCycle 最终状态。

017 可以展示 TradingCycle running / blocked / unknown / failed 等状态。

017 可以提示是否需要人工处理。

---

## 27. 与 014 的关系

017 Runtime Guard 页面读取 014 RuntimeGuardIssue。

017 可以通过后端 service 标记 issue 状态。

017 不替 014 巡检。

017 不修复 014 发现的业务问题。

017 不把 RuntimeGuard 巡检异常伪装成业务模块实时异常。

---

## 28. 与 012 的关系

017 Account Overview 页面读取 012 账户总览结果。

017 可以手动触发 012 一键同步。

017 不把 012 手动快照用于 015 收益计算。

017 不把 012 账户总览作为交易主流程 gate。

---

## 29. 与 007C 的关系

017 可以展示当前 active lock。

017 可以通过 007C 提供的人工入口处理锁。

017 不直接修改 OrderPlanActiveLock。

017 不在页面里直接把 lock 改成 released / failed。

---

## 30. 前端展示要求

017 前端应具备现代控制台体验。

建议包含：

```text
左侧导航；
顶部状态栏；
指标卡片；
折线图；
柱状图；
表格；
详情抽屉；
弹窗确认；
状态标签；
时间范围筛选；
深色模式可选。
```

页面应优先面向运维和复盘效率，而不是单纯 CRUD。

---

## 31. P0 不做范围

017 P0 不做：

```text
手机端完整适配；
公开用户系统；
多租户；
复杂权限后台；
实时 WebSocket 推送；
自动大模型复盘结论；
自动策略优化；
自动参数修改；
自动交易修复；
资金费精细分析；
回测系统。
```

这些后续可以扩展。

---

## 32. 测试要求

017 至少需要覆盖以下测试或验收点：

```text
1. 未登录不能访问 Web Console。
2. 只读用户不能执行人工补查。
3. Dashboard 能展示 015 周期浮动收益。
4. Dashboard 不把 011 realized_pnl 当成周期收益。
5. TradingCycle 列表能显示最近周期状态。
6. TradingCycle 详情能展开完整链路。
7. 订单详情能展开 009 / 010 / 011。
8. Account 页面能显示 012 同步结果。
9. RuntimeGuard 页面能展示 014 巡检异常。
10. Alert 页面能区分业务 Alert 和 RuntimeGuard 巡检 Alert。
11. 手动补查 010 必须调用后端 service。
12. 手动补查 011 必须调用后端 service。
13. 手动触发 012 必须调用后端 service。
14. 策略修改不能直接覆盖生产参数。
15. AI Review Export 不包含 API key / secret / token。
16. 危险操作必须二次确认。
17. 人工操作必须写审计。
18. 前端不能直接写数据库。
```

---

## 33. 最终结论

017 的最终定位是：

```text
独立 Web Console / 运维控制台 / 复盘工作台。
```

017 负责：

```text
看收益；
看周期；
看订单；
看账户；
看异常；
人工补查；
处理巡检 issue；
管理策略定义；
导出大模型复盘包。
```

017 不负责：

```text
自动交易；
自动修复；
自动下单；
自动撤单；
自动解锁；
自动改业务结果。
```

一句话：

```text
017 是用户日常查看系统、处理异常、复盘策略的现代化控制台，不是交易执行器。
```
