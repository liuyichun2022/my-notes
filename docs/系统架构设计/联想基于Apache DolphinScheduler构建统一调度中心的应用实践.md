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
当时，联想 TDP 团队是基于 DolphinScheduler 2.x 版本，以下是 2.x 版本的一些功能。

（1） 2.xDAG 可视化

![dag-view.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fdag-view.png)

DAG 是一个有向无环图，是 Dolphin 里面一个非常重要的概念。如上图可以看到，DAG 主要是由左侧的任务组成，通过箭头即可描述任务之间的关系。当多个任务之间有更复杂的关系时，就需要有可以描述任务间复杂依赖关系的逻辑任务。DolphinScheduler 的任务主要分为以下两种任务类型。

① 逻辑任务：表述任务之间的依赖关系，比如 Switch 任务、子工作流任务和依赖的任务；通过这些任务可以描述任务之间、工作流之间的依赖关系；

② 物理任务：在 Worker 上具体执行的任务，比如 Shell、SQL、Spark、MR、HTTP 任务等。

（2） 工作流定义

![work-flow.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fwork-flow.png)

绘制出的 DAG 保存后会生成工作流定义，可以对工作流定义进行多种操作，如，定时或手动启动，也可以查看工作流定义的各版本信息。

（3） 工作流实例

![work-flow-example.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fwork-flow-example.png)

无论是手动触发或者定时触发工作流定义，都会生成工作流实例。一个 DAG 里面有多个任务，因此，一个工作流实例会产生多个任务实例。

工作流实例也可以进行各种操作，比如，对正在运行实例进行暂停或 KILL 的操作，当工作流实例变成终态时，比如是失败的状态，也可以对工作流实例进行失败重跑的操作。

（4）任务实例

在任务实例的层面，可以查看任务的执行日志，也可以对实例设定强制成功，这是一个人工干预的功能。

（5）任务状态
![task-status.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Ftask-status.png)

任务有多种状态。当 Master 生成任务实例的时候，任务默认为提交成功的状态。当 Master 分发任务给 Worker，Worker 拿到后开始执行，会向 Master 汇报状态。这个时候 Master 会把提交成功的任务状态更改为运行中。当正在运行的工作流实例被暂停，任务就会显示为暂停状态。任务执行完成会显示最终状态，比如失败、成功或 KILL。

当某个 Worker 节点挂掉时，任务需要容错，这时需要容错的状态，由其他的 Worker 进行接管。当工作流实例被执行 KILL 操作，任务也会变成 KILL 状态。

在 DolphinScheduler 3.0 的时候增加了在提交成功与运行中之间的一个状态，分发状态。

（6）2.x 新增主要 Feature

![2.x-Feature.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2F2.x-Feature.png)

2.x 主要新增了五个特性：

- 任务结果参数传递；
- 工作流间的血缘关系，工作流之间的依赖关系可以通过依赖任务或者子流程表达；
- 增加数据同步主键 waterdrop、多分支等任务组件能力支持；
- 工作流定义和任务关系拆分，更易通过 openAPI 生成工作流；
- 将工作流定义和任务关系做了拆分，添加了工作流版本控制。


### 3. DolphinScheduler 架构设计
   以下是 DolphinScheduler 整个架构的演变过程，以及它是如何支撑 10 万级任务调度的。

（1）DolphinSchedule 1.2 架构

![DolphinSchedule-v1.2.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2FDolphinSchedule-v1.2.png)

Apache DolphinScheduler 从一开始就采用了无中心化的设计，这样可以保证调度集群性能的线性增长。

1.2 架构是 Apache DolphinScheduler 最初的架构，其中有两个重要的组件，即 MasterServer 和 WorkerServer。MasterServer 负责 DAG 任务的切分，将切分后的 DAG 任务生成工作流实例进行解析并提交。1.2 架构通过 zk 队列存储任务，Master 把任务存储到 zk 队列上，然后 Worker 再从 zk 队列去取，这种方式消耗比较大，同时，Worker 与 DB 产生交互后会更改 Task 任务的状态。因此，1.3 的架构对这两点进行了改进。

（2）DolphinScheduler 1.3.x 架构

![DolphinScheduler-v1.3.x.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2FDolphinScheduler-v1.3.x.png)

1.3.x 架构主要是对以上所说的两点问题进行了改善。首先是在 Master 与 Worker 之间引入了 Netty 通信，去除 zk 队列，大大减少了 Worker 获取任务运行的延迟。其次是在 Master 与 Worker 之间进行多种任务分发策略，Worker 只负责执行任务，职责变得更单一。Worker 把任务所执行的结果汇报给 Master，由 Master 统一对状态进行管理。

（3）ApacheDolphinScheduler 2.x 架构

![ApacheDolphinScheduler-v2.x.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2FApacheDolphinScheduler-v2.x.png)

在 2.x 架构里面主要是对 Master 进行了一些重构，并引入了 SPI 插件化的设计，Master 可以与 API 直接进行通信。

① 2.x 新增可扩展能力

![2.x-add.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2F2.x-add.png)

