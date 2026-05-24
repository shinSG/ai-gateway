# AI Gateway 与 Higress 架构对比分析

## 概述

本文档对比分析 AI Gateway 设计方案与 Higress 开源项目的架构异同，明确参考借鉴的边界和独立设计决策的依据。

---

## 1. 真正参考了 Higress 的部分

以下设计模式直接或间接来源于 Higress 的架构实践：

### 1.1 三合一统一控制面

| 维度 | Higress | AI Gateway |
|------|---------|-----------|
| **流量网关** | Envoy 承载南北向流量 | Spring Cloud Gateway 承载南北向流量 |
| **微服务网关** | Istio 服务网格 | Nacos 3.0 + Spring Cloud 负载均衡 |
| **AI 网关** | MCP 托管、多模型代理 | MCP 托管、多模型代理、A2A/ACP 协议 |

**忠实度**：高。三者合一的产品定位直接来自 Higress。

### 1.2 AI 流量一等公民

| 维度 | Higress | AI Gateway |
|------|---------|-----------|
| **LLM 协议** | OpenAI Compatible、Claude、Bedrock | OpenAI Compatible |
| **MCP 协议** | 原生托管（SSE + Streamable HTTP） | 原生托管（SSE + Streamable HTTP） |
| **流式处理** | 真正的流式（非 buffered 代理） | WebFlux SSE 响应式流 |
| **其他协议** | — | A2A、ACP 扩展 |

**忠实度**：高。核心理念一致，AI Gateway 额外扩展了 A2A/ACP 协议。

### 1.3 MCP SSE Session 管理

```
Higress 模式：
  MCP Client → /sse endpoint → SessionID 生成 → Redis pub/sub → 异步响应推送

AI Gateway 落地：
  McpSessionManager.createSession() → Redis store → Pub/Sub 监听 → Flux<SSE> 推送
```

**忠实度**：高。架构模式完全一致，仅 Redis 客户端实现不同（Higress 用 Go Redis，我们用 Spring Data Redis）。

### 1.4 多模型协议转换

```
Higress 模式：
  OpenAI SDK → Gateway → Protocol Translation Layer → 各厂商模型

AI Gateway 落地：
  Protocol Adapter → AIGatewayMessage 统一模型 → ProtocolTransformer → 目标协议
```

**忠实度**：中高。转换思路一致，但 Higress 的转换层更成熟（8+ Claude↔OpenAI 修复、角色降级、消息合并等），AI Gateway 目前仅定义了框架接口。

### 1.5 消费者粒度认证与限流

| 维度 | Higress | AI Gateway |
|------|---------|-----------|
| **认证粒度** | 按 Consumer（API Key） | 按 Consumer（API Key + JWT + OAuth2） |
| **限流粒度** | TPM、QPM、并发数 | TPM、QPM、并发数 |
| **存储后端** | Wasm + Redis | Spring Filter + Redis |
| **模型降级** | 多模型 fallback + 独立模型名映射 | ModelFallbackConfig + FallbackStrategy |

**忠实度**：高。限流模型和认证模型直接借鉴。

### 1.6 Wasm 插件扩展

| 维度 | Higress | AI Gateway |
|------|---------|-----------|
| **运行时** | Envoy 内置 Wasm VM | Chicory（JVM Wasm 运行时） |
| **语言** | Go、Rust、JavaScript | Go、Rust、JavaScript（相同） |
| **热加载** | Envoy Wasm 原生支持 | PluginClassLoader 隔离加载 |
| **沙箱** | 内存安全、进程隔离 | JVM 内 Wasm 沙箱 |

**忠实度**：中。概念借鉴自 Higress（Wasm 多语言插件），但运行时完全不同——Higress 的 Wasm 在 Envoy 进程内执行，AI Gateway 的 Wasm 在独立 Chicory 运行时中执行。

### 1.7 流式原生

**忠实度**：高。两者都强调全链路 SSE/streaming 支持，不做 buffered 代理。

---

## 2. 与 Higress 的根本性架构差异

以下差异源于底层技术栈选型的不同，导致工程架构产生本质区别：

### 2.1 数据面技术栈

| 维度 | Higress | AI Gateway | 影响 |
|------|---------|-----------|------|
| **语言** | C++（Envoy） | Java 21（Spring Boot） | 性能、内存模型、GC 行为完全不同 |
| **框架** | Envoy + Istio | Spring Cloud Gateway + WebFlux | Envoy 是 L4/L7 代理，SCG 是 L7 网关 |
| **并发模型** | 事件驱动（libevent） | 响应式（Project Reactor + Netty） | 线程模型和背压策略不同 |
| **性能** | 数十万 QPS（C++ 原生） | 数万 QPS（JVM 上限） | 吞吐量有量级差异 |
| **延迟** | 亚毫秒级 | 毫秒级 | AI 场景下延迟多被 LLM 推理掩盖 |

**结论**：这是最根本的架构差异。Higress 选择 C++ 数据面是为了极致性能；AI Gateway 选择 Java 是为了 Spring 生态集成和团队技能匹配。两者的目标场景不完全重叠——Higress 更适合超大规模、低延迟场景；AI Gateway 更适合企业级、生态丰富、快速迭代场景。

