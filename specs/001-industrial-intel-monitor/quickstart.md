# Quickstart — Industrial Intel Monitor (v0.0.0)

5 分钟跑通：采集 → 治理 → AI 加工 → 量化分析 → 生成 HTML 报告。

## 1. 环境要求
- Python 3.10+
- 一个可用的 LLM API（云端，已确认 Q4），设置环境变量：
  ```bash
  export LLM_API_KEY="你的key"
  export LLM_BASE_URL="https://..."   # 如使用兼容端点
  export LLM_MODEL="gpt-4o-mini"      # 或你选定的模型
  ```
- （可选）`playwright install chromium`，仅当目标源为 JS 动态渲染时需要。

## 2. 安装
```bash
git clone <repo> && cd industrial-intel
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

## 3. 配置（已含默认 5 家竞品）
```bash
# 确认/调整竞品与数据源
cat config/competitors.yaml
cat config/sources.yaml
```
> 首次运行建议先只启用 1 家竞品 + 1 个 RSS 源跑通链路，再逐步放开。

## 4. 运行
```bash
python run_all.py
```
阶段输出：
```
[fetch]   采集完成，新增 612 条，失败 3 条
[clean]   去重后 598 条，重复率 1.4%
[nlp]     摘要/实体完成（缓存命中 0，新调用 598）
[cluster] 聚出 12 个主题
[analyze] 生成 5 张图表
[report]  已生成 outputs/report_2026-11.html
```
- 二次运行命中缓存，AI 阶段几乎零成本。

## 5. 查看产物
- 报告：`outputs/report_2026-11.html`（浏览器打开）
- 图表：`outputs/figures/`
- 数据库：`data/intel.db`（可用 DB Browser / `sqlite3` 查看）
- 原始缓存：`data/raw/`（保证可离线复现）

## 6. 常见问题
| 现象 | 处理 |
|---|---|
| 某源抓取 0 条 | 站点反爬/改版 → 改用 Playwright 或调整 `parse_rule`；失败会记入 `run_logs` |
| 正文噪声多 | 检查该源是否命中 trafilatura，必要时在 `sources.yaml` 配 `detail_content` CSS |
| LLM 调用慢/贵 | 确认 `ai_cache` 命中；二次运行应跳过已处理条目 |
| 聚类主题乱 | 调 `min_topic_size`（5–10），并在 `cluster.py` 中人工给主题命名 |
| 报告无启示 | `report/build.py` 需补充规则或人工撰写 ≥3 条我方启示 |

## 7. 验收自检
- [ ] 近 12 个月去重情报 ≥600 条，每家 ≥50（SC-001）
- [ ] 一条命令跑通并产出报告（SC-002）
- [ ] 抽 30 条：正文 ≥85%、日期 ≥95%、重复 <2%（SC-003）
- [ ] 报告 ≥4 图 + 结论 + ≥3 条启示（SC-004）
- [ ] 5 分钟能讲清目标—方法—发现（SC-006）

> RAG 问答（SC-005）不在 v0.0.0，见 `mockups/rag-qa.html`（v0.1+ 预留）。
