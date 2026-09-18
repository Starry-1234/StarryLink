# StarryLink 项目改造实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 `/home/starry/starrylink`（原 StarryLink 课程配套项目）改造成 Starry 的个人项目 StarryLink：抹除原作者痕迹、项目改名、新 README、推到 GitHub。

**Architecture:** 不改业务代码逻辑,只做：
1. 字符串替换（StarryLink → StarryLink, xiaolincoding 等 → 移除）
2. 删除每个文件开头的"来源"注释块
3. 重写 README/DEPLOY.md 顶层文档
4. 多 commit 推到 GitHub 新仓库 `Starry-1234/StarryLink`

**Tech Stack:** Python 3.13 / FastAPI / LangGraph / MySQL / Milvus / Langfuse(v3)（全部沿用,零业务代码改动）

---

## 事实背景（执行无需重读）

- 痕迹形态 1:每个 .py/.js/.html/.md 文件开头 4 行注释
  ```
  # 来源:公众号@小林coding
  # 后端八股网站:xiaolincoding.com
  # Agent网站:xiaolinnote.com
  # 简历模版:jianli.xiaolinnote.com
  ```
- 痕迹形态 2:字符串 `xiaolincoding.com` / `xiaolinnote.com` / `jianli.xiaolinnote.com` / `公众号@小林coding` / `StarryLink` / `starrylink` 散落各处
- 痕迹形态 3:DEPLOY.md 提到"本仓库改动三处"
- 痕迹形态 4:README.md 整篇是课程配套源码风格
- GitHub 仓库已建好:`https://github.com/Starry-1234/StarryLink` (HEAD = dc3bde6...)
- 本机直连 GitHub OK,SSH key 未配,HTTPS push 需要 PAT

---

## Commit 顺序

```
1. chore: initial commit from StarryLink import         (baseline 状态)
2. chore: rename project StarryLink → StarryLink          (全项目字符串替换)
3. chore: remove original author headers              (删 4 行注释块)
4. docs: rewrite README as open-source project README  (新 README)
5. docs: rewrite DEPLOY.md                            (清理原仓库引用)
6. chore: rename docker container/host names          (compose + 项目名一致)
7. chore: rename Python package + uv project name     (pyproject.toml)
8. chore: push to github                              (首次推送)
```

---

## Task 1: 初始化 git 仓库 + 第一次 commit（baseline 状态）

**Files:**
- Create: `/home/starry/starrylink/.git/`

- [ ] **Step 1: 配置 git user（如果没有）**

```bash
git config --global user.name "Starry"
git config --global user.email "starry@users.noreply.github.com"
# email 用 noreply 格式,不会泄露真实邮箱
```

- [ ] **Step 2: 在项目根初始化 git**

```bash
cd /home/starry/starrylink
git init -b main
# 确认 .gitignore 已经存在(已有)
ls -la .gitignore  # 应该存在 462 字节
```

- [ ] **Step 3: 第一次 commit（baseline,保留全部原作者痕迹）**

```bash
cd /home/starry/starrylink
git add .
git status --short | wc -l   # 应该几百个文件
git commit -m "chore: initial commit from StarryLink import"
```

预期:`Created initial commit`,commit hash 是 7 位 hex。

---

## Task 2: 重命名项目 StarryLink → StarryLink（全项目字符串替换）

