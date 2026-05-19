# Agent 开发交付包

当用户明确要求“开发可开工”或需要转工程 owner 时，在 PRD 后追加本交付包。该部分由工程主责，PM 只确认产品范围、风险规则和验收口径。

| 交付项 | 内容 | 责任方 |
|---|---|---|
| Runtime 选择 | SDK / API / 自研 runner / ADK / MCP / 其他 | 工程 |
| 代码模块拆分 | agent、tool、state、eval、api、ui、storage | 工程 |
| 机器可读 schema | JSON Schema / Pydantic / Zod | 工程 |
| Tool mocks | mock 输入、输出、错误样例 | 工程 / QA |
| 状态与事件 | session state、event schema、状态转移 | 工程 |
| 安全策略 | 权限、确认、注入防护、敏感信息处理 | 工程 / 安全 / PM |
| Eval 资产 | 测试集、阈值、回归规则 | PM / QA / 工程 |
| 部署条件 | env、secret、存储、队列、监控、rollback | 工程 |

开工前必须明确：首版用户场景、工具和数据源范围、高风险动作规则、输入输出字段、失败和降级规则。MVP 及以上还要明确评估样本和阈值；Production 需要补部署和监控。
