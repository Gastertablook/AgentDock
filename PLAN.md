# AgentDock 开发方案总纲（PLAN.md）

> 版本 v1.0 · 2026-11-19 · 状态：已确认，作为后续开发的宪法性文档
> 本文档由 Claude Code 与项目所有者共同制定，融合了：前期规划对话中的有效结论（见 `docs/private/`）、2026 年 agent 基础设施行业调研、lumina-framework 代码审计结果。

---

## 1. 项目定位（一句话）

**AgentDock 是一条完整的"博客 agent 业务链"，但架构按"平台雏形"设计——它是未来演进为"管理 N 条 agent 链的 runtime 平台"的最小完整实例。**

三个目标的统一方式：

| 目标 | 落地方式 |
|---|---|
| 自用产品 | 博客文章是我的知识库数据源，检索助手/记忆/沙箱真实服务于我的写作与学习 |
| 求职作品 | 访问者打开博客 = 同时看到：文章深度 + 产品完成度 + 真实工程数据（trace 截图/评测跑分/缓存命中率） |
| 长期演进 | 控制面/数据面分离、agent 即微服务、MCP 注册表——平台化的接口从第一天就预留 |

### 1.1 北极星：个人 AI 助理（Jarvis 愿景，2026-11-24 纳入）

长期愿景：AgentDock 演进为**个人网络 AI 助理**——记住我所有的思考与开发经验（记忆层）、构建并管理各种特定功能的 agent（平台层）、获取适合我的行业前沿消息（信息采集管线）、向别人介绍我自己（个人画像 agent）。

**演进阶梯**（每一级都是上一级的真实使用方）：
1. 一条 agent 业务链（当前）→ 2. 管理 N 条 agent 链的平台 → 3. 平台之上的个人助理服务（Jarvis）

**现阶段纪律**：Jarvis 新增能力（信息采集管线、个人画像 agent、消息触达）只写入 M7+ backlog，不做细节规划——先让平台成型。这是"愿景验证可行性、同时防范围蔓延（风险 R1）"的两全方式：每一步都落在既有里程碑上，不新增里程碑。

---

## 2. 背景与动机

### 2.1 岗位画像与技术要求（决定了技术点的优先级）

> 完整面试背景与求职叙事见私有文档 `docs/private/求职叙事.md`（不入库）。

| 岗位方向 | 考察点 | 在方案中的对应 |
|---|---|---|
| AI Infra / Agent 平台（ToB 重型底座） | 状态恢复、高可用、弹性伸缩 | M1 checkpoint 断点续传 demo、M6 K8s 演练 |
| Agent 平台（ToC 低延迟/安全/记忆） | harness 安全沙箱、网络出站拦截、权限审批、长短期记忆 | M3 沙箱攻防（含 egress-proxy 网络出站分层拦截）、多 agent 工具权限物理隔离、M2 记忆 MCP |
| 目标技术点 | Claude Code 端侧 Agent、LangGraph、Langfuse、MCP | 全部用真技术重新落地（见 2.2） |

### 2.2 lumina-framework 的定位（"演进前传"）

lumina 的真实状态：**缓存/并发部分（singleflight、双层语义缓存、血缘失效）为真实实现；LangGraph（自研简化版）、Langfuse（基础埋点）、MCP（手写 SSE）为早期原型实现，未经过生产流量验证。**

处理方式：
1. **移植收窄**：只移植 singleflight、双层缓存、血缘失效三块真实代码（血缘失效的单机实现由 Kafka 降级为 Redis Pub/Sub，见 ADR-001）。
2. **重做而非升级**：LangGraph → 真 Python LangGraph；Langfuse → 真埋点 + 真实流量数据；MCP → 官方 SDK + Streamable HTTP。
3. **lumina 仓库零修改**（用户明确要求，2026-11-24）：lumina 作为只读参考代码库，不做任何改动；"演进前传"叙事只在 AgentDock 侧（README + 博客）讲述。移植 = 复制零件进新仓库适配，**不是沿用其架构**——lumina 的 Starter 架构撑不起平台；其 RAG 能力作为 rag-mcp 的素材来源，缓存/并发零件作为控制面的素材来源。