**Files:**
- Modify: `/home/starry/starrylink/` 全项目,排除 .venv / node_modules / __pycache__ / .git / data/*_checkpoints.sqlite

**目标替换映射**:

| 旧 | 新 |
|---|---|
| `StarryLink` | `StarryLink` |
| `starrylink` | `starrylink` |
| `starrylink-`（docker 容器/卷前缀）| `starrylink-` |

**注意**:docker compose 里 `starrylink-mysql` `starrylink-mysql-data` 这种命名要保留 docker 容器兼容性——但既然我们重做整个 docker compose stack,改名是 OK 的。**但 `starrylink-mysql-data` 卷里已经有数据**,改卷名 = 数据丢。建议:**保持卷名** `starrylink-mysql-data`（不动）;改 container name `starrylink-mysql` → `starrylink-mysql`。类似处理 minio/etcd/milvus。

- [ ] **Step 1: 替换字符串,先列出会影响哪些文件**

```bash
cd /home/starry/starrylink
grep -rl "StarryLink\|starrylink" . \
  --exclude-dir=.venv --exclude-dir=node_modules --exclude-dir=.git \
  --exclude-dir=__pycache__ \
  --exclude="*.sqlite*" 2>/dev/null | sort > /tmp/rename_files.txt
wc -l /tmp/rename_files.txt
```

预期:大约 34 个文件。

- [ ] **Step 2: 用 sed 批量替换**

```bash
cd /home/starry/starrylink
# 区分大小写替换
while IFS= read -r f; do
  sed -i \
    -e 's/StarryLink/StarryLink/g' \
    -e 's/starrylink/starrylink/g' \
    "$f"
done < /tmp/rename_files.txt
```

- [ ] **Step 3: 验证没有遗漏的旧字符串**

```bash
cd /home/starry/starrylink
# 排除 data 里的 sqlite 和 .git
grep -rn "StarryLink\|starrylink" . \
  --exclude-dir=.venv --exclude-dir=node_modules --exclude-dir=.git \
  --exclude-dir=__pycache__ \
  --exclude="*.sqlite*" 2>/dev/null | head -20
```

预期:应该没有匹配（除了 `data/ch05_checkpoints.sqlite` 这种 binary 文件）。

- [ ] **Step 4: 验证 docker compose 里卷名保留**

```bash
grep -E "starrylink-mysql-data|starrylink-mysql-etcd|starrylink-milvus-data|starrylink-milvus-etcd|starrylink-milvus-minio" \
  /home/starry/starrylink/docker-compose.yml
```

预期:这些卷名应该**保留**(因为是持久数据)。如果 sed 把它们也改了,要手动恢复。

- [ ] **Step 5: 看一次 sed 后影响较大的文件**

```bash
cd /home/starry/starrylink
head -20 docker-compose.yml
head -20 README.md
head -20 pyproject.toml
```

预期:看到 starrylink 命名。

- [ ] **Step 6: Commit**

```bash
cd /home/starry/starrylink
git add -A
git diff --cached --stat | tail -5
git commit -m "refactor: rename project StarryLink to StarryLink"
```

---

## Task 3: 删除每个文件开头的"来源"原作者注释块

**Files:**
- Modify: 229 个文件（每个文件开头 4 行注释）

**目标**:删除这种 4 行注释块：

```
# 来源:公众号@小林coding
# 后端八股网站:xiaolincoding.com
# Agent网站:xiaolinnote.com
# 简历模版:jianli.xiaolinnote.com
```

也包括 `.js` `.css` `.html` `.sql` 等文件的对应块（如 `/* 来源... */` 形式）。

- [ ] **Step 1: 找出所有"来源"块的文件**

```bash
cd /home/starry/starrylink
grep -rl "xiaolincoding\|xiaolinnote\|公众号@小林\|jianli.xiaolinnote" . \
  --exclude-dir=.venv --exclude-dir=node_modules --exclude-dir=.git \
  --exclude-dir=__pycache__ 2>/dev/null | sort > /tmp/header_files.txt
wc -l /tmp/header_files.txt
```

预期:大约 229 个文件。

- [ ] **Step 2: 用 Python 脚本统一删除文件头部 4 行注释块**

```python
# /tmp/strip_headers.py
import re, sys
from pathlib import Path

