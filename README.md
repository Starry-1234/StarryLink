# StarryLink

> 一个 AI 驱动的电商智能客服 Agent，基于 LangGraph + FastAPI + MySQL + Milvus 构建。支持订单/物流查询、知识库问答、退款工单创建等典型电商客服场景。

StarryLink 是 Starry 的个人项目。它把一个完整的"对话式 AI 客服系统"从底层到上层都做出来了：能查订单、能查物流、能答产品 FAQ、能开退款工单，还能从历史对话里自动挖问答来喂养自己的知识库。

## 这是什么 / 不是什么

- ✅ **是**：一个能在你本机跑起来的、可演示的电商客服 AI Agent 完整项目
- ✅ **是**：一个 LangGraph 状态机 + 工具调用 + 检索增强生成（RAG）的实战代码库
- ✅ **是**：一份"怎么把 LLM 接进业务"的参考实现（订单查 MySQL、知识库查 Milvus、对话全链路走 Langfuse 观测）
- ❌ **不是**：一个面向终端用户的成品 SaaS（前端是最简的聊天 UI）
- ❌ **不是**：一个 LLM 微调框架（虽然 `ch10/` 下有相关脚本，但那是离线实验，不是主流程）

## 核心能力

### 对话能力
- **多轮对话**：上下文延续 + 流式响应
- **指代消解**：你说"它的物流到哪了"，它知道"它"指上一句提到的订单
- **意图识别**：每条消息自动判断是查订单 / 查物流 / 问知识库 / 申请退款

### 5 个内置工具

| 工具 | 做什么 | 数据来源 |
|---|---|---|
| `query_order` | 查订单基本信息（状态、金额、商品） | MySQL |
| `query_logistics` | 查物流轨迹 | MCP server (:8101) |
| `query_faq` | 回答通用 FAQ | MySQL + Milvus 知识库 |
| `query_product` | 查商品详情、规格 | MySQL |
| `create_refund_ticket` | 创建退款工单 | MCP server (:8102) |

### 知识库（RAG）
- **混合检索**：BM25（关键词）+ 向量（语义）+ RRF（结果融合）+ 重排序
- **离线构建**：从 FAQ / 产品手册 / 历史对话挖出问答对，落 MySQL + Milvus 双写
- **飞轮**：每次对话结束后，把"用户问题 + 客服回答"挖出来喂给人工审核队列，审核通过后自动入库——知识库会越用越大

### Agent 编排
- **LangGraph 状态机**：节点 = 工具调用 / 检索 / 生成 / 路由，边 = 条件路由
- **退款中断/恢复**：用户说"我要退款"会触发中断、弹出确认；用户确认后继续走流程

### 可观测
- **自部署 Langfuse v3**：每条 LLM 调用、每个工具调用、每个 trace 都记录
- **数据不出门**：所有 trace 数据都在你本机的 ClickHouse + Postgres 里
- **成本统计**：按意图聚合 token 用量，算出"问物流花了多少钱"

### 其他
- **主题分类器**（ch10）：用 ONNX 跑离线训练的分类模型，给每条对话打主题标签

## 技术栈

| 层 | 技术 |
|---|---|
| Web 框架 | FastAPI + Uvicorn |
| Agent | LangGraph + LangChain |
| LLM 接入 | OpenAI 兼容协议（chat / embed / rerank，无 gateway） |
| 数据库 | MySQL 8 |
| 向量库 | Milvus Standalone + MinIO + etcd |
| 可观测 | Langfuse v3（自部署，OTLP） |
| 包管理 | uv |
| 容器编排 | Docker Compose v2 |

## 快速开始

### 1. 前置条件

- **uv**（Python 包管理）：`curl -LsSf https://astral.sh/uv/install.sh | sh`
- **Docker**：Docker Desktop / Colima / OrbStack 任一
- **make**：macOS/Linux 自带
- **三个 LLM 密钥**（chat / embed / rerank），需要 OpenAI 兼容协议的端点

### 2. 装依赖

```bash
uv sync
```

首次会下载所有 Python 依赖，等几分钟是正常的。

### 3. 配环境变量

```bash
cp .env.example .env
$EDITOR .env
```

要填的关键几项（其他都有默认值）：

```bash
CHAT_BASE_URL=https://api.deepseek.com/v1
CHAT_MODEL=deepseek-chat
CHAT_API_KEY=sk-xxx
EMBED_BASE_URL=https://api.siliconflow.cn/v1
EMBED_API_KEY=sk-xxx
RERANK_BASE_URL=https://api.siliconflow.cn/v1
RERANK_API_KEY=sk-xxx    # 通常和 EMBED 同一个
```

### 4. 起依赖容器

```bash
docker compose up -d
```

会拉起 MySQL / Milvus / MinIO / etcd。`sql/` 目录里所有 DDL 会自动按序执行。

