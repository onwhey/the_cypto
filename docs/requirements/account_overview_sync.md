# 012 Account Overview / One-Click Sync Requirements

> 建议路径：`docs/requirements/account_overview_sync.md`  
> 阶段：012  
> 模块：Account Overview / One-Click Sync  
> 基础能力来源：006 Binance Account Sync  
> 上游关系：不绑定 009 / 010 / 011  
> 下游关系：可供 UI、人工查看、运维排查、后续实时账户流模块使用  
> 状态：需求定稿草案  
> 核心结论：012 在 006 既有 `apps.binance_account_sync` 基础上新增“账户总览多账户域同步入口”；012 可以一次性同步全部支持账户域，但不改变 006 交易流程只消费 active domain 的规则。

---

## 1. 阶段目的

012 只负责一件事：

```text
基于 006 已有 Binance 账户 / 持仓同步能力，
提供“一键同步账户总览”的能力，
一次性拉取全部支持账户域的 Binance 原始账户 / 持仓快照，
并提供最新账户总览读取能力。
```

012 面向的是：

```text
用户查看当前 Binance 账户状态
UI 展示账户总览
人工运维排查
交易后手动查看账户状态
```

012 不面向某一笔订单。

012 不回答：

```text
某一笔 009 订单是否对账通过
某一笔 011 成交是否已经反映到持仓
是否可以释放订单锁
是否可以继续下单
```

012 只回答：

```text
当前 Binance 各账户域里有什么原始余额、什么持仓、保证金状态如何、同步时间是什么。
```

---

## 2. 012 与 006 的关系

012 不重新实现账户同步。

012 必须基于 006 已有能力继续扩展。

关系定义：

```text
006：
  交易流程内账户 / 持仓快照。
  用于给 OrderPlan / RiskCheck / ExecutionPreparation / OrderSubmission 提供交易前状态。
  正常交易流程调用时，只同步并服务当前 active account domain。
  正常交易流程调用失败时，可以 fail-closed 阻断后续交易。

012：
  账户总览 / 一键同步。
  用于人工查看、UI 展示、运维排查。
  可以一次性同步全部支持账户域。
  不具备交易阻断权。
```

代码与数据原则：

```text
1. 012 复用 006 的 apps.binance_account_sync。
2. 012 不新建第二套账户同步 app。
3. 012 不新建第二套账户 / 持仓快照表。
4. 012 不重新实现 Binance account / position client。
5. 012 写入 006 已有账户 / 持仓快照表。
6. 012 通过 market_type / account_domain 区分 U 本位和币本位数据。
```

这不是：

```text
006 一套账户模块
012 又一套账户模块
```

而是：

```text
同一个 apps.binance_account_sync
  ├── 006 交易入口：active domain 单域同步
  └── 012 总览入口：全部支持 domain 多域同步
```

---

## 3. 012 需要对 006 做的最小扩展

012 可以改动 006 既有代码，但必须保持 006 原交易语义不变。

建议把底层同步能力拆成三层：

```text
sync_domain(domain)
  同步指定 account domain。
  作为底层通用能力。

sync_active_for_trading()
  只同步当前 active account domain。
  服务 006 交易流程。
  可以 fail-closed 阻断后续交易。

sync_all_for_overview()
  依次同步全部支持 account domain。
  服务 012 账户总览。
  不具备交易阻断权。
```

如果现有代码只有一个 `sync_once()`，012 实现时可以重构，但要求：

```text
1. 006 原有命令 / 服务行为不能被改成默认同步全部账户域。
2. 006 交易流程仍然只同步并消费 active account domain。
3. 012 新增 overview 入口才允许同步全部支持账户域。
4. 012 的多域同步结果只能用于账户总览展示，不能被 OrderPlan / RiskCheck / OrderSubmission 消费。
```

---

## 4. 012 与订单链路的关系

012 不绑定订单链路。

012 不要求存在：

```text
009 订单提交记录
010 订单状态查询记录
011 成交明细
011 订单成交汇总
```

012 不保存：

```text
009 外键
010 外键
011 外键
```

012 可以记录本次同步的触发来源，例如：

