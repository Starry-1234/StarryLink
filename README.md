# StarryLink

> 一个基于 LangGraph + FastAPI + MySQL + Milvus 的 AI 电商客服 Agent。

## 概述

StarryLink 是一个端到端的对话式 AI 客服系统：能查订单、查物流、答产品 FAQ、开退款工单，并能通过对话飞轮持续喂养自己的知识库。

支撑它的核心是一个 LangGraph 状态机、5 个业务工具（含 2 个 MCP 服务）、一套 BM25 + 向量混合检索的知识库，以及自部署的 Langfuse 观测栈。

## 主要功能

- **多轮对话**：上下文延续 + 指代消解 + 流式响应
- **5 个工具**：订单、物流、FAQ、商品、退款工单（含 2 个 MCP 服务）
- **混合检索知识库**：BM25 + 向量 + RRF + 重排序
- **LangGraph 状态机**：意图路由、工具调用、中断/恢复（用于退款确认）
- **对话飞轮**：从历史对话挖问答 → 人工审核 → 自动入库
- **主题分类器**：ONNX 离线推理
- **自部署 Langfuse**：trace 与成本数据完全留在本机

## 技术栈

| 层 | 技术 |
|---|---|
| Web | FastAPI + Uvicorn |
| Agent | LangGraph + LangChain |
| LLM | OpenAI 兼容协议（chat / embed / rerank） |
| 存储 | MySQL 8 · Milvus Standalone · MinIO · etcd |
| 观测 | Langfuse v3（自部署，OTLP） |
| 工具 | MCP（Model Context Protocol） |
| 包管理 | uv |
| 容器 | Docker Compose v2 |

## 快速开始

> 在WSL，Ubuntu，docker配置好之后，可以让 Claude Code / Cursor 一键起：在 IDE 里说一句「按 DEPLOY.md 把项目部署起来」，
> AI 会照着 [DEPLOY.md](DEPLOY.md) 走完下面的全流程，密钥配置、容器启动、知识库构建、验收一把过。

```bash
# 1. 装依赖
uv sync

# 2. 配置环境变量
cp .env.example .env
$EDITOR .env   # 填 CHAT_API_KEY / EMBED_API_KEY / RERANK_API_KEY

# 3. 起数据服务 + 灌数据 + 建知识库
docker compose up -d
make seed
make kb-build
make kb-vectorize

# 4. 起应用
make dev
```

打开 <http://localhost:8000> 即可对话。

启用 Langfuse 观测（可选）：

```bash
make langfuse-up
# 打开 http://localhost:3000
```

## 项目结构

```
app/
  api/        HTTP 路由（chat / agent / kb / review / eval / cost）
  graph/      LangGraph 状态机
  core/       业务模块（LLM / 检索 / 意图 / 观测 / 飞轮 / 摘要）
  kb/         知识库（切片 / 向量化 / 双写 / 去重 / 挖问答）
  tools/      工具系统（@tool / MCP 客户端 / 注册表 / 执行器）
  db/         SQLAlchemy 模型与仓储层
  static/     前端 HTML 页面（SSR，无构建步骤）
mcp_servers/  两个业务 MCP 服务（物流 / 售后）
sql/          DDL 与种子 SQL（首启自动执行）
scripts/      离线脚本（评估 / 训练 / 报告）
docs/         设计文档与计划
```

## 页面

| URL | 用途 |
|---|---|
| `/` | 对话 |
| `/kb` | 知识库录入 |
| `/review` | 飞轮待审队列 |
| `/observability` | 成本与观测 |
| `/topics` | 主题分布 |
| `/acceptance` | 分类器验收 |
| `/admin` | 后台首页（聚合卡） |

## 端口

| 端口 | 服务 |
|---|---|
| 8000 | 主应用 |
| 8101 / 8102 | MCP 服务（物流 / 售后） |
| 19530 | Milvus |

完整端口表（含 Langfuse / ClickHouse / Postgres）见 [DEPLOY.md](DEPLOY.md)。

## 开发命令

| 命令 | 作用 |
|---|---|
| `make test` | 单元测试 |
| `make eval` | 对话质量评估 |
| `make eval-rag` | 检索评估 |
| `make eval-agent` | 工具调用评估 |
| `make flywheel` | 飞轮批处理 |
| `make eval-flywheel` | 评估趋势（落 eval_runs） |
| `make cost-report` | 按意图的 token 成本报告 |
| `make kb-build` / `make kb-vectorize` | 重建知识库 |
| `make dev` / `make dev-down` | 启停应用 |
| `make langfuse-up` / `make langfuse-down` | 启停 Langfuse |

完整命令与高级配置见 [DEPLOY.md](DEPLOY.md)。

## License

MIT — see [LICENSE](LICENSE).

## 作者

Starry · <https://github.com/Starry-1234>
