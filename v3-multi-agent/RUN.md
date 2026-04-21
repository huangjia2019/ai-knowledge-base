# V3 Multi-Agent 知识库 — 运行指南

> 本文档说明如何从零跑通 V3 的三种场景：单元测试、API 冒烟、端到端 LangGraph 工作流。

---

## 0. 前置条件

```bash
# 必须安装
pip install -r requirements.txt  # langgraph >= 0.2, langchain-core >= 0.3, openai >= 1.0

# 必须配置 .env
cp .env.example .env
# 编辑 .env 填入 DeepSeek / Qwen / OpenAI 的 API Key
```

`.env` 必填字段：

```ini
LLM_API_KEY=sk-xxxxxxxxxxxxxxxx
LLM_BASE_URL=https://api.deepseek.com
LLM_MODEL=deepseek-chat
PRICE_INPUT_PER_MILLION=1.0   # DeepSeek 输入单价
PRICE_OUTPUT_PER_MILLION=2.0  # DeepSeek 输出单价
PLANNER_TARGET_COUNT=10       # Planner 目标采集量（决定策略）
```

可选字段：

```ini
GITHUB_TOKEN=ghp_xxxxx  # GitHub API rate limit 更宽松
```

---

## 1. 单元测试 — 最快验证

所有不调 LLM 的模块测试：

```bash
cd ~/ai-knowledge-base/v3-multi-agent

# 测 Planner（零 LLM 调用）
python3 -m patterns.planner

# 测 CostGuard（零 LLM 调用）
python3 tests/cost_guard.py

# 测 Eval 结构（不带 --slow 的话零 LLM 调用）
python3 tests/eval_test.py
```

预期：每个脚本都应该打印测试输出并退出码 0。

---

## 2. API 冒烟测试 — 验证 .env 配置正确

```bash
cd ~/ai-knowledge-base/v3-multi-agent

python3 -c "
from dotenv import load_dotenv; load_dotenv()
from workflows.model_client import chat
text, usage = chat('用一句话说 DeepSeek 是什么？', max_tokens=100)
print('Response:', text[:200])
print('Usage:', usage)
"
```

预期输出：

```
Response: DeepSeek 是由深度求索公司开发的先进人工智能助手...
Usage: {'prompt_tokens': 21, 'completion_tokens': 23}
```

**如果失败**：

| 错误 | 原因 | 解决 |
|:-----|:-----|:-----|
| `AuthenticationError` | API Key 无效 | 检查 `.env` 的 `LLM_API_KEY` |
| `openai.APIConnectionError` | 网络或 base_url 错 | 检查 `LLM_BASE_URL` |
| `ModuleNotFoundError: openai` | 依赖未装 | `pip install openai python-dotenv` |

---

## 3. 端到端 LangGraph 工作流

### 3.1 默认运行（standard 策略）

```bash
cd ~/ai-knowledge-base/v3-multi-agent
python3 -m workflows.graph
```

默认 `PLANNER_TARGET_COUNT=10`（已在 .env），走 **standard 策略**：
- `per_source_limit=10`（每源抓 10 条）
- `relevance_threshold=0.5`（过滤阈值）
- `max_iterations=2`（审核最多 2 次）

### 3.2 切换策略

**Lite 策略**（成本优先，抓 5 条，审核 1 次）：

```bash
PLANNER_TARGET_COUNT=5 python3 -m workflows.graph
```

**Full 策略**（质量优先，抓 20 条，审核 3 次）：

```bash
PLANNER_TARGET_COUNT=30 python3 -m workflows.graph
```

### 3.3 预期输出

```
============================================================
AI 知识库 V3 — LangGraph 工作流启动
============================================================
[Planner] 策略=lite, 每源=5 条, 阈值=0.7, 目标 5 条，启用精简模式（成本优先）

--- [plan] 完成 ---
  策略: lite
[Collector] 采集到 5 条原始数据

--- [collect] 完成 ---
  采集数量: 5
[Analyzer] 完成 5 条分析

--- [analyze] 完成 ---
  分析数量: 5
  累计成本: ¥0.002611
[Reviewer] 加权总分: 7.2/10, 通过: True (第 1 次审核)

--- [review] 完成 ---
  审核结果: 通过
  迭代次数: 1/1
  累计成本: ¥0.005557
[Organizer] 整理出 5 条知识条目（准备入库）
[Organizer] 已写入 5 篇到磁盘
[Organizer] 本次运行总成本: ¥0.005557

--- [organize] 完成 ---
  入库数量: 5
  累计成本: ¥0.005557

============================================================
工作流执行完毕
============================================================
```

