# spark入门

## spark部署

### 源码编译

SBT编译

	SPARk_HADOOP_VERSION=2.2.0 SPARK_YARN=true sbt/sbt compile assembly
	
Maven编译

	export MAVEN_OPTS=
	
spark部署包生成命令`make-distribution.sh`(调用了Maven)

	参数：
	--hadoop VERSION
	--with-yarn
	--with-hiv	e
	--skip-java-test
	--with-tachyon
	--tgz
	--name NAME
	
	e.g
		./make-distribution.sh --hadoop 2.2.0 --with-yarn --tgz
	
	根目录生成spark-x.x.x-bin-x.x.x-tgz
	assembly/target/scala-x.x.x/spark-assembly.xxx.jar
	
### 集群部署

* java安装
* ssh无密码登录
* spark安装包解压

		tar zxf spark-x.x.x-bin-x.x.x-tgz
		mv spark-x.x.x-bin-x.x.x sparkxxx
		
* spark配置文件
		
		cd sparkxxx
		cd conf
		vim slaves 把集群的hostname添加进去
		cp spark-env.sh-template spark-env.sh
			export SPARK_MASTER_IP=hadoop1
			export SPARK_MASTER_PORT=7077
			export SPARK_WORKER_CORES=1
			export SPARK_WORKER_INSTANCES=1
			export SPARK_WORKER_MEMORY=3g
		
* 把sparkxxx目录拷贝到集群其他机器相应目录

* 浏览器访问hadoop1:8080	

* 拷贝sparkxxx到客户端机器，使用	shell应用程序

		cd sparkxxx
		bin/spark-shell --master spark://hadoop1:7077
		
### standalone HA部署

#### 基于文件系统的HA
	
spark-env.sh增加

	export SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy......=FILESYSTEM ......"
	
拷贝到各个节点

启动、停止

	sbin/start-all.sh			
	sbin/stop-all.sh
	
#### 基于zookeeper的HA

spark-env.sh增加

	export SPARK_DAEMON_JAVA_OPTS="......"
	内容包括：
		spark.deploy.recoverryMode=ZOOKEEPER
		spark.deploy.zookeeper.url=xxx
		spark.deploy.zookeeper.dir=xxx

同时删除master的配置参数，因为是从zookeeper中获取master参数

	export SPARK_MASTER_IP=hadoop1
	export SPARK_MASTER_PORT=7077
		
### 伪分布式部署

* java安装
* 自己对自己的ssh无密码
* 配置

		cd spark/conf
		vim slaves
		vim spakr-env.sh
* 启动spark

		sbin/start-all.sh
		
## spark使用工具

交互工具：spark-shell
应用部署工具：spark-submit

## spark快不仅使基于内存计算，还和其原理相关

* DAG
* Schedule
* Cache()		

## spark编程模型

spark应用程序组成：`Driver`和`Executor`

* application：基于spark的应用程序
* Driver program：运行application的main函数并创建sparkcontext
* Executor：application运行在worker node上的一个进程，该进程负责运行task，并负责将数据存在内存或磁盘
* Cluster manager：在集群上获取资源的外部服务，如standalone、mesos、yarn
* worker node：集群中任何可以运行application的节点
* task：被送到某个Executor的工作单元
* job：包含多个task组成的并行计算，往往有spark action催生
* stage：每个job会被拆分多组task，每组任务被称为stage或TaskSet
* RDD：spark基本计算单元，可以通过一系列算子进行操作，主要由transformation和action操作

### driver program
* 导入spark类和隐式转换
* 构建spark应用程序运行环境SparkConf
* 初始化SparkContext
* 关闭SparkContext

Spark-shell在启动的时候会自动构建SparkContext，名称为sc

#### 输入

* 并行化Scala结合

		Spark使用parallelize方法转换为RDD
		val rdd1 = sc.Parallelize(Array(1, 2, 3, 4, 5))
		val rdd2 = sc.Parallelize(List(0 to 10), 5)
		参数slice是对数据集切片，每一个slice启动一个task进行处理
		
* Hadoop数据集

Spark可以将任何Hadoop支持的存储资源转换为RDD，如本地文件、hdfs、cassandra、hbase、Amazon s3

		使用textFile()方法可以将本地文件或HDFS文件
		使用wholeTextFiles()读取目录里面小文件，返回（文件名，内容）对
		使用sequenceFile[K,V]()方法可将SequenceFile转换为RDD
		使用hadoopRDD()方法可以将任何Hadoop输入类型转换为RDD
		
