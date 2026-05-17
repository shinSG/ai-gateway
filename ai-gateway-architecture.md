# AI 网关架构设计文档

## 目录

- [1. 概述](#1-概述)
- [2. 系统架构](#2-系统架构)
- [3. 详细设计](#3-详细设计)
- [4. 注册中心元数据定义](#4-注册中心元数据定义)
  - [4.9 Nacos 注册元数据详细定义](#49-nacos-注册元数据详细定义)
- [5. 基础设施](#5-基础设施)
- [6. 开发工作流](#6-开发工作流)
- [7. 监控与可观测性](#7-监控与可观测性)
- [8. 风险评估](#8-风险评估)
- [9. 实施路线图](#9-实施路线图)
- [附录](#附录)

---

## 1. 概述

### 1.1 项目背景

本项目旨在构建一个企业级 AI 网关（AI Gateway），作为 AI 能力的统一入口和管理平台。网关负责管理 AI 资源的注册与发现、请求的智能路由、安全防护以及全链路观测，为企业提供统一、安全、可观测的 AI 服务接入层。

**核心价值：**
- **统一入口**：屏蔽后端 AI 服务差异，提供标准化接入接口
- **多协议支持**：统一管理 API、MCP、A2A 等多种 AI 协议
- **智能路由**：支持静态路由和基于语义理解的动态路由
- **安全可控**：统一鉴权、审计、限流，保障 AI 服务安全
- **可观测性**：全链路追踪、指标采集、日志聚合，运维无忧

**目标用户：**
- AI 服务提供者：注册和管理 Agent、Skill、Plugin、Model 等 AI 资源
- 业务开发团队：通过网关快速接入 AI 能力
- 运维团队：监控网关及 AI 服务运行状态
- 安全团队：审计 AI 调用行为，管控访问权限

### 1.2 设计原则

- **高可用性**：网关无单点故障，支持多实例部署和故障自动转移
- **高性能**：低延迟转发，支持高并发 AI 请求处理
- **可扩展性**：插件化架构，支持自定义扩展功能模块
- **协议兼容**：统一抽象多种 AI 协议，提供标准化接入
- **安全合规**：统一鉴权、审计日志、数据脱敏，符合企业安全规范
- **可观测性**：全链路追踪、实时指标、智能告警

### 1.3 技术栈

| 层级 | 技术 | 选择理由 |
|-----|------|---------|
| 编程语言 | Java 17+ | 企业级生态成熟，性能稳定 |
| 基础框架 | Spring Boot 3.x + Spring Cloud | 微服务架构标准，组件丰富 |
| 网关框架 | Spring Cloud Gateway | 响应式架构，高性能，与 Spring 生态无缝集成 |
| 服务注册 | Nacos 2.x | 阿里开源，支持服务发现与配置管理，社区活跃 |
| 配置中心 | Nacos Config | 动态配置管理，支持热更新 |
| RPC 框架 | gRPC + Dubbo | 高性能服务调用，支持流式传输 |
| 认证鉴权 | Spring Security + OAuth2 + JWT | 企业级安全框架，支持多种认证方式 |
| 缓存 | Redis 7+ Cluster | 高速缓存、分布式锁、限流计数 |
| 向量数据库 | Milvus / Weaviate | 语义路由向量存储与检索 |
| 消息队列 | RocketMQ / Kafka | 异步解耦、事件驱动、审计日志 |
| 数据库 | MySQL 8.0 + MyBatis-Plus | 关系型数据存储，ORM 框架 |
| 搜索引擎 | Elasticsearch 8+ | 日志聚合、全文检索 |
| 链路追踪 | SkyWalking / Zipkin | 分布式链路追踪，性能分析 |
| 指标监控 | Prometheus + Grafana | 指标采集与可视化 |
| 日志管理 | ELK Stack | 集中式日志管理与分析 |
| 容器化 | Docker + Kubernetes | 容器化部署，弹性伸缩 |
| API 文档 | Knife4j (Swagger) | 在线 API 文档，便于调试 |

---

## 2. 系统架构

### 2.1 高层架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            客户端层 (Client Layer)                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Web 应用  │  │ 移动端APP │  │ 小程序    │  │ 第三方系统 │  │ AI Agent    │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘ │
│       └──────────────┼──────────────┼──────────────┼──────────────┘         │
└──────────────────────┼──────────────┼──────────────┼────────────────────────┘
                       │              │              │
                       ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AI 网关层 (AI Gateway)                            │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        API 接入层 (Gateway)                           │  │
│  │   认证鉴权 · 限流熔断 · 协议转换 · 请求过滤 · 响应缓存                 │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │                                          │
│  ┌───────────────────────────────▼───────────────────────────────────────┐  │
│  │                        智能路由层 (Router)                             │  │
│  │   静态路由 · 语义路由 · 负载均衡 · 能力匹配 · 成本优化 · 故障转移     │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │                                          │
│  ┌───────────────────────────────▼───────────────────────────────────────┐  │
│  │                      注册中心 (Registry)                              │  │
│  │   Agent 注册 · Skill 注册 · Plugin 注册 · Model 注册                  │  │
│  │   API 协议 · MCP 协议 · A2A 协议 · 健康检查 · 元数据管理              │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │                                          │
│  ┌───────────────────────────────▼───────────────────────────────────────┐  │
│  │                      管理控制台 (Admin Console)                        │  │
│  │   资源管理 · 配置管理 · 权限管理 · 审计日志 · 监控大盘                  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                       │              │              │
                       ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AI 服务层 (AI Services)                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Agent    │  │ Skill    │  │ Plugin   │  │ Model    │  │ Workflow     │ │
│  │ 服务     │  │ 服务     │  │ 服务     │  │ 服务     │  │ 服务         │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                       │              │              │
                       ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       基础设施层 (Infrastructure)                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │  Nacos   │  │  Redis   │  │ MySQL    │  │ RocketMQ │  │ SkyWalking  │ │
│  │ 服务发现  │  │ 缓存/限流 │  │ 数据存储  │  │ 消息队列  │  │ 链路追踪    │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │
│  ┌──────────┐  ┌─────────────────────────────────────────────────────────┐  │
│  │ Milvus   │  │              Elasticsearch                              │  │
│  │ 向量检索  │  │              日志聚合                                    │  │
│  └──────────┘  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 组件概览

| 组件 | 职责 | 技术实现 |
|-----|------|---------|
| API 网关 | 请求接入、协议转换、认证鉴权、限流熔断 | Spring Cloud Gateway |
| 注册中心 | AI 资源注册发现、多协议支持、健康检查、元数据管理 | Nacos 2.x + 自研扩展 |
| 智能路由 | 静态路由、语义路由、负载均衡、能力匹配 | 自研路由引擎 + Milvus |
| 安全模块 | 认证鉴权、权限控制、审计日志 | Spring Security + OAuth2 |
| 管理控制台 | 资源管理、配置管理、监控展示 | Vue 3 + Element Plus |
| 链路追踪 | 全链路追踪、性能分析 | SkyWalking |
| 指标监控 | 指标采集、告警通知 | Prometheus + Grafana |
| 日志中心 | 日志聚合、检索分析 | ELK Stack |

### 2.3 数据流

**AI 请求处理流程：**
```
1. 客户端请求 → API 网关 → JWT Token 验证
2. 验证通过 → 限流检查 → 请求过滤
3. 通过过滤 → 智能路由引擎
   ├── 静态路由: 路径匹配 → 直接转发
   └── 语义路由: 语义分析 → 向量检索 → 能力匹配
4. 路由决策 → 协议转换 → 转发请求 → AI 服务处理
5. 服务响应 → 响应处理 → 缓存结果
6. 返回响应 → 客户端
```

**AI 资源注册流程：**
```
1. AI 服务启动 → 注册中心 → 发送注册请求
2. 注册中心 → 健康检查 → 验证服务状态
3. 注册成功 → 元数据存储 → 生成服务描述向量
4. 更新路由表 → 路由引擎 → 支持新服务路由
```

**MCP 协议处理流程：**
```
1. MCP 客户端 → AI 网关 → MCP 请求
2. 网关解析 MCP 消息 → 路由到目标 MCP Server
3. MCP Server 处理 → 返回工具调用结果
4. 网关转换响应 → 返回 MCP 客户端
```

**A2A 协议处理流程：**
```
1. Agent A → AI 网关 → A2A 请求 (Task/Send)
2. 网关路由 → 目标 Agent B
3. Agent B 处理 → 返回 A2A 响应
4. 网关转发 → 返回 Agent A
```

---

## 3. 详细设计

### 3.1 目录结构

```
ai-gateway/
├── gateway-bootstrap/              # 启动模块
│   └── src/main/java/
│       └── com/ai/gateway/
│           └── GatewayApplication.java
│
├── gateway-core/                   # 核心模块
│   └── src/main/java/
│       └── com/ai/gateway/core/
│           ├── config/             # 配置类
│           ├── filter/             # 网关过滤器
│           ├── route/              # 路由引擎
│           ├── registry/           # 注册中心客户端
│           ├── protocol/           # 协议处理
│           └── security/           # 安全模块
│
├── gateway-registry/               # 注册中心模块
│   └── src/main/java/
│       └── com/ai/gateway/registry/
│           ├── agent/              # Agent 注册
│           ├── skill/              # Skill 注册
│           ├── plugin/             # Plugin 注册
│           ├── model/              # Model 注册
│           ├── protocol/           # 协议处理器
│           │   ├── api/            # API 协议
│           │   ├── mcp/            # MCP 协议
│           │   └── a2a/            # A2A 协议
│           ├── health/             # 健康检查
│           └── metadata/           # 元数据管理
│
├── gateway-router/                 # 智能路由模块
│   └── src/main/java/
│       └── com/ai/gateway/router/
│           ├── static/             # 静态路由
│           ├── semantic/           # 语义路由
│           ├── loadbalance/        # 负载均衡
│           ├── matcher/            # 能力匹配
│           └── fallback/           # 故障转移
│
├── gateway-security/               # 安全模块
│   └── src/main/java/
│       └── com/ai/gateway/security/
│           ├── auth/               # 认证
│           ├── authorization/      # 授权
│           ├── audit/              # 审计
│           └── token/              # Token 管理
│
├── gateway-admin/                  # 管理控制台
│   └── src/main/java/
│       └── com/ai/gateway/admin/
│           ├── controller/         # API 控制器
│           ├── service/            # 业务服务
│           └── repository/         # 数据访问
│
├── gateway-common/                 # 公共模块
│   └── src/main/java/
│       └── com/ai/gateway/common/
│           ├── model/              # 数据模型
│           ├── exception/          # 异常定义
│           └── utils/              # 工具类
│
├── gateway-console/                # 前端控制台
│   ├── src/
│   │   ├── views/                  # 页面组件
│   │   ├── components/             # 公共组件
│   │   ├── api/                    # API 接口
│   │   └── store/                  # 状态管理
│   └── package.json
│
└── docs/                           # 文档
    ├── api/                        # API 文档
    └── design/                     # 设计文档
```

### 3.2 模块设计

#### 3.2.1 注册中心模块 (Registry)

**核心功能：**
- Agent 注册与发现
- Skill 注册与发现
- Plugin 注册与发现
- Model 注册与发现
- 多协议支持 (API/MCP/A2A)
- 服务健康检查
- 元数据管理与向量化

**资源类型枚举：**

```java
/**
 * 资源类型枚举
 */
public enum ResourceType {
    AGENT("Agent智能体", "具有自主决策能力的AI组件"),
    SKILL("Skill技能", "特定领域的AI能力单元"),
    PLUGIN("Plugin插件", "可扩展的功能模块"),
    MODEL("Model模型", "大语言模型或专用模型服务");
    
    private final String description;
    private final String detail;
}
```

**协议类型枚举：**

```java
/**
 * 协议类型枚举
 */
public enum ProtocolType {
    API("REST API", "标准RESTful接口"),
    GRPC("gRPC", "高性能RPC协议"),
    MCP("Model Context Protocol", "模型上下文协议，用于工具调用"),
    A2A("Agent-to-Agent", "Agent间通信协议"),
    WEBSOCKET("WebSocket", "长连接协议"),
    SSE("Server-Sent Events", "服务器推送事件");
    
    private final String name;
    private final String description;
}
```

**Nacos 集成配置：**

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER:localhost:8848}
        namespace: ${NACOS_NAMESPACE:ai-gateway}
        group: AI_GATEWAY_GROUP
        metadata:
          version: ${APP_VERSION:1.0.0}
          environment: ${APP_ENV:dev}
      config:
        server-addr: ${NACOS_SERVER:localhost:8848}
        namespace: ${NACOS_NAMESPACE:ai-gateway}
        group: AI_GATEWAY_GROUP
        file-extension: yaml
```

**Nacos 注册元数据规范：**

所有 AI 资源注册到 Nacos 时，使用统一的元数据结构。Nacos 实例元数据 (Instance Metadata) 采用扁平化 KV 结构，复杂数据使用 JSON 字符串序列化。

**Nacos 元数据通用字段：**

| 元数据 Key | 类型 | 必填 | 说明 |
|-----------|------|------|------|
| resourceId | String | 是 | 资源唯一标识 (UUID) |
| resourceType | String | 是 | 资源类型: AGENT/SKILL/PLUGIN/MODEL |
| name | String | 是 | 资源名称 |
| version | String | 是 | 资源版本 |
| description | String | 否 | 资源描述 |
| protocolType | String | 是 | 协议类型: API/MCP/A2A/GRPC/WEBSOCKET/SSE |
| serviceAddress | String | 是 | 服务地址 (host:port 或 URL) |
| capabilities | String | 否 | 能力标签列表 (JSON Array) |
| tags | String | 否 | 标签 (JSON Map) |
| status | String | 是 | 服务状态: UP/DOWN/STARTING/STOPPING |
| weight | Integer | 否 | 负载权重 (1-100)，默认 50 |
| healthCheckInterval | Integer | 否 | 健康检查间隔 (秒)，默认 30 |
| createTime | String | 是 | 创建时间 (ISO 8601) |
| updateTime | String | 是 | 更新时间 (ISO 8601) |

**Nacos 元数据扩展字段前缀规范：**

| 前缀 | 用途 | 说明 |
|------|------|------|
| agent.* | Agent 专用字段 | Agent 类型、能力定义、工具列表等 |
| skill.* | Skill 专用字段 | Skill 类型、输入输出参数等 |
| plugin.* | Plugin 专用字段 | Plugin 类型、配置参数等 |
| model.* | Model 专用字段 | 模型类型、能力、资源需求、定价等 |
| mcp.* | MCP 协议专用字段 | MCP 服务器能力、工具、资源等 |
| a2a.* | A2A 协议专用字段 | Agent Card、任务类型、认证等 |
| api.* | API 协议专用字段 | API 类型、限流配置等 |

---

#### 3.2.2 智能路由模块 (Router)

**核心功能：**
- 静态路由：基于路径、Header、Query 参数的确定性路由
- 语义路由：基于请求内容语义理解的智能路由
- 负载均衡策略
- 能力匹配
- 成本优化路由
- 故障转移机制

**路由类型枚举：**

```java
/**
 * 路由类型枚举
 */
public enum RouteType {
    STATIC("静态路由", "基于规则的确定性路由"),
    SEMANTIC("语义路由", "基于内容理解的智能路由"),
    HYBRID("混合路由", "静态路由优先，未匹配时使用语义路由");
    
    private final String name;
    private final String description;
}
```

**语义路由引擎：**

```java
/**
 * 语义路由引擎
 * 基于向量相似度匹配最合适的AI服务
 */
@Service
@Slf4j
@RequiredArgsConstructor
public class SemanticRouteEngine {
    
    private final EmbeddingService embeddingService;
    private final VectorStore vectorStore;
    private final AiResourceRegistryService registryService;
    
    /**
     * 语义路由决策
     */
    public RouteResult routeBySemantic(SemanticRouteContext context) {
        // 1. 生成请求文本的向量表示
        String requestText = buildRequestText(context);
        float[] queryEmbedding = embeddingService.embed(requestText);
        
        // 2. 向量检索相似服务
        List<VectorMatch> matches = vectorStore.search(
            queryEmbedding,
            context.getTargetType(),
            context.getTopK() != null ? context.getTopK() : 5
        );
        
        // 3. 过滤可用服务
        List<AiResourceRegistry> candidates = matches.stream()
            .map(match -> registryService.getByResourceId(match.getResourceId()))
            .filter(Objects::nonNull)
            .filter(r -> r.getStatus() == ServiceStatus.UP)
            .collect(Collectors.toList());
        
        // 4. 应用负载均衡
        if (candidates.isEmpty()) {
            return RouteResult.failure("No semantic match found");
        }
        
        AiResourceRegistry selected = loadBalanceStrategy.select(
            candidates, context);
        
        return RouteResult.success(selected, buildRouteTarget(selected));
    }
    
    /**
     * 构建请求文本用于向量化
     */
    private String buildRequestText(SemanticRouteContext context) {
        StringBuilder sb = new StringBuilder();
        
        // 添加能力标签
        if (CollectionUtils.isNotEmpty(context.getRequiredCapabilities())) {
            sb.append("capabilities: ")
              .append(String.join(", ", context.getRequiredCapabilities()));
        }
        
        // 添加请求描述
        if (StringUtils.isNotBlank(context.getDescription())) {
            sb.append(" description: ").append(context.getDescription());
        }
        
        // 添加查询内容
        if (StringUtils.isNotBlank(context.getQuery())) {
            sb.append(" query: ").append(context.getQuery());
        }
        
        return sb.toString();
    }
}
```

**路由策略模型：**

```java
/**
 * 路由策略枚举
 */
public enum RouteStrategy {
    ROUND_ROBIN("轮询"),
    WEIGHTED_ROUND_ROBIN("加权轮询"),
    LEAST_CONNECTION("最小连接数"),
    CAPABILITY_MATCH("能力匹配"),
    COST_OPTIMIZED("成本优化"),
    LATENCY_FIRST("延迟优先"),
    SEMANTIC_MATCH("语义匹配");
    
    private final String description;
}
```

---

## 4. 注册中心元数据定义

### 4.1 通用元数据结构

所有资源类型共享的基础元数据结构：

```java
/**
 * 基础元数据结构
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BaseMetadata {
    
    /**
     * 资源唯一标识
     */
    private String resourceId;
    
    /**
     * 资源名称
     */
    private String name;
    
    /**
     * 资源描述
     */
    private String description;
    
    /**
     * 资源版本
     */
    private String version;
    
    /**
     * 资源类型
     */
    private ResourceType resourceType;
    
    /**
     * 协议类型
     */
    private ProtocolType protocolType;
    
    /**
     * 服务地址 (host:port 或 URL)
     */
    private String serviceAddress;
    
    /**
     * 能力标签列表
     */
    private List<String> capabilities;
    
    /**
     * 标签 (用于分类和筛选)
     */
    private Map<String, String> tags;
    
    /**
     * 服务状态
     */
    private ServiceStatus status;
    
    /**
     * 负载权重 (1-100)
     */
    private Integer weight;
    
    /**
     * 健康检查间隔 (秒)
     */
    private Integer healthCheckInterval;
    
    /**
     * 创建时间
     */
    private LocalDateTime createTime;
    
    /**
     * 更新时间
     */
    private LocalDateTime updateTime;
    
    /**
     * 扩展元数据
     */
    private Map<String, Object> extensions;
}
```

### 4.2 Agent 资源元数据

```java
/**
 * Agent 元数据定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class AgentMetadata extends BaseMetadata {
    
    /**
     * Agent 类型
     */
    private AgentType agentType;
    
    /**
     * Agent 描述信息 (用于语义路由)
     */
    private AgentDescription agentDescription;
    
    /**
     * 能力定义
     */
    private AgentCapabilityDefinition capabilities;
    
    /**
     * 工具列表 (Agent 可调用的工具)
     */
    private List<ToolDefinition> tools;
    
    /**
     * 协议配置
     */
    private ProtocolConfig protocolConfig;
    
    /**
     * 部署配置
     */
    private DeploymentConfig deploymentConfig;
}

/**
 * Agent 类型枚举
 */
public enum AgentType {
    REACTIVE("反应式Agent", "基于规则的简单响应Agent"),
    PROACTIVE("主动式Agent", "具有规划能力的Agent"),
    AUTONOMOUS("自主式Agent", "完全自主决策的Agent"),
    MULTI_AGENT("多Agent", "协调多个子Agent的Agent"),
    WORKFLOW("工作流Agent", "基于工作流编排的Agent");
    
    private final String name;
    private final String description;
}

/**
 * Agent 描述信息 (用于语义路由)
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AgentDescription {
    
    /**
     * Agent 功能描述 (自然语言)
     */
    private String summary;
    
    /**
     * 使用场景描述
     */
    private List<String> useCases;
    
    /**
     * 输入描述
     */
    private String inputDescription;
    
    /**
     * 输出描述
     */
    private String outputDescription;
    
    /**
     * 示例对话
     */
    private List<DialogueExample> examples;
}

/**
 * 对话示例
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DialogueExample {
    
    /**
     * 用户输入
     */
    private String userInput;
    
    /**
     * 期望输出
     */
    private String expectedOutput;
    
    /**
     * 场景标签
     */
    private String scenario;
}

/**
 * Agent 能力定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AgentCapabilityDefinition {
    
    /**
     * 支持的任务类型
     */
    private List<String> taskTypes;
    
    /**
     * 支持的语言
     */
    private List<String> supportedLanguages;
    
    /**
     * 上下文窗口大小 (token数)
     */
    private Integer contextWindowSize;
    
    /**
     * 是否支持流式输出
     */
    private Boolean streamingSupport;
    
    /**
     * 是否支持多模态输入
     */
    private Boolean multiModalSupport;
    
    /**
     * 多模态支持类型
     */
    private List<String> supportedModalities;
}

/**
 * 工具定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ToolDefinition {
    
    /**
     * 工具名称
     */
    private String name;
    
    /**
     * 工具描述
     */
    private String description;
    
    /**
     * 参数定义 (JSON Schema)
     */
    private String parametersSchema;
    
    /**
     * 返回值描述
     */
    private String returnDescription;
}

/**
 * 协议配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProtocolConfig {
    
    /**
     * 协议类型
     */
    private ProtocolType protocolType;
    
    /**
     * 协议版本
     */
    private String protocolVersion;
    
    /**
     * 端点路径
     */
    private String endpointPath;
    
    /**
     * 认证方式
     */
    private String authenticationType;
    
    /**
     * 协议特定配置
     */
    private Map<String, Object> protocolSpecificConfig;
}

/**
 * 部署配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DeploymentConfig {
    
    /**
     * 部署模式
     */
    private String deploymentMode;
    
    /**
     * 实例数量
     */
    private Integer instanceCount;
    
    /**
     * CPU 需求
     */
    private String cpuRequirement;
    
    /**
     * 内存需求
     */
    private String memoryRequirement;
    
    /**
     * GPU 需求 (如有)
     */
    private String gpuRequirement;
    
    /**
     * 超时配置
     */
    private TimeoutConfig timeoutConfig;
}

/**
 * 超时配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class TimeoutConfig {
    
    /**
     * 连接超时 (毫秒)
     */
    private Long connectTimeout;
    
    /**
     * 读取超时 (毫秒)
     */
    private Long readTimeout;
    
    /**
     * 写入超时 (毫秒)
     */
    private Long writeTimeout;
}
```

### 4.3 Skill 资源元数据

```java
/**
 * Skill 元数据定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class SkillMetadata extends BaseMetadata {
    
    /**
     * Skill 类型
     */
    private SkillType skillType;
    
    /**
     * Skill 描述信息 (用于语义路由)
     */
    private SkillDescription skillDescription;
    
    /**
     * 输入参数定义
     */
    private List<ParameterDefinition> inputParameters;
    
    /**
     * 输出参数定义
     */
    private List<ParameterDefinition> outputParameters;
    
    /**
     * 依赖的模型
     */
    private List<String> requiredModels;
    
    /**
     * 依赖的工具
     */
    private List<String> requiredTools;
    
    /**
     * 协议配置
     */
    private ProtocolConfig protocolConfig;
}