> 拓扑：`plan → collect → analyze → review` 之后分三路——审核通过 → `organize` (END)，未通过且未超次 → `revise → review`（循环），超次 → `human_flag` (END)。organize 是正常终点，没有独立 save 节点。

### 3.4 成本对比（实测）

| 策略 | target_count | 抓取 | 分析 | 成本 | 循环数 |
|:-----|:-------------|:----|:----|:----|:--------|
| lite | 5 | 5 | 5 | ~¥0.0051 | 1/1 |
| standard | 15 | 10 | 10 | ~¥0.0089 | 1-2/2 |
| full | 30 | 20 | 20 | ~¥0.018 | 1-3/3 |

---

## 4. 查看生成的知识库

```bash
# 列出所有文章
ls knowledge/articles/

# 看某一篇
cat knowledge/articles/2026-04-11-000.json

# 看索引
cat knowledge/articles/index.json | python3 -m json.tool | head -30
```

每篇文章字段：

```json
{
  "id": "2026-04-11-000",
  "title": "langgenius/dify",
  "source": "github",
  "url": "https://github.com/langgenius/dify",
  "collected_at": "2026-04-11T13:19:51...",
  "summary": "Dify 是一个开源的 LLM 应用开发平台...",
  "tags": ["LLM 应用开发", "可视化工作流", "RAG", "开源平台", "智能体"],
  "relevance_score": 0.9,
  "category": "agent",
  "key_insight": "Dify 通过将复杂的智能体工作流开发抽象为可视化编排..."
}
```

---

## 5. 常见问题

### Q1: Reviewer 报 `Extra data: line 12 column 1`

LLM 在 JSON 后加了尾巴文本。`chat_json` 现在有三层容错（markdown → 直接 → 正则提取），应该自动恢复。如果还是报错，检查 `workflows/model_client.py::chat_json` 是否是最新版。

### Q2: GitHub API 返回 `API rate limit exceeded`

未配置 `GITHUB_TOKEN`。每小时未认证 60 次，认证后 5000 次。去 GitHub → Settings → Developer settings → Personal access tokens 生成一个。

### Q3: `BudgetExceededError` 熔断

成本超过 `BUDGET_LIMIT_YUAN`。正常跑一次 < ¥0.02，如果异常超标检查是不是 review 循环死锁。

### Q4: `knowledge/articles/` 没生成文件

- 相关性都 < threshold：降低 `PLANNER_TARGET_COUNT` 触发 lite 策略（threshold=0.7→0.5→0.4）
- GitHub API 失败：收集全空，organize 过滤后为空

### Q5: 想跑审核失败的分支

调高 `relevance_threshold` 到 `0.95` 或改 prompt 让 Reviewer 更严格。或者手动在 analyses 里注入一条垃圾数据看循环行为。

---

## 6. 完整运行链路图

```
.env + PLANNER_TARGET_COUNT
        │
        ▼
┌─── patterns/planner.py::plan_strategy() ───┐
│    返回 {strategy, per_source_limit,      │
│         relevance_threshold, max_iterations}│
└──────────────┬──────────────────────────────┘
               │ state["plan"]
               ▼
    workflows/nodes.py::collect_node
    (按 per_source_limit 抓 GitHub API)
               │
               ▼
    workflows/nodes.py::analyze_node
    (每条数据一次 LLM 分析)
               │
               ▼
    workflows/reviewer.py::review_node
    (LLM 四维评分，读 plan.max_iterations 兜底)
               │
        ┌──────┴──────┬─────────────┐
        ▼             ▼             ▼
    organize      revise        human_flag
    (通过, END)   (未通过,回 review) (超次, END)
        │
        ▼
    knowledge/articles/*.json
```

---

## 7. 运行演示脚本（一键跑三种策略）

```bash
# 创建 demo.sh
cat > /tmp/v3_demo.sh <<'SCRIPT'
#!/bin/bash
cd ~/ai-knowledge-base/v3-multi-agent

echo "====== Lite 策略 ======"
PLANNER_TARGET_COUNT=5 python3 -m workflows.graph 2>&1 | tail -20

echo ""
echo "====== Standard 策略 ======"
PLANNER_TARGET_COUNT=15 python3 -m workflows.graph 2>&1 | tail -20

# Full 模式需谨慎运行（成本较高且 GitHub API 可能触发限制）
# echo ""
# echo "====== Full 策略 ======"
# PLANNER_TARGET_COUNT=30 python3 -m workflows.graph 2>&1 | tail -20
SCRIPT
bash /tmp/v3_demo.sh
```