### 2.2 配置下发机制

| 维度 | Higress | AI Gateway |
|------|---------|-----------|
| **协议** | xDS（Envoy Discovery Service） | Nacos 3.0 Config HTTP 长轮询 |
| **推送模式** | gRPC 双向流（push） | HTTP 长轮询（pull/watch） |
| **实时性** | 接近实时（delta xDS） | 秒级延迟（1-3s 轮询间隔） |
| **无重启** | 原生无重启热更新 | Spring Cloud Gateway RouteRefreshListener |
| **一致性** | 最终一致性（eventual） | 最终一致性（Nacos CP） |

**结论**：xDS 是 Envoy 生态的核心竞争力，支持 delta 增量推送和按需订阅。Nacos 3.0 的配置监听在实时性上不如 xDS，但足以满足 AI 网关场景（路由表变更频率远低于微服务流量）。

### 2.3 服务注册与发现

| 维度 | Higress | AI Gateway |
|------|---------|-----------|
| **注册中心** | Kubernetes API Server + CRD | Nacos 3.0 + metadata |
| **路由定义** | HTTPRoute、InferencePool（标准 K8s CRD） | AIGatewayRouteDefinition（自定义模型） |
| **服务模型** | K8s Service + EndpointSlice | Nacos Instance + Group + Namespace |
| **扩展方式** | 标准 CRD + controller 模式 | Nacos metadata JSON 扩展 |
| **生态兼容** | 与 Istio、Linkerd 等服务网格兼容 | 与 Spring Cloud Alibaba 生态兼容 |

**结论**：这是 Higress 的"云原生原生"优势所在。Higress 直接复用 K8s 生态的标准 API（Gateway API、InferencePool），无需额外的注册中心。AI Gateway 选择 Nacos 3.0 是因为 Java 生态的惯用模式，但在 K8s 环境下增加了一个中间层。

### 2.4 AI 推理路由（GIE）

| 维度 | Higress | AI Gateway |
|------|---------|-----------|
| **标准** | K8s Gateway API Inference Extension (GIE) | 自定义 AIGatewayRoute |
| **路由决策** | KV Cache 命中率、任务队列深度、LoRA 状态 | AI_AWARE 策略（占位） |
| **集成方式** | 外部 gRPC Endpoint Picker | 未定义具体实现 |
| **生态** | K8s Gateway API 标准生态 | 私有模型 |

**结论**：Higress 支持 K8s 官方的 Inference Extension 标准，AI Gateway 目前仅有接口定义（AiAwareRouteDefinition），缺少具体的推理感知路由实现。这不是设计缺陷，而是因为 GIE 是 K8s 特有的扩展点，Nacos 生态中没有等价物。

### 2.5 Wasm 运行时架构

```
Higress Wasm 架构：
  Envoy Worker Thread
    └── Wasm VM (V8/WAMR)
          └── Plugin 1..N (in-process)
  
  - 零拷贝数据访问（共享 Envoy buffer）
  - 微秒级调用延迟
  - 故障可能影响整个 Worker 线程

AI Gateway Wasm 架构：
  JVM Thread
    └── Chicory Wasm Runtime (独立 JVM 进程或类加载器)
          └── Plugin 1..N (isolated)
  
  - 数据需序列化/反序列化传递
  - 毫秒级调用延迟（进程间通信）
  - 故障隔离性更好（独立进程）
```

**结论**：两者的 Wasm 集成方式由于宿主运行时不同而有本质区别。Higress 的 Wasm 更"原生"（与 Envoy 共享内存空间），AI Gateway 的 Wasm 更"安全"（强隔离）。

---

## 3. 缺失的 Higress 关键能力

以下 Higress 已实现的功能在 AI Gateway 设计中未覆盖：

| 能力 | 说明 | 缺失原因 |
|------|------|---------|
| **`mergeConsecutiveMessages`** | 同角色连续消息自动合并（GLM/Kimi 等严格模型的兼容） | 协议转换尚在框架阶段，未细化到具体厂商适配 |
| **`developer → system` 角色降级** | 后端不支持 OpenAI developer 角色时自动降级为 system | 同上，厂商兼容性细节未展开 |
| **SigV4 签名** | AWS Bedrock 认证签名支持 | 认证模块未覆盖云厂商专有签名 |
| **轻量级可观测模式** | 跳过大体积 prompt/response 缓存以节省内存 | 未覆盖 |
| **客户端断开时实时 Token 计费** | SSE 连接断开时立即统计已生成 Token，不丢失数据 | 计费追踪定义了 API 但未处理异常断开场景 |
| **Gateway API Inference Extension** | 基于 KV Cache 命中率的智能路由 | K8s 专用标准，Nacos 生态无等价物 |
| **Nginx Ingress 兼容模式** | Higress 可平替 Nginx Ingress | 不在设计范围内 |

---

## 4. 新增的差异化能力

以下是 AI Gateway 在 Higress 基础上的增强设计：