```text
manual
ui_one_click
scheduled
post_trade_view
system_check
```

但 `post_trade_view` 只表示“交易后人工或系统触发查看账户状态”，不表示 012 绑定某一笔订单。

---

## 5. 012 不做对账

012 不做订单对账。

012 不比较：

```text
009 提交记录
010 订单状态
011 成交汇总
账户持仓变化
```

012 不判断：

```text
某笔订单是否完成
某笔订单是否已经反映到账户
某笔成交是否和持仓匹配
某笔交易是否可以收尾
```

012 只做：

```text
向 Binance 拉取当前账户 / 持仓数据；
保存为账户 / 持仓快照；
展示最新成功快照。
```

如果未来需要订单诊断或对账，应单独设计诊断模块，不进入 012。

---

## 6. 012 不具备交易阻断权

012 的同步结果不得直接阻断交易。

要求：

```text
1. 012 同步失败，不得直接阻断 OrderPlan。
2. 012 同步失败，不得直接阻断 RiskCheck。
3. 012 同步失败，不得直接阻断 ExecutionPreparation。
4. 012 同步失败，不得直接阻断 OrderSubmission。
5. 012 不得修改全局交易允许状态。
6. 012 不得生成会被交易链路直接消费的阻断结论。
```

允许：

```text
1. 记录日志。
2. 写审计。
3. 写告警。
4. UI 显示同步失败。
```

唯一允许作为交易阻断依据的是：

```text
正常交易流程中，由 006 账户 / 持仓快照阶段生成的新交易前快照。
```

也就是说：

```text
006 是交易链路 gate。
012 是账户展示 / 一键同步工具。
```

---

## 7. 012 不维护 current 状态表

012 只保存账户 / 持仓快照。

012 不维护额外的：

```text
CurrentAccountState
CurrentPosition
CurrentBalance
CurrentAccountOverview
```

这类“当前状态表”。

UI 展示当前账户状态时，直接读取：

```text
最新一次成功同步快照
```

如果未来需要真正实时的当前状态，应单独做：

```text
Account WebSocket Stream / 账户实时流
```

实时状态不在 012 实现。

---

## 8. 012 不做 WebSocket

012 P0 不做 WebSocket。

012 不处理：

```text
listenKey 创建
listenKey keepalive
WebSocket 连接
WebSocket 断线重连
ACCOUNT_UPDATE 消息
ORDER_TRADE_UPDATE 消息
UI 实时推送
实时状态缓存
```

012 只做 REST 快照同步。

未来如果需要实时账户状态和 UI 实时渲染，应单独设计后续阶段：

```text
Account Realtime Stream
```

---

## 9. 012 不做估值

012 不做账户估值。

012 不保存：

```text
estimated_total_value_usdt
estimated_value_usdt
valuation_status
valuation_price
valuation_price_source
valuation_time
valuation_snapshot
AccountOverviewValuation
```

原因：

```text
估值不是 Binance 返回的原始账户数据；
估值依赖价格来源、价格时效和换算口径；
012 的职责是拉取和展示 Binance 原始账户 / 持仓事实，不负责估值计算。
```

如果未来 UI 需要展示总估值，可以由 UI 或后续展示层基于最新快照和实时价格临时计算。

该临时估值不得写回 006 / 012 的账户快照表。

---

## 10. 综合账户总览口径

012 展示的是综合账户总览，但总览只汇总展示原始账户块，不做统一估值。

012 展示分层：

```text
综合账户总览
  ├── U 本位账户原始数据
  └── 币本位账户原始数据
```

示例：

```text
账户总览：

U 本位账户：
  钱包余额：50 USDT
  可用余额：xx USDT
  保证金余额：xx USDT
  持仓：BTCUSDT 多/空 xx

币本位账户：
  钱包余额：0.1 BTC
  可用余额：xx BTC
  保证金资产：BTC
  持仓：BTCUSD_PERP 多/空 xx 张
```

要求：

