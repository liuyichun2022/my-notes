## 概述
随着业务不断增长以及定时任务类型的多样化，联想内部需要一个统一的调度中心对任务生命周期进行管理。Apache DolphinScheduler 是一个分布式、易扩展的可视化 DAG 工作流任务调度平台，致力于解决数据处理流程中错综复杂的依赖关系，使调度系统在数据处理流程中开箱即用。本次分享的主题是联想基于 Apache DolphinScheduler 构建统一调度中心的应用实践。

主要从以下4个方面展开
1. 背景需求

2. 为什么选择 Apache DolphinScheduler

3. Apache DolphinSheduler 在联想的落地实践

4. Apache DolphinSheduler 3.x 新特性 & Roadmap

## 01/背景需求
在日常工作场景中，不管是后端开发还是数据分析，或是运维的工作中都会用到定时任务，比如，定时清理集群上某一个冗余的日志或定时执行数据备份，又或是发日报周报。在一开始任务量小的时候可以通过 crontab 或在 Spring 里面开发定时任务，但当有几百上千个定时任务时，就涉及到很多问题，比如，如何统一管理多个任务的生命周期以及任务间的相互依赖等，这时就需要一个统一的调度中心。

![lenovo-dolphin-background.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Flenovo-dolphin-background.png)

对于统一的调度中心，联想内部当时收集了各个业务 Domain 的需求，主要分为三大类：

（1）支持任务的丰富性，除了日常需要的定时通知任务等功能，如日报周报、邮件通知等，也需要支持大数据领域的 ETL 任务和非大数据场景下的 HTTP 任务，具体包括数据集成、数据处理、调用 HTTP API 接口等。同时接口之间的请求参数也有依赖关系，这些都需要调度中心能够支持。

（2）任务管理和编排，当任务越来越多，任务依赖关系变得越来越复杂时，就需要进行统一管理和编排，并对他们的执行情况做监控，看任务是否按照逻辑顺序正确执行。此外一些在第三方业务系统里通过 Spring 开发的定时任务，也需要纳入统一管理。

（3）关注 SLA，当任务量越来越大的时候要保证调度按时可靠地执行，并且任务的上下游依赖要能够更快更实时的按序触发。

![requirement.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Frequirement.png)

通过对以上需求进行抽象分析，可以得出统一调度中心需要满足以下六个特性：

➊ 高可靠性。稳定性是一个调度中心的重中之重，因此，调度系统要高可用，调度集群要支持分布式。

➋ 丰富易用。根据上面的需求，调度中心需要能够支持多种任务，并且要对用户足够的友好，操作简单。一方面可以基于 API 去调用，另一方面可以通过拖拉拽的方式方便数据分析人员使用。

➌ 轻量化。用于调度流程、启动任务等的开销要非常小。

➍ 业务隔离。当多个业务域接入到调度中心，需要对业务进行隔离，同时，任务执行时互不干扰。

➎ 调度性能线性扩展。当业务域加入到调度中心以后，任务的量级一定会指数级地增长。当任务量越来越大的时候，要能够支持快速地线性扩展从而增加集群的能力。

➏ 业务易扩展。当有定制化的需求时，要能够快速响应，这对调度中心的扩展性有很高的要求。

## 02/为什么选择 DolphinScheduler
根据联想集团当时调研的需求以及背后的抽象，联想 TDP 团队对三个调度系统进行了调研，分别是 xxl-job、Apache DolphinScheduler、Apache Airflow。

### 1. 调度系统对比
![schedule-sys-compare.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fschedule-sys-compare.png)

如图，是三个调度系统的对比。

从定位上看，XXL-Job 是一个轻量级分布式的任务调度框架，在业务系统里面需要去写调度的逻辑，同时也能在 XXL-Job 上进行管理；Apache DolphinScheduler 是一个云原生的分布式易扩展的可视化的工作流调度平台，致力于解决任务编排以及错综复杂的依赖关系；Airflow 跟 Dolphin 都是基于 DAG 的调度，但更偏重以编程的方式去编写 DAG，然后对 DAG 进行调度和监控。

从对任务类型的支持上看，XXL-Job 支持 Java、Shell、Python 这三种任务类型，支持的任务类型比较有限；Dolphin 和 Airflow 支持的任务类型相对较多，Dolphin 可以支持近 20 种业务类型，覆盖大多数的业务场景；Airflow 也可以支持自定义地扩展。

对于可视化的能力，XXL-Job 可配置任务级联触发，但不支持 DAG；而 Dolphin-Scheduler 支持强大的 DAG 可视化拖拽，可以对其任务和依赖关系进行拖拽，并且 DolphinScheduler 在 2.x 也支持通过 Python 代码去构建 DAG；Airflow 更多是通过 Python 代码去构建 DAG。

对于调度中心扩展性的要求，当需要开发一些新的任务的时候，XXL-Job 需要通过 Java去开发执行器；DolphinScheduler 通过 SPI 实现自定义任务，更加容易扩展整个 DolphinScheduler 任务的支持能力；Airflow 也支持自定义任务类型，通过 Operator 来支持。在大数据这一块对多租户支持需求比较多，因为需要用到资源的调度，需要用到 Yarn 里面的队列。在 DolphinScheduler 里面就会通过租户对应到 Yarn 里面的队列，这样就可以尽量地保证生产集群上资源的合理分配和较高的资源利用率。

对于集群扩展的支持，如何能够线性地去增长，XXL-Job 和 Airflow 更多是在执行器上进行一个水平的扩展。而 DolphinScheduler 的架构设计是 Master 和 Worker，都是无中心化设计，所以都支持动态的伸缩。

从易用性上可以看到，DolphinScheduler 强调开箱即用，具备很高的易用性。

从社区活跃度来讲，DolphinScheduler 和 Airflow 都是 Apache 顶级项目。

从二次开发成本考虑，联想 TDP 团队更多的是采用 Java 语言。

综上，我们最终选择了 Apache DolphinScheduler 作为统一的调度中心。


### 2. DolphinScheduler 功能一览


