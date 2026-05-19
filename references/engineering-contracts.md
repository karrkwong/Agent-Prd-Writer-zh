# Agent 工程契约参考

本文件用于把 Agent PRD 从产品讨论稿补齐为工程可开工文档。写开发可落地 PRD 时必须使用。

## 1. Agent 运行契约

每个 Agent 必须写清以下字段：

| 字段 | 说明 | 示例 |
|---|---|---|
| 触发事件 | Agent 被唤起的事件。 | 用户点击、邮件进入、日程临近、上游 handoff。 |
| 输入 schema | 输入字段、类型、必填项、允许为空项。 | task_id、user_intent、source_context。 |
| 输出 schema | 输出字段、类型、来源、置信度和状态。 | summary、sources、missing_info、next_action。 |
| 可调用工具 | 允许调用的工具白名单。 | 邮箱读取、日历读取、文档检索。 |
| 工具调用上限 | 单次任务最多调用次数。 | 最多 5 次检索，最多 2 次重试。 |
| 超时策略 | 单工具和整体任务超时。 | 单工具 10 秒，整体任务 60 秒。 |
| 重试策略 | 可重试错误、重试次数、退避规则。 | 网络超时重试 2 次，权限错误不重试。 |
| 停止条件 | 任务成功、失败、取消、超时、信息不足。 | 输出完整结果或进入待确认。 |
| 需要用户确认的条件 | 哪些结果或动作需要用户确认。 | 发送、创建、共享、外部写入。 |
| 异常降级方式 | 异常时如何保持可用。 | 保留草稿、提示授权、改为手动上传。 |

## 2. 输入 / 输出 schema 写法

输入 schema 至少包含：

```json
{
  "task_id": "string",
  "user_id": "string",
  "user_intent": "string",
  "source_context": [
    {
      "source_id": "string",
      "source_type": "email | calendar | doc | file | transcript | task | other",
      "title": "string",
      "excerpt": "string",
      "permission_status": "authorized | unauthorized | partial",
      "confidence": "confirmed | tentative | insufficient"
    }
  ],
  "constraints": {
    "language": "zh-CN",
    "risk_level": "low | medium | high",
    "requires_user_confirmation": true
  }
}
```

输出 schema 至少包含：

```json
{
  "task_id": "string",
  "agent_name": "string",
  "status": "success | needs_user_confirmation | insufficient_info | failed",
  "result": {},
  "sources": [],
  "missing_info": [],
  "next_actions": [],
  "risk_level": "low | medium | high",
  "requires_user_confirmation": true,
  "error": null
}
```

## 3. 工具调用契约

每个工具必须写成可测试契约：

| 工具 / 数据源名称 | 真实工具或 mock 工具 | 用途 | 输入参数 | 输出字段 | 所需权限 | 是否写入外部系统 | 是否需要用户确认 | 失败类型 | 重试策略 | 是否幂等 | 是否支持回滚 | 日志记录要求 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|

字段说明：

- 真实工具或 mock 工具：标明当前阶段是否真实接入。
- 输入参数：写字段名、类型、必填项。
- 输出字段：写结构化返回字段。
- 所需权限：读取、写入、发送、共享、管理员配置等。
- 是否写入外部系统：只要改变外部状态就填“是”。
- 是否需要用户确认：高影响动作必须为“是”。
- 失败类型：权限不足、网络超时、参数错误、服务不可用、结果为空、部分成功。
- 重试策略：哪些失败可重试，最多几次。
- 是否幂等：重复调用是否产生重复结果。
- 是否支持回滚：执行后是否能撤销。
- 日志记录要求：记录请求 ID、参数摘要、结果摘要、耗时、错误码，不保存敏感原文。

## 4. Agent 交接 payload

所有 Agent 交接必须使用结构化 payload：

```json
{
  "task_id": "",
  "user_intent": "",
  "source_context": [],
  "current_agent": "",
  "next_agent": "",
  "handoff_reason": "",
  "required_tools": [],
  "risk_level": "",
  "needs_user_confirmation": true,
  "missing_info": [],
  "expected_output": ""
}
```

必填字段：task_id、user_intent、source_context、current_agent、next_agent、handoff_reason、required_tools、risk_level、needs_user_confirmation、missing_info、expected_output。

允许为空：

