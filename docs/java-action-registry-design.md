# 基于 NocoBase 插件化思路的 Java Action 注册与调用机制设计

## 1. 背景与目标

参考 NocoBase 的核心思想：
- **微内核 + 可扩展能力**：核心只负责生命周期、依赖与编排，业务能力以插件（Action）扩展。
- **统一管理平面**：通过统一的管理器（类似 PluginManager）完成注册、启停、查询和调用。
- **声明式元数据 + 运行时加载**：能力通过 metadata 描述，运行时按标准协议装配。

在 Java 微服务体系中，目标是设计一套统一的 Action 机制，满足：
1. 支持自定义 Action 注册；
2. 支持微服务远程调用；
3. 提供统一接口：注册、查询、调用；
4. 定义标准 Action 元数据 schema；
5. 定义统一调用协议；
6. 可观测、可治理、可演进。

---

## 2. 总体架构

```text
+------------------------- Control Plane --------------------------+
|  Action Registry Service                                         |
|  - ActionMetadata 存储（DB）                                     |
|  - Schema 校验                                                   |
|  - 版本治理（draft/active/deprecated）                           |
|  - 查询 API / 管理 API                                           |
+--------------------------^-------------------^-------------------+
                           |                   |
                   register/update       discovery/query
                           |                   |
+--------------------------+-------------------+-------------------+
|                     Service Runtime Plane                         |
|  Service A                Service B                Service C       |
|  +-------------------+    +-------------------+    +------------+ |
|  | Action SDK        |    | Action SDK        |    | Action SDK | |
|  | - 注解扫描        |    | - 手工注册        |    | - 远程代理 | |
|  | - 本地执行器      |    | - 本地执行器      |    |            | |
|  | - 调用客户端      |    | - 调用客户端      |    |            | |
|  +-------------------+    +-------------------+    +------------+ |
+--------------------------+-------------------+-------------------+
                           |
                           v
                 +--------------------------+
                 | Action Gateway (可选)    |
                 | - 鉴权、限流、路由       |
                 | - 灰度、熔断、审计       |
                 +--------------------------+
```

### 2.1 角色说明

- **Action Registry Service（控制面）**
  - 存储 Action 元数据、校验 schema、管理生命周期状态。
  - 负责“全局目录”和“能力发现”。

- **Action SDK（数据面）**
  - 每个业务服务内嵌 SDK：用于注册本服务 Action、处理调用请求、发起远程调用。
  - 对开发者暴露简单注解/接口。

- **Action Gateway（可选）**
  - 当系统规模大时，统一入口处理鉴权、流控、审计和多租户隔离。

---

## 3. 核心概念模型

## 3.1 Action 标识

建议统一标识：

```text
{domain}.{boundedContext}.{actionName}:{version}
```

示例：
- `crm.customer.create:1.0.0`
- `order.fulfillment.allocateStock:2.1.0`

字段拆分：
- `namespace`：`crm.customer`
- `name`：`create`
- `version`：SemVer
- `fullName`：`crm.customer.create`
- `key`：`crm.customer.create:1.0.0`

## 3.2 Action 生命周期状态

- `DRAFT`：草稿，未可用；
- `ACTIVE`：可调用；
- `DISABLED`：停用；
- `DEPRECATED`：不建议新接入，但可兼容调用；
- `REMOVED`：逻辑下线，仅保留历史记录。

## 3.3 Action 类型

- `SYNC`：请求-响应；
- `ASYNC`：异步投递（MQ/事件总线）；
- `STREAM`：流式响应（可选，后续演进）。

---

## 4. 标准 Metadata Schema（Action 规范）

