### 一、yarn模式

Flink 提供了两种在 yarn 上运行的模式，分别为 **Session-Cluster**和 **Per-Job-Cluster**模式。  

#### 1.Session-Cluster 

**（1）启动hadoop集群**

**（2）启动 yarn-session**  

~~~shell
bin/yarn-session.sh -n 2 -s 2 -jm 1024 -tm 1024 -nm test -d
~~~

**-n(--container)：** TaskManager 的数量。
**-s(--slots)：** 每个 TaskManager 的 slot 数量，默认一个 slot 一个 core，默认每个taskmanager 的 slot 的个数为 1，有时可以多一些 taskmanager，做冗余。
**-jm：** JobManager 的内存（单位 MB)。
**-tm：**每个 taskmanager 的内存（单位 MB)。
**-nm：** yarn 的 appName(现在 yarn 的 ui 上的名字)。
**-d：**后台执行  

**（3）提交任务**

~~~shell
bin/flink run -c com.atguigu.wc.StreamWordCount -p 3 FlinkTutorial-1.0-SNAPSHOT-jar-with-dependencies.jar  --host spark01 –port 8888
~~~

**-p:**  并行度

**(4)停止会话**

~~~shell
yarn application --kill application_1577588252906_0001
~~~

#### 2.Per Job Cluster  

**(1)启动hadoop集群**

**(2)不启动 yarn-session，直接执行 job**  

加入 **–m yarn-cluster**  

### 二、运行架构

#### 1.运行组件

**JobManager:**（1）JobGraph=》ExecutionGraph（2）向ResourceManager请求必要的资源（slot）(3)资源足够，将ExecutionGraph发送到TaskManager。

运行的过程中，还会负责中央的协调操作。

**ResourceManager:**当 JobManager 申请插槽资源时， ResourceManager会将有空闲插槽的 TaskManager 分配给 JobManager。如果 ResourceManager 没有足够的插槽来满足 JobManager 的请求，它还可以向资源提供平台发起会话，以提供启动 TaskManager进程的容器。另外， ResourceManager 还负责终止空闲的 TaskManager，释放计算资源。  

**TaskManager:**执行任务，还可以跟其他的TaskManager交换数据

**Dispatcher:**

#### 2.并行度

**并行度：**一个特定算子的子任务个数。

**整条流的并行度：**所有算子中最大的并行度。

**静态的并行度：**指 TaskManager 具有的并发执行能力，可以通过参数 **taskmanager.numberOfTaskSlots** 进行配置  。

**动态的并行度：**按照优先级由大到小，每个任务后面设置（局部） **>**   env后面设置（代码全局） **>**   提交job时-p指定  **>** conf中默认配置

**算子之间的传输数据形式：**One-to-one:maop,filter类似spark中的窄依赖；Redistributing： keyBy,rebalance(轮询),broadcast类似宽依赖。

Spark中的shuffle类似洗牌，而flink中是发牌，不会等待所有任务完成。

**任务链：**相同并行度的 one to one 操作  （同共享组）Flink 这样相连的算子链接在一起形成一个 task，原来的算子成为里面的一部分。 

不进行合并任务，但还可以在一个slot中：方法一：加入重分区操作（shuffle,rebalance）;方法二：disableChaining,任务前后，都断开。 

前断后不断：.startNewChain()

**思考1.一个job，需要占用多少slot？**

（1）先求每个组中的算子的最大并行度（2）对每个组求和即为占用的slot

**思考2.一个流处理程序，到底包含多少个任务？**

相同并行度的 one to one 操作  （同组）Flink 这样相连的算子链接在一起形成一个 task，原来的算子成为里面的一部分。 

#### 3.常用操作

**设置并行度：**setParallelism()

**设置共享组：**slotSharingGroup(“分组名”)，组内任务可以共享一个槽，不同组不可以。默认跟前一个任务为同一个共享组。

**不合并任务：**disableChaining，任务前后都拆分;              startNewChain(),前断后不断

### 三、API

* DataStream
  * SingleOutputStreamOperator
  * KeyedStream
  * SplitStream
* WindowedStream
* ConnectedStreams

#### 1. Source

* 从集合中读取数据 env.fromCollection()或者env.fromElements()
* 从文件中读取数据 env.readTextFile()
* 从kafka中读取数据 添加pom依赖，addSource()

#### 2.Transform

* **基本转换**  **DataStream ->SingleOutputStreamOperator** 
  * map
  * fliatMap
  * filter
* **keyBy**          **DataStream → KeyedStream**：逻辑地将一个流拆分成不相交的分区，每个分区包含具有相同 key 的元素，在内部以 hash 的形式实现的。  
* **滚动聚合算子** **KeyedStream → DataStream**  针对 KeyedStream 的每一个支流做聚合。  
  * sum,min,max()    可以针对多个字段，并且更新对应的字段
  * minBy,maxBy()    只能针对以一个字段，整条数据都更新。
  * reduce()     一个分组数据流的聚合操作，合并当前的元素和上次聚合的结果，产生一个新的值，返回的流中包含每一次聚合的结果，而不是只返回最后一次聚合的最终结果。  