/**
 * Skill 类型枚举
 */
public enum SkillType {
    TEXT_GENERATION("文本生成", "文本内容生成"),
    TEXT_ANALYSIS("文本分析", "文本内容分析与理解"),
    CODE_GENERATION("代码生成", "代码自动生成"),
    TRANSLATION("翻译", "多语言翻译"),
    SUMMARIZATION("摘要", "文本摘要生成"),
    EXTRACTION("信息抽取", "结构化信息抽取"),
    CLASSIFICATION("分类", "文本分类"),
    QA("问答", "问答系统"),
    CUSTOM("自定义", "自定义技能");
    
    private final String name;
    private final String description;
}

/**
 * Skill 描述信息
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SkillDescription {
    
    /**
     * 功能摘要
     */
    private String summary;
    
    /**
     * 适用场景
     */
    private List<String> useCases;
    
    /**
     * 输入格式描述
     */
    private String inputFormat;
    
    /**
     * 输出格式描述
     */
    private String outputFormat;
    
    /**
     * 示例
     */
    private List<SkillExample> examples;
}

/**
 * Skill 示例
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SkillExample {
    
    /**
     * 输入示例
     */
    private String input;
    
    /**
     * 输出示例
     */
    private String output;
    
    /**
     * 说明
     */
    private String description;
}