> **诚信原则（最高优先级）**：公开材料（README/简历/博客）上写的每一行，都必须能在 AgentDock 仓库里指出对应代码和真实数据。个人求职叙事与面试复盘维护在 `docs/private/`（不入库）。

---

## 3. 设计原则

1. **真实业务闭环优先**：不做没有真实流量的空壳框架。博客内容 = 知识库数据源，写代码的同时真实使用。
2. **用真技术，实做优先**：行业标准组件（LangGraph/MCP/Langfuse/评测体系），不用自研简化版冒充。
3. **每里程碑可交付**：2-4 周时间盒，"完成"的定义是**能用**而非完美；每里程碑强制产出一篇博客。
4. **单机务实 + 平台化演进**：当前 docker-compose 单机运行；每个 service 与未来 K8s Deployment 一一对应；平台化接口第一天预留。
5. **写作与开发并行**：文章不是项目结束后的总结，而是每个里程碑的验收产物。

---

## 4. 总体架构

```
┌────────────────────────────────────────────────────────────────┐
│  访问层：博客（VitePress 静态站 + 检索助手 + 沙箱演示页）          │
│         （M5 上线；检索助手限流开放，沙箱仅演示模式）              │
└──────────────────────────────┬─────────────────────────────────┘
                               │ HTTP / SSE（流式）
┌──────────────────────────────▼─────────────────────────────────┐
│  控制面 Control Plane（可信代码，正常部署）                       │
│                                                                │
│  agent-supervisor（Python FastAPI + LangGraph）                 │
│    Supervisor ──Handoff──→ Chat / RAG Expert / Code Runner      │
│    · Postgres checkpointer（thread_id 断点续传）                 │
│    · langchain-mcp-adapters 挂载全部 MCP                        │
│    · Langfuse callback 全链路埋点                                │
│                                                                │
│  agentdock-core（Java Spring Boot 3.x 平台控制面）               │
│    · API 网关（会话管理、SSE 转发、模型路由预留）                 │
│    · 语义缓存（移植 singleflight + 双层缓存 + 血缘失效）          │
│    · MCP 工具注册表（M7+ 演进为 Web UI）                         │
│    · 评测 runner（M4）                                          │
└────────┬──────────────────────────────┬────────────────────────┘
         │ MCP（Streamable HTTP）        │ Docker API
┌────────▼───────────────┐  ┌───────────▼────────────────────────┐
│  数据面 1：业务 MCP     │  │  数据面 2：沙箱（多 profile）         │
│  · rag-mcp（Java）     │  │  · isolated（默认）：--network none  │
│    博客知识库混合检索    │  │    --cap-drop ALL --memory 512m    │
│  · memory-mcp（Java）  │  │    --cpus 0.5 --pids-limit 64       │
│    长期画像+短期窗口    │  │    --read-only + 超时强杀           │
│                        │  │  · web-scrape：仅经 egress-proxy    │
│                        │  │    （SNI白名单+QPS限流+超时+审计）   │
│                        │  │  （演进蓝图：gVisor/Firecracker/eBPF）│
└────────────────────────┘  └────────────────────────────────────┘
        基建：PostgreSQL（checkpoint+元数据）· Redis（缓存+记忆）· ES（检索+画像）
        编排：docker-compose 单机（每个 service = 未来 K8s Deployment）
```

**零信任边界判定标准**：
- 沙箱只用于"执行大模型现场生成的代码"这一个点；
- 业务 MCP（rag/memory）是受控 API 调用，参数校验 + 权限控制即可，不需要沙箱；
- Supervisor 是可信代码，正常部署，必须有外网（调模型 API）。

**沙箱网络策略三层（零信任出口，见 ADR-003）**：
- `isolated`（默认断网档）：通用代码执行，绝对隔离；
- `web-scrape`（受控出口档）：爬虫/信息采集类任务，沙箱唯一网络路径是 egress-proxy——域名白名单 + TLS SNI 校验 + 每沙箱 QPS 限流 + idle 连接超时 + 并发连接上限 + 全量审计日志；
- 特权操作（演进蓝图）：MCP 代理模式——沙箱永不获权，宿主侧 Daemon 校验后代为执行。

