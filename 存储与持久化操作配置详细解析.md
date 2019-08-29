> 本文出处：https://blog.csdn.net/qwe6112071/article/details/50989192

# 1、内存存储RAMJobStore

​		Quartz默认使用RAMJobStore，它的优点是速度。因为所有的 Scheduler 信息都保存在计算机内存中，访问这些数据随着电脑而变快。而无须访问数据库或IO等操作，但它的缺点是将 Job 和 Trigger 信息存储在内存中的。因而我们每次重启程序，Scheduler 的状态，包括 Job 和 Trigger 信息都丢失了。 
​		Quartz 的内存 Job 存储的能力是由一个叫做 org.quartz.simple.RAMJobStore 类提供。在我们的quartz-2.x.x.jar包下的org.quartz包下即存储了我们的默认配置quartz.properties。打开这个配置文件，我们会看到如下信息：

```
# Default Properties file for use by StdSchedulerFactory
# to create a Quartz Scheduler Instance, if a different
# properties file is not explicitly specified.
#

org.quartz.scheduler.instanceName: DefaultQuartzScheduler
org.quartz.scheduler.rmi.export: false
org.quartz.scheduler.rmi.proxy: false
org.quartz.scheduler.wrapJobExecutionInUserTransaction: false

org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 10
org.quartz.threadPool.threadPriority: 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true

org.quartz.jobStore.misfireThreshold: 60000

org.quartz.jobStore.class: org.quartz.simpl.RAMJobStore
```

# 2、持久性JobStore

Quartz 提供了两种类型的持久性 JobStore，为JobStoreTX和JobStoreCMT，其中： 
1. JobStoreTX为独立环境中的持久性存储，它设计为用于独立环境中。这里的 “独立”，我们是指这样一个环境，在其中不存在与应用容器的事物集成。这里并不意味着你不能在一个容器中使用 JobStoreTX，只不过，它不是设计来让它的事特受容器管理。区别就在于 Quartz 的事物是否要参与到容器的事物中去。 
2. JobStoreCMT 为程序容器中的持久性存储，它设计为当你想要程序容器来为你的 JobStore 管理事物时，并且那些事物要参与到容器管理的事物边界时使用。它的名字明显是来源于容器管理的事物(Container Managed Transactions (CMT))。

# 3、持久化配置步骤

​		要将JobDetail等信息持久化我们的数据库中，我们可按一下步骤操作：

## 	1. 配置数据库

​		其中各表的含义如下所示：

| **表名**                 | **描述**                                                     |
| ------------------------ | ------------------------------------------------------------ |
| QRTZ_CALENDARS           | 以 Blob 类型存储 Quartz 的 Calendar 信息                     |
| QRTZ_CRON_TRIGGERS       | 存储 Cron Trigger，包括 Cron 表达式和时区信息                |
| QRTZ_FIRED_TRIGGERS      | 存储与已触发的 Trigger 相关的状态信息，以及相联 Job 的执行信息 |
| QRTZ_PAUSED_TRIGGER_GRPS | 存储已暂停的 Trigger 组的信息                                |
| QRTZ_SCHEDULER_STATE     | 存储少量的有关 Scheduler 的状态信息，和别的 Scheduler 实例(假如是用于一个集群中) |
| QRTZ_LOCKS               | 存储程序的非观锁的信息(假如使用了悲观锁)                     |
| QRTZ_JOB_DETAILS         | 存储每一个已配置的 Job 的详细信息                            |
| QRTZ_JOB_LISTENERS       | 存储有关已配置的 JobListener 的信息                          |
| QRTZ_SIMPLE_TRIGGERS     | 存储简单的 Trigger，包括重复次数，间隔，以及已触的次数       |
| QRTZ_BLOG_TRIGGERS       | 作为 Blob 类型存储(用于 Quartz 用户用 JDBC 创建他们自己定制的 Trigger 类型，JobStore 并不知道如何存储实例的时候) |
| QRTZ_TRIGGER_LISTENERS   | 存储已配置的 TriggerListener 的信息                          |
| QRTZ_TRIGGERS            | 存储已配置的 Trigger 的信息                                  |

## 2. 使用JobStoreTX

1. 首先，我们需要在我们的属性文件中表明使用JobStoreTX： 
   org.quartz.jobStore.class = org.quartz.ompl.jdbcjobstore.JobStoreTX
2. 然后我们需要配置能理解不同数据库系统中某一特定方言的驱动代理：

| **数据库平台**             | **Quartz 代理类**                                            |
| -------------------------- | ------------------------------------------------------------ |
| Cloudscape/Derby           | org.quartz.impl.jdbcjobstore.CloudscapeDelegate              |
| DB2 (version 6.x)          | org.quartz.impl.jdbcjobstore.DB2v6Delegate                   |
| DB2 (version 7.x)          | org.quartz.impl.jdbcjobstore.DB2v7Delegate                   |
| DB2 (version 8.x)          | org.quartz.impl.jdbcjobstore.DB2v8Delegate                   |
| HSQLDB                     | org.quartz.impl.jdbcjobstore.PostgreSQLDelegate              |
| MS SQL Server              | org.quartz.impl.jdbcjobstore.MSSQLDelegate                   |
| Pointbase                  | org.quartz.impl.jdbcjobstore.PointbaseDelegate               |
| PostgreSQL                 | org.quartz.impl.jdbcjobstore.PostgreSQLDelegate              |
| (WebLogic JDBC Driver)     | org.quartz.impl.jdbcjobstore.WebLogicDelegate                |
| (WebLogic 8.1 with Oracle) | org.quartz.impl.jdbcjobstore.oracle.weblogic.WebLogicOracleDelegate |
| Oracle                     | org.quartz.impl.jdbcjobstore.oracle.OracleDelegate           |

