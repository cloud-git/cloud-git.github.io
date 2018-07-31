# datax

https://github.com/alibaba/DataX

各种异构数据源之间高效的数据同步

## source code analysis

### 1. entry point

用stream测试分析。

    ./bin/datax.py job/1.json -d

datax.py 源码在core包下，主要是是构造java命令参数。mode默认standalone，jobid默认-1，jvm默认是DEFAULT_JVM，CLASS_PATH是要当前目录的lib加载了，DEFAULT_PROPERTY_CONF是一些通用配置（编码、日志等），job指定的是json文件（来源于args[0]，实例/home/mysql/datax/job/1.json）。

    ENGINE_COMMAND = "java -server ${jvm} %s -classpath %s  ${params} com.alibaba.datax.core.Engine -mode ${mode} -jobid ${jobid} -job ${job}" % (
                                      DEFAULT_PROPERTY_CONF, CLASS_PATH)

### 2. Engine

#### 2.1 配置
Engine.entry(args); 解析并且验证了参数。在ConfigParser.parse(jobPath);解析了很多配置文件，还merge了一些配置，比如core.json/plugin配置加载。具体是jsonobject（Configuration.root）和path（a.b.c）之间的转化。

standalone 时jobId可以为-1？

打印vmInfo。engine.start(configuration); 真正开始执行任务。

#### 2.2 job

这里有两种container,先分析job.
配置中core.container.model为空，代表是个job。设置为core.container.job.mode为standalone，启动JobContainer。
基本流程如下。

    this.preHandle();
    LOG.debug("jobContainer starts to do init ...");
    this.init();
    LOG.info("jobContainer starts to do prepare ...");
    this.prepare();
    LOG.info("jobContainer starts to do split ...");
    this.totalStage = this.split();
    LOG.info("jobContainer starts to do schedule ...");
    this.schedule();
    LOG.debug("jobContainer starts to do post ...");
    this.post();
    LOG.debug("jobContainer starts to do postHandle ...");
    this.postHandle();
    LOG.info("DataX jobId [{}] completed successfully.", this.jobId);
    this.invokeHooks();

preHandle阶段： 为空。job.preHandler相关参数。

init阶段把jobId<0的设置为0. 同时initJobReader和initJobWriter。initJobReader中加载了StreamReader，并jobReader.init()，加载时涉及到classloader的切换，使用Thread.currentThread().setContextClassLoader，自定义了JarLoader，继承自URLClassLoader。

prepare阶段：prepareJobReader和prepareJobWriter。调用了具体的实现，也涉及了classloader的切换。

split阶段：adjustChannelNumber中根据配置设置channel，可以根据byte，records，channel限速。byte需要设置 `job.setting.speed.byte` 和 `core.transport.channel.speed.byte` 两个值，然后计算channel数，record类似。channel直接根据job.setting.speed.channel。needChannelNumberByByte 和 needChannelNumberByRecord 两个值去较小值，如果设置了 channel 则最终用这个。
同时调用了reader、writer实现类的split生成了相关的task的配置，因为job.setting.speed.channel是5，所以生成了5个task。taskConfig设置到job.content属性。

schedule：计算taskGroup，根据taskNumber和core.container.taskGroup.channel(default 5)计算只需要一个group。JobAssignUtil.assignFairly构造taskGroupConfig，其中打乱了任务，为了更好的资源分配。
new了一个StandAloneScheduler，并设置了StandAloneJobContainerCommunicator（信息汇总）。scheduler.schedule执行任务。
根据taskGroup启动线程，执行TaskGroupContainerRunner，接着进入task流程，TaskGroupContainer是入口。

#### 2.3 task

runner里执行了 this.taskGroupContainer.start();进入taskGroup流程。根据core.container.taskGroup.channel（default 5）启动TaskExecutor，new TaskExecutor(taskConfigForRun, attemptCount)时生产对应的runner以及加载了reader、writer的plugin。runTasks在一个ArrayList里维护。
while还有监控相关taskMonitor的先不看。

taskExecutor里会初始化readerRunner和writerRunner以及相应的thread，taskExecutor.doStart()启动两个线程。先writerThread.start，后readerThread。