| 能力 | 说明 | 设计依据 |
|------|------|---------|
| **A2A 协议支持** | Google Agent-to-Agent 协议原生集成 | Higress 不支持 A2A |
| **ACP 协议支持** | IBM/Linux Foundation Agent Communication Protocol | Higress 不支持 ACP |
| **协议转换矩阵** | 4×4 协议双向透明转换 | 自有设计，Higress 仅做 OpenAI↔厂商 |
| **Agent 编排引擎** | ReAct Loop + 多 Agent 协调 + 对话记忆 | 来自 AgentScope 参考 |
| **Nacos 3.0 AI Registry** | 模型/工具/Prompt 注册管理 | 利用 Nacos 3.0 新特性，Higress 用 K8s CRD |
| **统一消息模型** | AIGatewayMessage 统一所有协议的消息表示 | 参考 AgentScope Msg，Higress 无此抽象 |
| **服务元数据协议** | Agent/Skill/Service/Plugin 四类完整 metadata 规范 | 自有设计，Higress 无此规范 |
| **Java/Wasm 双模插件** | 同时支持 Java 原生插件和 Wasm 多语言插件 | Higress 仅支持 Wasm |

---

## 5. 总结

### 5.1 参考程度判定

| 层次 | 参考程度 | 说明 |
|------|---------|------|
| **产品理念层** | 高 | 三合一、AI 一等公民、MCP 托管等产品定位直接借鉴 |
| **功能模式层** | 中高 | MCP Session 管理、协议转换、消费者限流等模式一致 |
| **工程架构层** | 低 | 底层技术栈完全不同，导致数据面、配置下发、服务发现机制有本质差异 |
| **生态定位层** | 低 | Higress 是 K8s+Envoy 生态，AI Gateway 是 Java+Nacos 生态 |

### 5.2 准确的关系表述

> AI Gateway 在 **产品理念和功能模式** 层面参考了 Higress 的 AI 原生网关设计（三合一、MCP 托管、多模型协议转换、Wasm 插件、消费者粒度治理）。但在 **工程架构** 层面，由于选择了 Java/Spring Cloud Gateway/Nacos 3.0 技术栈（而非 Envoy/C++/K8s CRD），实际的架构实现与 Higress 有根本性差异。
>
> 同时，AI Gateway 在协议覆盖面（新增 A2A/ACP）、Agent 编排能力（ReAct/多 Agent 协调）、服务元数据规范、Nacos 3.0 深度集成等方面，对 Higress 现有能力进行了扩展和补充。

### 5.3 关键差异一览

```
                    Higress                    AI Gateway
                    ────────                   ───────────
数据面语言           C++ (Envoy)               Java 21 (Spring Boot)
并发模型             事件驱动 (libevent)         响应式 (Reactor + Netty)
配置下发             xDS (gRPC stream push)     Nacos Config (HTTP poll)
服务发现             K8s CRD + API Server       Nacos 3.0 + metadata
Wasm 运行时          Envoy 进程内               Chicory 独立运行时
AI 路由标准          Gateway API GIE            自定义 AIGatewayRoute
MCP 协议             SSE + Streamable HTTP      SSE + Streamable HTTP ✓
A2A 协议             ✗                          ✓
ACP 协议             ✗                          ✓
多模型转换           OpenAI↔厂商 (成熟)          框架阶段 (接口定义)
Agent 编排           ✗ (纯网关无 Agent 能力)     ✓ (ReAct + 多Agent)
服务元数据规范       无                           Agent/Skill/Service/Plugin
```

### 5.4 技术栈选型的合理性论证

选择 Java/Nacos 而非 C++/Envoy/K8s CRD 的原因：

1. **团队技能匹配**：Java 是企业后端团队的主流技能，C++ 团队稀缺
2. **Spring 生态深度集成**：Spring AI、LangChain4j、Spring Security 开箱即用
3. **Nacos 3.0 AI 特性**：Nacos 3.0 新增的 AI Registry、MCP Registry 与 AI 网关需求高度匹配
4. **Agent 编排能力**：Java 可以通过 LangChain4j/Spring AI 实现复杂的 Agent 编排逻辑，这是纯网关（Higress）不擅长的
5. **协议扩展性**：Java 生态的 JSON-RPC、gRPC、SSE 库成熟度高于 C++ Wasm 插件开发体验
6. **AI 场景延迟容忍**：LLM 推理延迟通常在 500ms-30s，网关层的毫秒级差异不构成瓶颈

### 5.5 补充建议

如需增强对 Higress 架构的参考深度，可以在以下方面改进：

1. **增加 xDS 协议兼容**：使用 gRPC 实现 xDS 协议的 adapter，与 Envoy 生态打通
2. **支持 K8s Gateway API**：实现 Gateway API controller，将 Nacos 服务映射为标准 HTTPRoute
3. **引入原生 Wasm 运行时**：考虑使用 GraalWasm 替代 Chicory，性能更接近原生
4. **细化厂商兼容层**：实现 OpenAI↔通义/GLM/DeepSeek 的具体协议转换逻辑