```text
1. U 本位账户展示 USDT / USDC 等保证金资产的原始余额。
2. 币本位账户展示 BTC / ETH 等币种保证金资产的原始余额。
3. 持仓展示 Binance 返回的原始持仓数量、合约张数、方向、开仓均价、标记价格、未实现盈亏。
4. 不得为了统一展示而覆盖原始币种数值。
5. 不得把币本位原始余额强制换算成 USDT 后替代原始余额。
```

---

## 11. 市场域与账户展示规则

012 是账户总览，不是交易执行。

因此 012 一键同步和展示可以覆盖：

```text
USDS-M Futures / U 本位账户
COIN-M Futures / 币本位账户
```

但交易链路仍然只能消费当前 active market domain。

要求：

```text
1. 012 不按 U 本位和币本位拆成两个账户模块。
2. 012 在一个综合账户总览中展示 U 本位和币本位。
3. 012 overview 同步入口可以同步全部支持账户域。
4. active market domain 只用于自动交易链路选择。
5. active market domain 不限制 012 的账户总览展示。
6. 自动交易仍然只能消费当前 active domain 的账户状态，避免 U 本位和币本位混用下单。
```

示例：

```text
展示层：
  可以同步和显示 U 本位 + 币本位综合账户状态。

交易层：
  每次自动交易只允许一个 active domain 参与。
```

---

## 12. 一键同步范围

012 一键同步默认同步全部支持账户域：

```text
USDS-M Futures / U 本位
COIN-M Futures / 币本位
```

P0 默认：

```text
同步全部支持账户域。
```

允许后续扩展参数：

```text
sync all
sync USDS-M only
sync COIN-M only
```

如果某个账户域没有配置 API 权限或项目暂未启用，可记录为 skipped / unsupported，不应影响其他账户域成功同步。

注意：

```text
012 的 sync all 只适用于账户总览。
006 交易流程仍然只同步并消费 active account domain。
```

---

## 13. 部分成功处理

012 必须允许部分成功。

示例：

```text
U 本位同步成功。
币本位同步失败。
```

结果应为：

```text
partial_success
```

展示要求：

```text
U 本位：
  显示本次最新成功快照。

币本位：
  显示本次同步失败；
  同时可以继续展示上一次成功快照；
  必须标明上一次成功同步时间。
```

不得因为某一账户域失败，就丢弃其他账户域已成功同步的数据。

---

## 14. 同步结果分类

012 本地同步结果建议分为：

```text
success
partial_success
failed
skipped
unsupported
```

含义：

```text
success：
  全部目标账户域同步成功。

partial_success：
  至少一个目标账户域同步成功，至少一个目标账户域失败、跳过或不支持。

failed：
  全部目标账户域同步失败。

skipped：
  本次没有可同步账户域，或调用参数明确跳过。

unsupported：
  某个账户域当前项目未支持或未配置。
```

注意：

```text
以上结果只用于展示、审计、告警。
不得直接作为交易阻断依据。
```

---

## 15. 同步运行记录

012 不新增第二套账户 / 持仓表。

但可以新增或复用一类“账户总览同步运行记录”，记录一次一键同步行为。

运行记录至少应包含：

```text
触发来源
触发时间
目标账户域列表
每个账户域同步状态
整体同步结果
成功账户域数量
失败账户域数量
错误码
错误信息
trace_id
created_at
updated_at
```

账户余额、持仓等业务数据仍然写入 006 已有账户 / 持仓快照表。

注意：

```text
如果复用 006 的 sync run 表，需要能区分 trading sync 和 overview sync。
如果新增运行记录表，只记录 012 一键同步行为，不重复保存账户 / 持仓业务数据。
```

---

## 16. 最新账户总览读取

012 必须提供读取最新账户总览的能力。

读取规则：

```text
1. 按账户域读取最新成功账户 / 持仓快照。
2. 每个账户域显示自己的同步时间、同步状态和来源。
3. 如果某个账户域本次同步失败，可以继续展示该域上一次成功快照。
4. UI 必须能看出每个账户域数据的新鲜度。
5. UI 当前账户状态来自最新成功快照，不来自 current 状态表。
```

示例：