#### transfomation

* 在一个已有的RDD上创建一个新的RDD
* 延迟执行
		
	算子：
	* map(func)
	* filter(func)
	* flatMap(func)	
	* sample(withReplacement, fraction, seed)
	* union(otherDataset)
	* distinct([numTask])
	* groupByKey([numTasks])
	* reduceByKey(func, [numTasks])
	* sortByKey([ascending], [numTasks])
	* join(otherDataset, [numTasks])
	* cogroup(otherDataset, [numTasks])
	* cartesian(otherDataset)
	
#### action特点

* 对RDD数据集进行运算后，将结果传给驱动程序或写入存储系统，触发job
	
算子：
	
* reduce(func)
* collect()
* count()
* first()
* take(n)
* takeSample(withReplacement, fraction, seed)
* saveAsTextFile(path)
* saveAsSequenceFile(path)
* countByKey()
* foreach(func)	

#### 缓存

缓存特点

* 使用gpersist和cache将rdd缓存到内存中
* 缓存是容错的，可通过transformation自动筹够
* 被换成的rdd被使用时，存取速度加速10x

缓存等级
	
> storagelevel -- DISK_ONLY、MEMORY_ONLY、MEMORY_AND_DISK、OFF_HEAP......

cache：是persist的特例

Executor中60%做cache，40%做task

#### 广播变量和累加器

广播变量

* 广播变量缓存到各个节点的内存中
* 广播变量能在集群中任何函数调用
* 广播变量只读
* 使用方法

		val broadcastVar = sc.broadcast(Array(1, 2, 3))
		broadcastVal.value
		
累加器

* 累加器只支持加法操作
* 累加器可以高效并行，实现计算器和变量求和
* spark原生支持数值类型和标准可变集合计数器，但用户可添加新的类型
* 只有驱动程序才能获取累加器的值
* 使用方法

		val accum = sc.accumulator(0)
		sc.parralelize(Array(1, 2, 3, 4)).foreach(x => accum += x)
		accum.value	
	
## RDD

特性

* 分区
* 依赖
* 函数
* 分区策略（K，V类型的RDD）
* 本地性策略

## spark-shell调试程序