/**
 * 参数定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ParameterDefinition {
    
    /**
     * 参数名
     */
    private String name;
    
    /**
     * 参数描述
     */
    private String description;
    
    /**
     * 参数类型
     */
    private String type;
    
    /**
     * 是否必填
     */
    private Boolean required;
    
    /**
     * 默认值
     */
    private Object defaultValue;
    
    /**
     * 枚举值列表
     */
    private List<Object> enumValues;
}
```

### 4.4 Plugin 资源元数据

```java
/**
 * Plugin 元数据定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class PluginMetadata extends BaseMetadata {
    
    /**
     * Plugin 类型
     */
    private PluginType pluginType;
    
    /**
     * Plugin 描述信息
     */
    private PluginDescription pluginDescription;
    
    /**
     * 配置参数定义
     */
    private List<ParameterDefinition> configParameters;
    
    /**
     * 依赖的其他 Plugin
     */
    private List<String> dependencies;
    
    /**
     * 协议配置
     */
    private ProtocolConfig protocolConfig;
}

/**
 * Plugin 类型枚举
 */
public enum PluginType {
    TOOL("工具", "提供外部工具调用能力"),
    CONNECTOR("连接器", "连接外部系统"),
    TRANSFORMER("转换器", "数据格式转换"),
    ENRICHER("增强器", "数据增强处理"),
    FILTER("过滤器", "请求/响应过滤"),
    CACHE("缓存", "缓存插件"),
    CUSTOM("自定义", "自定义插件");
    
    private final String name;
    private final String description;
}

/**
 * Plugin 描述信息
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PluginDescription {
    
    /**
     * 功能摘要
     */
    private String summary;
    
    /**
     * 使用场景
     */
    private List<String> useCases;
    
    /**
     * 配置说明
     */
    private String configurationGuide;
}
```

### 4.5 Model 资源元数据

```java
/**
 * Model 元数据定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class ModelMetadata extends BaseMetadata {
    
    /**
     * Model 类型
     */
    private ModelType modelType;
    
    /**
     * Model 描述信息 (用于语义路由)
     */
    private ModelDescription modelDescription;
    
    /**
     * 模型能力
     */
    private ModelCapabilityDefinition capabilities;
    
    /**
     * 资源需求
     */
    private ResourceRequirements resourceRequirements;
    
    /**
     * 定价信息
     */
    private PricingInfo pricing;
    
    /**
     * 协议配置
     */
    private ProtocolConfig protocolConfig;
}

/**
 * Model 类型枚举
 */
public enum ModelType {
    LLM("大语言模型", "通用大语言模型"),
    VLM("视觉语言模型", "支持图像输入的多模态模型"),
    EMBEDDING("嵌入模型", "文本向量化模型"),
    RERANKER("重排模型", "检索结果重排模型"),
    CLASSIFIER("分类模型", "文本分类模型"),
    CUSTOM("自定义模型", "自定义模型服务");
    
    private final String name;
    private final String description;
}

/**
 * Model 描述信息 (用于语义路由)
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ModelDescription {
    
    /**
     * 模型功能摘要
     */
    private String summary;
    
    /**
     * 擅长任务
     */
    private List<String> bestFor;
    
    /**
     * 不擅长任务
     */
    private List<String> notRecommendedFor;
    
    /**
     * 适用领域
     */
    private List<String> domains;
    
    /**
     * 语言支持
     */
    private List<String> supportedLanguages;
}

/**
 * 模型能力定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ModelCapabilityDefinition {
    
    /**
     * 上下文窗口大小 (token数)
     */
    private Integer contextWindowSize;
    
    /**
     * 最大输出长度 (token数)
     */
    private Integer maxOutputLength;
    
    /**
     * 支持的输入模态
     */
    private List<String> inputModalities;
    
    /**
     * 支持的输出模态
     */
    private List<String> outputModalities;
    
    /**
     * 是否支持函数调用
     */
    private Boolean functionCallingSupport;
    
    /**
     * 是否支持流式输出
     */
    private Boolean streamingSupport;
    
    /**
     * 是否支持 JSON 模式
     */
    private Boolean jsonModeSupport;
    
    /**
     * 是否支持结构化输出
     */
    private Boolean structuredOutputSupport;
    
    /**
     * 温度支持范围
     */
    private Range temperatureRange;
    
    /**
     * Top-p 支持范围
     */
    private Range topPRange;
}

/**
 * 范围定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Range {
    private Double min;
    private Double max;
}

/**
 * 资源需求
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ResourceRequirements {
    
    /**
     * 最小 GPU 数量
     */
    private Integer minGpuCount;
    
    /**
     * GPU 类型
     */
    private String gpuType;
    
    /**
     * 最小 GPU 显存 (GB)
     */
    private Integer minGpuMemory;
    
    /**
     * 最小内存 (GB)
     */
    private Integer minMemory;
    
    /**
     * 最小 CPU 核数
     */
    private Integer minCpuCores;
}

/**
 * 定价信息
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PricingInfo {
    
    /**
     * 定价模型
     */
    private PricingModel pricingModel;
    
    /**
     * 每 1K 输入 token 价格
     */
    private BigDecimal inputPricePer1K;
    
    /**
     * 每 1K 输出 token 价格
     */
    private BigDecimal outputPricePer1K;
    
    /**
     * 每次请求固定价格
     */
    private BigDecimal pricePerRequest;
    
    /**
     * 货币单位
     */
    private String currency;
}

/**
 * 定价模型枚举
 */
public enum PricingModel {
    TOKEN_BASED("按Token计费"),
    REQUEST_BASED("按请求计费"),
    SUBSCRIPTION("订阅制"),
    FREE("免费");
    
    private final String description;
}
```

### 4.6 MCP 协议元数据

```java
/**
 * MCP 协议配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class McpProtocolConfig {
    
    /**
     * 协议版本
     */
    private String protocolVersion;
    
    /**
     * 服务器能力
     */
    private McpServerCapabilities serverCapabilities;
    
    /**
     * 客户端能力
     */
    private McpClientCapabilities clientCapabilities;
    
    /**
     * 支持的传输类型
     */
    private List<McpTransportType> transportTypes;
    
    /**
     * 工具列表
     */
    private List<McpToolDefinition> tools;
    
    /**
     * 资源列表
     */
    private List<McpResourceDefinition> resources;
    
    /**
     * 提示模板列表
     */
    private List<McpPromptDefinition> prompts;
}

