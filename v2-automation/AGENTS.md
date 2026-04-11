# AI 知识库 - Agent 配置

> V2 自动化版本：在 V1 骨架基础上增加 Pipeline、Hooks、CI/CD

## 项目概述

这是一个 AI 技术知识库，自动采集、分析、整理 AI 领域的技术文章和开源项目。

## Agent 角色

### @collector — 内容采集员

- **职责**：从 GitHub、RSS、技术博客等渠道采集 AI 领域内容
- **触发**：`@collector 采集本周 AI 热点` 或 `@collector 搜索 MCP 相关项目`
- **定义文件**：`.opencode/agents/collector.md`
- **关联 Skill**：`collect-articles`

### @analyzer — 内容分析师

- **职责**：对采集的内容进行摘要、打分、分类
- **触发**：`@analyzer 分析这篇文章的技术价值` 或 `@analyzer 对比这两个框架`
- **定义文件**：`.opencode/agents/analyzer.md`
- **关联 Skill**：`analyze-content`

### @organizer — 内容整理

- **职责**：格式化、去重、质量把关，输出标准化 JSON
- **触发**：`@organizer 整理今天采集的内容` 或 `@organizer 检查文章格式`
- **定义文件**：`.opencode/agents/organizer.md`
- **关联 Skill**：`format-output`

## V2 新增能力

### Pipeline 自动化

- `pipeline/pipeline.py` — 四步流水线（采集 → 分析 → 整理 → 保存）
- `pipeline/model_client.py` — 统一 LLM 客户端（DeepSeek/Qwen/OpenAI）
- `pipeline/rss_sources.yaml` — RSS 数据源配置
- 运行方式：`python pipeline/pipeline.py --sources github,rss --limit 20`

### Hooks 质量校验

- `hooks/validate_json.py` — JSON 格式校验（必填字段、ID 格式、状态值）
- `hooks/check_quality.py` — 五维质量评分（A/B/C 等级）
- `.opencode/plugins/validate.ts` — OpenCode Plugin Hook，写入文章时自动触发校验

### CI/CD 定时采集

- `.github/workflows/daily-collect.yml` — GitHub Actions 每日 8:00 UTC 自动采集
- 自动提交采集结果到仓库

## 工作流

```
手动模式：@collector → @analyzer → @organizer → JSON 文件
自动模式：Pipeline（python pipeline.py）→ Hook 校验 → 保存
定时模式：GitHub Actions → Pipeline → 自动提交
```

## 文件结构

```
v2-automation/
├── AGENTS.md                    ← 本文件
├── .env.example                 ← 环境变量模板
├── requirements.txt             ← Python 依赖
├── .opencode/
│   ├── agents/
│   │   ├── collector.md         ← 采集 Agent
│   │   ├── analyzer.md          ← 分析 Agent
│   │   └── organizer.md         ← 整理 Agent
│   ├── skills/
│   │   ├── collect-articles/SKILL.md
│   │   ├── analyze-content/SKILL.md
│   │   └── format-output/SKILL.md
│   └── plugins/
│       └── validate.ts          ← Hook 校验插件
├── pipeline/
│   ├── model_client.py          ← 统一 LLM 客户端
│   ├── pipeline.py              ← 四步流水线
│   └── rss_sources.yaml         ← RSS 源配置
├── hooks/
│   ├── validate_json.py         ← JSON 校验脚本
│   └── check_quality.py         ← 质量评分脚本
├── .github/workflows/
│   └── daily-collect.yml        ← 定时采集
└── knowledge/
    ├── raw/                     ← 原始采集数据
    └── articles/                ← 标准化文章 JSON
```
