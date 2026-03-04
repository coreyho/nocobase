# Java Action Registry 设计评估报告

> 评估说明：当前仓库中未检索到 `design/java-action-registry` 分支或 `design-docs/` 目录内容，故本次先基于现有设计文档 `docs/java-action-registry-design.md` 进行结构化评估，供你对“其他人设计”做评审时直接复用评估框架。

## 0. 评估输入与检索结果

本次按你的要求优先评估 `design-docs` 路径下“其他人设计”。

检索结果：当前仓库快照中未发现 `design-docs/` 目录，也不存在 `design/java-action-registry` 分支内容。为避免阻塞，本报告改为评估当前可见设计文档，并输出可复用评审框架。

已执行的检索（摘要）：
- `find . -type d -name 'design-docs' -o -path './design-docs'` -> 无结果
- `find . -maxdepth 4 -type d | rg 'design|action-registry|java-action'` -> 未发现目标路径
- `git log --all --name-only --pretty=format: | rg 'design-docs|java-action-registry'` -> 仅发现当前两份文档

## 1. 总体结论

该设计已经覆盖了从 **元数据规范**、**注册/查询/调用接口**、**Java SDK 抽象** 到 **LLM Skill 适配**、**热部署** 的完整链路，文档完整性较高，作为架构蓝图可落地。当前主要短板不在“有没有设计”，而在“可执行细节深度与边界条件”：

- **优势**：架构分层清晰、接口统一、治理面齐全。
- **风险**：跨协议一致性、Schema 演进、多租户隔离与热部署一致性尚缺可直接执行的工程约束。
- **建议**：把“概念型设计”推进到“实现级规范”（IDL、错误码字典、兼容矩阵、灰度回滚 SOP、基准测试指标）。

## 2. 分维度评分（10 分制）

| 维度 | 评分 | 评估要点 |
|---|---:|---|
| 架构清晰度 | 9 | 控制面/数据面/网关职责边界清晰。 |
| 接口规范性 | 8 | 已有统一调用 Envelope，但缺少强约束版本协商规则。 |
| 元数据完备性 | 8 | Schema 字段丰富，仍需补充“兼容等级”与“弃用策略字段”。 |
| 工程可实施性 | 7 | SDK 组件定义明确，但缺少线程模型、缓存一致性与并发约束细节。 |
| 治理与可观测 | 8 | 已覆盖审计、指标、追踪；需量化 SLO 与告警阈值。 |
| LLM 友好度 | 8 | Skill Adapter 思路正确；需进一步约束 prompt/tool 调用安全策略。 |
| 热部署可行性 | 7 | 方案完整，但未定义失败回滚自动化判定条件。 |

**综合评分：7.9 / 10**

## 3. 亮点（值得保留）

1. **微内核式能力抽象正确**：将 Action 看作受治理的“能力单元”，可支持插件化扩展。
2. **ActionMetadata 作为统一契约**：有助于跨语言、跨服务发现与调用一致性。
3. **调用 Envelope 标准化**：便于链路追踪、统一错误处理与平台治理。
4. **LLM Skill Adapter 补位及时**：已考虑模型消费能力发现与参数约束。
5. **热部署引入 DRAINING 状态**：体现了“在途请求不丢失”的生产意识。

## 4. 关键问题与风险

### 4.1 版本协商仍偏原则化

当前只描述“默认路由到 latest active”，但未定义：
- Minor/Patch 的自动升级边界；
- 客户端 pinned 版本的失效策略；
- 版本冲突时的 deterministic 决策顺序。

**风险**：不同 SDK 实现可能行为不一致，导致线上调用漂移。

### 4.2 Schema 演进缺少机器可判定规则

虽然提到向后兼容，但缺少“兼容类型”字段（如 `backward|forward|breaking`）和自动校验流程。

**风险**：发布阶段无法阻断破坏性变更，回归问题后置到运行时。

### 4.3 错误码体系尚未产品化

文档给了示例错误码，但没有完整错误码字典（模块归属、可重试性、HTTP/gRPC 映射）。

**风险**：调用方很难稳定做重试/降级策略。

### 4.4 多租户与授权策略需更细

目前以原则描述为主，未落到：
- tenant 级隔离字段是否参与唯一索引；
- action 级 scope 与用户级 scope 的合并规则；
- 代理调用（service-to-service）的授权链路。

**风险**：可能出现“跨租户发现/调用泄漏”。

### 4.5 热部署回滚触发条件缺失

已有流程但无自动化阈值：
- 错误率阈值、P99 延迟阈值；
- 灰度分桶策略和停止条件；
- 回滚后是否冻结发布窗口。

**风险**：热部署失败后恢复速度依赖人工经验。

## 5. 优先级改进建议（按落地价值排序）

### P0（上线前必须）

1. **补齐协议与错误码规范**
   - 输出 `error-codes.md`：错误码、可重试、建议动作、HTTP/gRPC 对照。
2. **定义版本协商算法**
   - 明确路由优先级：`pinned > canary > latest-active`。
3. **建立 schema gate**
   - CI 增加 metadata 兼容性校验，阻断 breaking 变更。

### P1（首个版本后）

1. **热部署自动回滚策略**
   - 定义阈值与窗口（例如 5 分钟错误率 > 2% 自动回滚）。
2. **多租户授权模型落地**
   - 引入 `tenant_scope`、`visibility`、`required_scopes` 的强校验。

### P2（规模化阶段）

1. **LLM 调用安全策略中心化**
   - 增加 tool 调用防注入规则与敏感 action 人审开关。
2. **跨区域容灾与目录一致性**
   - Registry 多活 + 最终一致性策略。

## 6. 建议补充的交付物清单

为从“设计文档”走向“可实施规范”，建议新增以下文档/资产：

- `api/action-registry-openapi.yaml`
- `api/action-invoke.proto`（或等价 gRPC IDL）
- `spec/error-codes.md`
- `spec/version-negotiation.md`
- `spec/metadata-compatibility-matrix.md`
- `runbook/hot-deploy-and-rollback.md`
- `security/llm-tool-policy.md`

## 7. 可执行下一步（2 周计划）

- 第 1 周：完成 OpenAPI + 错误码字典 + 版本协商规则，并在 SDK 中落一个 reference 实现。
- 第 2 周：完成 metadata 兼容性 CI 检查 + 热部署灰度回滚脚本草案。

---

如果你把“其他人的设计文档”路径给到我（或补充到当前仓库），我可以按同一框架输出 **逐章节逐条评审意见**，并给出“可合并/需修改/阻塞项”清单。