/**
 * MCP 传输类型枚举
 */
public enum McpTransportType {
    STDIO("标准输入输出"),
    HTTP("HTTP传输"),
    SSE("Server-Sent Events"),
    WEBSOCKET("WebSocket");
    
    private final String description;
}

/**
 * MCP 服务器能力
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class McpServerCapabilities {
    
    /**
     * 是否支持工具调用
     */
    private Boolean tools;
    
    /**
     * 是否支持资源访问
     */
    private Boolean resources;
    
    /**
     * 是否支持提示模板
     */
    private Boolean prompts;
    
    /**
     * 是否支持日志
     */
    private Boolean logging;
}

/**
 * MCP 客户端能力
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class McpClientCapabilities {
    
    /**
     * 是否支持工具调用
     */
    private Boolean tools;
    
    /**
     * 是否支持资源订阅
     */
    private Boolean resources;
    
    /**
     * 是否支持提示模板
     */
    private Boolean prompts;
}

/**
 * MCP 工具定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class McpToolDefinition {
    
    /**
     * 工具名称
     */
    private String name;
    
    /**
     * 工具描述
     */
    private String description;
    
    /**
     * 输入参数 Schema (JSON Schema)
     */
    private String inputSchema;
}

/**
 * MCP 资源定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class McpResourceDefinition {
    
    /**
     * 资源 URI
     */
    private String uri;
    
    /**
     * 资源名称
     */
    private String name;
    
    /**
     * 资源描述
     */
    private String description;
    
    /**
     * MIME 类型
     */
    private String mimeType;
}

/**
 * MCP 提示模板定义
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class McpPromptDefinition {
    
    /**
     * 提示名称
     */
    private String name;
    
    /**
     * 提示描述
     */
    private String description;
    
    /**
     * 参数列表
     */
    private List<McpPromptArgument> arguments;
}

/**
 * MCP 提示参数
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class McpPromptArgument {
    
    /**
     * 参数名
     */
    private String name;
    
    /**
     * 参数描述
     */
    private String description;
    
    /**
     * 是否必填
     */
    private Boolean required;
}
```

### 4.7 A2A 协议元数据

```java
/**
 * A2A 协议配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class A2aProtocolConfig {
    
    /**
     * Agent Card 信息
     */
    private AgentCard agentCard;
    
    /**
     * 支持的任务类型
     */
    private List<String> supportedTaskTypes;
    
    /**
     * 默认输入模式
     */
    private A2aInputMode defaultInputMode;
    
    /**
     * 支持的输入模式
     */
    private List<A2aInputMode> supportedInputModes;
    
    /**
     * 超时配置
     */
    private A2aTimeoutConfig timeoutConfig;
}

/**
 * Agent Card - A2A 协议的核心元数据
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AgentCard {
    
    /**
     * Agent 名称
     */
    private String name;
    
    /**
     * Agent 描述
     */
    private String description;
    
    /**
     * 服务 URL
     */
    private String url;
    
    /**
     * 版本
     */
    private String version;
    
    /**
     * 文档 URL
     */
    private String documentationUrl;
    
    /**
     * 能力声明
     */
    private A2aCapabilities capabilities;
    
    /**
     * 默认输入模式
     */
    private A2aInputMode defaultInputMode;
    
    /**
     * 支持的输入模式
     */
    private List<A2aInputMode> supportedInputModes;
    
    /**
     * 支持的输出模式
     */
    private List<A2aOutputMode> supportedOutputModes;
    
    /**
     * 认证信息
     */
    private A2aAuthentication authentication;
    
    /**
     * 标签
     */
    private List<String> tags;
}

/**
 * A2A 能力声明
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class A2aCapabilities {
    
    /**
     * 是否支持流式输出
     */
    private Boolean streaming;
    
    /**
     * 是否支持推送通知
     */
    private Boolean pushNotifications;
    
    /**
     * 是否支持状态历史
     */
    private Boolean stateTransitionHistory;
}

/**
 * A2A 输入模式枚举
 */
public enum A2aInputMode {
    TEXT("文本"),
    FILE("文件"),
    STRUCTURED_DATA("结构化数据");
    
    private final String description;
}

/**
 * A2A 输出模式枚举
 */
public enum A2aOutputMode {
    TEXT("文本"),
    FILE("文件"),
    STRUCTURED_DATA("结构化数据");
    
    private final String description;
}

/**
 * A2A 认证信息
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class A2aAuthentication {
    
    /**
     * 认证类型
     */
    private A2aAuthType authType;
    
    /**
     * OAuth2 配置 (如果使用 OAuth2)
     */
    private A2aOAuth2Config oauth2Config;
    
    /**
     * API Key 配置 (如果使用 API Key)
     */
    private A2aApiKeyConfig apiKeyConfig;
}

/**
 * A2A 认证类型枚举
 */
public enum A2aAuthType {
    API_KEY("API Key"),
    OAUTH2("OAuth2"),
    NONE("无认证");
    
    private final String description;
}

/**
 * A2A OAuth2 配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class A2aOAuth2Config {
    
    /**
     * 授权服务器 URL
     */
    private String authorizationServerUrl;
    
    /**
     * 客户端 ID
     */
    private String clientId;
    
    /**
     * Scopes
     */
    private List<String> scopes;
}

/**
 * A2A API Key 配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class A2aApiKeyConfig {
    
    /**
     * Header 名称
     */
    private String headerName;
    
    /**
     * Key 前缀 (可选)
     */
    private String keyPrefix;
}

/**
 * A2A 超时配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class A2aTimeoutConfig {
    
    /**
     * 任务执行超时 (秒)
     */
    private Integer taskExecutionTimeout;
    
    /**
     * 连接超时 (秒)
     */
    private Integer connectionTimeout;
}
```

### 4.8 API 协议元数据

```java
/**
 * API 协议配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ApiProtocolConfig {
    
    /**
     * API 类型
     */
    private ApiType apiType;
    
    /**
     * 基础路径
     */
    private String basePath;
    
    /**
     * 认证方式
     */
    private List<ApiAuthType> authTypes;
    
    /**
     * 请求示例
     */
    private List<ApiExample> examples;
    
    /**
     * 限流配置
     */
    private ApiRateLimitConfig rateLimitConfig;
}

/**
 * API 类型枚举
 */
public enum ApiType {
    REST("RESTful API"),
    GRAPHQL("GraphQL"),
    SOAP("SOAP");
    
    private final String description;
}

/**
 * API 认证类型枚举
 */
public enum ApiAuthType {
    API_KEY("API Key"),
    BEARER_TOKEN("Bearer Token"),
    OAUTH2("OAuth2"),
    BASIC("Basic Auth"),
    NONE("无认证");
    
    private final String description;
}

/**
 * API 示例
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ApiExample {
    
    /**
     * 示例名称
     */
    private String name;
    
    /**
     * 示例描述
     */
    private String description;
    
    /**
     * 请求示例 (JSON)
     */
    private String requestExample;
    
    /**
     * 响应示例 (JSON)
     */
    private String responseExample;
}

