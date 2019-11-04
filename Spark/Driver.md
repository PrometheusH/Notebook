# Driver

`Spark Driver Program`是运行Application的main函数并且新建SparkContext实例的程序。初始化SparkContext是为了准备Spark引用程序的运行环境，SparkContext负责与集群进行通信、资源的申请、任务的分匹配和监控等。当worker节点中的Executor运行完毕Task后，Driver同时负责将SparkContext关闭，通常使用sc来代表Driver。

#### SparkContext初始化

<img src=".\SparkContextDo.png" alt="SparkContextDo" style="zoom:50%;" />

1. 会创建SparkConf实例，来覆盖Spark默认配置文件中的参数
2. 会初始化DAGScheduler，它是高层调度器，将整个job划分成Stage，TaskScheduler，它是低层调度器，调度每个Stage的task进行处理，是一个接口，根据ClusterManager的不同，有不同的实现。Standalone下的是TaskSchedulerImpl。
3. SchedulerBackend，它是通信终端，管理整个集群中为当前这个Application分配的资源，是一个接口，根据ClusterManager的不同会有不同的实现。Standalone模式下是StandaloneSchedulerBackend。
4. MapOutputTrackerMaster？？？

高层：与具体的ClusterManager无关，低层：yarn和standalone模式不一样

#### StandaloneSchedulerBackend三大核心功能：

1. 负责与Master连接，注册当前的程序（RegisterWithMaster）
2. 接收集群中为当前应用程序分贝的计算资源——Executor的注册并管理Executor

3. 负责发送Task到具体的Executor执行





#### SparkContext初始化

调用createTaskScheduler（this，master，deployMode）

根据master的不同创建不同的TaskShceduler和SchedulerBackend

如果是Local就创建TaskShcedulerImpl和LocalShedulerBackend，





