```text
账户总览：

U 本位：
  同步状态：success
  同步来源：012 one-click
  同步时间：2026-06-17 10:00 UTC
  原始余额：50 USDT

币本位：
  同步状态：failed_this_time
  当前展示：上一次成功快照
  上次成功同步时间：2026-06-17 08:00 UTC
  原始余额：0.1 BTC
```

---

## 17. 006 快照与 012 展示的关系

如果 006 在正常交易流程中跑了一次账户 / 持仓快照，并成功落库：

```text
012 账户总览可以读取并展示这份快照。
```

如果 012 overview 入口同步了全部账户域：

```text
012 账户总览可以展示全部账户域的新快照；
但交易链路仍然只允许消费 active domain。
```

示例：

```text
active domain = USDS-M
012 一键同步全部账户域成功

012 账户总览显示：
  U 本位：刚刚由 012 overview 同步更新
  币本位：刚刚由 012 overview 同步更新

后续交易流程：
  仍然只能消费 active domain = USDS-M 的账户数据
  不能因为 012 刚同步了币本位，就让币本位参与 OrderPlan / RiskCheck
```

012 展示 006 或 012 快照时：

```text
1. 不继承 006 的交易阻断语义。
2. 不把账户快照解释为订单对账结果。
3. 只作为账户总览数据来源。
```

---

## 18. 交易链路隔离规则

012 同步全部账户域后，必须保证交易链路不被污染。

要求：

```text
1. OrderPlan 只能读取当前 active domain 的交易前账户上下文。
2. RiskCheck 只能读取当前 active domain 的交易前账户上下文。
3. ExecutionPreparation 只能使用当前 active domain 对应的交易参数。
4. OrderSubmission 只能提交当前 active domain 的订单。
5. 非 active domain 快照不得参与自动交易决策。
6. 012 overview sync 不得修改 active market domain。
7. 012 overview sync 不得改变交易流程的账户选择规则。
```

---

## 19. Alert / 审计规则

012 必须写审计。

建议规则：

```text
success：
  普通审计。

partial_success：
  普通审计 + UI 提示。
  如涉及关键账户域失败，可写告警。

failed：
  告警。

skipped：
  普通日志或普通审计。

unsupported：
  普通日志或普通审计。
```

再次强调：

```text
012 告警不等于阻断交易。
```

---

## 20. 012 禁止事项

012 严禁实现：

```text
1. 订单状态查询。
2. 成交明细同步。
3. 订单对账。
4. 持仓与某笔订单的对账。
5. 账户估值。
6. estimated_total_value_usdt。
7. valuation snapshot。
8. 重新下单。
9. 补单。
10. 缩单。
11. 主动撤单。
12. 自动撤单。
13. 订单锁释放。
14. 订单锁 completed / failed 收尾。
15. 修改交易允许状态。
16. 阻断 OrderPlan / RiskCheck / ExecutionPreparation / OrderSubmission。
17. 维护 CurrentAccountState / CurrentPosition 当前状态表。
18. WebSocket user data stream。
19. UI 实时推送。
20. 重新实现一套账户同步 app。
21. 重新实现一套账户 / 持仓快照表。
22. 改变 006 交易入口只服务 active domain 的语义。
23. 让非 active domain 快照参与自动交易决策。
```

---

## 21. 测试要求

### 21.1 012 一键同步全部账户域成功

期望：

```text
1. 012 触发 U 本位账户同步。
2. 012 触发币本位账户同步。
3. 两个账户域均成功。
4. 整体结果为 success。
5. 账户 / 持仓数据写入 006 已有快照表。
6. 不新建第二套账户 / 持仓表。
7. 不阻断交易。
```

### 21.2 012 U 本位成功，币本位失败

期望：

```text
1. U 本位成功落库。
2. 币本位记录失败。
3. 整体结果为 partial_success。
4. U 本位最新快照可展示。
5. 币本位可继续展示上一次成功快照，并标明时间。
6. 不阻断交易。
```

### 21.3 012 全部账户域失败

期望：

```text
1. 整体结果为 failed。
2. 写审计 / 告警。
3. 不修改交易允许状态。
4. 不阻断交易。
```

### 21.4 006 交易入口仍只同步 active domain