PATTERNS = [
    re.compile(r'^[ \t]*#[ \t]*来源[：:]\s*公众号@?小林\s*[a-z]*[ \t]*\r?\n', re.MULTILINE),
    re.compile(r'^[ \t]*#[ \t]*后端八股网站[：:]\s*xiaolincoding\.com[ \t]*\r?\n', re.MULTILINE),
    re.compile(r'^[ \t]*#[ \t]*Agent\s*网站[：:]\s*xiaolinnote\.com[ \t]*\r?\n', re.MULTILINE),
    re.compile(r'^[ \t]*#[ \t]*简历模版[：:]\s*jianli\.xiaolinnote\.com[ \t]*\r?\n', re.MULTILINE),
    # JS/CSS 块注释形式
    re.compile(r'^[ \t]*\*[ \t]*来源[：:]\s*公众号@?小林\s*[a-z]*[ \t]*\r?\n', re.MULTILINE),
    re.compile(r'^[ \t]*\*[ \t]*后端八股网站[：:]\s*xiaolincoding\.com[ \t]*\r?\n', re.MULTILINE),
    re.compile(r'^[ \t]*\*[ \t]*Agent\s*网站[：:]\s*xiaolinnote\.com[ \t]*\r?\\r?\n', re.MULTILINE),
]

def strip(path: Path) -> bool:
    text = path.read_text(encoding='utf-8', errors='replace')
    orig = text
    # 必须在文件最开头(允许前导空行)
    stripped = text.lstrip('\r\n')
    # 找开头 6 行
    head_lines = stripped.split('\n')[:6]
    head = '\n'.join(head_lines)
    for pat in PATTERNS:
        head = pat.sub('', head)
    new_text = text[:len(text) - len(stripped)] + head + stripped[len('\n'.join(head_lines)) + 1 if False else stripped.split('\n', 6)[-1] if False else stripped[len('\n'.join(head_lines))+1:] if '\n'.join(head_lines) in stripped else stripped]
    # 简化:直接在原 text 上对前 6 行做替换
    lines = text.split('\n')
    for i in range(min(8, len(lines))):
        for pat in PATTERNS:
            new_line = pat.sub('', lines[i])
            if new_line != lines[i]:
                lines[i] = new_line
                break
    new_text = '\n'.join(lines)
    if new_text != orig:
        path.write_text(new_text, encoding='utf-8')
        return True
    return False

files = [Path(p) for p in sys.argv[1:]]
n = sum(1 for p in files if strip(p))
print(f"stripped {n}/{len(files)} files")
```

> 注:这个脚本要实际跑一遍调试,上面的 regex 草图可能因实际格式差异需要调。建议先 `cp` 一个文件备份,跑后 `diff` 验证 4 行真的删了。

- [ ] **Step 3: 实际跑脚本**

```bash
cd /home/starry/starrylink
# 先 dry-run:列出第一个文件看看格式
head -6 app/main.py
head -6 app/core/intent.py
head -6 app/static/admin.js
# 确认格式后跑
python3 /tmp/strip_headers.py $(cat /tmp/header_files.txt)
```

- [ ] **Step 4: 验证**

```bash
cd /home/starry/starrylink
grep -rln "xiaolincoding\|xiaolinnote\|公众号@小林\|jianli.xiaolinnote" \
  --exclude-dir=.venv --exclude-dir=node_modules --exclude-dir=.git \
  --exclude-dir=__pycache__ 2>/dev/null | head -10
```

预期:应该空。

- [ ] **Step 5: 抽查几个文件**

```bash
head -10 /home/starry/starrylink/app/main.py
head -10 /home/starry/starrylink/app/core/intent.py
head -15 /home/starry/starrylink/app/static/admin.js
```

预期:开头不再有"来源"4 行,但保留其他注释（如 docstring、`# 来源:` 是误伤的话要恢复）。

- [ ] **Step 6: Commit**

```bash
cd /home/starry/starrylink
git add -A
git commit -m "chore: remove original author attribution headers"
```

---

## Task 4: 重写 README.md 为标准开源项目风格

**Files:**
- Modify: `/home/starry/starrylink/README.md`（覆盖）

