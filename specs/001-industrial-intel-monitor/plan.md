# Implementation Plan: 工业自动化竞品情报监测（Industrial Intel Monitor）

**Branch**: `001-industrial-intel-monitor` | **Date**: 2026-11-24 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-industrial-intel-monitor/spec.md`（v0.0.0，Status: Approved）

**Note**: 本计划由 `/speckit-plan` 产出。v0.0.0 **不含 RAG**（US3 / FR-027~FR-030 / SC-005 已移至 v0.1+，spec 中保留占位）。

## Summary

为求职作品搭建一条**工业自动化竞品情报批处理流水线**：按配置自动采集 5 家竞品近 12 个月的公开动态（官网 Newsroom / RSS / 行业媒体），完成正文抽取、去重、日期与语言解析、竞品与产品线打标；再经 AI 加工（≤120 字摘要、公司/产品/技术/地区实体抽取、embedding 主题聚类），做发布时序、产品线热度、主题趋势、竞品对比矩阵四类量化分析，最终生成一份含结论与 ≥3 条我方启示的**静态 HTML 情报报告**。全流程一条命令可复现，LLM 结果缓存以控成本。报告形态、LLM 选型、竞品名单、摘要长度均已在 spec 中闭合（Q1/Q3/Q4/Q5）。

## Technical Context

**Language/Version**: Python 3.10+

**Primary Dependencies**:
- 采集：`requests` / `httpx`、`feedparser`（RSS）、`beautifulsoup4` + `lxml`、`playwright`（JS 动态渲染页）、`tenacity`（重试）
- 正文抽取：`trafilatura`，回退 `readability-lxml`
- 数据：`pandas`、`sqlalchemy`（SQLite）
- NLP：`sentence-transformers`（`BAAI/bge-m3` 多语 embedding）、`bertopic`、`scikit-learn`、`jieba`、`openai`（或兼容 API，摘要/NER）
- 可视化：`matplotlib`
- 报告：`jinja2`（HTML），可选 `weasyprint`（PDF）
- 配置：`pyyaml`
- 测试：`pytest`

**Storage**: SQLite（`data/intel.db`）+ 原始页面文件缓存（`data/raw/`，按源/日期分桶）+ 中间产物 CSV/JSON（`data/processed/`）

**Testing**: `pytest` —— 仅覆盖核心纯函数（去重哈希、日期解析、产品线打标规则、schema 校验），不写端到端网络测试（依赖外部站点）。

**Target Platform**: Linux / macOS 单机（Windows 需额外处理 Playwright 与 weasyprint）

**Project Type**: CLI 批处理应用（入口 `python run_all.py`），无长驻服务、无前端（报告为静态 HTML 产物）

**Performance Goals**:
- 600–1000 条全流程（含 LLM 调用）在单机 < 30 分钟
- AI 结果缓存命中后，二次全流程运行 < 5 分钟
- 抓取单源限速 ≥ 1s/请求，失败重试 ≤ 3 次

**Constraints**:
- 非实时：按日/周批处理，无流式
- LLM 预算受限：必须批量 + 缓存，不重复计费
- 合规：仅抓公开数据、限速、不存个人隐私、报告注明来源链接
- 可离线复现：原始页面落盘缓存，二次运行不依赖网络
- 不引入生产级高可用、消息队列、容器编排

**Scale/Scope**: 5 竞品 × 3 类源 × 近 12 个月 ≈ 600–1000 条；无 RAG（v0.1+）；无社媒/付费源。

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

项目暂未建立 `.specify/memory/constitution.md`，采用以下默认门并逐项核对，**v0.0.0 全部通过**：

| 门 | 要求 | 满足情况 |
|---|---|---|
| 合规与数据边界 | 仅公开数据、限速、不采隐私、注明来源 | ✅ Assumptions + FR-005/006/025 |
| 最小依赖 | 优先 stdlib 与成熟库，不引入重型框架 | ✅ 无 FastAPI/前端/队列 |
| 可复现 | 一键运行 + 原始缓存，新环境可复跑 | ✅ SC-002 + raw 缓存 |
| 范围克制 | v0.0.0 不混入 RAG/生产级高可用 | ✅ Q2 已闭合 |
| 质量可度量 | 去重/抽检/图表/报告均有量化验收 | ✅ SC-001~SC-004、SC-006 |

> 若后续建立 constitution，需回填本处对应条目。

## Project Structure

### Documentation (this feature)

```text
specs/001-industrial-intel-monitor/
├── spec.md             # /speckit-specify 产出（已 Approved）
├── plan.md             # 本文件（/speckit-plan 产出）
├── data-model.md       # Phase 1 产出（本阶段已生成）
├── quickstart.md       # Phase 1 产出（本阶段已生成）
├── checklists/
│   └── requirements.md # 需求质量清单（已全闭合）
└── mockups/            # 报告 / RAG(预留) / 架构 原型
```

### Source Code (repository root)

```text
industrial-intel/
├── config/
│   ├── competitors.yaml        # 竞品清单 + 别名 + 关注产品线
│   └── sources.yaml            # 数据源：类型/URL/解析规则/频率
├── src/
│   ├── crawler/
│   │   ├── fetcher.py          # 限速/重试/缓存/UA
│   │   ├── spiders/            # 每站一个解析器（列表页+详情页）
│   │   └── scheduler.py        # APScheduler 或 cron 调度
│   ├── pipeline/
│   │   ├── schema.py           # 统一数据模型（见 data-model.md）
│   │   ├── clean.py            # 正文抽取/去重/日期/语言/打标
│   │   ├── nlp.py              # 摘要/NER（LLM，带缓存）
│   │   └── cluster.py          # embedding + BERTopic
│   ├── analysis/
│   │   ├── trend.py            # 发布时序
│   │   ├── compare.py          # 竞品 × 产品线矩阵
│   │   └── visualize.py        # matplotlib 出图
│   └── report/
│       ├── template.html       # Jinja2 报告模板
│       └── build.py            # 渲染 → HTML（可选 PDF）
├── data/
│   ├── raw/                    # 原始页面缓存（离线复现）
│   ├── processed/              # 中间 CSV/JSON
│   └── intel.db                # SQLite
├── outputs/
│   ├── figures/                # 图表 png
│   └── report_YYYY-MM.html     # 最终报告
├── tests/                      # pytest（纯函数单元）
├── run_all.py                  # 一键入口：编排全部阶段
├── requirements.txt
└── README.md
```

**Structure Decision**: 单项目 + `src/` 分层（Option 1）。无前后端分离（报告为静态 HTML 产物），无 API 服务层（v0.0.0 不做 RAG/在线问答），因此不引入 `backend/`、`frontend/` 与 `contracts/`。RAG 链路（v0.1+）将在 `src/rag/` 下新增，不影响本结构。

## Complexity Tracking

> **无 constitution 违反**，下表无需填充。总体复杂度评估如下。

| 维度 | 评分 | 说明 |
|---|---|---|
| 总体复杂度 | **Medium** | 链路长但每段技术成熟，风险集中在数据源适配与聚类效果 |

| 风险因子 | 影响 | 缓解措施 |
|---|---|---|
| 反爬 / 动态渲染 | 部分源抓不到 → 数据量不足 | Playwright 兜底；按站配 CSS 选择器；失败记录日志；目标量按“可用源”复核 |
| 正文抽取不准 | 报告内容噪声多 → 抽检不达标 | trafilatura 优先 + 按站规则兜底；SC-003 抽检 30 条强制校验 |
| 聚类效果差 | 主题不可解释 → 洞察弱 | 用 `bge-m3` 多语模型；`min_topic_size` 调参；主题名人工标注 |
| LLM 成本 / 不可用 | 超预算或中断 | 批量 + 结果缓存（按 `doc_id` 哈希）；失败回退 extractive 摘要 |
| 范围蔓延 | 工期失控 | 竞品 ≤5、时间窗 ≤12 月、源 ≤3 类，到量即停 |
| 合规 | 法律风险 | 仅公开数据、限速、不存隐私、报告附来源；遵守 robots/ToS |

## Implementation Phases

> 每个阶段结束有可检查的产物；建议每天 2–3 小时，约 10–12 个工作日完成 v0.0.0。

### Phase 0 · 定题与搭架子（Day 1）
- 建立仓库、`requirements.txt`、虚拟环境、`src/` 目录骨架
- 写 `config/competitors.yaml`（5 家）与 `config/sources.yaml`（3 类源占位）
- **产物**：可 `import` 不报错的空框架 + 配置文件

### Phase 1 · 采集（Day 2–4）
- `fetcher.py`：限速 1–2s、重试 3 次、raw 落盘缓存、UA 轮换
- 先跑通 **1 家官网**（列表页 + 详情页），再复制到其余 4 家
- 加 RSS 与 2–3 个行业媒体源
- **产物**：`data/raw/` 缓存 + 初始入库 ≥600 条；随机抽 20 条正文准确率 ≥85%

### Phase 2 · 清洗治理（Day 5）
- 正文抽取（trafilatura → readability 回退 → 按站 CSS）
- 去重：URL 规范化 + `title+content` 精确哈希 + simhash 近似（阈值 3）
- 日期多格式解析、语言识别、竞品/产品线关键词打标
- **产物**：`intel.db` 干净结构化表；重复率 <2%，日期有效率 >95%

### Phase 3 · AI 加工（Day 6–8）
- 摘要（≤120 字）+ 四类实体抽取（LLM，批量 + 缓存）
- `bge-m3` embedding → BERTopic 聚类 → 人工命名主题
- 抽样 30 条质检
- **产物**：每条带 `summary / entities / topic`

### Phase 4 · 量化分析与可视化（Day 9–10）
- 发布频率时序、产品线热度、主题趋势、竞品 × 产品线矩阵
- 每张图配一句结论
- **产物**：`outputs/figures/` 下 ≥4 张图

### Phase 5 · 报告生成（Day 11）
- Jinja2 渲染固定章节 → `outputs/report_YYYY-MM.html`
- 写 Highlights + ≥3 条对我方启示 + 附录来源链接
- **产物**：1 份可阅读的 HTML 报告（SC-004 / SC-006 达标）

### Phase 6 · 打包与演练（Day 12）
- `run_all.py` 一键跑通、写 README（架构图 + 复现步骤 + 样例截图）
- 按「目标—方法—发现」练 5 分钟讲述
- **产物**：可交付的 GitHub 仓库 + 演示材料

## Acceptance Mapping（plan → spec）

| spec 验收 | plan 对应阶段 |
|---|---|
| SC-001（≥600 条，每家 ≥50） | Phase 1–2 |
| SC-002（一条命令跑通） | Phase 6（`run_all.py`） |
| SC-003（抽检 85% / 95% / <2%） | Phase 1–2 校验 |
| SC-004（≥4 图 + 结论 + ≥3 启示） | Phase 4–5 |
| SC-006（5 分钟讲清） | Phase 6 演练 |
| SC-005 | **v0.1+，不在本计划** |
