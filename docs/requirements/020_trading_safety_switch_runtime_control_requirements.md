# 020 Trading Safety Switch / Runtime Control Requirements

> 建议路径：`docs/requirements/trading_safety_switch.md`  
> 阶段：020  
> 模块建议：`apps.trading_safety_switch` 或 `apps.runtime_control`  
> 状态：需求定稿草案  
> 上游依赖：001 Alert / 006 Account Sync / 007 OrderPlan & RiskCheck / 008 ExecutionPreparation / 009 OrderSubmission / 010 OrderStatusSync / 011 FillSync / 012 Account Overview / 013 Pipeline Orchestrator / 014 Runtime Guard / 017 Web Console  
> 核心结论：020 不修改 `.env`，不管理 API key，不热切换市场域，不热切换真实交易模式；020 只负责展示当前生效硬配置，并提供运行时安全开关和审计。

---

## 1. 阶段定位

020 的正式定位是：

```text
Trading Safety Switch / Runtime Control
```

020 是交易系统的运行时安全控制模块。

020 不是完整配置中心。

020 不是 `.env` 编辑器。

020 不是 API key 管理器。

020 不是交易策略参数管理模块。

020 只负责：

```text
读取并展示当前硬配置；
提供运行时安全开关；
限制自动交易行为；
限制人工补查行为；
记录配置变更审计；
为 013 / 017 / 014 等模块提供统一安全状态。
```

---

## 2. `.env` 是硬配置

以下配置属于部署级硬配置，继续放在 `.env` 中：

```text
Binance API key；
Binance secret；
Binance base url；
真实交易 / 测试交易硬开关；
active market domain；
默认交易市场；
交易所账户域；
硬性最大下单金额；
杠杆配置；
数据库配置；
Redis / Celery 配置；
系统基础环境配置。
```

020 不允许修改 `.env`。

017 后台不允许修改 `.env`。

任何前端页面都不允许写 `.env` 文件。

---

## 3. 020 对 `.env` 的权限

020 对 `.env` 只能做：

```text
读取；
解析；
展示脱敏后的当前生效配置；
提供配置健康检查；
提示需要重启服务才能生效的硬配置项。
```

020 不能做：

```text
修改 `.env`；
写入 `.env`；
通过数据库覆盖 `.env` 的高风险配置；
通过前端热切换 `.env` 中定义的交易模式；
通过前端热切换 `.env` 中定义的 active market domain。
```

---

## 4. API key / secret 处理

020 不管理 API key。

020 不展示完整 API key。

020 不展示 API secret。

020 不导出 API key。

020 不导出 API secret。

020 最多展示脱敏信息，例如：

```text
API key configured: true
API key prefix: abcd****
API secret configured: true
```

020 不允许通过 017 页面新增、修改、删除 API key / secret。

---

## 5. active market domain 规则

active market domain 属于硬配置。

例如：

```text
USDS-M
COIN-M
```

020 不允许后台热切换 active market domain。

如果需要从 U 本位切换到币本位，必须：

```text
修改 `.env`；
重启服务；
重新检查配置；
确认 006 / 007 / 008 / 009 / 010 / 011 均使用新 domain；
再允许自动交易。
```

017 页面只能展示当前 active market domain。

017 页面不能提供“切换 U 本位 / 币本位”的运行时按钮。

---

## 6. 真实交易模式规则

真实交易 / 测试交易属于硬配置。

如果 `.env` 中真实交易关闭，020 不能在后台打开真实交易。

如果 `.env` 中只允许 testnet / dry-run，017 页面不能绕过该限制。

020 的运行时开关只能进一步收紧权限，不能突破 `.env` 限制。

示例：

```text
.env 禁止真实交易：
  020 不能打开真实交易。

.env 允许真实交易：
  020 可以临时暂停真实交易动作。

.env 单轮最大下单金额为 100 USDT：
  020 可以临时降到 50 USDT；
  020 不能提高到 1000 USDT。
```

---

## 7. 运行时安全开关

020 可以提供以下运行时安全开关：

```text
暂停自动交易；
恢复自动交易；
进入只读观察模式；
退出只读观察模式；
允许 / 禁止 013 自动运行交易周期；
允许 / 禁止 009 真实订单提交；
允许 / 禁止人工补查 010；
允许 / 禁止人工补查 011；
允许 / 禁止手动触发 012；
设置运行时单轮最大下单金额上限；
设置运行时人工操作保护状态。
```

这些开关应存储在数据库中，而不是写入 `.env`。

---

## 8. 运行时开关只能收紧权限

020 的运行时开关只能比 `.env` 更保守。

020 不允许通过数据库配置突破 `.env` 的硬限制。

规则：

```text
runtime_permission = env_permission AND runtime_switch
```

也就是说：

```text
硬配置不允许，运行时永远不能允许；
硬配置允许，运行时可以临时禁止。
```

---

## 9. 自动交易总开关

020 必须提供自动交易总开关。

自动交易总开关用于控制：

```text
013 是否允许自动启动交易周期；
013 是否允许继续进入下单链路；
009 是否允许真实提交订单。
```