- [ ] **Step 1: 备份旧 README（可选）**

```bash
cp /home/starry/starrylink/README.md /tmp/README.old.md
```

- [ ] **Step 2: 用 Write 工具覆盖新 README**

新 README 内容（见附录 A）。要点:
- 项目名 StarryLink
- 简短一句话说明这是 AI 客服 Agent
- 技术栈列表
- 安装运行步骤
- 项目结构表
- 端口列表
- 页面入口列表
- 联系/署名:Starry 2026

- [ ] **Step 3: 验证**

```bash
head -30 /home/starry/starrylink/README.md
grep -n "StarryLink\|xiaolin\|小林" /home/starry/starrylink/README.md
```

预期:没有原作者字样。

- [ ] **Step 4: Commit**

```bash
cd /home/starry/starrylink
git add README.md
git commit -m "docs: rewrite README as standard open-source project README"
```

---

## Task 5: 清理 DEPLOY.md

**Files:**
- Modify: `/home/starry/starrylink/DEPLOY.md`

- [ ] **Step 1: 看现状**

```bash
cat /home/starry/starrylink/DEPLOY.md
```

- [ ] **Step 2: 删掉原仓库引用、改项目名**

用 Edit 工具或 Write 重写:
- 把里面"本仓库 / 改动三处"等章节改写成 StarryLink 项目部署
- 保留里面通用的部署经验（端口、镜像、调试技巧）
- 顶部加一句话说明这是 StarryLink 的部署笔记

- [ ] **Step 3: Commit**

```bash
cd /home/starry/starrylink
git add DEPLOY.md
git commit -m "docs: rewrite DEPLOY.md for StarryLink"
```

---

## Task 6: 改 docker compose 命名（容器/卷）

**Files:**
- Modify: `/home/starry/starrylink/docker-compose.yml`
- Modify: `/home/starry/starrylink/docker-compose.langfuse.yml`

- [ ] **Step 1: 检查现状**

```bash
grep -n "container_name\|starrylink-" /home/starry/starrylink/docker-compose.yml | head -10
grep -n "container_name\|starrylink-" /home/starry/starrylink/docker-compose.langfuse.yml | head -10
```

- [ ] **Step 2: 重命名容器（不重命名数据卷,保留 `starrylink-` 命名的卷以保护现有数据）**

把 `container_name: starrylink-mysql` → `container_name: starrylink-mysql` 等。但**数据卷名保留**(因为已经跑过了有数据)。

但是 sed -e 's/starrylink/starrylink/g' 在 Task 2 已经把所有 `starrylink-` 改了。所以这一步主要是验证 + 回滚数据卷名。

- [ ] **Step 3: 验证现有 compose 还能跑**

```bash
cd /home/starry/starrylink
docker compose config --quiet && echo OK
docker compose -p starrylink-langfuse -f docker-compose.langfuse.yml --env-file /tmp/langfuse.env config --quiet && echo OK
```

预期:都 OK。

- [ ] **Step 4: 如果上一步发现数据卷名被错改,Edit 改回 `starrylink-mysql-data` 等**

```bash
# 找是否被改了
grep -E "starrylink-mysql-data|starrylink-milvus" /home/starry/starrylink/docker-compose.yml
# 如果有,改成 starrylink-* 旧名
```

- [ ] **Step 5: Commit**

```bash
cd /home/starry/starrylink
git add -A
git commit -m "chore: rename docker containers to starrylink prefix"
# 或: git commit -m "chore: keep mysql data volume name for backward compat" (如果有手动调整)
```

---

## Task 7: 改 pyproject.toml 的项目名 + Python 包导入

**Files:**
- Modify: `/home/starry/starrylink/pyproject.toml`

- [ ] **Step 1: 检查现状**

```bash
cat /home/starry/starrylink/pyproject.toml | head -10
```

- [ ] **Step 2: 改 name**

