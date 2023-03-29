# 论文简析

#CloudComputing #MicroService #Serverless

## Synapse: A Microservices Architecture for Heterogeneous-Database Web Applications

#MicroService 

当常见网络应用微服务若采取不同架构的数据库实现时，需要一种机制在不同微服务之间共享数据，如今一般使用pub/sub模式进行消息传递与共享。

[Architecture](photos/EuroSys_Synapse_Fig6.png)

本项目在ORM和DB之间加入中间层，捕获Publisher对数据库的操作，判断操作数据是否有必要与远端共享，而后将操作封装为json传递给可靠的中间件，交付Subscriber后对消息解包输入数据库

Synapse Publisher与Subscriber都是SQL源语言相关的，使用interceptor拦截DB请求

数据可能更新，需要更新交付与版本控制

可借鉴：`pub/sub`数据共享模型

框架需明确API与架构