当自动交易总开关关闭时：

```text
013 可以记录 skipped / blocked；
013 不得继续真实下单；
017 应显示当前系统为暂停自动交易状态；
014 可巡检长期关闭状态但不强制恢复。
```

---

## 10. 只读观察模式

020 必须支持只读观察模式。

只读观察模式含义：

```text
系统可以读取数据；
系统可以生成快照；
系统可以计算信号；
系统可以记录 TradingCycle；
系统不得真实下单。
```

只读观察模式适合：

```text
上线前观察；
异常后暂停交易；
人工排查；
真实交易关闭但继续观察策略结果。
```

---

## 11. 真实订单提交保护

020 必须为 009 OrderSubmission 提供统一检查。

009 下单前必须检查：

```text
.env 是否允许真实交易；
020 自动交易是否开启；
020 是否处于只读观察模式；
020 是否允许 009 真实订单提交；
运行时单轮最大下单金额是否足够；
当前 active market domain 是否与 006 / 008 / 009 一致。
```

如果检查不通过，009 不得提交 Binance。

---

## 12. 人工补查保护

020 必须为 010 / 011 人工补查提供开关。

017 手动补查 010 前必须检查：

```text
020 是否允许人工补查 010。
```

017 手动补查 011 前必须检查：

```text
020 是否允许人工补查 011。
```

如果不允许，后端 service 必须拒绝执行。

不能只靠前端隐藏按钮。

---

## 13. 012 手动同步保护

020 必须为 012 账户总览一键同步提供开关。

017 手动触发 012 前必须检查：

```text
020 是否允许手动触发 012。
```

如果不允许，后端 service 必须拒绝执行。

012 仍然不参与交易主链路收益计算。

---

## 14. 单轮最大下单金额

020 可以提供运行时单轮最大下单金额上限。

该上限只能小于或等于 `.env` 中的硬上限。

如果 `.env` 硬上限为 100 USDT，则：

```text
020 允许运行时设置 50 USDT；
020 不允许运行时设置 200 USDT。
```

007 / 008 / 009 在计算或提交订单前，应使用最终生效上限。

最终生效上限应为：

```text
min(env_hard_order_limit, runtime_order_limit)
```

---

## 15. 当前系统安全状态

020 应提供一个统一的当前系统安全状态。

建议包含：

```text
env_allows_real_trading
runtime_auto_trading_enabled
runtime_read_only_mode
runtime_order_submission_enabled
effective_order_submission_allowed
active_market_domain
effective_single_cycle_order_limit
manual_order_status_sync_enabled
manual_fill_sync_enabled
manual_account_overview_sync_enabled
last_changed_at_utc
last_changed_by
```

017 Dashboard 应显示该安全状态。

013 / 009 等后端模块应读取该安全状态。

---

## 16. 配置变更审计

020 的所有运行时开关变更必须写审计。

审计至少记录：

```text
操作人；
操作时间；
操作来源；
配置项；
修改前值；
修改后值；
修改原因；
trace_id；
是否成功；
失败原因。
```

运行时安全配置审计不得被普通页面删除。

---

## 17. 危险操作二次确认

017 修改 020 开关时，以下操作必须二次确认：

```text
恢复自动交易；
退出只读观察模式；
允许 009 真实订单提交；
允许 010 人工补查；
允许 011 人工补查；
允许 012 手动同步；
提高运行时单轮最大下单金额；
关闭人工保护状态。
```

二次确认必须说明：

```text
当前 `.env` 硬配置；
当前运行时配置；
修改后最终生效状态；
是否可能导致真实下单；
是否可能调用 Binance；
是否可能改变系统交易行为。
```

---

## 18. 权限要求

020 配置页面和 API 必须有权限控制。

至少区分：

```text
只读查看权限；
运行时开关管理权限；
交易恢复权限；
超级管理员权限。
```

只读用户只能查看当前配置。

运维用户可以暂停交易，但不一定能恢复交易。

恢复自动交易应要求更高权限。

---

## 19. 后端强校验

020 的安全限制必须在后端生效。

不能只依赖 017 前端隐藏按钮。

以下模块调用前必须做后端校验：

```text
013 自动交易周期启动；
009 真实订单提交；
010 人工订单状态补查；
011 人工成交补查；
012 手动账户总览同步；
007C 人工锁处理；
017 策略启用 / 禁用操作。
```

---

## 20. 与 013 的关系

013 启动交易周期前必须读取 020 当前安全状态。

如果自动交易关闭，013 应记录本轮 skipped / blocked，并写 Alert。

如果只读观察模式开启，013 可以继续生成信号和计划，但不得进入真实下单。

013 不得绕过 020。

---

## 21. 与 009 的关系

009 提交 Binance 前必须读取 020 当前安全状态。

如果不允许真实提交，009 应返回 blocked，而不是提交订单。

009 blocked 需要记录原因，例如：

```text
runtime_auto_trading_disabled
runtime_read_only_mode
order_submission_disabled
env_real_trading_disabled
order_limit_exceeded
```

---

## 22. 与 010 / 011 的关系