示例：

	//parallelize演示
	val num=sc.parallelize(1 to 10)
	val doublenum = num.map(_*2)
	val threenum = doublenum.filter(_ % 3 == 0)
	threenum.collect
	threenum.toDebugString
	
	val num1=sc.parallelize(1 to 10,6)
	val doublenum1 = num1.map(_*2)
	val threenum1 = doublenum1.filter(_ % 3 == 0)
	threenum1.collect
	threenum1.toDebugString
	
	threenum.cache()
	val fournum = threenum.map(x=>x*x)
	fournum.collect
	fournum.toDebugString
	threenum.unpersist()
	
	num.reduce (_ + _)
	num.take(5)
	num.first
	num.count
	num.take(5).foreach(println)
	
	//K-V演示
	val kv1=sc.parallelize(List(("A",1),("B",2),("C",3),("A",4),("B",5)))
	kv1.sortByKey().collect //注意sortByKey的小括号不能省
	kv1.groupByKey().collect
	kv1.reduceByKey(_+_).collect
	
	val kv2=sc.parallelize(List(("A",4),("A",4),("C",3),("A",4),("B",5)))
	kv2.distinct.collect
	kv1.union(kv2).collect
	
	val kv3=sc.parallelize(List(("A",10),("B",20),("D",30)))
	kv1.join(kv3).collect
	kv1.cogroup(kv3).collect
	
	val kv4=sc.parallelize(List(List(1,2),List(3,4)))
	kv4.flatMap(x=>x.map(_+1)).collect
	
	//文件读取演示
	val rdd1 = sc.textFile("hdfs://hadoop1:8000/dataguru/week2/directory/")
	rdd1.toDebugString
	val words=rdd1.flatMap(_.split(" "))
	val wordscount=words.map(x=>(x,1)).reduceByKey(_+_)
	wordscount.collect
	wordscount.toDebugString
	
	val rdd2 = sc.textFile("hdfs://hadoop1:8000/dataguru/week2/directory/*.txt")
	rdd2.flatMap(_.split(" ")).map(x=>(x,1)).reduceByKey(_+_).collect
	
	//gzip压缩的文件
	val rdd3 = sc.textFile("hdfs://hadoop1:8000/dataguru/week2/test.txt.gz")
	rdd3.flatMap(_.split(" ")).map(x=>(x,1)).reduceByKey(_+_).collect
	
	//日志处理演示
	//http://download.labs.sogou.com/dl/q.html 完整版(2GB)：gz格式
	//访问时间\t用户ID\t[查询词]\t该URL在返回结果中的排名\t用户点击的顺序号\t用户点击的URL
	//SogouQ1.txt、SogouQ2.txt、SogouQ3.txt分别是用head -n 或者tail -n 从SogouQ数据日志文件中截取
	
	//搜索结果排名第1，但是点击次序排在第2的数据有多少?
	val rdd1 = sc.textFile("hdfs://hadoop1:8000/dataguru/data/SogouQ1.txt")
	val rdd2=rdd1.map(_.split("\t")).filter(_.length==6)
	rdd2.count()
	val rdd3=rdd2.filter(_(3).toInt==1).filter(_(4).toInt==2)
	rdd3.count()
	rdd3.toDebugString
	
	//session查询次数排行榜
	val rdd4=rdd2.map(x=>(x(1),1)).reduceByKey(_+_).map(x=>(x._2,x._1)).sortByKey(false).map(x=>(x._2,x._1))
	rdd4.toDebugString
	rdd4.saveAsTextFile("hdfs://hadoop1:8000/dataguru/week2/output1")
	
	
	//cache()演示
	//检查block命令：bin/hdfs fsck /dataguru/data/SogouQ3.txt -files -blocks -locations
	val rdd5 = sc.textFile("hdfs://hadoop1:8000/dataguru/data/SogouQ3.txt")
	rdd5.cache()
	rdd5.count()
	rdd5.count()  //比较时间
	
	
	//join演示
	val format = new java.text.SimpleDateFormat("yyyy-MM-dd")
	case class Register (d: java.util.Date, uuid: String, cust_id: String, lat: Float,lng: Float)
	case class Click (d: java.util.Date, uuid: String, landing_page: Int)
	val reg = sc.textFile("hdfs://hadoop1:8000/dataguru/week2/join/reg.tsv").map(_.split("\t")).map(r => (r(1), Register(format.parse(r(0)), r(1), r(2), r(3).toFloat, r(4).toFloat)))
	val clk = sc.textFile("hdfs://hadoop1:8000/dataguru/week2/join/clk.tsv").map(_.split("\t")).map(c => (c(1), Click(format.parse(c(0)), c(1), c(2).trim.toInt)))
	reg.join(clk).take(2)
	
## 在IDEA中的调试

	
	
	
## spark编程模型

### spark运行架构

* job： 包含多个task组成的并行计算，往往由Spark action催生
* stage：job的调度单位，对应于taskset
* taskset：一组关联的，相互之间没有shuffle依赖关系的人去组成的任务集
* task：被送到某个executor上的工作单元