##### 2.3.1 WriterRunner

    channelWaitRead.start();
    LOG.debug("task writer starts to do init ...");
    PerfRecord initPerfRecord = new PerfRecord(getTaskGroupId(), getTaskId(), PerfRecord.PHASE.WRITE_TASK_INIT);
    initPerfRecord.start();
    taskWriter.init();
    initPerfRecord.end();

    LOG.debug("task writer starts to do prepare ...");
    PerfRecord preparePerfRecord = new PerfRecord(getTaskGroupId(), getTaskId(), PerfRecord.PHASE.WRITE_TASK_PREPARE);
    preparePerfRecord.start();
    taskWriter.prepare();
    preparePerfRecord.end();
    LOG.debug("task writer starts to write ...");

    PerfRecord dataPerfRecord = new PerfRecord(getTaskGroupId(), getTaskId(), PerfRecord.PHASE.WRITE_TASK_DATA);
    dataPerfRecord.start();
    taskWriter.startWrite(recordReceiver);
    dataPerfRecord.addCount(CommunicationTool.getTotalReadRecords(super.getRunnerCommunication()));
    dataPerfRecord.addSize(CommunicationTool.getTotalReadBytes(super.getRunnerCommunication()));
    dataPerfRecord.end();

    LOG.debug("task writer starts to do post ...");
    PerfRecord postPerfRecord = new PerfRecord(getTaskGroupId(), getTaskId(), PerfRecord.PHASE.WRITE_TASK_POST);
    postPerfRecord.start();
    taskWriter.post();
    postPerfRecord.end();

    super.markSuccess();


依次执行taskWriter的init、prepare、startWrite、post方法。

writer和reader的通信用的是BufferedRecordExchanger recordReceiver、recordSender。

StreamWriter的核心代码如下。

```java
while ((record = recordReceiver.getFromReader()) != null) {
    if (this.print) {
        writer.write(recordToString(record));
    } else {
/* do nothing */
    }
}
writer.flush();
```
最后markSuccess把Communication的state由 RUNNING 改为 SUCCEEDED.

#### 2.3.2 ReaderRunner

类似Writer。

### 2.4 more

#### 2.4.1 MemoryChannel BufferedRecordExchanger

ReaderRunner 分析。

首先分析 BufferedRecordExchanger。在 ReaderRunner 中的核心流程。
```java
taskReader.startRead(recordSender);
recordSender.terminate();
```
BufferedRecordExchanger 持有MemoryChannel，BufferedRecordExchanger是有缓存的buffer=new ArrayList<Record>(bufferSize);
```java
if (record.getMemorySize() > this.byteCapacity) {
    this.pluginCollector.collectDirtyRecord(record, new Exception(String.format("单条记录超过大小限制，当前限制为:%s", this.byteCapacity)));
    return;
}

boolean isFull = (this.bufferIndex >= this.bufferSize || this.memoryBytes.get() + record.getMemorySize() > this.byteCapacity);
if (isFull) {
    flush();
}

this.buffer.add(record);
this.bufferIndex++;
memoryBytes.addAndGet(record.getMemorySize());
```

具体 core.transport.exchanger.bufferSize 配置为32. byteCapacity（64M，core.transport.channel.byteCapacity）， 单个record最大不能超过 byteCapacity，同时isFull判断两个条件，一个是bufferSize一个就是缓存的byteCapacity。满了就flush，否则就加入缓存。
flush时，把buffer数据放入channel，然后清空buffer，重置bufferIndex和memoryBytes。
recordSender.terminate()也会flush，然后向channel发送TerminateRecord。

字节数按Column统计，addColumn时会incrByteSize。StringColumn算的string的length,`hello，你好，世界-DataX`算17字节明显是有问题的。
LongColumn用的也是String的length。其实channel和 BufferedRecordExchanger 用的都是 getMemorySize。
memoryBytes还加了类head所占空间，具体 ClassSize.DefaultRecordHead 和 ClassSize.ColumnHead。

接着分析MemoryChannel。底层其实是一个ArrayBlockingQueue。上面的 byteCapacity 影响channel的pushAll，bufferSize影响 pullAll。
doPushAll的核心为lock和condition(notInsufficient, notEmpty)。queue放不下就循环等待200ms，byteCapacity限制64M(core.transport.channel.byteCapacity),队列大小为core.transport.channel.capacity=512。
doPush直接put到queue。
在父类channel里，普通push会有stat统计，而pushTerminate不统计。
```java
lock.lockInterruptibly();
int bytes = getRecordBytes(rs);
while (memoryBytes.get() + bytes > this.byteCapacity || rs.size() > this.queue.remainingCapacity()) {
    notInsufficient.await(200L, TimeUnit.MILLISECONDS);
}
this.queue.addAll(rs);
waitWriterTime += System.nanoTime() - startTime;
memoryBytes.addAndGet(bytes);
notEmpty.signalAll();

```

StreamWriter 的核心其实是从 BufferedRecordExchanger 中 getFromReader 。当前buffer读完了，从channel中再获取一批。一开始bufferIndex和buffer.size()都是0，满足isEmpty，获取一次。
这里TerminateRecord返回null，也就是结束逻辑。
```java
boolean isEmpty = (this.bufferIndex >= this.buffer.size());
if (isEmpty) {
    receive();
}

Record record = this.buffer.get(this.bufferIndex++);
if (record instanceof TerminateRecord) {
    record = null;
}
return record;
```

两个 WriterRunner、ReaderRunner 所持有的 BufferedRecordExchanger 是独立的，共享的是channel。

