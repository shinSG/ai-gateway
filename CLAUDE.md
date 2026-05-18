# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI Gateway — an enterprise-grade AI service gateway providing a unified entry point for AI capabilities. It manages service registration/discovery, intelligent routing (including semantic routing), security, and observability across multiple AI protocols.

## Project Status

This is currently a **design-only** repository. The only substantive file is `ai-gateway-architecture.md` containing the full architecture design. No implementation code exists yet. Future work should follow the module structure and tech stack defined in that document.

## Architecture

**Core modules** (as defined in the architecture doc):

- `gateway-bootstrap` — Spring Boot application entry point
- `gateway-core` — Filters, route engine, registry client, protocol handling, security
- `gateway-registry` — Unified service registration (ServiceType: API/MCP/ACP/A2A), role-based capability management, health checks, metadata management
- `gateway-router` — Static routing, semantic routing (vector-based), load balancing, capability matching, failover
- `gateway-security` — Authentication (OAuth2/JWT), authorization (RBAC), audit logging, token management
- `gateway-admin` — Management console backend (controllers, services, repositories)
- `gateway-common` — Shared models, exceptions, utilities
- `gateway-console` — Vue 3 + Element Plus frontend

**Key design decisions:**

- Semantic routing uses vector similarity (Milvus + Embedding service) to match incoming requests to the best AI service
- Services register to Nacos under a unified `ServiceType` (API/MCP/ACP/A2A) — not split by resource type and protocol separately
- Each service also has a `ServiceRole` (AGENT/SKILL/PLUGIN/MODEL) describing its internal role, used for semantic routing and capability matching
- Nacos metadata uses prefix conventions: `api.*`, `mcp.*`, `acp.*`, `a2a.*` for protocol-specific fields; `role.*` for role-specific fields
- Multi-protocol support: REST API, MCP (Model Context Protocol), ACP (Agent Communication Protocol), A2A (Agent-to-Agent)
- `ServiceType` enum: API, MCP, ACP, A2A
- `ServiceRole` enum: AGENT, SKILL, PLUGIN, MODEL

## Tech Stack

Java 17+, Spring Boot 3.x, Spring Cloud Gateway, Nacos 2.x, Redis 7+, MySQL 8.0 + MyBatis-Plus, Milvus/Weaviate (vector DB), RocketMQ/Kafka, SkyWalking, Prometheus + Grafana, Docker + Kubernetes, Vue 3 + Element Plus (frontend).

## Working with the Architecture Document

The file `ai-gateway-architecture.md` is the single source of truth. It contains:
- `ServiceType` (API/MCP/ACP/A2A) and `ServiceRole` (AGENT/SKILL/PLUGIN/MODEL) unified type system
- Protocol config classes: `ApiProtocolConfig`, `McpProtocolConfig`, `AcpProtocolConfig`, `A2aProtocolConfig`
- Role config classes: `AgentRoleConfig`, `SkillRoleConfig`, `PluginRoleConfig`, `ModelRoleConfig`
- Nacos metadata KV structures for each service type + role combination
- `NacosMetadataBuilder` and `MetadataBasedDiscovery` reference implementations
- Semantic routing engine pseudocode (`SemanticRouteEngine`)
- CI/CD pipeline configuration (GitLab CI)
- Deployment architecture (Kubernetes)
- Prometheus metrics, alerting rules, and log format specs
