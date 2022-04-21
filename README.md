# DataProcess

基于大数据平台的数据处理服务框架:fire:
> Make big data processing more convenient

## :clipboard: 什么是DataProcess

结合大数据项目实际使用场景，提取出的一些通用的功能，形成大数据平台数据处理框架。

目前主要实现的功能有：  

- 参数信息配置模块，可实现采用数据库进行配置和Properties文件进行配置
- 集成Kafka，实现了Kafka的生产者和消费者相关的功能
- 集成MongoDB，实现了MongoDB的数据读取、写入等，实现了SparkSQL通过DataFrame与MongoDB的数据进行交互，并且实现了分页读取、流式读取等特殊读取方式
- 集成Redis，实现了Redis的读取、写入等，实现了SparkSQL通过DataFrame与Redis的数据进行交互
- SparkStreaming流式处理Kafka、MongoDB的数据
- 手动记录Kafka的偏移量，实现了基于数据库进行记录和基于Zookeeper进行记录
- 集成了规则引擎，客户标签、客户画像等功能可基于规则引擎进行实现

## :airplane: 软件架构

软件结构如下：  
```
DataProcess      项目根目录
├── commons             公共功能模块，提供配置文件读取、数据库连接、日志打印、工具类等公共功能，以供其他模块调用。  
├── examples            样例模块，提供各个功能点的样例代码。  
├── kafka-clients       KafkaClients相关功能，比如生产者、消费者等。
├── kafka-streams       主题数据过滤模块
├── rule-engine         规则引擎功能。
├── spark-sql           SparkSQL相关功能，扩展了Dataset/DataFrame的方法，集成Redis数据的读写、MongoDB数据的读写。  
├── spark-streaming     SparkStreaming实时数据处理模块，通过SparkStreaming程序，准实时消费Kafka中的数据，流式方式处理MongoDB中的数据。
└── third-party         第三方源码
      ├── hammurabi     Scala规则引擎
      ├── mongodb       Spark操作MongoDB
      └── redislabs     Spark操作Redis
```

> **skafka-treams说明**
>
> 主题数据过滤模块，Kafka自带的流处理功能，业务系统记录的日志如果包含了大量的：程序异常日志、数据库操作日志、调试日志等日志信息。
>
> 而采集的数据只需要日志文件中的特定数据的日志记录，那么对于我们采集到的日志来说，可能会有90%以上的日志都是垃圾数据，但是Flume组件没有提供日志过滤功能，而Spark程序又不应该消费这些数据。
>
> 这时就需要提供一个中间层，将Flume采集到的Topic1的日志中满足条件的数据筛选出来放到Topic2中，Spark程序只需要消费Topic2的数据即可，过滤条件按照正则表达式进行配置。这样Spark消费Topic2的数据都是我们需要的数据，并且我们可以及时的清理掉Topic1的数据以释放空间。

## :microphone: 使用说明

###  :blush:开发环境准备

这里以Intellij IDEA为例。

打开IDEA，File -> Import Project…，选择项目目录DataService-FusionInsight下的pom.xml，打开项目。

![028](works/images/028.png) 

打开了项目之后，可以根据自己的喜好，采用Java或Scala语言进行功能开发、扩展。

### :pencil2: ​功能扩展

目前，软件实现了Flume数据采集、Kafka主题数据过滤、SparkStreaming实时数据处理。

但是SparkStreaming的数据处理只实现了代码值标准化等基础功能。

目前默认支持的采集日志格式只有两种：分隔符分隔字段的数据、JSON格式的数据。

功能扩展可以从两个方面进行：

- SparkStreaming程序扩展，可以继续增加程序处理功能，完成更复杂的数据处理，比如：指标加工、客户行为分析、客户画像等
- 日志格式扩展，目前只开发了支持两种类型的日志格式，可以自定义类实现com.service.data.spark.streaming.process.TopicValueProcess接口，以实现其他格式的日志内容的解析。自定义实现类后，需要在spark-kafka模块的resources/META-INF/services/com.service.data.spark.streaming.process.TopicValueProcess文件中添加一行记录类名称，并且在使用过程中将其配置到数据库中即可。

![029](works/images/029.png)

自定义类编写完成后，需要在指定文件中添加实现类的名称。

![030](works/images/030.png)

这个自定义类名，用于在配置表中配置，最终实现用该类解析对应的Topic的数据。

![031](works/images/020.png)

###  :cat:程序运行

#### :one: 数据源设置

在运行程序之前，需要设置配置信息所在的数据库的数据源，以及SparkStreaming程序处理后的数据的最终落地的数据库数据源信息。主要修改的是commons下的db.properties文件，修改里面的数据库驱动、连接、用户、密码等信息。

![031](works/images/031.png)

其中：

- db.config.* 配置的是配置信息所在的数据库的数据源，即InitData.sql中的表所在的数据库信息。

- db.mysql.*/db.oracle.*/db.other.* 根据实际需求配置多数据源，即SparkStreaming程序处理数据后需要落地到哪些数据库。

- 据源配置信息中，用户名、密码是需要加密处理的，使用commons的test下的TestEncrypt进行用户密码的加密，并将加密后的字符串复制填充到配置文件里即可。

![031](works/images/032.png)



#### :two: 本地运行

SparkStreaming程序要在本地运行，需要指定master，可以设置为local[2]。

![031](works/images/033.png)

并且需要将pom.xml中的依赖的provided注释掉，以保证依赖中有Spark相关的库。

![031](works/images/034.png)

然后直接运行StreamingFromKafka类即可。

#### :three: 打包部署

最终的SparkStreaming程序将运行在Yarn上，因此需要将master设置去掉。

![031](works/images/035.png)

并且为了减小Jar包的大小，在打包的时候可以排除掉集群环境中已有的Jar的依赖。

![031](works/images/036.png)

打开Project Structure，在Artifacts中选择“+”，并选择从已有模块添加。

![031](works/images/037.png)

选择spark-kafka模块。

![031](works/images/038.png)

在配置中将spark-kafka设置为Put into Output Root。

![031](works/images/039.png)

通过Build –> Build Artifacts…

![031](works/images/040.png)

选择Build进行编译打包。

![031](works/images/041.png)

最后，在out目录下有一个spark-kafka.jar的软件包。

![031](works/images/042.png)

这个包就是打包后的最终结果，可以直接提交到Yarn上运行。

> PS：更多使用说明请查看下方文档

- 数据端配置工具：[数据端配置工具.xlsx](works/docs/%E6%95%B0%E6%8D%AE%E7%AB%AF%E9%85%8D%E7%BD%AE%E5%B7%A5%E5%85%B7.xlsx) 

- 环境搭建部署文档：[环境搭建部署文档.docx](works/docs/%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E9%83%A8%E7%BD%B2%E6%96%87%E6%A1%A3.docx)  


##  🤝 如何贡献

[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Fstreamxhub%2Fstreamx%2Fpulls)

如果你希望参与贡献 欢迎 `Pull Request`，或给我们 `报告 Bug`。

这个包就是打包后的最终结果，可以直接提交到Yarn上运行。

> 强烈推荐阅读 [《提问的智慧》](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Fryanhanwu%2FHow-To-Ask-Questions-The-Smart-Way)(**本指南不提供此项目的实际支持服务！**)、[《如何有效地报告 Bug》](https://gitee.com/link?target=http%3A%2F%2Fwww.chiark.greenend.org.uk%2F%7Esgtatham%2Fbugs-cn.html)、[《如何向开源项目提交无法解答的问题》](https://gitee.com/link?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F25795393)，更好的问题更容易获得帮助。

感谢所有向 DataProcess贡献的朋友!

