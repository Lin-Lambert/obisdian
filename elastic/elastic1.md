# Elasticsearch 查询索引流程 (Query Then Fetch)

Elasticsearch 最常见、也是默认的分布式搜索执行逻辑被称为 **Query Then Fetch（先查询，后获取）**。这个过程分为两个主要阶段，以确保在分布式架构下能够高效、准确地拿到 Top-N 的结果。

## 流程图 (PlantUML)

```plantuml
@startuml
skinparam participant {
  BackgroundColor #dae8fc
  BorderColor #6c8ebf
  FontName "Microsoft YaHei"
  FontSize 13
}
skinparam sequence {
  ArrowColor #333333
  LifeLineBorderColor #999999
  GroupBackgroundColor #f5f5f5
  GroupBorderColor #cccccc
  DividerBackgroundColor #eeeeee
}
skinparam database {
  BackgroundColor #fff2cc
  BorderColor #d6b656
  FontName "Microsoft YaHei"
  FontSize 13
}
skinparam actor {
  FontName "Microsoft YaHei"
  FontSize 13
}
skinparam note {
  BackgroundColor #FFF9E6
  BorderColor #D6B656
  FontName "Microsoft YaHei"
  FontSize 12
}

title Elasticsearch 查询索引流程 (Query Then Fetch)

actor "客户端\n(Client)" as Client

box "Elasticsearch Cluster" #F5F5F5
  participant "协调节点\n(Coordinating Node)" as CoordNode #A8D08D
  database "数据分片 1...N\n(Data Shards)" as Shards #F4B183
end box

== 第一阶段：Query Phase (查询阶段) ==
autonumber 1
Client -> CoordNode : 发起 Search 请求
activate CoordNode
note right of Client: 查询条件被解析，确定目标索引

CoordNode -> Shards : 路由广播请求至所有相关分片
activate Shards
note right of CoordNode: 主分片或副本分片均可，轮询负载

Shards -> Shards : 本地执行搜索，构建优先队列
note right of Shards: 仅提取匹配文档的 ID 和 Score

Shards --> CoordNode : 返回本地 Top-N (Doc ID, Score)
deactivate Shards

CoordNode -> CoordNode : 执行全局归并排序
note right of CoordNode: 计算出全局真正的 Top-N 列表

== 第二阶段：Fetch Phase (获取阶段) ==
autonumber 5
CoordNode -> Shards : 带着 Top-N 的 ID 列表发起 Multi-Get 请求
activate Shards

Shards -> Shards : 读取对应 ID 的完整文档内容

Shards --> CoordNode : 返回包含 _source 的完整数据
deactivate Shards

CoordNode -> CoordNode : 拼接与装配最终结果

CoordNode --> Client : 返回最终 JSON 响应
deactivate CoordNode
@enduml
```

---

## 流程详细说明

### 第一阶段：Query Phase（查询阶段）
在这个阶段，系统主要是为了找出 **"哪些文档（Doc ID）符合条件"** 以及 **"它们的分数（Score）排在最前面"**。

1. **接收请求**：客户端（Client）发起一个 Search 请求，集群中接收到该请求的节点充当本次搜索的 **协调节点（Coordinating Node）**。
2. **路由广播**：协调节点解析查询条件，确定需要查询哪些索引。然后将请求转发到对应索引的每个分片（Shard）上。这里它会在主分片（Primary）和副本分片（Replica）中运用轮询策略随机选择一个以实现负载均衡。
3. **本地搜索**：每个收到请求的分片独立在本地执行查询。分片会在内部生成一个大小为 `from + size` 的优先队列（Priority Queue），里面仅包含匹配到的 **Doc ID（文档ID）** 和 **Score（相关性算分）**，**不包含具体的文档内容**（这样极大减少了网络带宽消耗）。
4. **返回与归并**：每个分片将自己的 Top-N `(Doc ID, Score)` 列表返回给协调节点。协调节点将收集到的所有结果集再次进行全局的归并排序运算，最终选出真正的全局 Top-N 文档列表。

### 第二阶段：Fetch Phase（获取阶段）
在明确知道了最终需要的文档 ID 以及它们所在的原始分片后，开始拉取真实的数据内容。

5. **定向拉取**：协调节点拿着全局排序后确认的最终的 Top-N 的 Doc ID 列表，向它们对应的各个数据分片发送 `Multi-Get` 请求获取详细数据。
6. **返回数据**：各个数据分片根据 ID 读取文档的完整内容（如 `_source`、高亮字段等），并将其返回给协调节点。
7. **客户端响应**：协调节点将这些完整的文档数据拼接、装配成最终的 JSON 响应格式，返回给客户端。

> **💡 为什么需要分两步？**
> 对于分布式系统来说，比如你想查排名前 10 的数据。如果直接查完整文档，每个分片都返回 10 条庞大的 JSON 给协调节点，如果有 10 个分片，网络里就会传输 100 条庞大的文档数据，而最终只保留 10 条，造成巨大浪费（**深度分页灾难**）。
> 采用 `Query Then Fetch` 能够让第一遍在网络流转的仅仅是体积极小的 ID 与分数，等最终裁死前 10 名局势后，再精准拉取实际内容，是最优解。