/**
 * API 限流配置
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ApiRateLimitConfig {
    
    /**
     * 每秒请求数限制
     */
    private Integer requestsPerSecond;
    
    /**
     * 每分钟请求数限制
     */
    private Integer requestsPerMinute;
    
    /**
     * 每日请求数限制
     */
    private Integer requestsPerDay;
    
    /**
     * 并发数限制
     */
    private Integer concurrentLimit;
}
```

### 4.9 Nacos 注册元数据详细定义

本节定义各服务类型、各协议在 Nacos 中注册时的完整元数据 KV 结构。

#### 4.9.1 Agent 服务 Nacos 注册元数据

```json
{
  "resourceId": "agent-uuid-001",
  "resourceType": "AGENT",
  "name": "customer-service-agent",
  "version": "1.0.0",
  "description": "客服智能体，支持多轮对话和问题解答",
  "protocolType": "API",
  "serviceAddress": "10.0.1.100:8080",
  "capabilities": "[\"multi-turn-dialogue\", \"intent-recognition\", \"knowledge-qa\"]",
  "tags": "{\"domain\":\"customer-service\", \"language\":\"zh-CN\", \"team\":\"ai-platform\"}",
  "status": "UP",
  "weight": "80",
  "healthCheckInterval": "30",
  "createTime": "2024-01-15T10:00:00Z",
  "updateTime": "2024-01-15T10:00:00Z",

  "agent.agentType": "PROACTIVE",
  "agent.summary": "智能客服Agent，能够理解用户意图并提供精准的问题解答",
  "agent.useCases": "[\"售前咨询\", \"售后服务\", \"投诉处理\", \"产品推荐\"]",
  "agent.inputDescription": "用户自然语言输入，支持文本和语音转文本",
  "agent.outputDescription": "结构化回复，包含回答内容、置信度、推荐操作",
  "agent.taskTypes": "[\"dialogue\", \"qa\", \"recommendation\"]",
  "agent.supportedLanguages": "[\"zh-CN\", \"en-US\"]",
  "agent.contextWindowSize": "32000",
  "agent.streamingSupport": "true",
  "agent.multiModalSupport": "false",
  "agent.tools": "[{\"name\":\"knowledge_search\",\"description\":\"知识库检索\",\"parametersSchema\":\"{\\\"type\\\":\\\"object\\\",\\\"properties\\\":{\\\"query\\\":{\\\"type\\\":\\\"string\\\"}}}\"}]",
  "agent.deploymentMode": "kubernetes",
  "agent.instanceCount": "3",
  "agent.connectTimeout": "5000",
  "agent.readTimeout": "30000",

  "protocol.version": "1.0",
  "protocol.endpointPath": "/api/v1/agent/chat",
  "protocol.authenticationType": "BEARER_TOKEN"
}
```

#### 4.9.2 Skill 服务 Nacos 注册元数据

```json
{
  "resourceId": "skill-uuid-001",
  "resourceType": "SKILL",
  "name": "text-summarization-skill",
  "version": "1.2.0",
  "description": "文本摘要技能，支持长文本自动摘要生成",
  "protocolType": "API",
  "serviceAddress": "10.0.1.101:8081",
  "capabilities": "[\"summarization\", \"text-generation\", \"chinese\", \"english\"]",
  "tags": "{\"domain\":\"nlp\", \"task\":\"summarization\", \"model\":\"gpt-4\"}",
  "status": "UP",
  "weight": "60",
  "healthCheckInterval": "30",
  "createTime": "2024-01-15T10:00:00Z",
  "updateTime": "2024-01-15T10:00:00Z",

  "skill.skillType": "SUMMARIZATION",
  "skill.summary": "自动摘要生成技能，支持多种长度和风格的摘要",
  "skill.useCases": "[\"新闻摘要\", \"文档摘要\", \"会议纪要生成\"]",
  "skill.inputFormat": "长文本字符串，最大支持 100K tokens",
  "skill.outputFormat": "摘要文本，支持自定义长度",
  "skill.inputParameters": "[{\"name\":\"text\",\"type\":\"string\",\"required\":true,\"description\":\"待摘要文本\"},{\"name\":\"maxLength\",\"type\":\"integer\",\"required\":false,\"description\":\"最大摘要长度\",\"defaultValue\":200}]",
  "skill.outputParameters": "[{\"name\":\"summary\",\"type\":\"string\",\"description\":\"生成的摘要\"},{\"name\":\"keyPoints\",\"type\":\"array\",\"description\":\"关键点列表\"}]",
  "skill.requiredModels": "[\"gpt-4\", \"claude-3\"]",
  "skill.examples": "[{\"input\":\"长文本内容...\",\"output\":\"摘要内容\",\"description\":\"新闻摘要示例\"}]",

  "protocol.version": "1.0",
  "protocol.endpointPath": "/api/v1/skill/summarize",
  "protocol.authenticationType": "API_KEY"
}
```

#### 4.9.3 Plugin 服务 Nacos 注册元数据

```json
{
  "resourceId": "plugin-uuid-001",
  "resourceType": "PLUGIN",
  "name": "web-search-plugin",
  "version": "2.0.0",
  "description": "Web搜索插件，提供实时网络搜索能力",
  "protocolType": "API",
  "serviceAddress": "10.0.1.102:8082",
  "capabilities": "[\"web-search\", \"real-time-data\", \"url-fetch\"]",
  "tags": "{\"category\":\"connector\", \"provider\":\"google\", \"rateLimit\":\"100/min\"}",
  "status": "UP",
  "weight": "70",
  "healthCheckInterval": "60",
  "createTime": "2024-01-15T10:00:00Z",
  "updateTime": "2024-01-15T10:00:00Z",

  "plugin.pluginType": "CONNECTOR",
  "plugin.summary": "实时Web搜索插件，支持Google、Bing等搜索引擎",
  "plugin.useCases": "[\"实时信息查询\", \"新闻搜索\", \"学术论文检索\"]",
  "plugin.configurationGuide": "需要配置API Key和搜索引擎偏好",
  "plugin.configParameters": "[{\"name\":\"apiKey\",\"type\":\"string\",\"required\":true,\"description\":\"搜索引擎API Key\"},{\"name\":\"searchEngine\",\"type\":\"string\",\"required\":false,\"description\":\"搜索引擎\",\"defaultValue\":\"google\",\"enumValues\":[\"google\",\"bing\",\"duckduckgo\"]}]",
  "plugin.dependencies": "[]",

  "protocol.version": "1.0",
  "protocol.endpointPath": "/api/v1/plugin/search",
  "protocol.authenticationType": "API_KEY"
}
```

#### 4.9.4 Model 服务 Nacos 注册元数据

```json
{
  "resourceId": "model-uuid-001",
  "resourceType": "MODEL",
  "name": "gpt-4-turbo",
  "version": "2024-04-09",
  "description": "GPT-4 Turbo 模型，支持128K上下文窗口",
  "protocolType": "API",
  "serviceAddress": "10.0.1.103:8083",
  "capabilities": "[\"text-generation\", \"function-calling\", \"vision\", \"128k-context\"]",
  "tags": "{\"provider\":\"openai\", \"tier\":\"premium\", \"availability\":\"high\"}",
  "status": "UP",
  "weight": "90",
  "healthCheckInterval": "30",
  "createTime": "2024-01-15T10:00:00Z",
  "updateTime": "2024-01-15T10:00:00Z",

  "model.modelType": "LLM",
  "model.summary": "OpenAI GPT-4 Turbo，支持文本生成、函数调用、视觉理解",
  "model.bestFor": "[\"复杂推理\", \"代码生成\", \"多轮对话\", \"内容创作\"]",
  "model.notRecommendedFor": "[\"简单分类\", \"关键词提取\"]",
  "model.domains": "[\"通用\", \"编程\", \"创意写作\", \"分析\"]",
  "model.supportedLanguages": "[\"zh-CN\", \"en-US\", \"ja-JP\", \"ko-KR\", \"fr-FR\", \"de-DE\", \"es-ES\"]",
  "model.contextWindowSize": "128000",
  "model.maxOutputLength": "4096",
  "model.inputModalities": "[\"text\", \"image\"]",
  "model.outputModalities": "[\"text\"]",
  "model.functionCallingSupport": "true",
  "model.streamingSupport": "true",
  "model.jsonModeSupport": "true",
  "model.structuredOutputSupport": "true",
  "model.temperatureRange": "{\"min\":0.0,\"max\":2.0}",
  "model.topPRange": "{\"min\":0.0,\"max\":1.0}",
  "model.minGpuCount": "8",
  "model.gpuType": "A100-80G",
  "model.minGpuMemory": "640",
  "model.minMemory": "128",
  "model.minCpuCores": "32",
  "model.pricingModel": "TOKEN_BASED",
  "model.inputPricePer1K": "0.01",
  "model.outputPricePer1K": "0.03",
  "model.currency": "USD",

  "protocol.version": "v1",
  "protocol.endpointPath": "/v1/chat/completions",
  "protocol.authenticationType": "BEARER_TOKEN"
}
```

#### 4.9.5 MCP Server 服务 Nacos 注册元数据

```json
{
  "resourceId": "mcp-uuid-001",
  "resourceType": "PLUGIN",
  "name": "filesystem-mcp-server",
  "version": "1.0.0",
  "description": "文件系统MCP Server，提供文件读写和目录操作能力",
  "protocolType": "MCP",
  "serviceAddress": "10.0.1.104:3000",
  "capabilities": "[\"file-read\", \"file-write\", \"directory-list\"]",
  "tags": "{\"category\":\"tool\", \"type\":\"filesystem\", \"mcp-version\":\"2024-11-05\"}",
  "status": "UP",
  "weight": "50",
  "healthCheckInterval": "30",
  "createTime": "2024-01-15T10:00:00Z",
  "updateTime": "2024-01-15T10:00:00Z",

  "mcp.protocolVersion": "2024-11-05",
  "mcp.transportTypes": "[\"STDIO\", \"HTTP\"]",
  "mcp.serverCapabilities.tools": "true",
  "mcp.serverCapabilities.resources": "true",
  "mcp.serverCapabilities.prompts": "false",
  "mcp.serverCapabilities.logging": "true",

  "mcp.tools": "[{\"name\":\"read_file\",\"description\":\"读取文件内容\",\"inputSchema\":\"{\\\"type\\\":\\\"object\\\",\\\"properties\\\":{\\\"path\\\":{\\\"type\\\":\\\"string\\\",\\\"description\\\":\\\"文件路径\\\"}},\\\"required\\\":[\\\"path\\\"]}\"},{\"name\":\"write_file\",\"description\":\"写入文件内容\",\"inputSchema\":\"{\\\"type\\\":\\\"object\\\",\\\"properties\\\":{\\\"path\\\":{\\\"type\\\":\\\"string\\\"},\\\"content\\\":{\\\"type\\\":\\\"string\\\"}},\\\"required\\\":[\\\"path\\\",\\\"content\\\"]}\"},{\"name\":\"list_directory\",\"description\":\"列出目录内容\",\"inputSchema\":\"{\\\"type\\\":\\\"object\\\",\\\"properties\\\":{\\\"path\\\":{\\\"type\\\":\\\"string\\\"}},\\\"required\\\":[\\\"path\\\"]}\"}]",

  "mcp.resources": "[{\"uri\":\"file:///workspace\",\"name\":\"workspace\",\"description\":\"工作空间根目录\",\"mimeType\":\"inode/directory\"}]",

  "mcp.prompts": "[]",

  "protocol.endpointPath": "/mcp",
  "protocol.authenticationType": "API_KEY"
}
```

#### 4.9.6 MCP Client (Agent with MCP) 服务 Nacos 注册元数据

```json
{
  "resourceId": "agent-mcp-uuid-001",
  "resourceType": "AGENT",
  "name": "coding-assistant-agent",
  "version": "1.0.0",
  "description": "编程助手Agent，通过MCP协议调用外部工具",
  "protocolType": "MCP",
  "serviceAddress": "10.0.1.105:8085",
  "capabilities": "[\"code-generation\", \"code-review\", \"refactoring\"]",
  "tags": "{\"domain\":\"coding\", \"mcp-client\":\"true\"}",
  "status": "UP",
  "weight": "75",
  "healthCheckInterval": "30",
  "createTime": "2024-01-15T10:00:00Z",
  "updateTime": "2024-01-15T10:00:00Z",

  "agent.agentType": "PROACTIVE",
  "agent.summary": "编程助手Agent，支持代码生成、审查和重构",
  "agent.useCases": "[\"代码生成\", \"代码审查\", \"Bug修复\", \"代码重构\"]",
  "agent.taskTypes": "[\"code-generation\", \"code-analysis\"]",
  "agent.contextWindowSize": "100000",
  "agent.streamingSupport": "true",
  "agent.tools": "[{\"name\":\"code_search\",\"description\":\"代码搜索\"},{\"name\":\"run_tests\",\"description\":\"运行测试\"}]",

  "mcp.protocolVersion": "2024-11-05",
  "mcp.clientCapabilities.tools": "true",
  "mcp.clientCapabilities.resources": "true",
  "mcp.clientCapabilities.prompts": "true",

  "protocol.endpointPath": "/mcp",
  "protocol.authenticationType": "BEARER_TOKEN"
}
```

#### 4.9.7 A2A Agent 服务 Nacos 注册元数据

```json
{
  "resourceId": "a2a-uuid-001",
  "resourceType": "AGENT",
  "name": "research-assistant-agent",
  "version": "1.0.0",
  "description": "研究助手Agent，支持A2A协议与其他Agent协作",
  "protocolType": "A2A",
  "serviceAddress": "10.0.1.106:8086",
  "capabilities": "[\"research\", \"analysis\", \"report-generation\", \"a2a-collaboration\"]",
  "tags": "{\"domain\":\"research\", \"a2a-enabled\":\"true\", \"collaboration\":\"true\"}",
  "status": "UP",
  "weight": "65",
  "healthCheckInterval": "30",
  "createTime": "2024-01-15T10:00:00Z",
  "updateTime": "2024-01-15T10:00:00Z",

  "agent.agentType": "MULTI_AGENT",
  "agent.summary": "研究助手Agent，擅长信息收集、分析和报告生成",
  "agent.useCases": "[\"市场调研\", \"竞品分析\", \"技术调研\", \"报告撰写\"]",
  "agent.taskTypes": "[\"research\", \"analysis\", \"writing\"]",

  "a2a.agentCard.name": "Research Assistant Agent",
  "a2a.agentCard.description": "专业的研究助手，能够进行深度信息收集和分析",
  "a2a.agentCard.url": "https://agent.example.com/a2a/research-assistant",
  "a2a.agentCard.version": "1.0.0",
  "a2a.agentCard.documentationUrl": "https://docs.example.com/research-assistant",
  "a2a.agentCard.capabilities.streaming": "true",
  "a2a.agentCard.capabilities.pushNotifications": "true",
  "a2a.agentCard.capabilities.stateTransitionHistory": "true",
  "a2a.agentCard.defaultInputMode": "TEXT",
  "a2a.agentCard.supportedInputModes": "[\"TEXT\", \"FILE\"]",
  "a2a.agentCard.supportedOutputModes": "[\"TEXT\", \"FILE\"]",
  "a2a.agentCard.tags": "[\"research\", \"analysis\", \"reporting\"]",
  "a2a.agentCard.authentication.authType": "OAUTH2",
  "a2a.agentCard.authentication.oauth2Config.authorizationServerUrl": "https://auth.example.com",
  "a2a.agentCard.authentication.oauth2Config.clientId": "research-agent-client",
  "a2a.agentCard.authentication.oauth2Config.scopes": "[\"agent:read\", \"agent:execute\"]",

  "a2a.supportedTaskTypes": "[\"research\", \"analysis\", \"report-generation\"]",
  "a2a.defaultInputMode": "TEXT",
  "a2a.supportedInputModes": "[\"TEXT\", \"FILE\"]",
  "a2a.taskExecutionTimeout": "300",
  "a2a.connectionTimeout": "10",

  "protocol.endpointPath": "/a2a",
  "protocol.authenticationType": "OAUTH2"
}
```

#### 4.9.8 API 协议服务 Nacos 注册元数据

```json
{
  "resourceId": "api-uuid-001",
  "resourceType": "SKILL",
  "name": "sentiment-analysis-api",
  "version": "1.0.0",
  "description": "情感分析API，提供文本情感判断能力",
  "protocolType": "API",
  "serviceAddress": "10.0.1.107:8087",
  "capabilities": "[\"sentiment-analysis\", \"emotion-detection\", \"text-classification\"]",
  "tags": "{\"category\":\"nlp\", \"task\":\"sentiment\", \"format\":\"rest\"}",
  "status": "UP",
  "weight": "55",
  "healthCheckInterval": "30",
  "createTime": "2024-01-15T10:00:00Z",
  "updateTime": "2024-01-15T10:00:00Z",

  "api.apiType": "REST",
  "api.basePath": "/api/v1/sentiment",
  "api.authTypes": "[\"API_KEY\", \"BEARER_TOKEN\"]",
  "api.requestsPerSecond": "100",
  "api.requestsPerMinute": "5000",
  "api.requestsPerDay": "100000",
  "api.concurrentLimit": "50",

  "api.examples": "[{\"name\":\"基本情感分析\",\"description\":\"分析文本情感倾向\",\"requestExample\":\"{\\\"text\\\":\\\"这个产品非常好用！\\\"}\",\"responseExample\":\"{\\\"sentiment\\\":\\\"positive\\\",\\\"confidence\\\":0.95,\\\"emotions\\\":[{\\\"name\\\":\\\"joy\\\",\\\"score\\\":0.8}]}\"}]",

  "protocol.version": "1.0",
  "protocol.endpointPath": "/api/v1/sentiment/analyze",
  "protocol.authenticationType": "API_KEY"
}
```

#### 4.9.9 Nacos 注册元数据使用示例

**Java 代码示例 - 构建 Nacos 元数据：**

```java
/**
 * Nacos 元数据构建器
 */