期望：

```text
1. active domain = USDS-M 时，006 交易入口只同步 U 本位。
2. active domain = COIN-M 时，006 交易入口只同步币本位。
3. 006 交易入口不会因为 012 新增 overview sync 而默认同步全部账户域。
```

### 21.5 012 多域快照不污染交易链路

期望：

```text
1. 012 同步 U 本位和币本位后，两个账户域快照都可展示。
2. OrderPlan 仍只读取 active domain。
3. RiskCheck 仍只读取 active domain。
4. OrderSubmission 仍只提交 active domain 订单。
5. 非 active domain 不参与交易决策。
```

### 21.6 读取最新账户总览

期望：

```text
1. 按账户域读取最新成功快照。
2. 展示每个账户域的同步时间。
3. 展示每个账户域的同步来源。
4. 展示每个账户域的同步状态。
5. 不要求所有账户域同一时间更新。
```

### 21.7 006 跑出快照后，012 可展示

期望：

```text
1. 006 正常交易流程同步成功后产生账户 / 持仓快照。
2. 012 最新账户总览可以读取该快照。
3. 012 展示该快照时不继承 006 的阻断语义。
4. 012 不把该快照解释为订单对账结果。
```

### 21.8 原始账户值展示

期望：

```text
1. 分账户显示 Binance 原始余额和原始持仓。
2. U 本位显示 USDT / USDC 等原始余额。
3. 币本位显示 BTC / ETH 等原始余额。
4. 不计算 estimated_total_value_usdt。
5. 不保存 valuation 字段。
6. 不用估值覆盖原始数值。
```

### 21.9 不维护 current 状态表

期望：

```text
1. 012 不新增 CurrentAccountState。
2. 012 不新增 CurrentPosition。
3. UI 当前账户状态来自最新成功快照。
```

### 21.10 不做 WebSocket

期望：

```text
1. 不创建 listenKey。
2. 不维护 keepalive。
3. 不建立 WebSocket 连接。
4. 不处理 ACCOUNT_UPDATE。
5. 不做 UI 实时推送。
```

### 21.11 不做订单对账

期望：

```text
1. 不读取 009 / 010 / 011 进行对账。
2. 不判断某笔订单是否完成。
3. 不释放订单锁。
4. 不更新订单状态。
```

### 21.12 012 不阻断交易

期望：

```text
1. 012 同步失败只记录失败 / 告警。
2. 012 不修改交易允许状态。
3. 012 不阻断 OrderPlan。
4. 012 不阻断 RiskCheck。
5. 012 不阻断 ExecutionPreparation。
6. 012 不阻断 OrderSubmission。
```

### 21.13 敏感信息过滤

期望：

```text
1. 不落库 API secret。
2. 不落库 signature。
3. 不落库完整认证 header。
```

---

## 22. 与后续阶段关系

012 完成后，后续可以继续：

```text
Account Realtime Stream：
  使用 Binance User Data Stream 实时接收账户 / 持仓变化，用于 UI 实时渲染和实时风险监控。

UI / Admin Account Overview：
  基于 012 的最新账户总览读取能力，提供页面展示和一键同步按钮。
  如需账户总估值，可在 UI 层基于最新快照和实时价格临时计算，不写回 012 快照。

Runtime Guard：
  后续如需运行保护，应独立实现，不由 012 直接阻断交易。
```

012 不得提前实现这些内容。

---

## 23. 最终结论

012 的职责是：

```text
基于 006 已有 Binance 账户同步能力，
新增账户总览的一键多账户域同步入口；
一次性同步全部支持账户域；
综合账户中展示 U 本位和币本位的 Binance 原始账户 / 持仓数据；
允许部分成功；
不做账户估值；
不做订单对账；
不绑定 009；
不阻断交易；
不维护 current 状态表；
不做 WebSocket；
不新建第二套账户模块；
不改变 006 交易入口只同步 active domain 的语义；
不允许非 active domain 快照参与自动交易决策。
```

一句话：

```text
006 是交易链路里的 active domain 账户快照 gate；
012 是账户总览里的全部账户域一键同步和查看工具。
```
