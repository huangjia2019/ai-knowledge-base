# V4 Production 知识库 — 运行指南

> V4 = V3 LangGraph 工作流 + 分发层（formatter/publisher）+ OpenClaw 网关 + Docker 容器化
>
> **核心设计：V4 不重写 V3，而是继承**。workflows/ 和 patterns/ 是从 v3-multi-agent 拷贝过来的。

---

## 0. 前置条件

```bash
cd ~/ai-knowledge-base/v4-production

# 必装依赖
pip install -r requirements.txt
# langgraph, langchain-core, openai, python-dotenv, aiohttp

# 配置 .env（复制模板）
cp .env.example .env
# 编辑 .env 至少填：LLM_API_KEY, LLM_BASE_URL, LLM_MODEL
```

---

## 1. 单元测试 — 验证 v4 和 v3 的集成

```bash
cd ~/ai-knowledge-base/v4-production

# 验证 workflows 和 patterns 从 v3 继承成功
python3 -c "
from workflows.graph import build_graph
from workflows.state import KBState
from patterns.planner import plan_strategy
from distribution.formatter import generate_daily_digest
from distribution.publisher import publish_daily_digest
print('All v4 imports OK')
g = build_graph()
print(f'LangGraph nodes: {list(g.nodes.keys())}')
"
```

预期输出：

```
All v4 imports OK
LangGraph nodes: ['plan', 'collect', 'analyze', 'review', 'revise', 'organize', 'human_flag']
```

> 7 节点拓扑：`plan → collect → analyze → review` 之后分三路——
> 通过 → `organize` (END)，未通过且未超次 → `revise → review`（循环），超次 → `human_flag` (END)。
> 没有独立的 `save` 节点，入库由 `organize` 完成。

---

## 2. 只跑流水线（不发布）— 生成知识条目

```bash
cd ~/ai-knowledge-base/v4-production
python3 -m pipeline.pipeline --no-publish
```

流水线 = V3 LangGraph 工作流（`plan → collect → analyze → review` ⇒ `organize` / `revise↺review` / `human_flag`）
- 调用 DeepSeek API
- 成本约 ¥0.005 (lite) / ¥0.01 (standard) / ¥0.02 (full)
- 结果写到 `knowledge/articles/*.json`

**策略切换**：

```bash
PLANNER_TARGET_COUNT=5  python3 -m pipeline.pipeline --no-publish  # lite
PLANNER_TARGET_COUNT=15 python3 -m pipeline.pipeline --no-publish  # standard
PLANNER_TARGET_COUNT=30 python3 -m pipeline.pipeline --no-publish  # full
```

---

## 3. 只发布简报（不跑流水线）

```bash
cd ~/ai-knowledge-base/v4-production
python3 daily_digest.py
```

`daily_digest.py` 是独立发布入口：
- 从 `knowledge/articles/` 读现有文章
- 用 `distribution/formatter.py` 生成三种格式简报
- 用 `distribution/publisher.py` 并发推送到 Telegram + 飞书 + 文件

**必须先配置**：

```ini
TELEGRAM_BOT_TOKEN=123456789:ABCxxx
TELEGRAM_CHAT_ID=-1001234567890
FEISHU_WEBHOOK_URL=https://open.feishu.cn/open-apis/bot/v2/hook/xxx
```

如果没配置，对应渠道会失败但不会崩溃——其他渠道继续推送。

---

## 4. 完整流水线（跑流水线 + 发布）

```bash
cd ~/ai-knowledge-base/v4-production
python3 -m pipeline.pipeline
```

**调用链**：

```
python -m pipeline.pipeline
    │
    ▼ Stage A: V3 LangGraph 工作流
    workflows.graph.app.invoke()
        plan → collect → analyze → review ┬ organize (END)
                                          ├ revise → review (循环)
                                          └ human_flag (END)
    │
    ▼ Stage B: V4 分发层（新增）
    distribution.publisher.publish_daily_digest()
        formatter → {telegram, feishu, file} 三种格式
        并发推送到对应渠道
```

---

## 5. OpenClaw 网关（交互式 Bot）

```bash
# 启动 OpenClaw 网关守护进程
openclaw daemon start
openclaw daemon status
```

OpenClaw 读取 `openclaw/openclaw.json5`（注意是 JSON5 不是 JSON），
根据用户消息的关键词路由到不同 Agent：

