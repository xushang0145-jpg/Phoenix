# Data Model — Industrial Intel Monitor (v0.0.0)

本文件定义 v0.0.0 的持久化数据模型（SQLite）与配置文件 schema，对应 spec 的 Key Entities。RAG 相关表（向量索引）不在 v0.0.0，移至 v0.1+。

## 1. SQLite 表结构

### `competitors` — 竞品
| 列 | 类型 | 说明 |
|---|---|---|
| id | INTEGER PK | 自增 |
| name | TEXT UNIQUE | 标准名，如 `Weidmüller` |
| aliases | TEXT(JSON) | 别名数组，如 `["魏德米勒","Weidmuller"]` |
| website | TEXT | 官网域名 |
| product_lines | TEXT(JSON) | 关注产品线数组 |
| enabled | INTEGER | 0/1 |

### `sources` — 数据源
| 列 | 类型 | 说明 |
|---|---|---|
| id | INTEGER PK | 自增 |
| competitor_id | INTEGER FK | 关联 competitors |
| type | TEXT | `newsroom` / `rss` / `media` |
| list_url | TEXT | 列表页 URL |
| parse_rule | TEXT(JSON) | 选择器/字段映射，如 `{"item":"article","title":"h2","date":"time","link":"a@href"}` |
| frequency | TEXT | `daily` / `weekly` |
| enabled | INTEGER | 0/1 |

### `intel_items` — 情报条目（核心表）
| 列 | 类型 | 说明 |
|---|---|---|
| doc_id | TEXT PK | `md5(url)` |
| competitor_id | INTEGER FK | 打标得到 |
| source_id | INTEGER FK | 来源 |
| source_type | TEXT | newsroom/rss/media |
| source_url | TEXT | 原文链接（报告附录引用） |
| title | TEXT | 标题 |
| content | TEXT | 正文（去噪后） |
| summary | TEXT | ≤120 字摘要（AI 生成） |
| publish_date | TEXT | ISO 日期，解析失败留空 |
| lang | TEXT | `zh` / `en` |
| product_line | TEXT | 打标得到，可空 |
| topic_id | INTEGER FK | 聚类后回填，可空 |
| entities | TEXT(JSON) | `{"companies":[],"products":[],"tech":[],"regions":[]}` |
| sentiment | TEXT | `pos`/`neg`/`neu`，v0.0.0 可选 |
| content_hash | TEXT | 用于精确去重 |
| fetched_at | TEXT | ISO 时间戳 |
| status | TEXT | `ok` / `failed` |

**索引**：`publish_date`、`competitor_id`、`product_line`、`topic_id`；`doc_id` 唯一。

### `topics` — 主题
| 列 | 类型 | 说明 |
|---|---|---|
| id | INTEGER PK | BERTopic 主题编号 |
| name | TEXT | 人工命名 |
| keywords | TEXT(JSON) | 代表词 |
| item_count | INTEGER | 关联情报数 |
| created_at | TEXT |  |

### `ai_cache` — LLM 结果缓存（控成本，FR-015）
| 列 | 类型 | 说明 |
|---|---|---|
| id | INTEGER PK |  |
| doc_id | TEXT FK |  |
| field | TEXT | `summary` / `entities` |
| model | TEXT | 模型标识 |
| input_hash | TEXT | 输入哈希，内容未变则复用 |
| result | TEXT(JSON) |  |
| created_at | TEXT |  |

**键**：`(doc_id, field)` 唯一；命中即跳过 LLM 调用。

### `run_logs` — 运行日志（FR-033）
| 列 | 类型 | 说明 |
|---|---|---|
| id | INTEGER PK |  |
| phase | TEXT | fetch/clean/nlp/cluster/analyze/report |
| started_at / finished_at | TEXT |  |
| item_count | INTEGER | 本阶段处理条数 |
| fail_count | INTEGER |  |
| status | TEXT | `ok`/`failed` |

## 2. 配置文件 schema

### `config/competitors.yaml`
```yaml
- name: Weidmüller
  aliases: ["魏德米勒", "Weidmuller"]
  website: weidmuller.com
  product_lines: ["工业连接器", "端子", "工业以太网", "电源", "EV充电"]
  enabled: true
#  万可 WAGO / 哈亭 Harting / 泰科 TE / 西门子 同结构
```

### `config/sources.yaml`
```yaml
- competitor: Weidmüller
  type: newsroom
  list_url: https://www.weidmuller.com/.../news
  parse_rule:
    item: "div.news-item"
    title: "h3"
    date: "time"
    link: "a@href"
    detail_content: "article"
  frequency: weekly
  enabled: true
- competitor: Weidmüller
  type: rss
  list_url: https://.../feed.xml
  enabled: true
```

## 3. 数据流
`data/raw/`（原始页面）→ `clean.py` → `intel_items`（清洗后）→ `nlp.py` 回填 `summary/entities`（查 `ai_cache`）→ `cluster.py` 回填 `topic_id` → `analysis/*.py` 出图 → `report/build.py` 渲染。
