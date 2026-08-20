# GustoBot 代码评审报告

> 审查对象：[skygazer42/GustoBot](https://github.com/skygazer42/GustoBot)（develop 分支，克隆自 GitHub）
> 审查日期：2026-08-18
> 审查方式：全文通读 + 4 路并行深度分析（主路由层 / 知识库检索 / 图谱子图 / 工程实践）
> 规模：约 2.3 万行 Python（gustobot 主包 19943 行 + kb_ingest 2970 行）+ Vue3 前端 + 389 个文件

---

## 一、项目概览：它做了什么

GustoBot 是一个基于 LangGraph 的**中华菜谱垂直领域 Multi-Agent 客服系统**，三层架构：

| 层级 | 职责 | 实现 |
|------|------|------|
| L1 主路由层 | 意图识别与任务分发 | `lg_builder.py`（1034 行）：LLM 结构化输出（Router Pydantic model）+ 启发式关键词兜底"双保险" |
| L2 多工具子图层 | 任务分解与工具编排 | `multi_tool.py`（855 行）：guardrails → planner → tool_selection → 4 工具并行 → summarize → final_answer |
| L3 原子工具层 | 具体执行 | Neo4j（Cypher 图谱推理）/ MySQL（Text2SQL 统计）/ PostgreSQL pgvector + Milvus（双路语义检索）/ LightRAG（GraphRAG 社区检索）/ Vision + CogView-4（多模态） |

**核心能力**：菜谱做法查询、历史典故知识问答、菜系统计、图谱关联推理、图片识别/生成、Excel 文件导入分析、多轮对话。

**值得学习的亮点**（对 Agent 开发方向很有参考价值）：
1. **PostgreSQL 优先兜底链**：结构化数据优先 → 向量兜底 → 外部搜索，多级兜底保证答案质量（`multi_tool.py:578-701`）
2. **Reranker 双阈值过滤**：相似度 ≥0.2 粗筛 → rerank 得分 ≥0.8 精排，两段式质量闸门（`knowledge_service.py:152-164`）
3. **启发式 + LLM 双保险路由**：LLM 失效时关键词兜底，不会裸奔
4. **Text2Cypher 全链路**：生成 → 校验 → 纠错 → 执行，带格式校验器（validators）与纠错循环
5. **预定义 Cypher 模板库**：高频查询走模板，降低 LLM 生成错误的概率
6. **领域可迁移设计**：替换知识源与图谱 schema 即可迁移到其他垂直领域（宝可梦百科/中医药典/法律咨询）

---

## 二、问题清单（按严重程度分级）

### 🔴 P0 安全风险（必须立即处理）

| # | 问题 | 证据 | 影响 |
|---|------|------|------|
| 1 | **真实密码文件被提交到公开仓库** | `uploads/3d819b6f-1cea-47c7-861c-c0abc2f22c74_密码(1).txt` 被 git 跟踪并推送到 `github.com/skygazer42/GustoBot`，内含 **Ubuntu 服务器密码、MySQL root 密码、邮箱密码、Apple ID 密保、以太坊助记词**等大量明文凭据 | 仓库是公开的，任何人可克隆看到这些凭据。对应账号/服务器/钱包全部面临被盗风险 |
| 2 | `.gitignore` 未忽略 `uploads/` 与 `data/` | `.gitignore:213-335` 只有前端/模型/数据库忽略规则；`uploads/密码(1).txt`、`data/lightrag/*.json`、`data/tiktoken_cache/`、`kb_ingest/data/历史菜谱源头.xlsx` 全部入库 | 运行时生成物、向量缓存、业务数据、上传文件污染仓库 |
| 3 | CORS 全放开 | `gustobot/main.py:36` `allow_origins=["*"]` | 任意网站可跨域调用 API |
| 4 | Neo4j 认证被禁用 | `docker-compose.yml` 中 Neo4j 配置 | 图数据库无鉴权访问 |
| 5 | **Cypher 注入面** | text2cypher 校验环节拼接 label/property_key 时**无白名单校验**（`validation/` 目录） | LLM 生成的 Cypher 可能被注入恶意查询 |
| 6 | 注释/脚本泄漏凭据与内网信息 | 代码注释泄漏 Neo4j 密码；`scripts/vllm_start_*.sh` 硬编码 `/home/kdsoft/miniconda3`、`/data/models` 等个人路径；Dockerfile 硬编码公网 IP | 暴露部署环境与基础设施信息 |

### 🟠 P1 架构问题（影响系统可靠性）

| # | 问题 | 证据 |
|---|------|------|
| 7 | **双套检索链路并存** | `kb_tools/node.py` 的 knowledge_query（Milvus 单路）与 `multi_tool.py` 的 local_search（PG 优先 + Milvus 兜底）是两条并行路径，最终都走 LLM 合成答案——维护双倍代码、行为不一致 |
| 8 | **新旧 QA 管线职责重叠** | `infrastructure/knowledge/recipe_kg/` 下旧规则管线（neo4j_qa_service、query_parser_service 等）仍通过 HTTP 暴露，新 agent 又反向复用其分类器/解析器——两代架构并存 |
| 9 | 子图节点未注册导致运行时报错风险 | `tool_selection` 可动态路由到 `error_tool_selection`，但主图 `create_multi_tool_workflow` 未注册该节点 |
| 10 | 生产 `cypher_query` 是"伪节点" | 在单个函数内串行调用 生成→校验→执行，并非真实 LangGraph 子图，无法利用图编排的容错与重试 |
| 11 | 条件边定义后未使用 | `single_agent/text2cypher.py:88` 跳过 validation/correction 环节，条件边形同虚设 |
| 12 | 双套工作流重复 | `workflows/multi_agent/` 与 `workflows/single_agent/` 各有一套 text2cypher，State/validators/generation 三处重复（validators 双份约 500 行） |
| 13 | **README 声称的 Redis Checkpointer 实际是 MemorySaver** | `lg_builder.py:955` `checkpointer = MemorySaver()`——进程内内存，重启即丢；Redis 只做语义缓存。与文档严重不符，且多实例部署时无共享记忆 |

### 🟡 P2 代码质量问题

| # | 问题 | 证据 |
|---|------|------|
| 14 | 超大文件 | `lg_builder.py` 1034 行、`multi_tool.py` 855 行、`ChatWidget.vue` 740 行 |
| 15 | 对象重复实例化 | `ChatOpenAI` 重复实例化 13 处；`temperature=0.7` 魔法值重复 7 处；`scope_description` 三处复制粘贴 |
| 16 | **假流式** | `chat.py:211-224` 用 `asyncio.sleep(0.05)` 模拟流式，注释自己写着 "Simulate streaming delay" |
| 17 | 死代码 | `check_hallucinations`（`lg_builder.py:913`）定义后未入图；3 个 0 字节空文件 |
| 18 | 同步阻塞事件循环 | `kb_tools/node.py:140` 在 async 函数里同步调用 `SearchTool.search()` |
| 19 | 数据库连接管理薄弱 | `kb_ingest/kb_service/services/vector_store.py:49`、`processor.py:468/559` 多处 PG 连接**无 try/finally**、无连接池；Milvus 全局单例连接 |
| 20 | Rerank 链路缺陷 | rerank 失败**静默降级**，0.8 阈值把结果清空；存在**双重 rerank 调用**；embedding 维度两套配置不一致（1024 vs 其他） |
| 21 | 双套 PG 表结构代码并存 | `kb_ingest` 与主包各自定义一套 |
| 22 | 启发式路由过于简陋 | `_heuristic_router`（`lg_builder.py:986-1024`）仅 15 个关键词，"多少"会被路由到 text2sql（如"这个菜放多少克盐"会被误判为统计查询）；text2sql 关键词优先级高于 graphrag |
| 23 | 记忆截断逻辑不一致 | HTTP 层与 CLI 层的 `GUSTOBOT_MEMORY_TURNS` 处理方式不同 |
| 24 | Legacy 端点残留 | 4 个 `include_in_schema=False` 的旧别名端点（/chat/chat、/chat/stream 等） |

### 🟢 P3 工程化问题

| # | 问题 | 证据 |
|---|------|------|
| 25 | **无 CI/CD** | 无 `.github/` 目录 |
| 26 | 无 pre-commit / ruff 配置 | 只有 Makefile 里的 flake8/black/mypy 命令 |
| 27 | 测试严重不足 | 9 个测试文件仅 3 个有真实断言（共 26 个），其余为脚本式 smoke test；301 个 Python 文件只有 1322 行测试 |
| 28 | 依赖陈旧 | `langgraph==0.2.60`（2024-12 发布）、`langchain==0.3.7`、`pymilvus==2.3.7`、`numpy<2.0.0`——现在 LangGraph 已 1.x，升级 API 迁移成本高 |
| 29 | 依赖无锁文件 | requirements.txt 有精确版本但无 hash lock；两个 Dockerfile 基础镜像版本不一致 |
| 30 | 文档与代码脱节 | README 声称 **React 18**（实际 Vue3 + Vite + TS）、Redis Checkpointer（实际 MemorySaver）、`lg_builder.py` 在根目录（实际在包内）；`pyproject.toml` authors 还是 "Your Name" 占位符 |

---

## 三、改进路线图（按优先级）

### 第一步：止血（1-2 天）
1. **处理密码泄露**（如果这是你的仓库）：
   - 立即从仓库删除该文件，**用 `git filter-repo` 或 BFG 重写 git 历史**（只删文件不彻底，历史里仍可翻到）
   - **轮换所有暴露的凭据**：服务器密码、MySQL root 密码、邮箱密码、Apple ID 密码与密保、以太坊助记词（助记词泄露=钱包失控，必须迁移资产）
   - 联系 GitHub 无济于事——公开仓库的提交记录无法"收回"，只能换凭据
2. `.gitignore` 补充：`uploads/`、`data/`、`neo4j_plugins/*.jar`、`*.xlsx`、`data/lightrag/`
3. CORS 改白名单（`.env` 里配 `CORS_ORIGINS`，默认只允许 `http://localhost:5173`）
4. 开启 Neo4j 认证；清除代码注释中的密码
5. text2cypher 校验加 label/property 白名单，阻断注入

### 第二步：架构收敛（1-2 周）
1. **统一检索链路**：废弃 `kb_tools/node.py` 的 Milvus 单路实现，全走 `multi_tool.py` 的 PG 优先 + Milvus 兜底；或者反过来，二选一
2. **清理 recipe_kg 旧管线**：保留纯工具函数（分类器/解析器），删除 HTTP 暴露的旧 QA 端点
3. **删除 single_agent 重复工作流**，或明确标注 deprecated
4. 修复 `error_tool_selection` 未注册 bug；把"伪节点" cypher_query 改造成真实子图
5. **决定记忆策略**：要么升级 Redis/Postgres checkpointer（多实例共享记忆），要么在 README 里诚实说明是 MemorySaver 并评估单实例场景够不够用

### 第三步：质量提升（1 个月滚动）
1. 拆 `lg_builder.py`（1034 行）→ 按职责拆成 router/guardrails/multimodal/kb 模块
2. 抽 `ChatOpenAI` 工厂函数（`get_llm()` 单例 + model 参数），消灭 13 处重复
3. 真流式：`langchain_core` 的 `stream` API 或 SSE 逐 token 推送，替换 sleep 假流式
4. PG 连接池（SQLAlchemy `pool_size`）+ try/finally；删死代码
5. 统一 embedding 维度、修双重 rerank、rerank 失败要显式降级（不要静默清空）
6. `SearchTool` 异步化（httpx async）

### 第四步：工程化（持续）
1. 上 GitHub Actions：`lint + pytest` 最小 CI（30 分钟能配好）
2. pre-commit + ruff
3. 补测试：优先给路由层、检索链路、Text2Cypher 校验器写单测（这是最容易出 bug 的地方）
4. 依赖升级计划：langgraph 0.2 → 1.x 迁移（API 变动较大，建议先锁死当前版本跑通 CI 再升级）
5. 修正 README（Vue3、MemorySaver、真实文件结构），加一张当前架构图

---

## 四、给你的学习提示（与求职方向相关）

这个项目和你的 LangGraph+Agent 智能客服项目（意图分诊动态路由、两级记忆、混合检索、多模型兼容）**高度同构**，值得对照学习：

1. **它的检索兜底链设计**（PG 优先 → 向量兜底 → 外部搜索）可以反哺你简历里的"混合检索"描述——你现在只有 BM25+向量，可以补"多级兜底"这个点
2. **它的双保险路由**（LLM + 关键词 fallback）是你"意图分诊动态路由"的强化版，面试可讲
3. **反面教材同样有价值**：假流式、双套链路并存、MemorySaver 冒充 Redis Checkpointer——面试官问"你踩过什么坑"时，这些是现成的技术债案例

---

*本报告基于 2026-08-18 克隆的 develop 分支代码。P0 项引用路径均已本地验证。*
