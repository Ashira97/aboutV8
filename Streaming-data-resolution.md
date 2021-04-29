# 利用 AWS 与 Amazon Kinesis 流数据解决方案

## 摘要

数据工程师、数据分析师和大数据开发者正在寻求将分析对象从批处理数据转向实时数据的方案，因为实时数据能够帮助他们了解客户、应用和产品现在的行为并且及时做出反应。

该白皮书主要讲述从批处理数据分析转向实时数据分析的过程，它描述了 Amazon Kinesis Streams/Firehost/Amalytics 是如何用于实现实时数据分析并为这些服务提供通用设计模式的。

## 引言

得益于能够产生流数据的数据源的爆炸性增长，今天的商业分析能够以前所未有的速度收到大量数据。

无论是来自于应用服务器的日志数据、来自于网页和移动应用的点击流数据或者是来自于物联网设备的遥感数据，这些数据都能够帮助你了解你的应用、客户和产品当前时刻的行为。

能够实时地处理和分析流数据能够帮助我们持续监控应用的行为，从而保证服务正常运行、个人定制服务或者是产品定制化推荐。

实时流数据在几分钟或几秒而不是几小时或几天内传递数据的特性也能够帮助诸如网页数据分析或机器学习应得到更真实和可操作的数据。

### 实时数据处理场景

实时流数据应用有两种使用场景：

> 从批数据处理迁移到流数据分析：我们可以将数据仓库或 Hadoop 框架中运行的传统批数据处理过程潜移到实时数据分析过程中。
> 该场景下最常见的使用案例有数据湖、数据科学或者机器学习案例。你能够连续使用流数据解决方案从数据湖中加载实时数据
> 你也可以在有新数据到来的时候更新机器学习模型，确保输出数据的准确性和可靠性。
> 例如，Zillow 使用 Amazon Kineses Streams 收集记录数据和 MLS 列表，然后为房屋卖家和买家提供近乎实时的房产价格估计数据。
> Zillow 同样使用 Kinesis Streams 将桐言的数据传到 Amazon Simple Storage Service 数据湖中，来保证所有的应用程序都基于同样的实时数据工作。

> 构建实时数据处理应用：我们可以将实时流数据服务应用于应用监控、欺诈探测或者实时排行榜。
> 这些案例从数据注入到数据处理等向目标系统和数据存储服务传递数据的操作都要求毫秒级的端到端延迟。
> 例如，Netflix 使用 Kinesie Streams 来监控所有应用之间的通信数据从而能够尽快探测到并修复问题，确保了服务可用性以及用户可达性。
> 虽然该技术最常用的使用案例是应用性能监控，现在广告技术、游戏技术和物联网领域也出现了越来越多的实时应用。

### 批数据处理和流数据处理的不同

不同于批数据分析任务，你需要一套不同的工具来收集、准备和处理实时流数据。在传统的批数据分析中，我们把数据定时地加载到数据库，以小时、天和周为维度来分析这些数据。
分析实时数据则要求不同的方法。流数据处理应用甚至会在数据存储之前实时地连续处理数据而不是对已存储的数据执行数据库查询语句。
流数据快速进入应用，可能在不同时间流入数据量相差巨大，流数据处理平台能够处理这种传入数据在速度上的差异并且在数据到达时即进行处理，因此通常每小时执行几百万甚至几千万个事件。

### 流数据处理面临的挑战

在数据到达时处理实时流数据相比传统数据分析技术能够帮助你更快速的做出决定。然而，构建和管理自己的流数据管道是一件复杂并且耗费资源的事情。
你需要搭建一套能够在代价尽量小的情况下收集、准备和传递数据的系统。你需要调整存储桶和计算资源的设置，来保证数据在低延时和最大吞吐量的情况下能够高效的打包和传递。
你需要部署和管理一系列服务器来衡量这一系统来保证你能够处理到来的不同速度的数据。
在你构建这一平台之后，你应该监控它并从通过在流中不同结点抓取合适的数据来保证能够任何服务器或者网络错误中恢复数据。
以上的所有事情都是耗时耗力的事情，所以大部分的公司必须为系统设置定额并且以小时和天的维度分析商业数据。

## 案例：从批数据到实时数据

为了更好的理解大型组织时如何使用 AWS 将批数据处理转到流数据处理的，我们看一个案例。
在该案例中，我们在一个场景下详细讨论如何使用 AWS 的服务来解决问题。
