# AgentDock

> 一条完整的"博客 agent 业务链"，按"平台雏形"设计——未来演进为管理 N 条 agent 链的 runtime 平台的最小完整实例。

- 📋 开发方案总纲：见 [`PLAN.md`](PLAN.md)（宪法性文档）
- 🧭 架构决策记录：见 `docs/decisions/`（ADR）

## 架构总览

```
┌────────────────────────────────────────────────────────────────┐
│  访问层：博客（VitePress 静态站 + 检索助手 + 沙箱演示页）          │
└──────────────────────────────┬─────────────────────────────────┘
                               │ HTTP / SSE（流式）
┌──────────────────────────────▼─────────────────────────────────┐
│  控制面 Control Plane（可信代码，正常部署）                       │
│                                                                │
│  agent-supervisor（Python FastAPI + LangGraph）                 │
│    Supervisor ──Handoff──→ Chat / RAG Expert / Code Runner      │
│    · Postgres checkpointer（thread_id 断点续传）                 │
│    · Langfuse 全链路埋点                                        │
│                                                                │
│  agentdock-core（Java Spring Boot 3.x 平台控制面）               │
│    · API 网关 · 语义缓存 · MCP 工具注册表 · 评测 runner          │
└────────┬──────────────────────────────┬────────────────────────┘
         │ MCP（Streamable HTTP）        │ Docker API
┌────────▼───────────────┐  ┌───────────▼────────────────────────┐
│  数据面 1：业务 MCP     │  │  数据面 2：沙箱（多 profile）         │
│  · rag-mcp（Java）     │  │  · isolated：断网/降权/限资源+超时   │
│  · memory-mcp（Java）  │  │  · web-scrape：仅经 egress-proxy    │
│                        │  │    （SNI白名单+限流+超时+审计）      │
└────────────────────────┘  └────────────────────────────────────┘
        基建：PostgreSQL · Redis · Elasticsearch
        编排：docker-compose 单机（每 service = 未来 K8s Deployment）
```

## 进度

| 里程碑 | 主题 | 状态 |
|---|---|---|
| M0 | 地基：控制面骨架 + lumina 精华移植 | 🚧 进行中 |
| M1 | 大脑：LangGraph Supervisor + Checkpoint | ⬜ |
| M2 | 数据面 MCP 双件套：RAG + 记忆 | ⬜ |
| M3 | 沙箱：多 profile 隔离执行 + 网络攻防实录 | ⬜ |
| M4 | 观测 + 评测：数据说话 | ⬜ |
| M5 | 博客即产品入口 | ⬜ |
| M6 | 云原生演练：Minikube + HPA | ⬜ |
| M7+ | 平台化演进 | ⬜ |

> 📊 数据区（M4 填充）：Langfuse trace 截图 / eval 跑分 / 缓存命中率 benchmark / 沙箱攻击测试报告

## 演进前传：lumina-framework

本项目的知识库检索、语义缓存与并发护城河经验来自我的 v0 项目 [lumina-framework](https://github.com/Gastertablook/lumina-framework)（Spring Boot Agentic RAG Starter）。

lumina 中：singleflight、双层语义缓存、血缘失效为真实实现（已移植至 AgentDock 的 `agentdock-core`）；LangGraph（自研简化版）、Langfuse（基础埋点）、MCP（手写 SSE）为早期原型实现——这三者在 AgentDock 中均改用行业标准技术重新实现（真 LangGraph / 真 Langfuse 链路 / 官方 MCP SDK + Streamable HTTP）。

> 注：lumina 仓库保持原样、零修改（只读参考）；移植 = 复制零件适配，而非沿用其 Starter 架构。

## 快速开始

```bash
docker compose -f deploy/docker-compose.yml up -d   # M0 完成前暂不可用
```

## 许可证

Apache-2.0