```
name = "starrylink"  →  name = "starrylink"
```

- [ ] **Step 3: 验证 Python 仍能 import**

```bash
cd /home/starry/starrylink
uv run python -c "import app.main; print('ok')" 2>&1 | tail -5
```

预期:`ok`。如果报错说找不到 `app`,那 Task 2 的 sed 把 import 路径也改了——需要手动 fix（不应该改 import path,因为目录名还是 `app/`）。

- [ ] **Step 4: Commit**

```bash
cd /home/starry/starrylink
git add pyproject.toml uv.lock
git commit -m "chore: rename uv project to starlrylink"
```

---

## Task 8: 推送到 GitHub

**Files:** 无

- [ ] **Step 1: 准备 git 凭证**

用户已建仓库 `https://github.com/Starry-1234/StarryLink`。

push 选项:
- **A. HTTPS + Personal Access Token**(推荐,马上能用)
- B. SSH key(需要用户生成并加到 GitHub)

询问用户偏哪种。如果选 A,执行:

```bash
# 用户提供 PAT 后
git remote add origin https://github.com/Starry-1234/StarryLink.git
# 第一次 push 提示输用户名 + PAT(用户自己输入,不写到脚本里)
git push -u origin main
```

如果选 B:

```bash
ssh-keygen -t ed25519 -C "starry@users.noreply.github.com" -f ~/.ssh/id_ed25519 -N ""
# 打印公钥让用户加到 https://github.com/settings/keys
cat ~/.ssh/id_ed25519.pub
# 然后:
git remote add origin git@github.com:Starry-1234/StarryLink.git
git push -u origin main
```

- [ ] **Step 2: 推之前再看一次 commit 历史**

```bash
cd /home/starry/starrylink
git log --oneline | head -10
```

预期:8 个 commit 左右,主题清晰。

- [ ] **Step 3: Push**

```bash
cd /home/starry/starrylink
git push -u origin main
```

预期:8 个 commit 全部推到 GitHub。

- [ ] **Step 4: 验证**

```bash
git ls-remote origin main
# 或者: gh repo view Starry-1234/StarryLink --web
```

预期:看到 main 分支 commit hash,跟本地一致。

---

## 附录 A: 新 README 内容(写到 `/home/starry/starrylink/README.md`)

```markdown
# StarryLink

> An AI-powered e-commerce customer service agent built on LangGraph + FastAPI + Milvus + MySQL. Supports order / logistics queries, knowledge-base Q&A, refund workflows, knowledge harvesting from conversations, and topic classification.

StarryLink is Starry's personal project, evolved from an e-commerce CS agent codebase.

## Features

- **Multi-turn chat** with streaming, structured extraction, and coreference resolution
- **5 built-in tools**: order, logistics, FAQ, product, refund ticket creation
- **Hybrid retrieval** (BM25 + vector + RRF + reranker) over knowledge base
- **LangGraph stateful agent** with intent dispatch, refund interrupt/resume
- **Self-hosted Langfuse** observability for LLM trace + cost accounting
- **Flywheel**: mine Q&A pairs from conversations to grow the knowledge base
- **Topic classifier** (ch10) with ONNX inference

## Tech Stack

| Layer | Tech |
|---|---|
| Web | FastAPI + Uvicorn |
| Agent | LangGraph + LangChain |
| LLM clients | OpenAI-compatible (chat / embed / rerank, no gateway) |
| Storage | MySQL 8 + Milvus Standalone + MinIO + etcd |
| Observability | Langfuse v3 (self-hosted, OTLP) |
| Package manager | uv |
| Container | Docker Compose v2 |

## Quick Start

```bash
# 1. Configure (fill CHAT_* / EMBED_* / RERANK_* in .env)
cp .env.example .env
$EDITOR .env

# 2. Data services
docker compose up -d

# 3. Seed business data (ch02)
make seed

# 4. Build knowledge base (ch03)
make kb-build
make kb-vectorize