* **Split 和 Select**  两个操作才能实现拆流操作
  * Spalit    **DataStream → SplitStream**  根据需求给数据打上标签，实际上任是一个流。
  * Select   **SplitStream→ DataStream**   根据标签拆出一个流。
* **Connect 和 CoMap**  
  * Connect **DataStream → ConnectedStreams：**  放入同一个流中，内部仍保持着各自的数据格式，两个流相互独立。
  * CoMap、CoFlatMap  **ConnectedStreams → DataStream**   对 ConnectedStreams 中的每一个 Stream 分别进行 map 和 flatMap
    处理 。
* **Union**  支持多个流合并，但是要求每个流的类型一致

#### 3.Sink  

* kafka addSink()  **DataStream →DataStreamSink**
* Redis
* ......

### 四、Window

一般在使用window时，前面会使用keyBy,计算窗口内key对应的结果。

#### 1.window类型

* Window  **KeyedStream->WindowedStream**(window前加keyBy操作)
  * TimeWindow
    * Tumbling Window 时间对齐，窗口长度固定，没有重叠 timeWindow（size,）
    * Sliding Window 时间对齐，窗口长度固定， 可以有重叠   timeWindow（size,slide）
    * Session Window 时间无对齐  定义多久无数据就关闭窗口
  * CountWindow
    * Tumbling Window
    * Sliding Window

#### **2.window function**  

* window function  **WindowedStream->SingleOutputStreamOperator**
  * 增量聚合函数  
    * ReduceFunction  
    * AggregateFunction  
    * max,min,maxBy...........
  * 全窗口函数  拿到的信息更多
    * apply(new WindowFunction(){})

#### 3.其他API

* other APIs  **WindowedStream->WindowedStream**
  * trigger 触发器  定义 window 什么时候关闭，触发计算并输出结果  
  * evitor  移除器  定义移除某些数据的逻辑  
  * allowedLateness  允许处理迟到的数据  
  * sideOutputLateData  将迟到的数据放入侧输出流  结合getSideOutput 获取滞后的数据
  * getSideOutput  获取侧输出流  **SingleOutputStreamOperator->DataStream**

### 五、时间语义和Wartermark

#### 1.时间语义

* 时间语义
  * Event Time  事件创建时间（数据内容中的时间戳）
  * Ingestion Time  数据进入Flink的时间
  * Processing Time  每一个执行基于时间操作的算子的本地系统时间  

**使用：**

~~~java
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
~~~

#### 2.Wartermark（毫秒）

思想：整体上把时间调慢，有新的时间（大的时间戳数值）就更新，旧的不更新，按照调慢的时间触发计算。

**2.1Wartermark计算：**Wartermark = maxEventTime  -  延迟时长  

**2.2Wartermark特点**：（1）递增，有新的值才会改变。（2）Watermark 比当前未触发的窗口的停止时间要晚，那么就会触发相应窗口的执行。  

**2.3使用：**  **dataStream->SingleOutputStreamOperator**

抽取数据中的时间戳 ,计算为watermark,将其插入到流中。

Time.milliseconds(1000)为延迟时间，element.getTimeStamp() * 1000L指定时间戳

~~~java
dataStream.assignTimestampsAndWatermarks(
    new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.milliseconds(1000)) {
             @Override
             public long extractTimestamp(SensorReading element) {
             	return element.getTimeStamp() * 1000L;
             }
    }
)
~~~

里面需要传一个TimestampAssigner

* TimestampAssigner

  * AssignerWithPeriodicWatermarks

    周期性的生成 watermark  默认周期是 200 毫秒  。时间戳大于之前的水位的时间戳，新的water插入，否则，不会产生新的。

    * BoundedOutOfOrdernessTimestampExtractor
    * ....

  * AssignerWithPunctuatedWatermarks

    有数据来就会进行判断

**2.4计算窗口的起始时间**

starttime = timestamp - (timestamp - offset + windowSize) % windowSize

 timestamp为数据中的实际的timestamp

**思考**

（1）处理延迟的数据（带窗口,针对的是Event Time），三道保障 ？

都是以watermark为时间线计算。

第一道：watermark到达窗口的结束时间，触发窗口计算。

dataStream.assignTimestampsAndWatermarks()

第二道：设置延迟关闭窗口时间，延迟期间每来一个该窗口的数据都会触发计算。

windowStream.allowedLateness() 

第三道：设置测输出流用于接收窗口迟到的数据，此时窗口为关闭状态。

windowStream.sideOutputLateData()