- 知识 / 搜索 / 查询 → `knowledge-query` Agent → `skills/knowledge-query/SKILL.md`
- 简报 / 今日 / daily → `daily-briefing` Agent → `skills/daily-digest/SKILL.md`
- 订阅 / subscribe → `subscription-manager` Agent
- 其他 → `general-chat` Agent

---

## 6. Docker 部署（生产）

```bash
cd ~/ai-knowledge-base/v4-production

# 构建镜像
docker compose build

# 启动全部三个服务：pipeline + bot + openclaw
docker compose up -d

# 查看日志
docker compose logs -f pipeline
docker compose logs -f bot
docker compose logs -f openclaw

# 停止
docker compose down
```

三个服务：

| 服务 | 职责 | 触发方式 |
|:-----|:-----|:---------|
| `pipeline` | 跑 V3 LangGraph 工作流 + 发布 | Cron（08:00 和 20:00） |
| `bot` | Telegram 交互 Bot | 常驻 |
| `openclaw` | 消息网关 | 常驻，监听 3000 端口 |

Cron 配置在 `docker-compose.yml` 的 pipeline 服务 command 字段里。

---

## 7. 常见问题

### Q1: V4 的 workflows/ 和 V3 的 workflows/ 是不是重复？

是的，这是**有意的继承**。V4 把 V3 的 workflows/ 和 patterns/ 完整拷贝过来，好处：

- V4 可以独立部署（Docker 镜像只 COPY 一个项目）
- V4 可以单独迭代而不影响 V3 的教学版本
- 学生可以看到"继承"在目录层面的体现

如果 V3 改了 workflows/，V4 需要手动同步（或者写脚本自动同步）。

### Q2: pipeline.py 为什么是一个薄封装？

历史原因 + 教学清晰：

- 历史：V2 时代 pipeline.py 是自带采集/分析/整理逻辑的完整脚本
- 教学：V3 引入 LangGraph 后，pipeline 变成"入口壳子"——只负责 import graph 并跑
- V4 的 pipeline.py 保留了这个壳子，但内部彻底替换为 `workflows.graph.app.invoke()`
- 好处：调用方式没变（`python -m pipeline.pipeline`），实现从线性升级到 LangGraph

### Q3: 发布失败后会卡住流水线吗？

不会。`run_pipeline` 里发布阶段包了 `try/except`，失败只记录日志，articles 已经写磁盘。

### Q4: 想只测试 V3 工作流部分怎么办？

两种方式：

```bash
# 方式 A：直接跑 v3 的 workflows/graph.py
cd ~/ai-knowledge-base/v3-multi-agent
python3 -m workflows.graph

# 方式 B：跑 v4 的 pipeline 但不发布
cd ~/ai-knowledge-base/v4-production
python3 -m pipeline.pipeline --no-publish
```

两种方式底层调用的是同一套 LangGraph 代码。

---

## 8. V3 → V4 差异一览

| 维度 | V3 | V4 |
|:-----|:---|:---|
| workflows/ | 原生 | 从 V3 拷贝（同步维护） |
| patterns/ | 原生 | 从 V3 拷贝 |
| pipeline.py | 无（v3 直接用 workflows/graph.py） | 薄封装 + 发布阶段 |
| distribution/ | 无 | formatter.py + publisher.py |
| bot/ | 无 | knowledge_bot.py |
| openclaw/ | 无 | AGENTS.md + SOUL.md + openclaw.json5 + skills/ |
| tests/ | cost_guard / security / eval_test | 继承自 v3（通过 sys.path） |
| Dockerfile | 无 | 多阶段构建，COPY v3 + v4 所有代码 |
| docker-compose.yml | 无 | pipeline + bot + openclaw 三服务 |
| daily_digest.py | 无 | 独立发布入口（Cron 可单独调用） |

---

## 9. 端到端冒烟测试（~2 分钟）

```bash
cd ~/ai-knowledge-base/v4-production

# Step 1: 测 imports
python3 -c "from workflows.graph import build_graph; from distribution.publisher import publish_daily_digest; print('imports OK')"

# Step 2: 跑流水线（lite 策略，最省）
PLANNER_TARGET_COUNT=5 python3 -m pipeline.pipeline --no-publish

# Step 3: 验证生成的知识库
ls knowledge/articles/ | head
cat knowledge/articles/index.json | python3 -m json.tool | head -20
```

预期：`knowledge/articles/` 有 5 个 JSON 文件 + 一个 `index.json`。