**核心认知**：沙箱的本质不是"永远断网"，而是"默认拒绝 + 按需受控"。断网只是默认档位；爬虫需要联网 ≠ 沙箱无用，而是需要第二档位。

---

## 5. 技术栈清单

| 组件 | 选型 | 理由 | 备选 |
|---|---|---|---|
| 编排层 | Python 3.12 + LangGraph（官方） | 行业标准，checkpointer/Langfuse 生态最成熟，岗位对齐 | langgraph.js（不选：放弃 Java 存量故事） |
| 编排服务框架 | FastAPI | 轻量、SSE 流式友好 | Flask |
| 控制面 | Java 17/21 + Spring Boot 3.x | 你的存量优势；**禁用 2.7（已 EOL）** | Quarkus |
| MCP 实现 | 官方 MCP Java SDK（io.modelcontextprotocol.sdk） | 协议级掌控，Streamable HTTP（2026 spec 基线，SSE 已废弃） | Spring AI MCP starter（封装厚，不利于讲协议细节） |
| Checkpoint | LangGraph PostgresSaver | 官方持久化方案 | RedisSaver（仅限小型状态） |
| 检索 | Elasticsearch 8.x（BM25 + 向量混合检索） | 复用 lumina 的检索经验，中文分词成熟 | pgvector（简化，但失去混合检索故事） |
| 缓存 | Redis 7（语义缓存 + 短期记忆） | 与 lumina 移植代码一致 | — |
| 沙箱 | Docker（docker-java 库调度） | 零成本，行业普遍底座 | Firecracker/gVisor（写入演进蓝图，不实现） |
| 观测 | Langfuse（Cloud 免费版优先，自托管备选） | 零运维起步；数据可导出 | 自托管（需 PG+ClickHouse+Redis+S3，单机重） |
| 评测 | 自建评测集 + LLM-as-judge + 规则校验 | 2026 年 agent 工程的核心能力 | LangSmith（收费） |
| 博客 | VitePress 静态站 | 免费托管（GitHub Pages）、零运维 | Hexo |
| 模型 | 默认 GLM-4-Flash（便宜+国内直连），网关层预留多模型路由 | lumina 已验证可用 | DeepSeek/Qwen 按实测切换 |
| 消息 | Redis Pub/Sub（单机版血缘失效） | Kafka 在单机是过度工程 | 生产环境再换 Kafka（ADR-001 记录） |
| 开发环境 | Windows 11 + WSL2 + Docker Desktop（前置条件） | 当前机器 | — |

---

## 6. 仓库结构（monorepo）

```
AgentDock/
├── PLAN.md                    # 本文档（宪法）
├── README.md                  # 项目总览：架构图 + 真实数据截图 + lumina 演进说明
├── docs/
│   ├── decisions/             # ADR：每个技术决策写一句为什么
│   ├── attack-report/         # M3 沙箱攻击用例测试报告
│   └── private/               # 个人过程文档，gitignore 不入库
├── agent-supervisor/          # Python：LangGraph 编排（大脑）
│   ├── pyproject.toml
│   └── src/agentdock_supervisor/
│       ├── api.py             # FastAPI /chat 流式接口
│       ├── graph.py           # 状态图
│       ├── agents/            # supervisor / chat / rag / coder
│       └── checkpointer.py
├── agentdock-core/            # Java：平台控制面（网关/缓存/注册表/评测）
├── mcp-servers/
│   ├── rag-mcp/               # 博客知识库检索
│   ├── memory-mcp/            # 长短期记忆
│   └── sandbox-mcp/           # Docker 沙箱调度
├── blog/                      # VitePress（M5 起）
├── evals/                     # 评测集（jsonl）+ 跑分脚本（版本化！）
├── sandbox-images/            # 沙箱最小镜像 Dockerfile
└── deploy/
    ├── docker-compose.yml     # 单机编排（逻辑边界 = 未来 K8s 边界）
    └── k8s/                   # M6：Deployment/Service/HPA yaml
```