010 / 011 自动同步和人工补查需要区分。

如果是 013 自动链路中的 010 / 011，可以按交易链路规则运行。

如果是 017 人工触发补查，必须检查 020 的人工补查开关。

020 不修改 010 / 011 的业务结果。

020 只决定是否允许执行补查动作。

---

## 23. 与 012 的关系

012 是账户总览同步模块。

017 触发 012 时必须检查 020 是否允许手动触发 012。

020 不改变 012 的账户同步逻辑。

020 不把 012 变成交易主链路 gate。

---

## 24. 与 014 的关系

014 可以巡检 020 配置异常。

例如：

```text
.env 允许真实交易，但 runtime 处于长期开启状态且无人工确认；
active market domain 缺失；
运行时开关状态不一致；
自动交易关闭超过很久；
order_submission_enabled 开启但 auto_trading_enabled 关闭；
配置最近被修改但没有审计记录。
```

014 发现问题只写 RuntimeGuard 巡检异常 Alert。

014 不自动修改 020 配置。

---

## 25. 与 017 的关系

017 提供 020 的可视化管理页面。

017 可以展示：

```text
当前 `.env` 硬配置摘要；
当前运行时开关；
最终生效交易状态；
最近配置变更记录；
配置风险提示。
```

017 可以操作：

```text
暂停 / 恢复自动交易；
进入 / 退出只读观察模式；
允许 / 禁止人工补查；
允许 / 禁止手动 012 同步；
降低运行时单轮最大下单金额。
```

017 不能操作：

```text
修改 `.env`；
修改 API key；
修改 API secret；
热切换 active market domain；
突破 `.env` 硬上限；
热切换真实交易模式。
```

---

## 26. Alert 要求

020 发生关键运行时配置变更时，应写 AlertEvent。

至少包括：

```text
auto_trading_paused
auto_trading_resumed
read_only_mode_enabled
read_only_mode_disabled
order_submission_disabled
order_submission_enabled
manual_order_status_sync_disabled
manual_order_status_sync_enabled
manual_fill_sync_disabled
manual_fill_sync_enabled
runtime_order_limit_changed
```

Alert 中应包含：

```text
操作人；
修改前值；
修改后值；
修改原因；
是否影响真实交易；
trace_id。
```

---

## 27. P0 不做范围

020 P0 不做：

```text
修改 `.env`；
API key 管理；
secret 管理；
active market domain 热切换；
真实交易模式热切换；
复杂配置中心；
多环境配置发布系统；
灰度发布；
多用户审批流；
自动恢复交易；
自动纠正异常配置。
```

---

## 28. 建议模型

020 可以有以下模型：

```text
RuntimeTradingControl
RuntimeTradingControlAudit
```

RuntimeTradingControl 保存当前运行时开关。

RuntimeTradingControlAudit 保存每次变更审计。

实际模型名可在 plan 中根据项目命名调整。

---

## 29. 建议字段

RuntimeTradingControl 建议包含：

```text
auto_trading_enabled
read_only_mode
order_submission_enabled
manual_order_status_sync_enabled
manual_fill_sync_enabled
manual_account_overview_sync_enabled
runtime_single_cycle_order_limit
protection_reason
updated_by
updated_at_utc
trace_id
```

RuntimeTradingControlAudit 建议包含：

```text
control_key
old_value
new_value
changed_by
changed_at_utc
change_reason
source
trace_id
success
error_code
error_message
```

需求层面要求业务含义，不强制最终字段名完全一致。

---

## 30. 测试要求

020 至少覆盖以下测试：

```text
1. 020 不允许修改 `.env`。
2. 020 不返回 API secret。
3. `.env` 禁止真实交易时，020 不能打开真实交易。
4. `.env` 单轮最大下单金额为 100 时，020 不能设置运行时上限为 200。
5. 020 可以把运行时上限从 100 降到 50。
6. auto_trading_enabled=false 时，013 不得进入真实下单链路。
7. read_only_mode=true 时，009 不得提交 Binance。
8. order_submission_enabled=false 时，009 不得提交 Binance。
9. manual_order_status_sync_enabled=false 时，017 不能触发 010 人工补查。
10. manual_fill_sync_enabled=false 时，017 不能触发 011 人工补查。
11. manual_account_overview_sync_enabled=false 时，017 不能触发 012 手动同步。
12. 运行时配置变更必须写审计。
13. 关键配置变更必须写 AlertEvent。
14. 只读用户不能修改运行时开关。
15. 恢复自动交易需要高权限和二次确认。
```

---

## 31. 最终结论

020 的最终定位是：

```text
运行时交易安全开关。
```

020 只做：

```text
展示硬配置；
管理运行时安全开关；
限制交易动作；
限制人工补查；
写配置审计；
写关键配置变更 Alert。
```

020 不做：

```text
修改 `.env`；
修改 API key；
修改 API secret；
热切换市场域；
热切换真实交易模式；
突破硬配置限制。
```

一句话：

```text
.env 是硬配置，020 只能看；020 能改的只是更保守的运行时安全开关。
```
