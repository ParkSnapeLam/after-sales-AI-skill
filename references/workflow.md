# 流程与状态

## 路由与状态转换

状态名供讨论和实现时对照，已有系统可映射到自己的命名。状态转换由会话编排代码管理，不以模型自述作为执行凭据。

| 当前状态 | 条件或事件 | 动作 | 下一状态 |
| --- | --- | --- | --- |
| `intent` | 明确非退货 | 交回既有客服 | `existing_support` |
| `intent` | 退货意图不确定 | 一次清晰澄清，保持零业务写入 | `intent` |
| `intent` | 明确退货 | 开始订单查询 | `order_lookup` |
| `order_lookup` | 无匹配 | 请求订单号或邮箱等必要信息 | `awaiting_order_info` |
| `awaiting_order_info` | 顾客补充信息 | 重新查询，执行归属验证 | `order_lookup` |
| `order_lookup` | 多笔匹配 | 仅展示经授权的最小信息，请顾客选择 | `awaiting_order_choice` |
| `awaiting_order_choice` | 顾客选定 | 重新确认订单、归属与目标商品 | `order_lookup` |
| `order_lookup` | 唯一且归属已验证 | 保存确定订单及商品 | `reason` |
| `order_lookup` | 查询异常或归属无法验证 | 不泄露订单详情，记录原因并转交 | `human_handoff` |
| `reason` | 原因仍不清晰 | 询问必要问题；识别尺码、质量、不会用、不喜欢或其他 | `reason` |
| `reason` | 原因足够计算方案 | 读取真实政策及业务数据 | `policy_evaluation` |
| `policy_evaluation` | 计算成功，白名单为空 | 进入既有正常退货流程 | `normal_return` |
| `policy_evaluation` | 数据缺失、过期或计算失败 | 记录故障并转交，不伪装成无资格 | `human_handoff` |
| `policy_evaluation` | 有可用方案 | 保存政策、订单及方案快照 | `offer_generation` |
| `offer_generation` | 模型返回草稿 | 解析 JSON，并执行确定性业务校验 | `offer_validation` |
| `offer_validation` | 任何校验失败 | 发送经过批准的兜底模板、转人工、记录日志 | `human_handoff` |
| `offer_validation` | 校验通过 | 发送已校验文案，保存实际展示的方案及条款 | `awaiting_decision` |
| `awaiting_decision` | 明确拒绝，或要求直接退货 | 停止挽留，进入既有正常退货流程 | `normal_return` |
| `awaiting_decision` | 接受含糊、无法关联到展示记录 | 澄清目标方案和条款，不执行 | `awaiting_decision` |
| `awaiting_decision` | 首次要求再谈 | `renegotiation_count` 从 0 加到 1，刷新仍可用的方案 | `policy_evaluation` |
| `awaiting_decision` | 再谈额度已用完 | 默认进入既有正常退货流程 | `normal_return` |
| `awaiting_decision` | 明确接受已展示的具体方案 | 固化接受证据；开始执行前再验证 | `pre_execution` |
| `pre_execution` | 订单、库存、政策或条款发生实质变化 | 重新计算与展示，旧的接受不授权新条款 | `policy_evaluation` |
| `pre_execution` | 接受证据有效，条款仍有效 | 持久化执行记录和幂等键，再请求业务接口 | `execution_pending` |
| `execution_pending` | 外部结果明确成功 | 保存结果凭据，按真实完成程度通知顾客 | `execution_confirmed` |
| `execution_pending` | 明确失败 | 通知未完成，记录原因并转人工 | `human_handoff` |
| `execution_pending` | 超时或状态未知 | 保存未知状态，查询原操作，不重新创建业务操作 | `execution_unknown` |
| `execution_unknown` | 查询确认成功 | 保存结果凭据，去重通知和统计 | `execution_confirmed` |
| `execution_unknown` | 查询确认失败 | 依据接口约定恢复或人工处理 | `human_handoff` |
| `execution_unknown` | 仍未知 | 有界查询后转人工对账，继续保留未知执行记录 | `human_handoff` |

`normal_return` 表示进入正常退货流程，不表示退款已批准或已经到账。`human_handoff` 表示自动处理停止，不表示外部未知操作已撤销。两条路径必须携带上下文并记录转交结果。

同一会话出现并发事件时：已发起而结果未知的业务操作先对账，暂停新增权益操作；缺少旧展示或接受证据时记录缺口，不能用后来补取的同意追认旧操作。顾客明确拒绝或要求退货时停止挽留并保留请求；若同时存在未决操作，将其上下文交给正常退货或人工处理，不能丢失该请求，也不能重复发放。政策查询故障仍单独记为故障，不把拒绝路由的结果解释成有效空名单。

## 确定性与模型边界

模型可提取意图、原因，选择允许方案并解释；模型不能自行确定政策有效性、退款资格、库存、金额、比例、接受证据或执行成功。

方案引擎以真实政策、商品库存、订单状态、退货期限和金额上限为输入，输出成功状态与白名单；`offers: []` 仅在计算成功时具有“无可用方案”的含义。错误状态不许经转换变成空名单。

回复中列出多个候选时，先由顾客选择并把选择绑定为一个具体 `offer_id`。示例的单方案回复不授权同时执行其他候选方案。

## 设计默认与接入配置

以下为实现默认，不能冒充商家政策：

- 首次展示后最多再谈一次。重新计算、模型重生成、渠道重发不会把计数清零；顾客拒绝优先于剩余轮次。
- 查找订单、澄清原因及外部查询都应设有限的尝试次数或时限，耗尽后转人工；具体值由接入方配置，不使用无限循环。
- 没有真实政策、权限或执行状态时转人工；未获授权不补发权益。缺少接口时可完成设计和模拟，但不能声称已经处理订单。
- 执行前若需要重新展示条款，不自动恢复再谈额度；不可持续变换方案阻碍正常退货。
- 质量或其他特殊原因是否允许某类挽留由真实政策决定，不能从示例推导通用规则。

## 结果出口

每条出口都记录路由与原因。非退货只记录必要的转交信息；正常退货、人工接管、执行失败和未知状态保留关联标识，以便后续查询和对账。

换码的“申请受理”与“完成换码”、余额的“创建请求”与“已确认入账”、教程的“发送成功”与“问题解决”分别记录。通知内容使用接口确认的实际阶段。

状态机不能通过“顾客点击接受”直接跳到“最终解决”；只有接入方定义的完成证据才可将结果认定为已解决。
