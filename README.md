<div align="center">
  <img
    src="./docs/assets/gustobot-logo.png"
    alt="GustoBot Logo"
    width="360"
  />
  
  <h3>智能菜谱客服 · 增强版</h3>
  <p>基于 Multi-Agent 架构的菜谱知识问答与多模态服务系统</p>

  ![Python](https://img.shields.io/badge/Python-3.10-blue?style=flat&logo=python)
  ![FastAPI](https://img.shields.io/badge/FastAPI-005571?style=flat&logo=fastapi)
  ![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=ffffff)
  ![LangGraph](https://img.shields.io/badge/LangGraph-Flow-green?style=flat&logo=databricks)
  ![GraphRAG](https://img.shields.io/badge/GraphRAG-KG-blueviolet?style=flat)
  ![Agent](https://img.shields.io/badge/Agent-System-orange?style=flat)
  ![Vue3](https://img.shields.io/badge/Vue3-3.4-42b883?style=flat&logo=vuedotjs)
  ![Neo4j](https://img.shields.io/badge/Neo4j-5.27-008CC1?style=flat&logo=neo4j)
  ![Milvus](https://img.shields.io/badge/Milvus-2.3-00A9E0?style=flat)
  ![License](https://img.shields.io/badge/License-Apache%202.0-yellow.svg?style=flat)
</div>

> **🏷️ 本项目说明**：本仓库是 [skygazer42/GustoBot](https://github.com/skygazer42/GustoBot)（Apache-2.0）的**增强版（Enhanced Edition）**，在保留原三层 Multi-Agent 架构的基础上，完成了 P0 安全加固与工程化修正。详情见 [增强版改动清单](#-增强版改动清单)。

## 项目简介

中华菜谱作为世界饮食文化的瑰宝之一，拥有深厚的历史底蕴与丰富的知识体系。从八大菜系的地域特色，到食材搭配的营养学原理，再到烹饪技法的代际传承，菜谱知识既具有高度结构化的特点（如食材用量、烹饪步骤），又包含大量非结构化的文化典故与经验性描述。这种复杂性使得传统的关键词搜索难以满足用户对菜谱知识的深度探索需求。

本项目以中华菜谱数据为基础，构建出覆盖菜谱名称、食材、烹饪步骤、营养成分、菜系流派、历史典故等元素的**多层次知识图谱**，并结合大模型的理解与生成能力，打造了一个**专注于菜谱领域的智能客服**——「GustoBot」。

技术架构融合了 **LangGraph 多智能体编排**、**GraphRAG 图谱检索增强**、**Text2SQL 结构化查询**、**多源知识融合**，支持做法查询、历史典故问答、菜系统计、图谱关联推理、图片识别/生成、Excel 文件导入分析、多轮对话等能力。

### 核心价值

- **可迁移的垂直领域模板**：三层架构（主路由层 → 多工具子图层 → 原子工具层）把「编排」与「领域知识」解耦，换知识源与图谱 schema 即可迁移至宝可梦百科 / 中医药典 / 法律咨询 / 政务服务等专域客服
- **PostgreSQL 优先兜底链**：结构化数据优先 → 向量兜底 → 外部搜索，多级兜底保证答案质量
- **Reranker 双阈值质量闸门**：相似度粗筛 + 重排精排两段式过滤
- **多模态交互**：文本问答、图片识别/生成、文件解析
- **知识来源可追溯**：每个答案标注来源
- **安全防护**：Guardrails 层拒绝越界查询 + 本增强版补全的注入白名单校验

### 核心功能

| 功能模块 | 说明 | 技术实现 |
|---------|------|---------|
| **智能路由** | 自动识别问题类型（启发式关键词 + LLM 结构化输出双保险） | 主路由层 |
| **知识库查询** | 历史文化、菜谱典故等知识查询 | Milvus + PostgreSQL pgvector + Reranker |
| **图谱推理** | 基于 Neo4j 的菜谱关系推理（Text2Cypher 全链路） | Cypher 生成 → 校验 → 纠错 → 执行 |
| **统计分析** | MySQL 数据库的统计和聚合查询 | Text2SQL + LLM 自然语言转换 |
| **社区检索** | 社区摘要检索 | LightRAG Global/Local/Hybrid Search |
| **图片处理** | 菜品图片识别与生成 | Vision Model + CogView-4 |
| **文件处理** | Excel、TXT、Markdown 等文件上传分析 | Ingest Service + Knowledge Service |
| **对话管理** | 多轮对话与会话历史 | MemorySaver Checkpointer + SQLAlchemy 持久化 |

### 技术架构

<div align="center">
  <img src="data/1.png" alt="GustoBot Multi-Agent 架构图" width="100%">
  <p><i>图：GustoBot Multi-Agent 系统架构总览</i></p>
</div>

**三层 Multi-Agent 架构**：

| 层级 | 职责 | 实现 |
|------|------|------|
| **L1 主路由层** | 意图识别与任务分发 | `lg_builder.py`：LLM 结构化输出 + 启发式关键词兜底 |
| **L2 多工具子图层** | 任务分解与工具编排 | `multi_tool.py`：Guardrails → Planner → Tool Selection → 并行调用 → Summarize |
| **L3 原子工具层** | 具体执行 | Neo4j(Cypher) / MySQL(Text2SQL) / PG+Milvus(语义) / LightRAG(社区) / Vision+CogView(多模态) |

核心执行流：用户提问 → 路由意图识别 → 分发子图 → Guardrails 检查 → Planner 分解 → 并行工具调用 → Summarize 聚合 → 带来源标注的答案

### 技术栈

- **语言/框架**：Python 3.10 · FastAPI · **Vue3 + Vite + TypeScript**（前端）
- **Agent 编排**：LangGraph 0.2.60 · LangChain 0.3.x · Pydantic 结构化输出
- **数据/检索**：Neo4j 5.27（图谱+Text2Cypher）· MySQL（统计+Text2SQL）· PostgreSQL pgvector + Milvus（向量）· LightRAG（GraphRAG）
- **检索增强**：Reranker（Cohere/Jina/Voyage/BGE/qwen3-rerank）双阈值
- **多模态**：Vision Model + CogView-4
- **工程化**：Docker & Docker Compose（8 服务）· Redis（语义缓存）· loguru · prometheus-client

## 🚀 快速开始

### 前置要求

Python 3.10+ · Node.js 16+ · Docker & Docker Compose

### 1. 克隆

```bash
git clone https://github.com/<your-name>/GustoBot.git
cd GustoBot
```

### 2. 后端

```bash
cp .env.example .env   # 配置 LLM_API_KEY、数据库连接等
docker-compose up -d   # 启动全部 8 个服务（含 Neo4j/MySQL/Milvus/pgvector）
```

或本地开发：

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt   # 网络不稳时建议分阶段安装
python scripts/run.py --mode dev
```

### 3. 前端

```bash
cd web
npm install
npm run dev    # http://localhost:5173
```

> 前端与后端联调时，`web/.env.local` 中 `VITE_API_BASE_URL` 需指向后端地址（默认 `http://localhost:8000`，端口冲突时改 8001 等并同步修改）。

### 4. 知识库导入（可选）

- **Excel 导入**：调用 Ingest Service `POST /api/ingest/excel`
- **图谱初始化**：参考 `scripts/` 下的 init 脚本与 `docs/` 部署文档

## ⚙️ 配置说明

复制 `.env.example` 为 `.env` 后按需修改：

| 配置项 | 说明 | 默认 |
|---|---|---|
| `LLM_MODEL` / `LLM_API_KEY` | 大模型与密钥 | qwen3-max / 必填 |
| `LLM_EMBEDDING_*` | Embedding 模型（1024 维） | text-embedding-v4 |
| `RERANK_*` | Reranker 配置与双阈值 | sim≥0.5 / rerank≥0.8 |
| `NEO4J_USER` / `NEO4J_PASSWORD` | 图库认证（本增强版已启用） | neo4j / recipepass |
| `DATABASE_URL` | MySQL 连接 | — |
| `GUSTOBOT_MEMORY_TURNS` | 记忆轮数 | 5 |
| `CORS_ORIGINS` | 前端跨域白名单 | http://localhost:5173 |

## 🔒 增强版改动清单

本仓库在保留上游全部功能的基础上，完成以下改进（2026-08-20）：

### P0 安全加固

1. **Text2Cypher 标签/关系白名单校验**（`validation/validators.py`）：新增 `validate_labels_and_types_in_cypher_query`，阻止 LLM 生成的 Cypher 引用 schema 之外的未知/恶意标签，堵住注入面
2. **Neo4j 认证启用**（`docker-compose.yml`）：`NEO4J_AUTH: none` → `${NEO4J_USER:-neo4j}/${NEO4J_PASSWORD:-recipepass}`
3. **敏感文件移除 git 跟踪**：`uploads/密码(1).txt`、`data/lightrag/`、`data/tiktoken_cache/`、`neo4j_plugins/*.jar` 等 18 个运行时/凭据文件已 `git rm --cached`，并补齐 `.gitignore` 规则
4. **CORS 白名单化**（`main.py` + `settings.py`）：`allow_origins=["*"]` → `settings.cors_origins_list` 可配置
5. **默认模型修正**：`LLM_MODEL` 默认改为 `qwen3-max`

### 稳定性修复

6. **修复子图节点未注册 bug**（`multi_tool.py`）：补注册 `error_tool_selection` 节点，消除 `tool_selection` 路由到未注册节点的运行时崩溃
7. **修复 19 处 loguru f-string 占位符 bug**：异常消息含 `{'error'}` 花括号与 loguru 格式化冲突导致 KeyError 掩盖真实错误，修复后错误信息正常透传

### 文档修正（与上游 README 偏差）

8. 前端技术栈：~~React 18~~ → **Vue3 + Vite + TypeScript**
9. Checkpointer：~~Redis Checkpointer~~ → **MemorySaver**（进程内存，Redis 仅做语义缓存）
10. Neo4j 版本：~~4.4~~ → **5.27**

> 📋 完整问题清单（30 项，P0-P3 分级）见 `docs/CODE_REVIEW.md`（个人代码评审报告）。

## 📁 目录结构

```
GustoBot/
├── gustobot/               # 后端主包（FastAPI + LangGraph）
│   ├── application/agents/ # 三层 Agent 核心（lg_builder / multi_tool / text2sql / text2cypher）
│   ├── config/             # pydantic-settings 配置
│   ├── infrastructure/     # Neo4j / Milvus / PG / 向量检索
│   └── interfaces/http/    # API 层（chat / sessions / upload / lightrag）
├── kb_ingest/              # 独立知识库导入服务（Excel → pgvector）
├── web/                    # Vue3 + Vite 前端
├── scripts/                # 启动 / 导入脚本
├── docker/                 # Neo4j / pgvector 构建
├── docker-compose.yml      # 8 服务编排
└── docs/                   # 文档与代码评审报告
```

## 📚 文档

- `docs/DEPLOYMENT.md` — 部署指南
- `docs/QUICK_START.md` — 快速开始
- `docs/ROUTER_PROMPT_FIX.md` — 路由提示词 P0 修复记录
- `docs/TEXT2SQL_IMPLEMENTATION.md` — Text2SQL LangGraph 化实现
- `docs/CODE_REVIEW.md` — 代码评审报告（30 项问题与改进路线图）

## License

[Apache License 2.0](LICENSE) — 本增强版基于 [skygazer42/GustoBot](https://github.com/skygazer42/GustoBot)，保留上游版权声明。