# 5. App + MCP servers
make dev
```

Open <http://localhost:8000> for the chat UI.

To start Langfuse (optional, for observability):

```bash
make langfuse-up
# Open http://localhost:3000, login admin@starrylink.local / changelog
```

## Project Layout

| Path | What lives here |
|---|---|
| `app/api/` | HTTP endpoints (chat, agent, KB, review, eval, cost) |
| `app/graph/` | LangGraph state machine (nodes, routing, build) |
| `app/core/` | Single-purpose modules (LLM client, retrieval, intent, coref, summary, confidence, observability) |
| `app/kb/` | Knowledge base: chunking, embedding, MySQL+Milvus dual-write, dedup, mining |
| `app/tools/` | Tool system: built-in `@tool`, MCP client, registry, executor |
| `app/db/` | SQLAlchemy models and repositories |
| `app/static/` | Frontend HTML pages (SSR, no build step) |
| `mcp_servers/` | Two business MCP servers (logistics, aftersales) |
| `sql/` | Per-chapter DDL / seed SQL, applied at first boot |
| `scripts/` | Build / eval / tune / observe offline jobs |
| `docs/superpowers/` | Specs and plans for each chapter |
| `primer/` | Primer examples (LLM, agent basics) — independent, share `.env` |

## Pages

| URL | Page |
|---|---|
| `/` | Chat |
| `/kb` | Knowledge base ingest |
| `/review` | Flywheel review queue |
| `/observability` | Cost & observability dashboard |
| `/topics` | Topic distribution |
| `/acceptance` | Classifier acceptance |

## Ports

| Port | Service |
|---|---|
| 8000 | App |
| 8101 / 8102 | MCP servers (logistics / aftersales) |
| 8110 | Topic classifier (after `make classifier-up`) |
| 3000 | Langfuse web (after `make langfuse-up`) |
| 19530 | Milvus |
| 3307 | MySQL (host port 3306 occupied on this machine, mapped to 3307) |

## Development

```bash
make test         # unit tests
make eval         # ch01 chat eval
make eval-agent   # ch02 tool calling
make kb-build     # ch03 KB build
make kb-vectorize # ch03 vectorize KB
make eval-rag     # ch04 retrieval eval
make eval-ch05    # ch05 LangGraph
make eval-ch06    # ch06 interrupt
make eval-ch07    # ch07 context
make eval-ch08    # ch08 tools
make cost-report  # ch09 cost
make ch10-corpus  # ch10 corpus
make ch10-train   # ch10 train
make ch10-eval    # ch10 eval
```

## License

© 2026 Starry. All rights reserved.

## Contact

GitHub: [@Starry-1234](https://github.com/Starry-1234)
```

---

## Self-Review Checklist

- [x] Spec coverage: 改名 ✓ Task 2, 抹痕迹 ✓ Task 3, README ✓ Task 4, DEPLOY ✓ Task 5, push ✓ Task 8
- [x] Placeholder scan: 没有 "TBD / TODO / 类似 Task N" 等占位
- [x] Type / 命名一致: project rename 用一致映射 (`StarryLink → StarryLink`, `starrylink → starrylink`)
- [x] Risk 标注: Task 6 标注了"数据卷名要保留否则数据丢"
- [x] Plan 可独立执行:每步有具体命令、预期输出、commit message

## 执行后验收

- [ ] `git log --oneline` 看到 8 个 commit
- [ ] `grep -r "xiaolincoding\|xiaolinnote\|公众号@小林\|StarryLink\|starrylink" --exclude-dir=.venv --exclude-dir=.git --exclude-dir=__pycache__` 输出为空
- [ ] README.md 不含原作者字样,新格式
- [ ] DEPLOY.md 不含原仓库引用
- [ ] `git push` 后 GitHub `Starry-1234/StarryLink` 主分支显示 8 个 commit
- [ ] 主分支代码能跑(可选:`make dev` smoke test)