---

## 7. 里程碑总览

| 里程碑 | 主题 | 周期 | 核心验收 |
|---|---|---|---|
| M0 | 地基：控制面骨架 + lumina 精华移植 | 第 1-2 周 | compose 一键起全栈，移植代码有测试 |
| M1 | 大脑：LangGraph Supervisor + Checkpoint | 第 3-5 周 | 真 LangGraph 多 agent 路由 + kill 进程断点续传 |
| M2 | 数据面 MCP 双件套：RAG + 记忆 | 第 6-8 周 | 协议级 MCP 可发现；检索准确；重启不失忆 |
| M3 | 沙箱：多 profile 隔离执行 + 网络攻防实录 | 第 9-12 周 | 爬虫 agent 端到端；恶意代码与网络攻击全部阻断 |
| M4 | 观测 + 评测：数据说话 | 第 13-15 周 | Langfuse 真实 trace 截图；eval 跑分可重复 |
| M5 | 博客即产品入口（形态 B） | 第 16-18 周 | 访问者 10 分钟看到内容+产品+数据 |
| M6 | 云原生演练：Minikube + HPA（独立） | 第 19-20 周 | 压测下自动扩缩容记录 |
| M7+ | 平台化演进（长期 backlog） | 无压力 | 见第 9 节 |

---

## 8. 里程碑详情（任务 + 验收标准 + 产出）

### M0 地基（第 1-2 周）
**任务**
1. `git init`；Apache-2.0 许可证；README 骨架（架构图 + 演进说明，数据截图位置留空待 M4 填充）
2. 隐私材料（对话记录、面试复盘）移入 `docs/private/`（已 gitignore，不入库）
3. 通读 lumina 代码，整理移植清单（**lumina 仓库零修改，只读参考**；RAG 能力 → rag-mcp 素材，缓存/并发 → 控制面零件）
4. `agentdock-core` 工程初始化（Spring Boot 3.x + Maven）
5. `deploy/docker-compose.yml`：PostgreSQL 16 / Redis 7 / ES 8（限制 JVM heap）
6. 移植 lumina：`RequestDeduplicator`、双层语义缓存、血缘失效（Redis Pub/Sub 替代 Kafka）+ 单元测试
7. 写 ADR-001（Kafka→Pub/Sub 降级理由）、ADR-002（SSE→Streamable HTTP 理由）

**验收标准（DoD）**
- [ ] `docker compose up` 一键起全部基建，`/health` 通过
- [ ] 移植代码单测全绿
- [ ] 本地可打出缓存命中率数字

### M1 大脑（第 3-5 周）
**任务**
1. Python 工程 `agent-supervisor`：FastAPI + 官方 LangGraph
2. Supervisor + Chat/RAG Expert（占位）/Code Runner（占位），transfer 工具 + `Command(goto=...)` 实现 Handoff
3. PostgresSaver checkpointer，thread_id 会话管理
4. 网关 → supervisor 的 SSE 流式链路打通（第一天就用 curl 验证流式，见风险 R10）
5. 断点续传演示：长任务执行中 kill 进程 → 重启恢复 → 录屏

**验收标准（DoD）**
- [ ] 跑的是真 LangGraph（非自研简化版）
- [ ] 多 agent 路由正确（闲聊/检索/执行三类意图各 5 条测试）
- [ ] 断点续传录屏（状态恢复场景）
- [ ] 产出博客 #1《从自研状态机到 LangGraph：手写图引擎错在哪》（呼应 lumina 演进故事）

### M2 数据面 MCP 双件套（第 6-8 周）
**任务**
1. 语料导入：技术复盘、技术笔记、lumina 文档 → ES（BM25 + 向量混合检索）
2. `rag-mcp`（官方 MCP Java SDK，Streamable HTTP）：`search_articles` / `get_article`
3. `memory-mcp`：短期 = Redis 滑动窗口（最近 10 轮）；长期 = ES 用户画像向量（长期画像能力）
4. Supervisor 经 langchain-mcp-adapters 挂载两个 MCP
5. 协议级验证：用 MCP Inspector 检查 `tools/list` 可发现性