@Component
public class NacosMetadataBuilder {
    
    /**
     * 构建 Agent 服务的 Nacos 元数据
     */
    public Map<String, String> buildAgentMetadata(AgentMetadata agentMetadata) {
        Map<String, String> metadata = new HashMap<>();
        
        // 通用字段
        metadata.put("resourceId", agentMetadata.getResourceId());
        metadata.put("resourceType", ResourceType.AGENT.name());
        metadata.put("name", agentMetadata.getName());
        metadata.put("version", agentMetadata.getVersion());
        metadata.put("description", agentMetadata.getDescription());
        metadata.put("protocolType", agentMetadata.getProtocolType().name());
        metadata.put("serviceAddress", agentMetadata.getServiceAddress());
        metadata.put("capabilities", JSON.toJSONString(agentMetadata.getCapabilities()));
        metadata.put("tags", JSON.toJSONString(agentMetadata.getTags()));
        metadata.put("status", agentMetadata.getStatus().name());
        metadata.put("weight", String.valueOf(agentMetadata.getWeight()));
        metadata.put("healthCheckInterval", String.valueOf(agentMetadata.getHealthCheckInterval()));
        metadata.put("createTime", agentMetadata.getCreateTime().toString());
        metadata.put("updateTime", agentMetadata.getUpdateTime().toString());
        
        // Agent 专用字段
        if (agentMetadata.getAgentType() != null) {
            metadata.put("agent.agentType", agentMetadata.getAgentType().name());
        }
        if (agentMetadata.getAgentDescription() != null) {
            AgentDescription desc = agentMetadata.getAgentDescription();
            metadata.put("agent.summary", desc.getSummary());
            metadata.put("agent.useCases", JSON.toJSONString(desc.getUseCases()));
            metadata.put("agent.inputDescription", desc.getInputDescription());
            metadata.put("agent.outputDescription", desc.getOutputDescription());
        }
        if (agentMetadata.getCapabilities() != null) {
            AgentCapabilityDefinition caps = agentMetadata.getCapabilities();
            metadata.put("agent.taskTypes", JSON.toJSONString(caps.getTaskTypes()));
            metadata.put("agent.supportedLanguages", JSON.toJSONString(caps.getSupportedLanguages()));
            metadata.put("agent.contextWindowSize", String.valueOf(caps.getContextWindowSize()));
            metadata.put("agent.streamingSupport", String.valueOf(caps.getStreamingSupport()));
            metadata.put("agent.multiModalSupport", String.valueOf(caps.getMultiModalSupport()));
        }
        if (agentMetadata.getTools() != null) {
            metadata.put("agent.tools", JSON.toJSONString(agentMetadata.getTools()));
        }
        
        // 协议字段
        if (agentMetadata.getProtocolConfig() != null) {
            ProtocolConfig proto = agentMetadata.getProtocolConfig();
            metadata.put("protocol.version", proto.getProtocolVersion());
            metadata.put("protocol.endpointPath", proto.getEndpointPath());
            metadata.put("protocol.authenticationType", proto.getAuthenticationType());
        }
        
        return metadata;
    }
    
    /**
     * 构建 Model 服务的 Nacos 元数据
     */
    public Map<String, String> buildModelMetadata(ModelMetadata modelMetadata) {
        Map<String, String> metadata = new HashMap<>();
        
        // 通用字段
        metadata.put("resourceId", modelMetadata.getResourceId());
        metadata.put("resourceType", ResourceType.MODEL.name());
        metadata.put("name", modelMetadata.getName());
        metadata.put("version", modelMetadata.getVersion());
        metadata.put("description", modelMetadata.getDescription());
        metadata.put("protocolType", modelMetadata.getProtocolType().name());
        metadata.put("serviceAddress", modelMetadata.getServiceAddress());
        metadata.put("capabilities", JSON.toJSONString(modelMetadata.getCapabilities()));
        metadata.put("tags", JSON.toJSONString(modelMetadata.getTags()));
        metadata.put("status", modelMetadata.getStatus().name());
        metadata.put("weight", String.valueOf(modelMetadata.getWeight()));
        
        // Model 专用字段
        if (modelMetadata.getModelType() != null) {
            metadata.put("model.modelType", modelMetadata.getModelType().name());
        }
        if (modelMetadata.getModelDescription() != null) {
            ModelDescription desc = modelMetadata.getModelDescription();
            metadata.put("model.summary", desc.getSummary());
            metadata.put("model.bestFor", JSON.toJSONString(desc.getBestFor()));
            metadata.put("model.notRecommendedFor", JSON.toJSONString(desc.getNotRecommendedFor()));
            metadata.put("model.domains", JSON.toJSONString(desc.getDomains()));
            metadata.put("model.supportedLanguages", JSON.toJSONString(desc.getSupportedLanguages()));
        }
        if (modelMetadata.getCapabilities() != null) {
            ModelCapabilityDefinition caps = modelMetadata.getCapabilities();
            metadata.put("model.contextWindowSize", String.valueOf(caps.getContextWindowSize()));
            metadata.put("model.maxOutputLength", String.valueOf(caps.getMaxOutputLength()));
            metadata.put("model.inputModalities", JSON.toJSONString(caps.getInputModalities()));
            metadata.put("model.outputModalities", JSON.toJSONString(caps.getOutputModalities()));
            metadata.put("model.functionCallingSupport", String.valueOf(caps.getFunctionCallingSupport()));
            metadata.put("model.streamingSupport", String.valueOf(caps.getStreamingSupport()));
            metadata.put("model.jsonModeSupport", String.valueOf(caps.getJsonModeSupport()));
        }
        if (modelMetadata.getPricing() != null) {
            PricingInfo pricing = modelMetadata.getPricing();
            metadata.put("model.pricingModel", pricing.getPricingModel().name());
            metadata.put("model.inputPricePer1K", pricing.getInputPricePer1K().toString());
            metadata.put("model.outputPricePer1K", pricing.getOutputPricePer1K().toString());
            metadata.put("model.currency", pricing.getCurrency());
        }
        
        return metadata;
    }
    