**如果我们的数据库平台没在上面列出，那么最好的选择就是，直接使用标准的 JDBC 代理 org.quartz.impl.jdbcjobstore.StdDriverDelegate 就能正常的工作。**

1. 以下是一些相关常用的配置属性及其说明：

| **属性**                                         |                      **默认值**                       | **描述**                                                     |
| ------------------------------------------------ | :---------------------------------------------------: | ------------------------------------------------------------ |
| org.quartz.jobStore.dataSource                   |                          无                           | 用于 quartz.properties 中数据源的名称                        |
| org.quartz.jobStore.tablePrefix                  |                         QRTZ_                         | 指定用于 Scheduler 的一套数据库表名的前缀。假如有不同的前缀，Scheduler 就能在同一数据库中使用不同的表。 |
| org.quartz.jobStore.userProperties               |                         False                         | “use properties” 标记指示着持久性 JobStore 所有在 JobDataMap 中的值都是字符串，因此能以 名-值 对的形式存储，而不用让更复杂的对象以序列化的形式存入 BLOB 列中。这样会更方便，因为让你避免了发生于序列化你的非字符串的类到 BLOB 时的有关类版本的问题。 |
| org.quartz.jobStore.misfireThreshold             |                         60000                         | 在 Trigger 被认为是错过触发之前，Scheduler 还容许 Trigger 通过它的下次触发时间的毫秒数。默认值(假如你未在配置中存在这一属性条目) 是 60000(60 秒)。这个不仅限于 JDBC-JobStore；它也可作为 RAMJobStore 的参数 |
| org.quartz.jobStore.isClustered                  |                         False                         | 设置为 true 打开集群特性。如果你有多个 Quartz 实例在用同一套数据库时，这个属性就必须设置为 true。 |
| org.quartz.jobStore.clusterCheckinInterval       |                         15000                         | 设置一个频度(毫秒)，用于实例报告给集群中的其他实例。这会影响到侦测失败实例的敏捷度。它只用于设置了 isClustered 为 true 的时候。 |
| org.quartz.jobStore.maxMisfiresToHandleAtATime   |                          20                           | 这是 JobStore 能处理的错过触发的 Trigger 的最大数量。处理太多(超过两打) 很快会导致数据库表被锁定够长的时间，这样就妨碍了触发别的(还未错过触发) trigger 执行的性能。 |
| org.quartz.jobStore.dontSetAutoCommitFalse       |                         False                         | 设置这个参数为 true 会告诉 Quartz 从数据源获取的连接后不要调用它的 setAutoCommit(false) 方法。这在少些情况下是有帮助的，比如假如你有这样一个驱动，它会抱怨本来就是关闭的又来调用这个方法。这个属性默认值是 false，因为大多数的驱动都要求调用 setAutoCommit(false)。 |
| org.quartz.jobStore.selectWithLockSQL            | SELECT * FROM {0}LOCKS WHERE LOCK_NAME = ? FOR UPDATE | 这必须是一个从 LOCKS 表查询一行并对这行记录加锁的 SQL 语句。假如未设置，默认值就是 SELECT * FROM {0}LOCKS WHERE LOCK_NAME = ? FOR UPDATE，这能在大部分数据库上工作。{0} 会在运行期间被前面你配置的 TABLE_PREFIX 所替换。 |
| org.quartz.jobStore.txIsolationLevelSerializable |                         False                         | 值为 true 时告知 Quartz(当使用 JobStoreTX 或 CMT) 调用 JDBC 连接的 setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE) 方法。这有助于阻止某些数据库在高负载和长时间事物时锁的超时。 |

4. 我们还需要配置Datasource 属性

| **属性**                                   | **必须** | **说明**                                                     |
| ------------------------------------------ | -------- | ------------------------------------------------------------ |
| org.quartz.dataSource.NAME.driver          | 是       | JDBC 驱动类的全限名                                          |
| org.quartz.dataSource.NAME.URL             | 是       | 连接到你的数据库的 URL(主机，端口等)                         |
| org.quartz.dataSource.NAME.user            | 否       | 用于连接你的数据库的用户名                                   |
| org.quartz.dataSource.NAME.password        | 否       | 用于连接你的数据库的密码                                     |
| org.quartz.dataSource.NAME.maxConnections  | 否       | DataSource 在连接接中创建的最大连接数                        |
| org.quartz.dataSource.NAME.validationQuary | 否       | 一个可选的 SQL 查询字串，DataSource 用它来侦测并替换失败/断开的连接。例如，Oracle 用户可选用 select table_name from user_tables，这个查询应当永远不会失败，除非直的就是连接不上了。 |

下面是我们的一个quartz.properties属性文件配置实例：

```
org.quartz.scheduler.instanceName = MyScheduler
org.quartz.threadPool.threadCount = 3
org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
org.quartz.jobStore.tablePrefix = QRTZ_
org.quartz.jobStore.dataSource = myDS

org.quartz.dataSource.myDS.driver = com.mysql.jdbc.Driver
org.quartz.dataSource.myDS.URL = jdbc:mysql://localhost:3306/quartz?characterEncoding=utf-8
org.quartz.dataSource.myDS.user = root
org.quartz.dataSource.myDS.password = root
org.quartz.dataSource.myDS.maxConnections =5
```

配置好quartz.properties属性文件后，我们只要将它放在类路径下，然后运行我们的程序，即可覆盖在quartz.jar包中默认的配置文件。