2.x 新增的可扩展能力主要是基于 SPI 插件化的设计。它不仅对任务进行了插件化，也对告警插件进行了 SPI 设计，包括注册中心和数据源，后期还会对资源的中心进行插件化的设计。采用 SPI 可以使代码变得更加的简洁，可增加调度系统的可扩展的能力，也便于二次任务的改造与开发，在多个的任务插件里面，各个任务插件的 pom 依赖互不影响。

② 2.0 重构
![2.0-rebuild.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2F2.0-rebuild.png)

- 去分布式锁
2.0 重构大大提升了 2.0 的并发能力。虽然在 1.x 架构里面是多个 Master 同时运行，但与 DB 进行交互的时候，会从数据库里取得 command，但需要采用分布式锁。2.0 去除了这个分布式锁，当 Master 动态上下线的时候，会根据自己的分片编号计算属于自己的槽位，然后根据槽位查询数据库取到属于自己的 command。

![2.0-remove-fenbushisuo.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2F2.0-remove-fenbushisuo.png)

- Master 中线程池
在 1.x 的版本里面，线程池的使用数量相对较多一点，在 2.0 重构主要是对 Master 中线程池进行了一些优化，引入了事件驱动的机制：

第一，MasterSchedulerService 线程去掉了分布式锁，从 DB 中取到 command 生成工作流实例进行拆分、提交任务。

第二，在 WorkflowExecuteThread 线程里会维护一个事件的队列，然后去不断地处理变化，当有 Worker 发来的任务状态变化或 Master 发来流程实例状态的变化，WorkflowExecuteThread 线程就会进行相应处理。当 API 的界面上触发了如 KILL 或暂停操作这样的事件，会通过事件的机制，由 Master 统一去进行处理。


### 4. 社区发展

![dolphin-dicrud.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fdolphin-dicrud.png)

如图是 DolphinScheduler 的社区发展指标。可以看到 DolphinScheduler 从进入孵化器，到从孵化器毕业，再到现在，社区发展的各项指标都是呈指数级的增长，是一个具备很强生命力的 Community。其中，社区的用户数、开发者的数量、贡献者的数量，以及 Committer PMC 都有几何倍的增长，并且还在持续不断地上升。这些都说明了 DolphinScheduler 是一个健康的可持续发展的社区。

基于以上原因，最终联想选择了 Apache DolphinScheduler，目前已接入了多个业务 Domain 到生产环境。


## 03/在联想的落地实践

在联想内部，各个业务里面有不同的需求，比如，不同的任务类型，以及任务间需要依赖、参数传递等。所以，在接入 DolphinScheduler 后进行一些功能的改造。

### 1. HTTP 任务参数传递
![http-task.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fhttp-task.png)

在一些非大数据的场景里面更多是 HTTP 任务，并且任务的请求参数之间存在依赖的关系，所以就提出了 HTTP 参数传递的功能。当时在社区 2.x 的版本里面只有 SQL 任务和 Shell 任务是支持参数传递，而 HTTP 任务不支持。

因此，对调度进行了一些改造，在 HTTP 的任务节点里面的自定义的参数增加了 OUT 参数，也就是使任务的执行结果可以作为变量输出，并增加字段的解析。这样，就可以在 API 接口返回的结果取某个字段定义为输出变量，下一个 HTTP 任务就可以把其作为请求参数使用。

### 2. Java 任务插件开发

![java-method.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fjava-method.png)
第二个改造是 Java 任务插件的开发。在之前的一些业务部门有一些 XXL-Job 的任务，需要对这些任务进行迁移。这些作业主要是通过 Spring Bean 里面的方法实现定时。对于 DolphinScheduler 而言，如果进行改造需要增加执行器的概念，由执行器和 Worker 进行通信，增加执行器与 Worker 之间的任务状态的汇报，最后再由 Worker 把最终的状态汇报给 Master。

这个实现的原理是通过提供 SDK 开发执行器，提供一个注解给业务人员添加到自己编写的调度任务里，当程序启动的时候会扫描注解下面所有的方法，并在运行过程中将所执行的状态汇报给 Worker，由 Worker 汇报给 Master 进行统一管理。因此需要去抽象出一些必要的任务参数，并将其作为 Java 任务插件，从而实现在 DAG 上支持 Java 任务。

![java-pluign.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fjava-pluign.png)

如图，是 Java 任务插件的实现，在 DAG 上去拖拽 Java Method 任务时填上参数即可实现 Worker 对执行器的调用。


### 3. 项目全局参数

在 DolphinScheduler 里有很多参数的概念，方便对任务进行各种操作，比如，任务的自定义的参数、流程的全局的参数、启动的参数、还有一些内置的参数。在操作 HTTP 任务时，在 Header 里有 token 字段，并在多个工作流定义里面都使用同样的字段，因此，就有了项目全局参数这样的需求。

![gloable-param.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fgloable-param.png)

当一个普通任务运行时，任务内的参数根据参数替换优先级规则依次覆盖，参数优先级从高到低为：

任务自定义参数 > 流程定义全局参数 > 项目全局参数

![gloable-param2.png](..%2Fdatalake%2Fimg%2Flenovo-dolphin%2Fgloable-param2.png)