### 5. 灌种子数据 + 建知识库

```bash
make seed            # 灌 FAQ / 历史会话
make kb-build        # 切 chunk 落 MySQL
make kb-vectorize    # 向量化写进 Milvus
```

### 6. 启动应用

```bash
make dev
```

这条会拉两个 MCP server（:8101 物流 / :8102 售后）+ 应用（:8000）。前台常驻进程，停的时候 `make dev-down`。

### 7. 试一下

浏览器打开 <http://localhost:8000>，问一句：

```
订单 1001 的物流到哪了？
```

应该看到两个工具调用标记（`query_order` + `query_logistics`）和一个回答。

### 8.（可选）起 Langfuse 看 trace

```bash
make langfuse-up
# 打开 http://localhost:3000
# 账号：admin@starrylink.local / starrylink123
```

## 项目结构

| 路径 | 职责 |
|---|---|
| `app/api/` | HTTP 路由（chat / agent / KB / review / eval / cost） |
| `app/graph/` | LangGraph 状态机（节点、路由、编译） |
| `app/core/` | 单职责模块（LLM client、检索、意图、指代消解、摘要、置信度、观测） |
| `app/kb/` | 知识库：切片、向量化、MySQL+Milvus 双写、去重、挖问答 |
| `app/tools/` | 工具系统：内置 `@tool`、MCP 客户端、注册表、执行器 |
| `app/db/` | SQLAlchemy 模型和仓储层 |
| `app/static/` | 前端 HTML 页面（SSR，无构建步骤） |
| `mcp_servers/` | 两个业务 MCP server：物流 (:8101)、售后 (:8102) |
| `sql/` | 按章节组织的 DDL 和种子 SQL，首次启动自动执行 |
| `scripts/` | 离线脚本：评估、调参、报告生成 |
| `docs/superpowers/` | 项目的 spec 和 plan |
| `primer/` | 入门示例（LLM、agent 基础）——独立模块，共享 `.env` |

## 页面入口

| URL | 页面 | 用途 |
|---|---|---|
| `/` | 对话 | 和 AI 客服聊天的主界面 |
| `/kb` | 知识库 | 手动录入 / 审核问答对 |
| `/review` | 飞轮审核 | 审核"从对话里挖出来的问答" |
| `/observability` | 成本 / 观测 | 看 token 用量、意图分布、trace 统计 |
| `/topics` | 主题分布 | 主题分类结果可视化 |
| `/acceptance` | 分类器验收 | 主题分类器的人工验收界面 |

## 端口一览

| 端口 | 服务 |
|---|---|
| 8000 | 应用主进程 |
| 8101 | MCP server：物流查询 |
| 8102 | MCP server：售后（退款工单） |
| 8110 | 主题分类器（`make classifier-up` 后） |
| 3000 | Langfuse Web（`make langfuse-up` 后） |
| 19530 | Milvus |
| 3307 | MySQL（暴露给主机的端口） |
| 9090 / 9091 | MinIO API / 控制台 |
| 8123 / 9000 | ClickHouse HTTP / TCP（Langfuse 用） |
| 5432 | Postgres（Langfuse 用） |

## 开发命令

```bash
make test         # 单元测试
make eval         # 对话质量评估
make eval-agent   # 工具调用准确性
make eval-rag     # 检索质量
make eval-ch05    # LangGraph 章节
make eval-ch06    # 中断 / 恢复章节
make eval-ch07    # 上下文工程章节
make eval-ch08    # 工具章节

make kb-build     # 重建知识库
make kb-vectorize # 重新向量化

make cost-report  # 跑一次成本报告
make ch10-corpus  # ch10 语料准备
make ch10-train   # ch10 分类器训练
make ch10-eval    # ch10 评估
```

## 数据流（一条请求怎么走完）

```
用户消息 → POST /api/agent
        ↓
   意图识别（chat LLM）
        ↓
   路由：订单？物流？知识？退款？
        ↓
   ┌─────────────────────────────────────────────┐
   │ 查订单    │ 查物流    │ 知识库检索    │ 退款表单 │
   │ MySQL     │ MCP 8101  │ BM25+向量    │ 中断+确认│
   └─────────────────────────────────────────────┘
        ↓
   生成回答（chat LLM，带工具结果 / 检索结果）
        ↓
   写回 Langfuse trace（同步）
        ↓
   POST /api/agent 响应 → 前端流式渲染
        ↓
   对话结束 → 挖问答 → 飞轮审核队列（异步）
```

## 部署

开发流程见本 README。**生产部署** / **镜像构建** / **代理配置** 等更细节的话题见 [DEPLOY.md](DEPLOY.md)。

## License

MIT — see [LICENSE](LICENSE).

## 联系

GitHub: [@Starry-1234](https://github.com/Starry-1234)