- source_context 可为空数组，但必须在 missing_info 中说明原因。
- required_tools 可为空数组，表示下游不需要工具。
- missing_info 可为空数组，表示当前无缺口。

来源不足时：

- risk_level 至少为 medium。
- needs_user_confirmation 设为 true。
- missing_info 写清缺少的来源、权限或用户判断。
- expected_output 不得要求下游输出确定性结论，只能要求输出草稿、候选项或待确认问题。

## 5. 状态转移规则

状态机不得只列状态，必须写清转移规则：

| 状态 | 进入条件 | 可进入下一状态 | 用户可执行动作 | 系统可执行动作 | 是否允许取消 | 是否允许重试 | 是否会产生外部写入 | 是否必须记录审计日志 |
|---|---|---|---|---|---|---|---|---|

要求：

- 进入条件必须可判断。
- 外部写入状态必须记录审计日志。
- 执行中状态不允许静默失败，必须进入已完成或失败。
- 失败状态必须说明可重试和不可重试条件。
- 用户取消后不得继续执行外部动作。

## 6. 记忆与数据策略

必须区分任务上下文、来源引用、长期偏好和敏感信息。

| 数据 / 记忆类型 | 是否可保存 | 保存期限 | 是否参与生成 | 用户控制 | 脱敏要求 | 阶段策略 |
|---|---|---|---|---|---|---|

规则：

- 可以记：用户明确授权的偏好、任务状态、来源引用、确认记录。
- 不能记：未经授权的邮件正文、会议敏感原文、个人隐私、商业机密、密码、密钥、未确认的长期偏好。
- 记忆保存期限必须按 Demo / MVP / Production 分开定义。
- 用户必须能查看、修改、删除长期记忆。
- 记忆参与生成时必须可追溯。
- 敏感信息必须脱敏或摘要化。
- Demo 阶段默认不做长期记忆。
- MVP 阶段只保存任务级状态和来源引用。
- Production 阶段才考虑用户授权下的长期偏好。

## 7. 日志、审计与可观测性

任务日志至少记录：

- task_id。
- user_id 或匿名用户标识。
- agent_name。
- 当前状态。
- 触发事件。
- 输入摘要。
- 输出摘要。
- 风险等级。
- 是否需要用户确认。
- 创建时间、更新时间、耗时。

工具调用日志至少记录：

- task_id。
- tool_name。
- call_id。
- 输入参数摘要。
- 输出摘要。
- 是否写入外部系统。
- 是否用户确认。
- 成功 / 失败。
- 错误码。
- 耗时。
- 重试次数。

用户确认日志至少记录：

- confirmation_id。
- task_id。
- 用户确认时间。
- 确认动作。
- 预览内容摘要。
- 最终执行内容摘要。
- 是否修改过草稿。

日志限制：

- 不保存邮件正文、会议原文、完整附件内容、密钥、密码、个人敏感信息。
- 必须对联系人、邮箱、手机号、客户名称、合同金额等敏感字段脱敏。
- 错误日志分级：info、warning、error、critical。
- 能通过 task_id 追踪一次错误建议或错误执行的完整链路。

## 8. 工程测试计划

开发前必须定义测试类型：

| 测试类型 | 测试目标 | 样本 / 方法 | 通过标准 |
|---|---|---|---|
| 单 Agent 单元测试 | 验证单个 Agent 的输入输出和异常处理。 | 结构化样本。 | 核心路径通过。 |
| 多 Agent 交接测试 | 验证 handoff payload 和状态继承。 | 多轮任务样本。 | 字段完整，交接正确。 |
| 工具调用测试 | 验证工具参数、返回值、超时和重试。 | mock + 真实工具。 | 调用符合契约。 |
| 权限与确认测试 | 验证高影响动作不能绕过用户确认。 | 权限矩阵。 | 误执行次数为 0。 |
| 失败恢复测试 | 验证权限不足、工具失败、来源不足时可降级。 | 失败注入。 | 用户可继续处理。 |
| 安全与隐私测试 | 验证敏感信息不外显、不被错误保存。 | 敏感样本。 | 不违反展示和存储规则。 |
| 回归测试 | 验证新版本不破坏已通过样本。 | 固定回归集。 | 通过率达到版本阈值。 |
| 端到端验收测试 | 验证完整用户主线。 | 真实或近真实任务。 | 主流程可完成。 |