**验收标准（DoD）**
- [ ] MCP 协议符合 2026 spec（Streamable HTTP，非 SSE）
- [ ] 检索人工测试 10 问 ≥ 8 中
- [ ] 重启后仍记得用户偏好（记忆生效）
- [ ] 产出博客 #2《MCP 2026：SSE 已死，Streamable HTTP 当立》

### M3 沙箱（第 9-12 周）
**任务**
1. `sandbox-mcp`（docker-java 调度）支持多 profile：
   - `isolated`（默认）：`--network none --cap-drop ALL --memory 512m --cpus 0.5 --pids-limit 64 --read-only + tmpfs`，超时 30s 强杀，stdout/stderr 截断
   - `web-scrape`：沙箱容器唯一网络路径是 egress-proxy（见任务 3）
2. Code Runner agent 正式接入，按任务类型选择 profile
3. **egress-proxy（受控出口代理）**：自研 Java 轻量实现——HTTP CONNECT + TLS SNI 解析 + 域名白名单 + 每沙箱 token bucket 限流 + idle 连接超时 + 最大并发连接数 + 全量审计日志；实现顺序：先用 Squid 配置快速验证链路，再替换为自研版
4. 端到端验收场景升级："开发一个'采集几大科技公司最新文章'的爬虫 agent"（贾维斯信息采集管线的爬取环节，对应 M7+ news-digest agent）
5. 攻击用例测试（全部录屏，报告归档 `docs/attack-report/`）：
   - 恶意代码：`rm -rf /`、base64 混淆绕过、挖矿脚本、死循环
   - 网络攻防：外连请求（isolated 档）、非白名单域名访问（web-scrape 档）、高并发请求（QPS 限流生效）、长连接挂起（idle 超时生效）、内网 SSRF 探测（网络隔离生效）

**验收标准（DoD）**
- [ ] isolated 档恶意代码全部阻断；web-scrape 档白名单外流量 100% 拒绝
- [ ] 爬虫 agent 在白名单内正常采集；限流/超时/连接上限均被攻击用例触发验证
- [ ] 产出博客 #3《给不可信代码的牢笼：agent 沙箱攻防实录》（含网络策略分层章节）
- [ ] README 架构图标注演进蓝图（gVisor/Firecracker/eBPF = 终局，不实现）

### M4 观测 + 评测（第 13-15 周）
**任务**
1. Langfuse 全链路埋点（LangGraph callback + Java 侧 SDK），收集真实流量
2. 截图三件套进 README：trace 瀑布图、token 消耗趋势、节点耗时分布
3. 评测体系：`evals/` 下 ≥30 条评测集（意图路由 10 / 检索质量 10 / 代码执行 5 / 安全负例 5），版本化；跑分脚本；LLM-as-judge + 人工抽检
4. 缓存命中率 benchmark（证明移植代码的价值）

**验收标准（DoD）**
- [ ] 修改一个 prompt 前后，eval 分数可见差异（评测体系真实可用）
- [ ] Langfuse 截图 + eval 跑分进入 README
- [ ] 产出博客 #4《没有评测的 agent 等于没有测试》

### M5 博客即产品入口（第 16-18 周）
**任务**
1. VitePress 博客上线：行业思考 + 开发过程文章；"关于 AgentDock"页（架构图 + M4 真实数据）
2. 检索助手接入（rag-mcp，Redis token bucket 限流开放）
3. 沙箱演示页：只允许预置 demo 脚本 + 录屏，**不开放任意代码执行**（防滥用/成本）
4. 托管决策（OQ1）：静态站 GitHub Pages（免备案）；助手后端本地演示 + 录屏兜底，上云方案备案后再定

**验收标准（DoD）**
- [ ] 访问者 10 分钟内看到：内容深度 + 产品可用 + 真实数据
- [ ] 产出博客 #5《AgentDock：一条 agent 业务链如何长成平台雏形》

### M6 云原生演练（第 19-20 周，独立于主链路）
**任务**
1. 全部服务 Dockerfile 化 → Minikube：Deployment/Service
2. HPA（CPU 指标 + 自定义指标：排队 agent 任务数）+ JMeter 压测
3. 扩缩容过程录屏