运行图
![运行图](http://image17-c.poco.cn/mypoco/myphoto/20160116/10/1734971822016011610513903_640.jpg?2048x869_120)

简易运行图
![运行图1](http://image17-c.poco.cn/mypoco/myphoto/20160116/10/17349718220160116105218084_640.jpg?1186x650_130)

复杂运行图
![运行图2](http://image17-c.poco.cn/mypoco/myphoto/20160116/11/17349718220160116111752043.png?1524x854_130)

复杂运行图
![运行图3](http://image17-c.poco.cn/mypoco/myphoto/20160116/11/1734971822016011611104206_640.jpg?2048x1042_120)

深入运行图
![运行图4](http://image17-c.poco.cn/mypoco/myphoto/20160116/11/17349718220160116111118071_640.jpg?2048x934_120)

更深入
![运行图5](http://image17-c.poco.cn/mypoco/myphoto/20160116/11/17349718220160116111221039_640.jpg?2048x915_130)

wordcount例子解析
![spark workcount](http://image17-c.poco.cn/mypoco/myphoto/20160116/11/1734971822016011611130904_640.jpg?2048x1759_120)


## SparkSQL原理

### hive

* ETL（extration-transformation-loading）工具
	
![hive-架构图](http://image17-c.poco.cn/mypoco/myphoto/20160116/11/17349718220160116115302051_640.jpg?1856x990_130)	

#### 元数据存储（metastore）
* derby
* mysql

#### 驱动

* 编译器（hive核心）

	* 语义解析器（parsedriver）
	* 语法分析器（semanticanalyzer）
	* 逻辑计划生成器（logical plan generator）
	* 查询计划生成器（query plan generator）
* 优化器
* 执行器

#### 接口

* CLI：命令行接口：bin/hive
* hwi: web接口， bin/hive --service hwi 默认端口9999
* thriftserver： bin/hive --service hiveserver 默认端口10000

* 其他服务：metastore，hiveserver2

## hive安装

[hive安装](https://github.com/zhuwei05/blog/blob/master/Spark/hive%E5%AE%89%E8%A3%85.md)


## spark stream

### 常用流处理系统
* storm
* trlent
* s4
* spark stream

### storm
* nimbus：负责资源分配和任务调度
* supervisor：负责接收nimbus分配的任务，启动和停止属于自己管理的workers
* worker：运行具体处理组件逻辑的进程
* executor：运行spout/bolt的线程
* task：worker中每一个spout/bolt的线程称为一个task

* topology：storm中运行的实时应用程序，消息在各个组件中流动形成逻辑上的拓扑结构
* spout：在一个topology中产生源数据流的组件
* bolt：在一个topology中接受数据然后执行处理的组件。

* tuple：消息传递的基本单元
* stream：源源不断传递的tuple组成了steam
* stream grouping：即消息的partition方法。storm提供多种group的方法：shuffle，fields hash，All，global，none，direct，localOrShuffle

* 多语言编程：clojure，java，ruby，python。如果要增加其他语言，只需实现一个简单的storm通信协议
* 容错性
* 水平扩展
* 快速
* 系统可靠性

### spark steaming原理

![spark-stream运行图](http://image17-c.poco.cn/mypoco/myphoto/20160118/07/17349718220160118075746026_640.jpg?1794x922_130)

#### 编程模型DStream
* discretized stream
* 表示为数据流，实现为RDD序列
* 从流输入源创建，从现有DStreams通过transformation转换而来

#### 输入源
kafka、flume、zeromq、mqtt、自定义数据源

#### output
* print
* foreachRDD(func)
* saveAsObjectFiles(prefix, [suffix])
* saveAsTextFiles(prefix, [suffix])
* saveAsHadoopFile(prefix, [suffix])

#### transformation
* map
* flatMap
* filter
* ......
* transform(func)
* updateStateByKey(fuc)
* window
* ......

### 持久化和容错

持久化：

* persist
* 默认持久化：MEMORY_ONLY_SER
* 对于来自网络数据源：MEMORY_AND_DISK_SER_2
* 对于window和stateful默认持久化

checkpoint

* 对于window和stateful操作必须checkpoint
* 通过streamingContext的checkpoint指定目录
* 通过DStream的checkpoint指定间隔事件
* 间隔必须是slide interval的倍数

DStream基于RDD组成，RDD容错性依旧有效

### 优化

* 利用集群资源，减少处理每个批次的数据时间

	* 控制reduce数据
	* 序列化：输入数据、RDD、Task序列化
	* 在standalone和coarse-grained模式下的任务启动要比fine-grained省时

* 给每个批次的数据量设定一个合适的大小：原则：要来得及笑话流进入系统的数据
* 内存调优
	* 清理缓存的RDD

### 实例


## 机器学习

### k-means
聚类的一个算法，是一个无监督学习，目标是讲一部分实体根据某种意义上的相似度和另一部实体聚在一起。聚类通常用于探索性的分析。

算法：

1. 选择k个点作为初始中心
2. 将每个点指派到最近的中心，形成k个族
3. 重新计算每个族的中心
4. 重复2-3直至中心不再发生变化

距离：

1. 绝对值距离
2. 欧式距离
3. 闵可夫斯基距离
4. 切比雪夫距离
5. 马氏距离

### MLlib的K-Means
实现中包含一个k-means++方法的并行化变体kmeans||

### 协同过滤

通常用于推荐系统。旨在补充用户-商品关联矩阵中所缺失的部分。


## GraphX

提供表和图的转化以及计算引擎。

![graphx架构](http://image17-c.poco.cn/mypoco/myphoto/20160120/08/17349718220160120083726034_165.jpg?2048x1067_130)

![graphx操作](http://image17-c.poco.cn/mypoco/myphoto/20160120/08/17349718220160120083817034_165.jpg?2048x1051_120)


















	
	


		
	



	
	
	
	