采用 JSON Schema（2020-12）定义元数据，便于跨语言兼容。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/schema/action-metadata.v1.json",
  "title": "ActionMetadata",
  "type": "object",
  "required": [
    "apiVersion", "kind", "metadata", "spec"
  ],
  "properties": {
    "apiVersion": { "type": "string", "const": "action.example.io/v1" },
    "kind": { "type": "string", "const": "Action" },
    "metadata": {
      "type": "object",
      "required": ["name", "namespace", "version"],
      "properties": {
        "name": { "type": "string", "pattern": "^[a-zA-Z][a-zA-Z0-9_]{2,63}$" },
        "namespace": { "type": "string", "pattern": "^[a-z][a-z0-9_.-]{1,127}$" },
        "version": { "type": "string", "pattern": "^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)(?:-[0-9A-Za-z.-]+)?(?:\\+[0-9A-Za-z.-]+)?$" },
        "labels": { "type": "object", "additionalProperties": { "type": "string" } },
        "owner": { "type": "string" },
        "description": { "type": "string", "maxLength": 2048 }
      }
    },
    "spec": {
      "type": "object",
      "required": ["type", "protocol", "endpoint", "inputSchema", "outputSchema", "timeoutMs"],
      "properties": {
        "type": { "type": "string", "enum": ["SYNC", "ASYNC", "STREAM"] },
        "protocol": { "type": "string", "enum": ["HTTP", "GRPC", "MQ"] },
        "endpoint": {
          "type": "object",
          "required": ["service", "path"],
          "properties": {
            "service": { "type": "string" },
            "path": { "type": "string" },
            "method": { "type": "string", "enum": ["GET", "POST", "PUT", "DELETE", "PATCH"] },
            "topic": { "type": "string" }
          }
        },
        "inputSchema": { "type": "object" },
        "outputSchema": { "type": "object" },
        "errorSchema": { "type": "object" },
        "timeoutMs": { "type": "integer", "minimum": 1, "maximum": 60000 },
        "retryPolicy": {
          "type": "object",
          "properties": {
            "maxAttempts": { "type": "integer", "minimum": 0, "maximum": 10 },
            "backoffMs": { "type": "integer", "minimum": 0, "maximum": 60000 },
            "retryableCodes": { "type": "array", "items": { "type": "string" } }
          }
        },
        "auth": {
          "type": "object",
          "properties": {
            "mode": { "type": "string", "enum": ["NONE", "JWT", "MTLS", "AKSK"] },
            "scopes": { "type": "array", "items": { "type": "string" } }
          }
        },
        "idempotency": {
          "type": "object",
          "properties": {
            "enabled": { "type": "boolean" },
            "keyHeader": { "type": "string" },
            "ttlSeconds": { "type": "integer", "minimum": 1, "maximum": 86400 }
          }
        }
      }
    },
    "status": {
      "type": "object",
      "properties": {
        "state": { "type": "string", "enum": ["DRAFT", "ACTIVE", "DISABLED", "DEPRECATED", "REMOVED"] },
        "lastUpdatedAt": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

---

## 5. 统一接口设计（注册 / 查询 / 调用）

## 5.1 Registry API（控制面）

基于 REST（管理 API），可补充 gRPC Admin API。

### 5.1.1 注册/更新
- `POST /api/actions`
- `PUT /api/actions/{namespace}/{name}/{version}`

请求体：`ActionMetadata`
响应：标准 Envelope

### 5.1.2 状态变更
- `POST /api/actions/{key}/activate`
- `POST /api/actions/{key}/disable`
- `POST /api/actions/{key}/deprecate`
- `DELETE /api/actions/{key}`（软删除为主）

### 5.1.3 查询
- `GET /api/actions/{key}`
- `GET /api/actions?namespace=crm.customer&state=ACTIVE&type=SYNC&label.env=prod`
- `GET /api/actions/{fullName}/versions`

## 5.2 Invoke API（数据面）

统一调用入口（推荐）：

- `POST /api/action-invoke/{fullName}`
- Header: `X-Action-Version: 1.0.0`（可选，不传默认路由到最新 ACTIVE）
- Header: `X-Request-Id`, `X-Tenant-Id`, `Authorization`, `Idempotency-Key`

请求体统一：

```json
{
  "context": {
    "tenantId": "t1",
    "userId": "u1001",
    "traceId": "7f6c...",
    "locale": "zh-CN"
  },
  "input": {
    "customerName": "Acme"
  },
  "options": {
    "timeoutMs": 3000,
    "dryRun": false
  }
}
```

响应体统一：

```json
{
  "code": "OK",
  "message": "success",
  "requestId": "req-123",
  "data": {
    "customerId": "c_001"
  },
  "error": null,
  "meta": {
    "actionKey": "crm.customer.create:1.0.0",
    "durationMs": 78,
    "provider": "crm-service"
  }
}
```

错误响应（统一结构）：

```json
{
  "code": "ACTION_TIMEOUT",
  "message": "invoke timeout",
  "requestId": "req-123",
  "data": null,
  "error": {
    "type": "TimeoutException",
    "retryable": true,
    "details": {}
  },
  "meta": {
    "actionKey": "crm.customer.create:1.0.0"
  }
}
```

---

## 6. Java SDK 设计

## 6.1 开发者接口（SPI + 注解）

```java
public interface ActionHandler<I, O> {
    O execute(ActionContext context, I input) throws Exception;
}

public record ActionContext(
    String requestId,
    String tenantId,
    String userId,
    String traceId,
    Map<String, Object> attributes
) {}
```

注解定义：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ActionDefinition {
    String namespace();
    String name();
    String version();
    ActionType type() default ActionType.SYNC;
    String description() default "";
}
```

示例：

```java
@ActionDefinition(namespace = "crm.customer", name = "create", version = "1.0.0")
public class CreateCustomerAction implements ActionHandler<CreateCustomerInput, CreateCustomerOutput> {
    @Override
    public CreateCustomerOutput execute(ActionContext context, CreateCustomerInput input) {
        // business logic
        return new CreateCustomerOutput("c_001");
    }
}
```

## 6.2 SDK 核心组件

- `ActionScanner`：扫描注解生成 metadata；
- `ActionRegistryClient`：向 Registry 服务注册/续租；
- `ActionCatalog`：本地缓存目录（带 TTL + watch）；
- `ActionInvoker`：统一调用入口（本地优先、远程兜底）；
- `ActionDispatcher`：接收 invoke 请求并分发到 handler；
- `ActionCodec`：请求/响应、错误码、schema 校验。

## 6.3 统一 Java 接口

```java
public interface ActionRegistry {
    void register(ActionMetadata metadata);
    Optional<ActionMetadata> findByKey(String actionKey);
    List<ActionMetadata> query(ActionQuery query);
    void activate(String actionKey);
    void disable(String actionKey);
}

public interface UnifiedActionInvoker {
    <I, O> ActionResponse<O> invoke(String fullName, String version, I input, InvokeOptions options);
}
```

---

## 7. 远程调用标准（微服务）

## 7.1 路由规则

1. 通过 `fullName + version` 查询 Registry；
2. 取 `state=ACTIVE` 且路由策略匹配实例（zone/tenant/tag）；
3. 使用 `protocol` 选择 HTTP/gRPC/MQ 客户端；
4. 透传标准上下文头：`traceId/requestId/tenantId/auth`。

## 7.2 协议建议

- **内部同步调用优先 gRPC**（强类型、性能、超时控制）。
- **外部开放或异构接入用 HTTP/JSON**。
- **异步流程（耗时任务）用 MQ**，返回 `accepted + taskId`。

## 7.3 弹性治理

- 超时：以调用方 `InvokeOptions.timeoutMs` 与 metadata `timeoutMs` 取最小值。
- 重试：仅对幂等 Action 生效；遵循 `retryPolicy`。
- 熔断：按 `actionKey` 粒度熔断。
- 限流：按 `tenant + actionKey` 限流。
- 降级：允许配置 fallback action。

---

## 8. 数据存储设计（Registry）

建议表结构（简化）：

### 8.1 `action_definition`
- `id`
- `namespace`
- `name`
- `version`
- `full_name`
- `state`
- `protocol`
- `endpoint_json`
- `input_schema_json`
- `output_schema_json`
- `retry_policy_json`
- `auth_json`
- `labels_json`
- `owner`
- `created_at`
- `updated_at`

唯一索引：`(namespace, name, version)`

### 8.2 `action_audit_log`
- `action_key`
- `operator`
- `operation`（REGISTER/ACTIVATE/...）
- `before_json` / `after_json`
- `created_at`

---

## 9. 安全与多租户

- 鉴权：Registry 管理接口与 Invoke 接口分离权限；
- 鉴权模式：支持 JWT / mTLS / AKSK；
- 多租户：metadata 可带 `tenantScope`（global / tenant-specific）；
- 数据隔离：查询与调用均校验 tenant 边界；
- 审计：注册、状态变更、调用失败均可追踪。

---

## 10. 可观测性标准

- 指标（Prometheus）：
  - `action_invoke_total{actionKey,code}`
  - `action_invoke_latency_ms_bucket{actionKey}`
  - `action_registry_register_total{result}`
- 日志：结构化日志统一字段：`requestId`, `traceId`, `tenantId`, `actionKey`, `durationMs`, `code`。
- 链路追踪：OpenTelemetry span name 规范：`action.invoke/{fullName}`。

---

## 11. 版本与兼容策略

- 版本遵循 SemVer；
- 默认路由到最新 `ACTIVE` 小版本；
- 大版本变更需显式指定 `X-Action-Version`；
- `DEPRECATED` 阶段持续告警并给出迁移窗口；
- schema 变更需保证向后兼容（新增字段可选、避免删除必填）。

---

## 12. 典型时序

## 12.1 注册时序

1. 服务启动 -> SDK 扫描 Action；
2. 生成 metadata 并做本地 schema 校验；
3. 调用 Registry `POST /api/actions`；
4. Registry 落库并返回 actionKey；
5. Action 被 `activate` 后进入可调用态。

## 12.2 调用时序

1. 调用方发起 `invoke(fullName, version, input)`；
2. SDK 查询本地缓存目录，不命中则拉取 Registry；
3. 按 protocol 路由到目标服务；
4. 目标服务 Dispatcher 校验 input schema；
5. 执行 handler 并返回统一响应；
6. SDK 记录 metrics + trace + audit。

---

## 13. 最小可行落地（MVP）

### 阶段 1（2~3 周）
- 完成 metadata schema v1；
- 完成 Registry 基础 CRUD + activate/disable；
- 完成 Java SDK：注解扫描 + 注册 + HTTP 调用；
- 完成统一响应结构与基础错误码。

### 阶段 2（2~4 周）
- 增加 gRPC 协议适配；
- 增加重试、限流、熔断；
- 增加审计日志与调用统计看板。

### 阶段 3（持续演进）
- 异步 Action（MQ）
- 多租户策略增强
- Gateway 与策略编排（灰度/AB 测试）

---

## 14. 关键设计决策（总结）

- 借鉴 NocoBase 的“**核心管理器 + 生命周期 + 元数据驱动**”思想，将 Java Action 治理做成可插拔体系；
- 将“注册与发现”放在控制面，将“执行与调用”放在数据面；
- 用统一 metadata schema 和统一调用 envelope 解决跨服务、跨语言一致性；
- 通过版本、状态、审计和可观测能力，保证该机制在生产可用。

---

## 15. 面向大模型的能力暴露（参考 Claude Skills 定义）

为方便大模型“发现并正确调用 Action”，建议在 Registry 之上增加一个 **LLM Skill Adapter**，将 ActionMetadata 映射为模型可消费的 Skill 定义（类似 Claude Skills 的 `name + description + usage instructions`）。

### 15.1 目标

- 让模型先做“能力发现”，再做“参数补齐与调用”；
- 降低模型直接理解底层 RPC/HTTP 细节的成本；
- 将 Action 的治理能力（版本、权限、租户、限流）透出给模型编排层。

### 15.2 Skill 视图（由 Action 自动生成）

建议新增只读接口：

- `GET /api/llm/skills`
- `GET /api/llm/skills/{skillName}`

返回结构建议（示例）：

```json
{
  "name": "crm-customer-create",
  "description": "Create a CRM customer record. Use when user asks to add a new customer.",
  "input_schema": {
    "type": "object",
    "required": ["customerName"],
    "properties": {
      "customerName": { "type": "string" },
      "phone": { "type": "string" },
      "source": { "type": "string" }
    }
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "customerId": { "type": "string" }
    }
  },
  "invoke": {
    "method": "POST",
    "path": "/api/action-invoke/crm.customer.create",
    "version": "1.0.0"
  },
  "policy": {
    "auth": "JWT",
    "idempotent": true,
    "timeoutMs": 3000
  }
}
```

### 15.3 ActionMetadata 到 Skill 的映射规则

- `metadata.namespace + metadata.name` -> `skill.name`（建议转 kebab-case）；
- `metadata.description` + 业务标签 -> `skill.description`；
- `spec.inputSchema` -> `skill.input_schema`；
- `spec.outputSchema` -> `skill.output_schema`；
- `spec.endpoint + version` -> `skill.invoke`；
- `spec.auth / idempotency / timeoutMs` -> `skill.policy`。

### 15.4 给模型的调用约束（标准化提示片段）

建议在 Skill 详情中附带 `usage_guidelines`，用于约束模型行为：

1. 仅调用 `state=ACTIVE` 的 action；
2. 调用前必须按 `input_schema` 做参数校验；
3. 缺少必填参数时，先追问用户，不得猜测；
4. 幂等 action 必须传 `Idempotency-Key`；
5. 失败时按统一错误码决定是否重试（仅 `retryable=true` 可重试）。

### 15.5 多模型兼容建议

- 对外暴露统一 Skill DTO，不绑定单一模型厂商；
- 通过 Adapter 层输出不同协议：
  - `Claude Skill` 风格（name/description/input schema/usage）；
  - `OpenAI Tool` 风格（function name + JSON schema）；
  - `MCP Tool` 风格（tool list + invoke）。

这样 Action 一次注册，可被不同大模型以各自“工具/技能”协议消费。

---

## 16. 如何支持热部署（Hot Deployment）

为保证 Action 在不停机条件下上线、下线与升级，建议引入“**控制面版本切换 + 数据面无状态执行器热更新**”机制。

### 16.1 热部署目标

- 新 Action 可在不重启业务服务的情况下生效；
- Action 新版本可灰度发布并快速回滚；
- 不中断已有调用（in-flight 请求可完成）；
- 对模型侧 Skill 目录可实时刷新。

### 16.2 热部署分层策略

1. **元数据热更新（必选）**
   - Registry 中 Action metadata 变更后，通过事件（SSE/WebSocket/MQ）推送到各服务 SDK；
   - SDK 的 `ActionCatalog` 增量刷新本地缓存并更新路由表；
   - 适用于 endpoint、策略、状态（ACTIVE/DISABLED）切换。

2. **执行器热装载（推荐）**
   - Java 服务内将 Action Handler 以“可替换 Bean/插件 ClassLoader”方式加载；
   - 新版本加载成功后原子替换路由；
   - 老版本进入 drain 状态，等待在途请求结束后卸载。

3. **进程级无损切换（兜底）**
   - 如执行器无法安全热替换，则采用滚动发布 + 版本路由；
   - 通过 Service Mesh/Gateway 做流量切换，逻辑上保持“热部署体验”。

### 16.3 SDK 侧关键机制

- `ActionCatalog.watch()`：订阅 Registry 变更事件；
- `ActionDispatcher.swap(actionKey, handlerRef)`：原子切换处理器引用；
- `ActionDispatcher.drain(oldVersion)`：旧版本停止接新流量，仅处理在途请求；
- `ActionHealthProbe`：新版本加载后先健康检查再接流量。

建议新增本地状态：
- `WARMING`：新版本加载中；
- `ACTIVE`：可接收流量；
- `DRAINING`：仅处理在途请求；
- `INACTIVE`：已卸载。

### 16.4 标准热部署流程

1. 发布新版本 metadata（`DRAFT`）；
2. 在目标服务节点预热加载新 handler（`WARMING`）；
3. 健康检查通过后，切换到 `ACTIVE`；
4. 按灰度策略（1% -> 10% -> 50% -> 100%）提升流量；
5. 旧版本进入 `DRAINING`，等待超时窗口（如 30s/60s）；
6. 完成后标记旧版本 `DEPRECATED` 或 `DISABLED`。

### 16.5 统一调用协议扩展（建议）

为支持灰度/回滚，调用接口增加可选头：

- `X-Action-Version`: 显式版本；
- `X-Action-Canary`: 灰度标签（如 `beta`, `tenantA`）；
- `X-Action-Drain-Timeout`: 调用方可接受的切换等待上限。

返回 `meta` 增加：
- `resolvedVersion`: 实际命中版本；
- `deploymentState`: `ACTIVE`/`DRAINING`；
- `routePolicy`: `latest-active`/`canary`/`pinned`。

### 16.6 与 LLM Skill Adapter 的联动

- Skill 查询接口默认仅返回 `ACTIVE` 版本；
- 若模型或编排层指定 `X-Action-Canary`，可返回对应灰度版本；
- Skill 缓存需设置短 TTL（如 10~30s）或基于事件实时失效；
- 当 Action 进入 `DRAINING`，Skill 侧应标记“即将下线”，避免模型继续推荐旧能力。

### 16.7 风险与防护

- **类加载泄漏**：使用独立 ClassLoader 并在卸载后释放引用；
- **并发切换竞态**：通过 CAS/读写锁保证 handler 原子替换；
- **长事务调用**：设置最大执行时长，超时后强制失败并告警；
- **回滚失败**：保留上一个稳定版本快照，支持一键回切。

### 16.8 最小落地建议

- 第一阶段先做“元数据热更新 + 路由热切换”（不做 class 热替换）；
- 第二阶段增加“执行器热装载 + drain 机制”；
- 第三阶段接入 Mesh/Gateway 灰度与全链路回滚编排。

