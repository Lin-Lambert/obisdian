# 简单的业务流程图示例

这个文件包含了一个使用 PlantUML 和 BPMN 的简单业务审批流程图。

```plantuml
@startuml
left to right direction

mxgraph.bpmn.event.start "开始" as start
rectangle "提交申请" as submit
mxgraph.bpmn.user_task "部门经理审批" as review
mxgraph.bpmn.gateway2.exclusive "是否通过?" as gw
rectangle "系统处理生成工单" as process
mxgraph.bpmn.user_task "驳回修改" as revise
mxgraph.bpmn.event.end "流程结束" as end_ok

start --> submit
submit --> review
review --> gw
gw --> process : "是"
gw --> revise : "否"
revise --> review : "修改后重新提交"
process --> end_ok
@enduml
```
