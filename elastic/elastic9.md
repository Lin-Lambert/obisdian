# Elasticsearch 写入流程 (Index/Write) 时序图

结合 `uml` Skill 与 PlantUML 语法规范，我们在此展现一个典型的 Elasticsearch 文档写入（Index）的交互时序图。本次更新特别针对 UML 规范加入了 `skinparam` 全局样式以及个体 `#color` 的配色要求，让角色和流程展示更直观。

```plantuml
@startuml
autonumber

' 全局样式配色 (遵循 uml skill 规范中的 skinparam 说明)
skinparam sequence {
    ArrowColor DeepSkyBlue
    LifeLineBorderColor DeepSkyBlue
    LifeLineBackgroundColor #E0FFFF
    
    ParticipantBorderColor DarkBlue
    ParticipantFontColor White
    
    NoteBackgroundColor #FEFEE2
    NoteBorderColor #E5D539
}

' 个体元素的专用配色 (遵循 uml skill 规范中的 #color 说明)
actor "客户端" as client #LightYellow
participant "协调节点\n(Coordinating Node)" as coordNode #SeaGreen
participant "主分片节点\n(Primary Node)" as primaryNode #SteelBlue
participant "副本分片节点 A\n(Replica Node A)" as replicaNodeA #LightPink
participant "副本分片节点 B\n(Replica Node B)" as replicaNodeB #LightPink

client -> coordNode: "发送写请求 (PUT /index/_doc/1)"
note right of coordNode: "通过哈希算法计算文档归属的主分片"

coordNode -> primaryNode: "转发写入请求至主分片所在节点"
note right of primaryNode: "执行写入、验证并建立底层索引"

par "并行同步至副本"
    primaryNode -> replicaNodeA: "转发请求到副本分片 A"
    primaryNode -> replicaNodeB: "转发请求到副本分片 B"
end

replicaNodeA --> primaryNode: "返回副本 A 写入成功"
replicaNodeB --> primaryNode: "返回副本 B 写入成功"

primaryNode --> coordNode: "主分片确认所有副本写入完毕，回复成功"

coordNode --> client: "向客户端返回操作成功的响应"

@enduml
```

**流程要点解析**：
1. 写请求首先经过**协调节点**进行哈希路由（基于 Document ID），定位到对应主分片。
2. **主分片**本地写入成功后，会**并行(par)**将写操作同步至所有副本分片。
3. 必须等到所有配置要求的副本分片返回成功，主节点才会确认，随后系统最终向客户端反馈写入成功，以此保证分布式架构下的数据一致性和高可用性。
