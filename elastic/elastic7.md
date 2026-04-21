# Elasticsearch 核心补全：六大实用 API 补充指南

> 本篇是对 `elastic6.md`（核心 API 全面指南）的**补充与补全**。`elastic6.md` 已系统讲解了 Index API、Document API、Search API、Cluster API、Analyze API 五大核心 API。本篇将覆盖六大高频实用 API：**Ingest API、Multi-Search API、Index Template API、Snapshot API、Reindex API、Point In Time API**，每个 API 都配有真实场景与代码示例。

## 📑 目录

- [一、Ingest API（数据预处理管道）](#ingest-api)
  - [1. Pipeline 概念与核心功能](#pipeline-concept)
  - [2. 内置 Processor 速览](#built-in-processors)
  - [3.实战示例：日志写入前自动解析与 GeoIP 补全](#ingest-example)
- [二、Multi-Search API（批量搜索）](#msearch-api)
  - [1. 基本用法](#msearch-basic)
  - [2. 实战示例：同时搜索多个索引](#msearch-example)
- [三、Index Template API（索引模板）](#index-template-api)
  - [1. 组件模板（Component Template）](#component-template)
  - [2. 可组合索引模板（Composable Index Template）](#composable-template)
  - [3. 实战示例：日志索引的标准化模板](#template-example)
- [四、Snapshot & Restore API（快照与恢复）](#snapshot-api)
  - [1. 注册仓库](#register-repo)
  - [2. 创建与恢复快照](#create-restore-snapshot)
  - [3. 实战示例：集群数据迁移](#snapshot-example)
- [五、Reindex API（跨索引数据迁移）](#reindex-api)
  - [1. 基础用法](#reindex-basic)
  - [2. 远程 Reindex](#reindex-remote)
  - [3. 实战示例：Mapping 变更后的数据迁移](#reindex-example)
- [六、Point In Time API（时间点视图）](#pit-api)
  - [1. 创建与使用 PIT](#pit-usage)
  - [2. 实战示例：一致性深度分页](#pit-example)
- [七、补充 API 总览对比表](#api-summary)

---

<a id="ingest-api"></a>

## 一、Ingest API（数据预处理管道）

Ingest Pipeline 是 Elasticsearch 内置的 **轻量级 ETL 引擎**。文档在正式写入 Data Node 之前，可以在 Ingest Node 上被拦截、转换和丰富——无需额外部署 Logstash。

> 💡 **通俗类比：**
> Ingest Pipeline 就像是仓库入口的 **安检与包装流水线**——货物（文档）正式入库之前，先过一遍 X 光机（解析）、贴标签（补字段）、扔掉违禁品（删字段），然后才放入货架。

<a id="pipeline-concept"></a>

### 1. Pipeline 概念与核心功能

- **关键端点**：`PUT /_ingest/pipeline/<pipeline_name>`
- **核心流程**：一个 Pipeline 由多个有序的 **Processor（处理器）** 组成，文档依次流经每个 Processor 进行处理。
- **触发方式**：写入文档时通过 `pipeline` 参数指定，或者设置索引的 `default_pipeline` 自动触发。

```json
// 创建一个简单的 Pipeline
PUT /_ingest/pipeline/my_pipeline
{
  "description": "写入前预处理",
  "processors": [
    {
      "set": {
        "field": "ingested_at",
        "value": "{{_ingest.timestamp}}"
      }
    }
  ]
}

// 写入文档时指定 Pipeline
POST /my_index/_doc?pipeline=my_pipeline
{
  "title": "测试文档"
}
// 结果：文档会自动多出一个 "ingested_at" 字段
```

<a id="built-in-processors"></a>

### 2. 内置 Processor 速览

| Processor | 功能 | 典型场景 |
| :--- | :--- | :--- |
| `set` | 设置或覆盖字段值 | 自动注入时间戳、默认值 |
| `remove` | 删除指定字段 | 去除敏感信息 |
| `rename` | 重命名字段 | 字段名规范化 |
| `convert` | 转换字段类型 | 字符串转数字 |
| `grok` | 正则提取结构化数据 | 解析 Apache/Nginx 日志 |
| `date` | 解析日期字符串为标准格式 | 统一时间格式 |
| `geoip` | 根据 IP 地址补全地理位置 | 访问日志地理分析 |
| `script` | 执行 Painless 脚本 | 自定义复杂转换逻辑 |
| `lowercase` / `uppercase` | 大小写转换 | 数据规范化 |
| `split` | 按分隔符拆分为数组 | 标签字符串拆分 |
| `join` | 数组合并为字符串 | 反向操作 |
| `pipeline` | 嵌套调用其他 Pipeline | 复用已有处理逻辑 |

<a id="ingest-example"></a>

### 3. 实战示例：日志写入前自动解析与 GeoIP 补全

```json
// 1. 创建日志处理 Pipeline
PUT /_ingest/pipeline/access_log_pipeline
{
  "description": "解析 Nginx 访问日志并补全地理位置",
  "processors": [
    // 第一步：用 Grok 正则从日志原文中提取结构化字段
    {
      "grok": {
        "field": "message",
        "patterns": ["%{IP:client_ip} %{WORD:method} %{URIPATH:path} %{NUMBER:status:int} %{NUMBER:bytes:int}"]
      }
    },
    // 第二步：将 IP 地址转换为地理位置
    {
      "geoip": {
        "field": "client_ip",
        "target_field": "geo_info"
      }
    },
    // 第三步：自动注入处理时间戳
    {
      "set": {
        "field": "@timestamp",
        "value": "{{_ingest.timestamp}}"
      }
    },
    // 第四步：删除原始日志原文（节省存储）
    {
      "remove": {
        "field": "message"
      }
    }
  ]
}

// 2. 写入原始日志，Pipeline 自动处理
POST /access_logs/_doc?pipeline=access_log_pipeline
{
  "message": "192.168.1.100 GET /api/users 200 1536"
}

// 3. 写入后文档结构（自动生成）
// {
//   "client_ip": "192.168.1.100",
//   "method": "GET",
//   "path": "/api/users",
//   "status": 200,
//   "bytes": 1536,
//   "geo_info": {
//     "city_name": "Shanghai",
//     "country_name": "China",
//     "location": { "lat": 31.2304, "lon": 121.4737 }
//   },
//   "@timestamp": "2024-09-20T10:30:00.000Z"
// }
```

- **注意事项**：
  - `geoip` Processor 需要安装 Elasticsearch 内置的 GeoIP 数据库插件。
  - Pipeline 中 Processor 的执行顺序很重要——比如先 `grok` 解析出 `client_ip`，后续的 `geoip` 才能使用这个字段。
  - 可通过 `POST /_ingest/pipeline/my_pipeline/_simulate` 端点 **模拟测试** Pipeline 的处理结果，不上传真实数据。

---

<a id="msearch-api"></a>

## 二、Multi-Search API（批量搜索）

Multi-Search API 允许在 **一次 HTTP 请求中执行多个搜索查询**，大幅减少客户端与 ES 集群之间的网络往返次数。

> 💡 **核心认知：**
> Multi-Search 的请求格式与 Bulk API 类似，使用 **NDJSON 格式**——每个查询由两行组成：第一行是请求头（指定索引和可选参数），第二行是查询体。

<a id="msearch-basic"></a>

### 1. 基本用法

- **关键端点**：`GET /_msearch`、`POST /_msearch`、`POST /<index>/_msearch`

```json
// 请求格式：NDJSON（每行一个独立 JSON）
POST /_msearch
{"index": "products"}
{"query": {"match": {"name": "手机"}}, "size": 2}
{"index": "orders"}
{"query": {"range": {"amount": {"gte": 1000}}}, "size": 2}
```

<a id="msearch-example"></a>

### 2. 实战示例：同时搜索多个索引

```json
// 场景：电商首页需要同时展示 "搜索结果" + "推荐商品" + "最近订单"
POST /_msearch
// 第 1 个查询：搜索商品
{"index": "products", "search_type": "query_then_fetch"}
{"query": {"match": {"name": "蓝牙耳机"}}, "size": 5, "_source": ["name", "price"]}

// 第 2 个查询：热门推荐
{"index": "products"}
{"query": {"term": {"tags": "热卖"}}, "size": 3, "sort": [{"sales": {"order": "desc"}}]}

// 第 3 个查询：用户最近订单
{"index": "orders", "routing": "user_123"}
{"query": {"term": {"user_id": "user_123"}}, "size": 5, "sort": [{"created_at": {"order": "desc"}}]}
```

- **注意事项**：
  - 多个查询之间 **互不影响**——某个查询失败不会导致其他查询中断。
  - 建议控制单次 `_msearch` 的查询数量（建议不超过 100 个），避免协调节点内存压力过大。
  - 每个 `hits` 数组中的 `status` 字段会标记该查询是否成功（`200` 表示成功）。

---

<a id="index-template-api"></a>

## 三、Index Template API（索引模板）

Index Template 是 Elasticsearch 中实现 **索引自动标准化** 的核心机制。当你按照日期模式（如 `log-2024-09-20`）或业务前缀（如 `orders-v1`）动态创建索引时，模板会自动为新索引注入预定义的 settings 和 mappings。

> 💡 **通俗类比：**
> Index Template 就像是 Word 的 **文档模板**——每次新建文档时，字体、字号、页边距都已经预设好了。同理，每次新索引创建时，分片策略、字段类型、分析器配置都自动应用。

Elasticsearch 8.x 推荐使用 **可组合模板（Composable Template）** 体系，由两层组成：**组件模板（Component Template）** 和 **可组合索引模板（Composable Index Template）**。

<a id="component-template"></a>

### 1. 组件模板（Component Template）

组件模板是可复用的 **配置积木块**，可以包含 settings、mappings 或 alias 配置。

```json
// 组件模板 1：通用 settings 配置
PUT /_component_template/logs_settings_template
{
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}

// 组件模板 2：日志字段 mappings 配置
PUT /_component_template/logs_mappings_template
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" },
        "message":    { "type": "text", "analyzer": "ik_max_word" },
        "service":    { "type": "keyword" }
      }
    }
  }
}
```

<a id="composable-template"></a>

### 2. 可组合索引模板（Composable Index Template）

可组合索引模板定义 **匹配规则**，将多个组件模板组装在一起。

```json
// 可组合索引模板：匹配 log-* 开头的索引名
PUT /_index_template/logs_template
{
  "index_patterns": ["log-*"],
  "composed_of": [
    "logs_settings_template",
    "logs_mappings_template"
  ],
  "priority": 200,
  "template": {
    "aliases": {
      "all_logs": {}
    }
  }
}
```

<a id="template-example"></a>

### 3. 实战示例：日志索引的标准化模板

```json
// 完整流程：创建模板 → 自动创建索引 → 验证

// 1. 创建可组合模板（匹配 orders-* 索引）
PUT /_index_template/orders_template
{
  "index_patterns": ["orders-*"],
  "composed_of": [],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "order_id":    { "type": "keyword" },
        "user_id":     { "type": "keyword" },
        "amount":      { "type": "double" },
        "status":      { "type": "keyword" },
        "created_at":  { "type": "date", "format": "yyyy-MM-dd HH:mm:ss" }
      }
    },
    "aliases": {
      "orders_current": {}
    }
  }
}

// 2. 当创建匹配模式的索引时，模板自动生效
PUT /orders-2024-09
// 无需手动指定 settings/mappings，模板自动应用！

// 3. 验证模板是否生效
GET /orders-2024-09
// 返回的结果将包含模板中定义的所有配置
```

- **注意事项**：
  - `priority` 值越高优先级越高，当多个模板匹配同一索引时，高优先级覆盖低优先级。
  - 模板在 **索引创建时** 生效，对已存在的索引无效。
  - 可通过 `GET /_index_template/orders_template` 查看模板定义，`DELETE /_index_template/orders_template` 删除模板。

---

<a id="snapshot-api"></a>

## 四、Snapshot & Restore API（快照与恢复）

Snapshot API 提供了 Elasticsearch 数据的 **备份与恢复** 能力，是生产环境数据安全保障的核心手段。

> 💡 **核心认知：**
> Elasticsearch 的快照不是"整盘复制"——它是增量的。第一次快照是全量，后续快照只备份 **新增或变更的段文件 (Segment)**，因此速度快且节省磁盘空间。

<a id="register-repo"></a>

### 1. 注册仓库

快照必须先注册一个 **仓库 (Repository)**，支持本地共享文件系统、HDFS、AWS S3、Azure Blob 等多种后端。

```json
// 1. 注册本地文件系统仓库（最常用）
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "/data/es_backup"
  }
}

// 2. 注册 S3 仓库（云环境常用）
PUT /_snapshot/s3_backup
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backup-bucket",
    "region": "us-east-1"
  }
}

// 3. 验证仓库是否注册成功
POST /_snapshot/my_backup/_verify
```

<a id="create-restore-snapshot"></a>

### 2. 创建与恢复快照

```json
// 1. 创建快照（备份 products 索引）
PUT /_snapshot/my_backup/snapshot_20240920
{
  "indices": "products,orders",
  "ignore_unavailable": true,
  "include_global_state": false
}

// 2. 查看快照进度
GET /_snapshot/my_backup/snapshot_20240920/_status

// 3. 查看仓库中所有快照
GET /_snapshot/my_backup/_all

// 4. 从快照恢复索引
POST /_snapshot/my_backup/snapshot_20240920/_restore
{
  "indices": "products",
  "include_global_state": false,
  "rename_pattern": "products",
  "rename_replacement": "products_restored"
}

// 5. 删除旧快照（释放空间）
DELETE /_snapshot/my_backup/snapshot_20240901
```

<a id="snapshot-example"></a>

### 3. 实战示例：集群数据迁移

```json
// 场景：将旧集群的数据迁移到新集群

// === 在旧集群上操作 ===

// 1. 注册共享存储仓库（两个集群共享同一块 NFS/S3 存储）
PUT /_snapshot/migration_repo
{
  "type": "fs",
  "settings": {
    "location": "/shared/es_migration"
  }
}

// 2. 创建全量快照
PUT /_snapshot/migration_repo/full_backup
{
  "indices": "*",
  "ignore_unavailable": true,
  "include_global_state": true
}

// 等待快照完成（轮询状态）
GET /_snapshot/migration_repo/full_backup/_status

// === 在新集群上操作 ===

// 3. 注册同一仓库
PUT /_snapshot/migration_repo
{
  "type": "fs",
  "settings": {
    "location": "/shared/es_migration"
  }
}

// 4. 恢复全部索引
POST /_snapshot/migration_repo/full_backup/_restore
{
  "include_global_state": true
}
```

- **注意事项**：
  - `include_global_state: true` 会备份集群元数据（索引模板、ILM 策略等），跨集群迁移时通常需要开启。
  - 快照过程是增量的，不会阻塞索引的读写操作。
  - 建议配合 ILM（Index Lifecycle Management）策略自动定期创建快照。

---

<a id="reindex-api"></a>

## 五、Reindex API（跨索引数据迁移）

Reindex API 可以将 **一个或多个索引中的文档复制到另一个索引**，非常适合 Mapping 变更、字段重组、索引合并等场景。

> 💡 **核心认知：**
> Reindex 本质上是对源索引执行一次 Scroll 查询，然后通过 Bulk API 写入目标索引。它是 **在线迁移**——源索引的读写不受影响。

<a id="reindex-basic"></a>

### 1. 基础用法

- **关键端点**：`POST /_reindex`

```json
// 将 products_v1 的数据全部迁移到 products_v2
POST /_reindex
{
  "source": {
    "index": "products_v1"
  },
  "dest": {
    "index": "products_v2"
  }
}
```

<a id="reindex-remote"></a>

### 2. 远程 Reindex

支持从另一个 Elasticsearch 集群直接拉取数据，无需中间存储。

```json
// 从远程集群拉取数据
POST /_reindex
{
  "source": {
    "remote": {
      "host": "https://old-cluster:9200",
      "username": "admin",
      "password": "password"
    },
    "index": "products"
  },
  "dest": {
    "index": "products_migrated"
  }
}
```

<a id="reindex-example"></a>

### 3. 实战示例：Mapping 变更后的数据迁移

```json
// 场景：products 索引的 name 字段从 text 改为 text + keyword 子字段，
//       同时删除废弃字段 old_field，新增 sales_count 字段

// 1. 创建新索引（新 Mapping）
PUT /products_v2
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "price": { "type": "double" },
      "sales_count": { "type": "integer" }
    }
  }
}

// 2. Reindex 迁移数据（排除 old_field，补默认值）
POST /_reindex
{
  "source": {
    "index": "products",
    "_source": ["name", "price"]
  },
  "dest": {
    "index": "products_v2"
  },
  "script": {
    "source": """
      ctx._source.sales_count = 0;
    """
  }
}

// 3. 原子切换别名（零停机）
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products",  "alias": "products_current" } },
    { "add":    { "index": "products_v2", "alias": "products_current" } }
  ]
}
```

- **注意事项**：
  - 大数据量 Reindex 时建议使用 `POST /_reindex?wait_for_completion=false` 异步执行，并通过 `GET /_tasks/<task_id>` 查看进度。
  - 可以在 `source` 中使用 `query` 参数只迁移符合条件的数据（如只迁移最近 30 天的订单）。
  - `script` 处理器可以修改每个文档的内容，但会显著降低迁移速度。

---

<a id="pit-api"></a>

## 六、Point In Time API（时间点视图）

Point In Time (PIT) API 创建索引的一个 **只读一致性快照视图**。在 PIT 存续期间，即使有新数据写入或删除，搜索结果也不会受到影响。

> 💡 **核心认知：**
> PIT 解决的核心问题是 **数据漂移 (Data Drift)**。当使用 `search_after` 进行深度翻页时，如果翻页过程中有新文档写入，可能导致：
> - 某些文档在不同页被重复返回
> - 某些文档被完全遗漏
>
> PIT 通过锁定一个"数据时间点"快照，确保整个翻页过程中数据视图完全一致。

<a id="pit-usage"></a>

### 1. 创建与使用 PIT

```json
// 1. 创建 PIT 视图（keep_alive 控制存活时间）
POST /products/_search/pit
{
  "keep_alive": "5m"
}
// 返回：{ "id": "my_pit_id_xxx" }

// 2. 在搜索中使用 PIT
GET /_search
{
  "pit": {
    "id": "my_pit_id_xxx",
    "keep_alive": "5m"
  },
  "query": {
    "match": { "name": "手机" }
  },
  "size": 100,
  "sort": [{ "price": "asc" }]
}

// 3. 使用 search_after 翻页（带上 PIT）
GET /_search
{
  "pit": {
    "id": "上一页返回的新 pit_id",
    "keep_alive": "5m"
  },
  "query": {
    "match": { "name": "手机" }
  },
  "size": 100,
  "search_after": [上一页最后一条的 price 值]
}

// 4. 翻页结束后释放 PIT（重要！）
DELETE /_pit
{
  "id": "my_pit_id_xxx"
}
```

<a id="pit-example"></a>

### 2. 实战示例：一致性深度分页

```json
// 场景：导出 orders 索引中 status="completed" 的所有订单数据
//       数据量可能超过 100 万条

// 第 1 步：创建 PIT 并执行首次查询
POST /orders/_search/pit?keep_alive=10m
// 获得 pit_id: "abc123..."

// 第 2 步：使用 PIT + search_after 执行第一页查询
GET /_search
{
  "pit": {
    "id": "abc123...",
    "keep_alive": "10m"
  },
  "query": {
    "term": { "status": "completed" }
  },
  "size": 5000,
  "sort": [
    { "created_at": "asc" },
    { "_shard_doc": "asc" }   // 分片文档排序，保证翻页确定性
  ]
}
// 记录：最后一条的 sort 值 + 响应头中的新 pit_id

// 第 3 步：循环翻页（伪代码逻辑）
// while (有数据返回):
//   GET /_search
//   {
//     "pit": { "id": "上一次返回的 pit_id", "keep_alive": "10m" },
//     "query": { "term": { "status": "completed" } },
//     "size": 5000,
//     "search_after": [上一页最后的 sort 值],
//     "sort": [{ "created_at": "asc" }, { "_shard_doc": "asc" }]
//   }

// 第 4 步：数据导出完毕后释放 PIT
DELETE /_pit
{
  "id": "最后一次的 pit_id"
}
```

- **注意事项**：
  - PIT 是有 **资源成本** 的——每个 PIT 会占用集群内存来维护搜索上下文。使用完毕必须及时释放。
  - `keep_alive` 每次翻页都会续期，但不要设置过长（建议 `5m~10m`）。
  - 排序字段建议加上 `_shard_doc` 作为 tiebreaker，避免排序值相同的文档翻页时顺序不确定。
  - **PIT + `search_after` 是 Elasticsearch 8.x 中深度分页的官方推荐方案**，已完全取代了 `scroll` API。

---

<a id="api-summary"></a>

## 七、补充 API 总览对比表

| API 类别 | 核心端点 | 功能类比 | 典型使用场景 | 关键注意事项 |
| :--- | :--- | :--- | :--- | :--- |
| **Ingest API** | `PUT /_ingest/pipeline/<name>` | 数据预处理流水线 | 日志解析、字段补全、数据清洗 | 用 `_simulate` 测试后再上线 |
| **Multi-Search API** | `POST /_msearch` | 批量 SELECT | 首页多模块并行搜索、仪表盘 | NDJSON 格式，控制单次查询数量 |
| **Index Template API** | `PUT /_index_template/<name>` | 自动建表模板 | 按日期滚动创建的日志索引 | 组件模板可复用，priority 控制优先级 |
| **Snapshot API** | `PUT /_snapshot/<repo>/<snap>` | 全库备份/恢复 | 数据安全备份、跨集群迁移 | 增量快照，需先注册仓库 |
| **Reindex API** | `POST /_reindex` | SELECT INTO + INSERT | Mapping 变更迁移、索引合并 | 大量数据用异步模式，搭配别名切换 |
| **Point In Time API** | `POST /<index>/_search/pit` | 一致性快照视图 | 深度分页防数据漂移、数据导出 | 必须手动释放 PIT，搭配 `search_after` |

> 💡 **与 elastic6.md 的五大核心 API 对比：**
> 本篇的六大 API 属于 **进阶实用型 API**——它们不是日常 CRUD 的必需品，但在生产环境的日志处理、数据迁移、深度分页、索引自动化管理等场景中是不可或缺的关键能力。结合 `elastic6.md` 的五大核心 API，覆盖了 Elasticsearch 全部高频使用场景。