    /**
     * 构建 MCP Server 的 Nacos 元数据
     */
    public Map<String, String> buildMcpServerMetadata(
            PluginMetadata pluginMetadata, 
            McpProtocolConfig mcpConfig) {
        Map<String, String> metadata = new HashMap<>();
        
        // 通用字段
        metadata.put("resourceId", pluginMetadata.getResourceId());
        metadata.put("resourceType", ResourceType.PLUGIN.name());
        metadata.put("name", pluginMetadata.getName());
        metadata.put("version", pluginMetadata.getVersion());
        metadata.put("protocolType", ProtocolType.MCP.name());
        metadata.put("serviceAddress", pluginMetadata.getServiceAddress());
        
        // MCP 协议字段
        if (mcpConfig != null) {
            metadata.put("mcp.protocolVersion", mcpConfig.getProtocolVersion());
            metadata.put("mcp.transportTypes", JSON.toJSONString(mcpConfig.getTransportTypes()));
            
            if (mcpConfig.getServerCapabilities() != null) {
                McpServerCapabilities caps = mcpConfig.getServerCapabilities();
                metadata.put("mcp.serverCapabilities.tools", String.valueOf(caps.getTools()));
                metadata.put("mcp.serverCapabilities.resources", String.valueOf(caps.getResources()));
                metadata.put("mcp.serverCapabilities.prompts", String.valueOf(caps.getPrompts()));
                metadata.put("mcp.serverCapabilities.logging", String.valueOf(caps.getLogging()));
            }
            
            if (mcpConfig.getTools() != null) {
                metadata.put("mcp.tools", JSON.toJSONString(mcpConfig.getTools()));
            }
            if (mcpConfig.getResources() != null) {
                metadata.put("mcp.resources", JSON.toJSONString(mcpConfig.getResources()));
            }
            if (mcpConfig.getPrompts() != null) {
                metadata.put("mcp.prompts", JSON.toJSONString(mcpConfig.getPrompts()));
            }
        }
        
        return metadata;
    }
    
    /**
     * 构建 A2A Agent 的 Nacos 元数据
     */
    public Map<String, String> buildA2aAgentMetadata(
            AgentMetadata agentMetadata,
            A2aProtocolConfig a2aConfig) {
        Map<String, String> metadata = new HashMap<>();
        
        // 通用字段
        metadata.put("resourceId", agentMetadata.getResourceId());
        metadata.put("resourceType", ResourceType.AGENT.name());
        metadata.put("name", agentMetadata.getName());
        metadata.put("version", agentMetadata.getVersion());
        metadata.put("protocolType", ProtocolType.A2A.name());
        metadata.put("serviceAddress", agentMetadata.getServiceAddress());
        
        // Agent 通用字段
        if (agentMetadata.getAgentType() != null) {
            metadata.put("agent.agentType", agentMetadata.getAgentType().name());
        }
        
        // A2A 协议字段
        if (a2aConfig != null && a2aConfig.getAgentCard() != null) {
            AgentCard card = a2aConfig.getAgentCard();
            metadata.put("a2a.agentCard.name", card.getName());
            metadata.put("a2a.agentCard.description", card.getDescription());
            metadata.put("a2a.agentCard.url", card.getUrl());
            metadata.put("a2a.agentCard.version", card.getVersion());
            
            if (card.getCapabilities() != null) {
                metadata.put("a2a.agentCard.capabilities.streaming", 
                    String.valueOf(card.getCapabilities().getStreaming()));
                metadata.put("a2a.agentCard.capabilities.pushNotifications", 
                    String.valueOf(card.getCapabilities().getPushNotifications()));
            }
            
            metadata.put("a2a.agentCard.defaultInputMode", 
                card.getDefaultInputMode().name());
            metadata.put("a2a.agentCard.supportedInputModes", 
                JSON.toJSONString(card.getSupportedInputModes()));
            metadata.put("a2a.agentCard.supportedOutputModes", 
                JSON.toJSONString(card.getSupportedOutputModes()));
            
            if (card.getAuthentication() != null) {
                metadata.put("a2a.agentCard.authentication.authType", 
                    card.getAuthentication().getAuthType().name());
            }
        }
        
        if (a2aConfig.getSupportedTaskTypes() != null) {
            metadata.put("a2a.supportedTaskTypes", 
                JSON.toJSONString(a2aConfig.getSupportedTaskTypes()));
        }
        
        return metadata;
    }
    
    /**
     * 从 Nacos 元数据解析为 AgentMetadata
     */
    public AgentMetadata parseAgentMetadata(Map<String, String> nacosMetadata) {
        return AgentMetadata.builder()
            .resourceId(nacosMetadata.get("resourceId"))
            .name(nacosMetadata.get("name"))
            .version(nacosMetadata.get("version"))
            .description(nacosMetadata.get("description"))
            .serviceAddress(nacosMetadata.get("serviceAddress"))
            .agentType(AgentType.valueOf(nacosMetadata.get("agent.agentType")))
            .agentDescription(AgentDescription.builder()
                .summary(nacosMetadata.get("agent.summary"))
                .useCases(JSON.parseArray(nacosMetadata.get("agent.useCases"), String.class))
                .inputDescription(nacosMetadata.get("agent.inputDescription"))
                .outputDescription(nacosMetadata.get("agent.outputDescription"))
                .build())
            .capabilities(AgentCapabilityDefinition.builder()
                .taskTypes(JSON.parseArray(nacosMetadata.get("agent.taskTypes"), String.class))
                .supportedLanguages(JSON.parseArray(nacosMetadata.get("agent.supportedLanguages"), String.class))
                .contextWindowSize(Integer.valueOf(nacosMetadata.get("agent.contextWindowSize")))
                .streamingSupport(Boolean.valueOf(nacosMetadata.get("agent.streamingSupport")))
                .build())
            .status(ServiceStatus.valueOf(nacosMetadata.get("status")))
            .weight(Integer.valueOf(nacosMetadata.get("weight")))
            .build();
    }
}
```

#### 4.9.10 Nacos 元数据查询示例

```java
/**
 * 基于元数据的服务发现
 */
@Service
@Slf4j
@RequiredArgsConstructor
public class MetadataBasedDiscovery {
    
    private final NamingService namingService;
    
    /**
     * 根据资源类型发现服务
     */
    public List<Instance> discoverByResourceType(String group, ResourceType resourceType) {
        try {
            // 使用 Nacos 元数据过滤查询
            return namingService.selectInstances(
                group,
                instance -> resourceType.name().equals(
                    instance.getMetadata().get("resourceType"))
            );
        } catch (NacosException e) {
            log.error("Failed to discover instances: {}", e.getMessage(), e);
            return Collections.emptyList();
        }
    }
    
    /**
     * 根据协议类型发现服务
     */
    public List<Instance> discoverByProtocolType(String group, ProtocolType protocolType) {
        try {
            return namingService.selectInstances(
                group,
                instance -> protocolType.name().equals(
                    instance.getMetadata().get("protocolType"))
            );
        } catch (NacosException e) {
            log.error("Failed to discover instances: {}", e.getMessage(), e);
            return Collections.emptyList();
        }
    }
    
    /**
     * 根据能力标签发现服务
     */
    public List<Instance> discoverByCapability(String group, String capability) {
        try {
            return namingService.selectInstances(
                group,
                instance -> {
                    String capabilities = instance.getMetadata().get("capabilities");
                    if (StringUtils.isBlank(capabilities)) {
                        return false;
                    }
                    List<String> caps = JSON.parseArray(capabilities, String.class);
                    return caps.contains(capability);
                }
            );
        } catch (NacosException e) {
            log.error("Failed to discover instances: {}", e.getMessage(), e);
            return Collections.emptyList();
        }
    }
    
    /**
     * 组合条件发现服务
     */
    public List<Instance> discoverWithConditions(String group, DiscoveryConditions conditions) {
        try {
            return namingService.selectInstances(
                group,
                instance -> matchConditions(instance.getMetadata(), conditions)
            );
        } catch (NacosException e) {
            log.error("Failed to discover instances: {}", e.getMessage(), e);
            return Collections.emptyList();
        }
    }
    
    private boolean matchConditions(Map<String, String> metadata, 
                                   DiscoveryConditions conditions) {
        if (conditions.getResourceType() != null) {
            if (!conditions.getResourceType().name().equals(metadata.get("resourceType"))) {
                return false;
            }
        }
        if (conditions.getProtocolType() != null) {
            if (!conditions.getProtocolType().name().equals(metadata.get("protocolType"))) {
                return false;
            }
        }
        if (conditions.getRequiredCapabilities() != null) {
            String capabilities = metadata.get("capabilities");
            if (StringUtils.isBlank(capabilities)) {
                return false;
            }
            List<String> caps = JSON.parseArray(capabilities, String.class);
            if (!caps.containsAll(conditions.getRequiredCapabilities())) {
                return false;
            }
        }
        if (conditions.getMinWeight() != null) {
            Integer weight = Integer.valueOf(metadata.getOrDefault("weight", "50"));
            if (weight < conditions.getMinWeight()) {
                return false;
            }
        }
        return true;
    }
}