**验收标准（DoD）**
- [ ] 压测下 Pod 1→N 自动扩容记录（弹性伸缩场景）
- [ ] 产出博客 #6《从 docker-compose 到 K8s：单机平台的平移术》

---

## 9. M7+ 平台化演进 backlog（无压力长期，按需认领）

- [ ] Agent 定义 manifest（`agent.yaml` / AGENTS.md 风格）：声明模型/工具/权限 → 注册进平台
- [ ] MCP 工具注册表 Web UI：挂载/卸载/权限（平台雏形的核心卖点）
- [ ] 多租户全链路隔离：tenant_id 贯穿会话/缓存 key/checkpoint 命名空间（M0 起数据模型就要带，见风险 R11）
- [ ] Human-in-the-loop 审批门（权限审批能力）
- [ ] 信息采集管线（news-digest agent，贾维斯第一步）：定时采集白名单科技媒体 → 摘要 → 按个人画像过滤 → 推送（M3 爬虫场景的完整化）
- [ ] 个人画像 agent（"介绍我自己"）：个人知识库 + 画像维护 + 问答
- [ ] 消息触达通道（邮件/微信等，个人使用；需考虑公网常驻服务的托管与备案，见 OQ1）
- [ ] 开发经验沉淀闭环：每里程碑复盘 → 知识库回流 → 记忆层（贾维斯的记忆）
- [ ] "爬虫流水线"沉淀为平台第一个可复用 agent 模板
- [ ] 多模型路由 + 降级（网关层预留接口）
- [ ] A2A 协议互通 demo（agent 间互操作，2026 年 3 月已 GA，了解为主）
- [ ] 沙箱升级探索：gVisor / Kata（学习性质，不承诺落地）

---

## 10. 风险登记册（复盘结果）

| # | 风险 | 概率×影响 | 缓解措施 |
|---|---|---|---|
| R1 | **范围蔓延**（第三次"没概念"） | 高×高 | 本文档即宪法；新想法一律进 M7+ backlog，禁止插入进行中的里程碑；每里程碑开始只认领当周任务 |
| R2 | **拖延烂尾**（lumina 前车之鉴） | 高×高 | 2-4 周硬时间盒；DoD 是"能用"不是"完美"；每里程碑一篇博客是强制产出 |
| R3 | **对外宣称与实际实现的一致性** | 高×高 | 公开材料（README/简历/博客）只写仓库里可验证的内容（诚信原则见 2.2）；求职叙事维护在 docs/private/ |
| R4 | **Python 学习曲线** | 中×中 | M1 预留学习时间（第 3 周前以官方教程+Claude Code 辅助为主）；不追求 Python 技巧，只求 LangGraph 模式熟练 |
| R5 | **沙箱安全错觉**（含受控出口的绕过面：DNS rebinding、非 HTTP 协议、IP 直连） | 中×中 | 文档明确 Docker 局限；沙箱无敏感数据、降权；web-scrape 档靠 egress-proxy 的 SNI 校验 + 协议白名单 + 审计日志兜底；回答口径见 docs/private/求职叙事.md 追问 #2/#11；博客演示模式 |
| R6 | **单机资源压力**（ES+PG+Redis 同机） | 中×中 | compose 限制 ES JVM heap（512m 起步）；内存紧张时 Langfuse 用 Cloud 免费版（OQ2）；沙箱内存 512m 上限 |
| R7 | **LLM API 成本失控** | 低×中 | 默认 GLM-4-Flash；语义缓存 + prompt caching；每月充值 50-100 元限额；沙箱断网（agent 内代码无法外呼 API） |
| R8 | **博客托管/备案合规** | 中×中 | 静态站 GitHub Pages 免备案；助手后端上云前按 OQ1 决策（国内轻量+备案 vs 境外 VPS vs 本地演示+录屏） |
| R9 | **评测自欺** | 中×高 | 评测集版本化 + 负面用例 + LLM-judge 人工抽检 + CI 跑分；不用"我跑通了"代替"分数变了" |
| R10 | **SSE 流式链路工程复杂度**（前端→网关→supervisor 三级转发易翻车） | 中×中 | M1 第一天用 curl 验证流式；网关透传不缓冲；e2e 冒烟脚本随里程碑演进 |
| R11 | **多租户返工**（平台化时才发现会话/缓存没带租户维度） | 中×中 | M0 起数据模型强制带 session_id/tenant_id；缓存 key、checkpoint thread_id 命名空间规范写进 ADR |
| R12 | **Windows 环境坑**（Docker Desktop/WSL2） | 中×低 | WSL2 + Docker Desktop 为前置条件（M0 第一天验证）；沙箱功能依赖 Docker 引擎，异常时备选云沙箱机 |
| R13 | **爬虫演示合规** | 低×低 | 只抓公开数据（天气等）；演示页免责声明；不提供任意 URL 抓取 |
| R14 | **求职追问储备不足** | 中×高 | 追问清单维护在 docs/private/求职叙事.md（不入库），每里程碑完成后自测一遍；技术性答案落进 `docs/decisions/` |