#### 2.4.2 statistics 信息汇总统计用的。

Channel父类进行统计信息。
- statPush里面有速度控制，太快就sleep一会。限速计算间隔是 flowControlInterval = 20 ms。
- statPull直接记录统计信息。

1. job

先分析job。job的schedule阶段设置 StandAloneJobContainerCommunicator 。
StandAloneJobContainerCommunicator 继承自 AbstractContainerCommunicator 。而AbstractContainerCommunicator 有  VMInfo vmInfo; AbstractCollector collector;AbstractReporter reporter; 这三个核心参数。

collector 用的是 ProcessInnerCollector，reporter 用的是 ProcessInnerReporter。
StandAloneScheduler 的父类 AbstractScheduler schedule while循环。 从 ProcessInnerCollector.collectFromTaskGroup 收集 nowJobContainerCommunication 最终来自 LocalTGCommunicationManager.getJobCommunication(); 注意这个是下面的 TaskGroupContainer 更新的。

nowJobContainerCommunication.getState() 成功结束job，dealKillingStat 和 dealFailedStat 抛出异常。
jobSleepIntervalInMillSec(10000) 循环，同时 jobReportIntervalInMillSec(10000) 进行报告,输出 汇总信息(10s)和 vminfo (300000 5分钟)。

2. taskGroup

schedule 启动 TaskGroupContainerRunner ，runner 中其实是 TaskGroupContainer ， TaskGroupContainer 构造时 设置了 StandaloneTGContainerCommunicator ，用的是ProcessInnerCollector和ProcessInnerReporter。TaskGroupContainer 还有个 TaskMonitor。
core.statistics.collector.plugin.taskClass 配置为 StdoutPluginCollector 。 BufferedRecordExchanger 里用来收集脏数据的。StdoutPluginCollector 输出到 log 文件中。 而父类 AbstractTaskPluginCollector 更新 communication (来自 TaskExecutor 的 taskCommunication ) .
sleepIntervalInMillSec (100 ms) reportIntervalInMillSec (10000 10s)

有任务未执行，且正在运行的任务数小于最大通道限制 runTasks.size() < channelNumber 新建一个 TaskExecutor ，并添加到 taskMonitor 。 当前tg完成立刻汇报并break或者10s 汇报一次状态，最终 LocalTGCommunicationManager.updateTaskGroupCommunication(taskGroupId, communication); communication 来自  this.containerCommunicator.collect(); 收集的task信息和 lastTaskGroupContainerCommunication ，根据两个的数据差，时间差算速率。 state 用的是 nowTaskGroupContainerCommunication 的状态，也就是收集来的。

整个流程也就是 job 从 collectFromTaskGroup taskGroup LocalTGCommunicationManager 信息 并且report 到 控制台。ProcessInnerReporter 里  // do nothing。
taskGroup 从 task 里 collectFromTask 信息 并且report 给 LocalTGCommunicationManager。


3. 任务成功。

WriterRunner 执行完 会 markSuccess ，把 runnerCommunication 设置为 SUCCEEDED。 ReaderRunner 不标记。其实两个都持有 同一个 taskCommunication。

这里 communication 变量 有个引用链。
WriterRunner 的runnerCommunication 来自 TaskExecutor 的taskCommunication 来自 TaskGroupContainer 中 对应 taskId 的 communication .

4. taskMonitor
目前看是监控超时的 EXPIRED_TIME =  172800 * 1000; // 2day?
感觉没啥卵用。

5. PerfTrace
core.container.trace.enable

#### 2.4.3 jdbc

## misc

1. tg的计算

    int taskGroupNumber = (int) Math.ceil(1.0 * channelNumber / channelsPerTaskGroup);

channelsPerTaskGroup 这个 "core.container.taskGroup.channel 配置的是5.
channelNumber 来自 this.needChannelNumber = Math.min(this.needChannelNumber, taskNumber); 而needChannelNumber 来自之前的配置计算，如job.setting.speed.channel=3.
taskNumber 来自 split。

mysql 计算 split 任务 单表时 task数 为 adviceNumber*5; querySql 配置 就是一个task，没有并行。

2. vmInfo 打印的地方

Engine entry 开始时 会打印一次。
job 中 report 周期中会打印。 这个是 job的vm。
job 完成时 finally 会 打印一次。


看代码，如果TaskGroupContainer 是跑在 distribute 模式下 finally 也会打印一次。

## todo

- [x] standalone 时jobId可以为-1。开源版本只有standalone，看参数还有DISTRIBUTE。
- [ ] 打印vmInfo。
- [ ] job.preHandler相关参数
- [x] 不同的scheduler.schedule。目前开源就一种
- [ ] runnerCommunication的统计信息从哪来
- [ ] jdbc reader writer.
- [ ] 执行数据转化。com.alibaba.datax.core.transport.transformer相关。
