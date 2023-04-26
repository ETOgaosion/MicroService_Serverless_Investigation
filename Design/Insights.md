# Insights

根据调研，一个云原生框架需要考虑几个方面的问题：

1. 计算模式的支持
2. 数据传输模式的支持
3. 工作流支持
4. 运行辅助的支持

## 计算模式的支持

这部分主要是考虑框架对分布式运算模式的支持性，模式通常有以下几种：

- MapReduce：适用于处理大规模数据集的计算任务，如网页排名、搜索引擎索引等。
- BSP: Bulk Synchronous Parallel，分阶段的计算方式，将计算分为多伦迭代，每轮迭代包含三个步骤：计算、通信和同步。BSP模型适用于迭代计算任务，如Page Rank算法
- Actor: 基于消息传递的并行计算模型，将计算过程中的组件抽象为actor，每个actor可以接收消息并发送消息给其他actor。Actor模型适用于处理复杂的交互式应用，如实时推荐系统、游戏引擎等
- 数据流模型：基于数据流的计算模型，将计算过程看作是数据流在计算节点之间流动的过程。数据流模型适用于流式计算任务，如实时监控、实时数据分析等
- DAG：基于有向无环图的计算模型，将计算过程表示为一张有向无环图。每个节点代表一个计算任务，节点的边代表数据依赖关系。DAG模型适用于处理复杂的计算任务，如机器学习、图像处理等

## 数据传输模式的支持

- Database, Relational/KV
- topic queue
- pub/sub/fanout
- Producer/Consumer
- Push/Pull
- Data Stream
- Event Sourcing/CQRS

## 工作流支持

- Serverless Workflow
- dapr Workflow
- argo Workflow
- Apache Airflow
- KubeFlow Pipelines

## 运行辅助的支持

类似dapr，SpringBoot, k8s等等

## 重点关注的框架

Apache Flink
Apache Spark
Beam
Ray

## 语言支持

Java
Scala
Python
Go