**开放决策点（Open Questions）**
- OQ1（M5 前定）：助手后端的托管方案——本地演示+录屏 / 境外 VPS（免备案）/ 国内轻量云（需 ICP 备案）
- OQ2（M4 前定）：Langfuse Cloud 免费版 vs 自托管（先实测 Cloud 在国内的访问速度）
- OQ3（持续）：主模型组合——GLM-4-Flash 起步，按成本/质量实测切换 DeepSeek/Qwen
- OQ4（M5 前定）：检索助手公网开放策略——默认限流开放（Redis token bucket），之后可按需收紧

---

## 11. 成本预算

| 项目 | 金额 | 说明 |
|---|---|---|
| 开发期基础设施 | 0 元 | 本机 Docker |
| LLM API | 50-100 元/月 | GLM-4-Flash 等国产模型个人量级；充值设上限 |
| 博客静态托管 | 0 元 | GitHub Pages / Vercel |
| Langfuse | 0 元 | Cloud 免费版（若自托管则为机器内存成本） |
| 可选：助手后端上云 | 50-100 元/月 | 仅当 OQ1 选择云方案时发生 |

---

## 12. 非目标（明确不做，防止范围蔓延）

- 手写 eBPF / Firecracker（写入演进蓝图，不实现）
- 自研 agent 编排框架（用 LangGraph，不再维护自研简化版）
- 低代码画布 / 拖拽式 agent 构建器
- 多租户计费、商业化
- Kafka 消息总线（单机版用 Redis Pub/Sub，ADR-001）
- A2A 协议完整实现（M7+ 了解性 demo 即可）
- 公网开放任意代码执行（博客沙箱只做演示模式）

---

## 13. 工作协议（开发纪律）

1. **时间盒**：里程碑 2-4 周；到期未完成 → 砍需求保 DoD，不延期不追加。
2. **可运行**：每个 commit 后 compose 全栈可起（至少 e2e 冒烟脚本通过）。
3. **决策留痕**：每个技术取舍写一条 ADR（`docs/decisions/`）。
4. **数据先行**：观测截图、eval 跑分、缓存命中率、攻击测试报告——全部版本化入库，README 引用。
5. **博客同步**：每里程碑一篇文，选题见各里程碑产出；文章即复盘，复盘即沉淀。
6. **诚实边界**：公开材料只写仓库里指得出来的东西；个人叙事与求职材料一律放 `docs/private/`（不入库）。

---

## 附录 A：参考资源（官方文档优先）

- LangGraph：langchain-ai.github.io/langgraph（concepts + checkpointers）
- MCP 规范：modelcontextprotocol.io（2026 spec：Streamable HTTP / OAuth 2.1 / stateless core）
- MCP Java SDK：github.com/modelcontextprotocol/java-sdk
- Langfuse：langfuse.com/docs（trace + evals）
- Docker 安全加固：docs.docker.com/engine/security（seccomp/capabilities 章节）
- K8s HPA：kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale
- kagent（平台化参照）：github.com/kagent-dev/kagent
