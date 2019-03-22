# Store模块的设计与实现

- [1、存储目录](https://github.com/wbear1/rocket_blog/hellword)
- [2、存储设计](https://github.com/wbear1/rocket_blog/arch)
- [3、存储实现](https://github.com/wbear1/rocket_blog/remoting)
  - [内存映射文件MappedFile](https://github.com/wbear1/rocket_blog/remoting)
  - [MappedFileQueue](https://github.com/wbear1/rocket_blog/remoting)
  - [消息存储CommitLog](https://github.com/wbear1/rocket_blog/remoting)
  - [消息元数据ConsumeQueue](https://github.com/wbear1/rocket_blog/remoting)
  - [简单文件Config](https://github.com/wbear1/rocket_blog/remoting)
 
####1、存储目录

先来看看存储目录下具体有哪些文件，对RocketMQ的存储模块有个直观的认识。
![dir](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/dir.png)

+ commitlog  目录下存储了该broker接收的mq的消息，文件名按消息偏移量命名，文件内容按一定格式编码，详细编码后文介绍。如下所示：每个文件大小为1GB，00000000222264557568文件为第207个文件，00000000223338299392文件为第208个文件，前面的0~206个文件，因满足删除策略已被删除，关于删除策略在后文介绍。
![commitlog](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/commitlog.png)

+ consumequeue 目录下存储了该broker的各个queue消息的offset，文件按{topicName}/{queueId}/{offset}目录存储，文件内容按一定格式编码。如下所示：topictest下有4个queue，queueId=0下面有一个文件，该文件记录了topictest下面的第0个queue中的消息在commitLog中的offset。
![consumequeue](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/consumequeue.png)

+ conifg 目录下存储各种配置信息，包括：topic的配置、consumer提交的offset、consumerFilter的配置、subscriptionGroup的配置等等，文件内容为json字符串，因文件都比较小，对文件的读写直接通过inputstream和outputstream进行操作。
![config1](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/config1.png)
![config2](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/config2.png)

* index 目录下存储了消息索引，用于快速查找消息
* checkpoint文件
* lock文件

####2、存储设计

核心设计如下图所示
![arch](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/arch.png)

其中messageStore为存储模块对外提供的功能接口，DefaultMessageStore为RokcetMQ的默认实现。
CommitLog、ConsumeQUeue、config、index、checkpoint为内部实现的几类存储。
最下面的黑色虚框表示使用内存映射文件读写文件，MappedFileQueue表示对一个目录的读写，底层都是使用MappedFile对应一个实际物理文件，出于效率的考虑，设计了AllocateMappedFileService用于提前创建文件。

MessageStore提供的主要方法：写消息、读消息、其中MessageExtBrokerInner为单条消息，MessageExtBatch为多条封装的批量消息
![MessageStore](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/MessageStore.png)
 
####3、存储实现

##### 内存映射文件MappedFile
初始化MappedFile，主要是将文件映射到MappedByteBuffer，对文件的读写操作就变成对MappedByteBuffer的操作，关于文件的nio操作相关资料比较多，此处不展开。
![MappedFile](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/MappedFile.png)

##### MappedFileQueue

##### 消息存储CommitLog

##### 消息元数据ConsumeQueue

##### 简单文件Config