/**
 * 服务发现条件
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DiscoveryConditions {
    private ResourceType resourceType;
    private ProtocolType protocolType;
    private List<String> requiredCapabilities;
    private Integer minWeight;
    private Map<String, String> tags;
}
```

---

## 5. 基础设施

### 5.1 部署架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Kubernetes 集群                                    │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        Ingress Controller                            │  │
│  │                      (Nginx / Traefik)                               │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │                                          │
│  ┌───────────────────────────────▼───────────────────────────────────────┐  │
│  │                         AI Gateway Pods                               │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │  │
│  │  │ Gateway  │  │ Gateway  │  │ Gateway  │  │ Gateway  │             │  │
│  │  │ Pod 1    │  │ Pod 2    │  │ Pod 3    │  │ Pod N    │             │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                  │                                          │
│  ┌───────────────────────────────▼───────────────────────────────────────┐  │
│  │                      AI 服务 Pods                                     │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │  │
│  │  │ Agent    │  │ Skill    │  │ Plugin   │  │ Model    │             │  │
│  │  │ Service  │  │ Service  │  │ Service  │  │ Service  │             │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      基础设施服务                                      │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │  │
│  │  │  Nacos   │  │  Redis   │  │  MySQL   │  │ RocketMQ │             │  │
│  │  │ Cluster  │  │ Cluster  │  │ Cluster  │  │ Cluster  │             │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      语义路由基础设施                                   │  │
│  │  ┌──────────┐  ┌─────────────────────────────────────────────────┐   │  │
│  │  │  Milvus  │  │           Embedding Service                      │   │  │
│  │  │ Cluster  │  │    (text-embedding-ada-002 / BGE / etc.)         │   │  │
│  │  └──────────┘  └─────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 可扩展性策略

**水平扩展：**
- AI Gateway：无状态设计，支持 Pod 水平自动扩缩容
- AI 服务：独立部署，按需扩缩容
- Nacos 集群：3 节点部署，支持动态扩缩容
- Milvus 集群：支持水平扩展向量存储和检索

**垂直扩展：**
- 网关 Pod：JVM 参数调优，GC 策略优化
- 数据库：读写分离，分库分表
- 向量数据库：分片存储，提升检索性能

### 5.3 安全考虑

**网络安全：**
- Pod 间通信使用 mTLS
- 外部访问通过 Ingress 统一入口
- 网络策略限制 Pod 间访问

**数据安全：**
- 敏感数据加密存储（AES-256）
- Token 使用 RS512 签名
- 配置中心敏感配置加密

**访问控制：**
- RBAC 权限模型
- API Key + JWT 双重认证
- IP 白名单访问控制

---

## 6. 开发工作流

### 6.1 本地开发

**环境要求：**
- JDK 17+
- Maven 3.8+
- Docker & Docker Compose
- Nacos 2.x (本地或 Docker)

**快速启动：**

```bash
# 1. 克隆项目
git clone https://github.com/your-org/ai-gateway.git
cd ai-gateway

# 2. 启动依赖服务
docker-compose up -d nacos redis mysql milvus

# 3. 初始化数据库
mysql -h localhost -u root -p < docs/sql/init.sql

# 4. 导入 Nacos 配置
nacos config import -n ai-gateway -f configs/

# 5. 启动网关服务
mvn clean install -DskipTests
cd gateway-bootstrap
mvn spring-boot:run

# 6. 启动管理控制台
cd gateway-console
pnpm install
pnpm dev
```

### 6.2 CI/CD 流水线

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - sonar
  - docker
  - deploy

build:
  stage: build
  script:
    - mvn clean compile
  only:
    - merge_requests
    - main

test:
  stage: test
  script:
    - mvn test
  only:
    - merge_requests
    - main

sonar:
  stage: sonar
  script:
    - mvn sonar:sonar
  only:
    - main

docker:
  stage: docker
  script:
    - docker build -t ai-gateway:${CI_COMMIT_SHA} .
    - docker push registry.cn-hangzhou.aliyuncs.com/ai/ai-gateway:${CI_COMMIT_SHA}
  only:
    - main

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/ai-gateway ai-gateway=registry.cn-hangzhou.aliyuncs.com/ai/ai-gateway:${CI_COMMIT_SHA}
  only:
    - main
  when: manual
```

### 6.3 代码质量

**代码检查：**
- SonarQube 静态分析
- Checkstyle 代码规范
- SpotBugs 潜在缺陷检测

**测试要求：**
- 单元测试覆盖率 >= 80%
- 集成测试覆盖核心流程
- 性能测试验证高并发场景

---

## 7. 监控与可观测性

### 7.1 日志

**日志格式：**
```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "INFO",
  "traceId": "abc123",
  "spanId": "def456",
  "userId": "user001",
  "username": "zhangsan",
  "method": "POST",
  "path": "/api/v1/chat",
  "statusCode": 200,
  "duration": 150,
  "resourceType": "AGENT",
  "resourceId": "agent-001",
  "protocol": "MCP",
  "message": "Request processed successfully"
}
```

**日志配置：**
```yaml
logging:
  level:
    root: INFO
    com.ai.gateway: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{50} - %msg%n"
  file:
    name: /var/log/ai-gateway/gateway.log
```

### 7.2 指标

**核心指标：**

| 指标名称 | 类型 | 描述 | 告警阈值 |
|---------|------|------|---------|
| gateway_requests_total | Counter | 请求总数 | - |
| gateway_requests_duration_seconds | Histogram | 请求延迟 | P99 > 1s |
| gateway_requests_success_rate | Gauge | 成功率 | < 99% |
| gateway_active_connections | Gauge | 活跃连接数 | > 10000 |
| gateway_rate_limit_rejected_total | Counter | 限流拒绝数 | > 100/min |
| gateway_auth_failures_total | Counter | 认证失败数 | > 50/min |
| registry_resources_total | Gauge | 注册资源数 | - |
| registry_healthy_resources | Gauge | 健康资源数 | < total * 0.8 |
| semantic_route_latency_seconds | Histogram | 语义路由延迟 | P99 > 500ms |
| semantic_route_accuracy | Gauge | 语义路由准确率 | < 90% |
| protocol_requests_total | Counter | 各协议请求总数 | - |
| protocol_errors_total | Counter | 各协议错误总数 | - |

**Prometheus 配置：**
```yaml
scrape_configs:
  - job_name: 'ai-gateway'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['gateway:8080']
```

### 7.3 告警

**告警规则：**
```yaml
groups:
  - name: ai-gateway
    rules:
      - alert: HighErrorRate
        expr: rate(gateway_requests_success_rate[5m]) < 0.99
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "AI 网关错误率过高"
          
      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(gateway_requests_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "AI 网关延迟过高"
          
      - alert: LowHealthyResources
        expr: registry_healthy_resources / registry_resources_total < 0.8
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "健康 AI 资源不足"
          
      - alert: SemanticRouteHighLatency
        expr: histogram_quantile(0.99, rate(semantic_route_latency_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "语义路由延迟过高"
```

---

## 8. 风险评估

| 风险 | 影响 | 可能性 | 缓解措施 |
|-----|------|-------|---------|
| Nacos 集群故障 | 高 | 低 | 3 节点部署，定期备份，监控告警 |
| 高并发下性能瓶颈 | 高 | 中 | 水平扩展，缓存优化，JVM 调优 |
| AI 服务不可用 | 高 | 中 | 故障转移，熔断降级，多活部署 |
| 语义路由准确率不足 | 中 | 中 | 持续优化 embedding 模型，提供反馈机制 |
| MCP/A2A 协议兼容性问题 | 中 | 中 | 协议版本管理，兼容性测试 |
| 安全漏洞 | 高 | 低 | 安全审计，渗透测试，及时更新 |
| 配置错误导致故障 | 中 | 中 | 配置版本管理，灰度发布，回滚机制 |
| 数据丢失 | 高 | 低 | 定期备份，主从复制，异地容灾 |
| 向量数据库性能瓶颈 | 中 | 低 | 分片存储，索引优化，缓存热门向量 |

---

## 9. 实施路线图

### 第一阶段：基础框架（第 1-2 周）

- [ ] 项目脚手架搭建
- [ ] Nacos 集成配置
- [ ] 基础网关过滤器链
- [ ] 数据库设计与初始化
- [ ] 本地开发环境搭建

### 第二阶段：注册中心（第 3-4 周）

- [ ] Agent/Skill/Plugin/Model 注册模块开发
- [ ] API 协议支持
- [ ] 健康检查机制
- [ ] 元数据管理基础功能

### 第三阶段：MCP/A2A 协议（第 5-6 周）

- [ ] MCP 协议适配器开发
- [ ] A2A 协议适配器开发
- [ ] 协议转换层实现
- [ ] 协议兼容性测试

### 第四阶段：智能路由（第 7-8 周）

- [ ] 静态路由引擎开发
- [ ] 语义路由引擎开发
- [ ] Milvus 集成
- [ ] 负载均衡策略实现

### 第五阶段：安全与运维（第 9-10 周）

- [ ] 认证鉴权模块开发
- [ ] RBAC 权限控制
- [ ] 审计日志系统
- [ ] 限流熔断机制

### 第六阶段：监控与可观测性（第 11-12 周）

- [ ] SkyWalking 链路追踪集成
- [ ] Prometheus 指标采集
- [ ] Grafana 监控大盘
- [ ] 告警规则配置

### 第七阶段：生产就绪（第 13-14 周）

- [ ] 性能测试与优化
- [ ] 安全测试与加固
- [ ] 部署文档编写
- [ ] 灰度发布上线

---

## 附录

### A. 术语表

| 术语 | 定义 |
|-----|------|
| AI Gateway | AI 网关，AI 能力的统一接入层 |
| Agent | 智能体，具有自主决策能力的 AI 组件 |
| Skill | 技能，特定领域的 AI 能力单元 |
| Plugin | 插件，可扩展的功能模块 |
| Model | 模型，大语言模型或专用模型服务 |
| MCP | Model Context Protocol，模型上下文协议 |
| A2A | Agent-to-Agent，Agent 间通信协议 |
| Nacos | 阿里开源的服务发现与配置管理平台 |
| RBAC | 基于角色的访问控制 |
| JWT | JSON Web Token，用于身份认证 |
| 语义路由 | 基于请求内容语义理解的智能路由 |
| 向量检索 | 基于向量相似度的信息检索 |
| 熔断 | 当服务故障率超过阈值时，自动停止调用以防止雪崩 |
| 限流 | 控制请求速率，防止系统过载 |

### B. 参考资料

- [Spring Cloud Gateway 官方文档](https://docs.spring.io/spring-cloud-gateway/docs/)
- [Nacos 官方文档](https://nacos.io/docs/)
- [Spring Security 官方文档](https://docs.spring.io/spring-security/reference/)
- [SkyWalking 官方文档](https://skywalking.apache.org/docs/)
- [MCP 协议规范](https://spec.modelcontextprotocol.io/)
- [A2A 协议规范](https://github.com/google/a2a)

### C. 配置示例

```yaml
# application.yml
server:
  port: 8080

spring:
  application:
    name: ai-gateway
  cloud:
    nacos:
      discovery:
        server-addr: nacos-server:8848
        namespace: ai-gateway
        group: AI_GATEWAY_GROUP
      config:
        server-addr: nacos-server:8848
        namespace: ai-gateway
        file-extension: yaml
    gateway:
      routes:
        - id: ai-agent-route
          uri: lb://ai-agent-service
          predicates:
            - Path=/api/v1/agent/**
          filters:
            - StripPrefix=2
      default-filters:
        - name: CircuitBreaker
          args:
            name: ai-service-cb
            fallbackUri: forward:/fallback/ai-service

# JWT 配置
jwt:
  secret: ${JWT_SECRET:your-256-bit-secret-key}
  access-expiration: 3600
  refresh-expiration: 86400

# 限流配置
rate-limit:
  enabled: true
  default-limit: 100
  default-window: 60

# 语义路由配置
semantic-route:
  enabled: true
  embedding-service:
    url: ${EMBEDDING_SERVICE_URL:http://localhost:8081}
    model: text-embedding-ada-002
  vector-store:
    type: milvus
    host: milvus-server
    port: 19530
    collection: ai_resource_vectors
  top-k: 5
  similarity-threshold: 0.7

# MCP 协议配置
mcp:
  enabled: true
  default-transport: HTTP
  timeout: 30000

# A2A 协议配置
a2a:
  enabled: true
  default-timeout: 60000
```

---

**文档版本**: v2.0
**最后更新**: 2024-01-15
**作者**: AI 架构师团队
