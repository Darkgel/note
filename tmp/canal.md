# canal相关

## 官方文档笔记

更高级的功能：

- kafka消息投递
- 消息队列
- docker
- prometheus监控
- Canal-Admin WebUI
- HA(High Availability高可用性)

### 架构

server代表一个canal运行实例，对应于一个jvm
instance对应于一个数据队列 （1个server对应1..n个instance)

instance模块：

- eventParser (数据源接入，模拟slave协议和master进行交互，协议解析)
- eventSink (Parser和Store链接器，进行数据过滤，加工，分发的工作)
- eventStore (数据存储)
- metaManager (增量订阅&消费信息管理器)

查找start position ： conf/cash_deposit/meta.data -> conf/cash_deposit/instance.properties

parse\src\main\java\com\alibaba\otter\canal\parse\inbound\AbstractEventParser.java
parser中一直循环执行 erosaConnection.dump(startPosition.getJournalName(),startPosition.getPosition(),multiStageCoprocessor);//（dump所在parse\src\main\java\com\alibaba\otter\canal\parse\inbound\mysql\MysqlConnection.java）

binlog的存储：

- 传递给EventSink模块进行数据存储，是一个阻塞操作，直到存储成功（没有空位的时候会阻塞）
- 目前仅实现了Memory内存模式，后续计划增加本地file存储，mixed混合模式
- 借鉴了Disruptor的RingBuffer的实现思路

## mysql binlog

binlog中主要是记录数据库**变动**的“events”

binlog 的两个主要作用

- 复制（replication）
- 某些场景下的数据修复

The binary log is generally resilient to unexpected halts because only complete transactions are logged or read back. （只有已完成的事务才会被记录被读回）

A binary log file may become larger than max_binlog_size if you are using large transactions because a transaction is written to the file in one piece, never split between files.（同一个事务会写入到同一个文件中，不会切分开）

除了binlog文件，还有一个索引文件（通常在相同的目录下，扩展名为.index）用来记录哪些binary logs 文件已经被使用。 

binlog文件中记录events的三种格式：

- row-based logging ： --binlog-format=ROW，the master writes events to the binary log that indicate how individual table rows are affected
- statement-based logging ：  --binlog-format=STATEMENT
- mixed-base logging ：--binlog-format=MIXED，statement-based logging is used by default, but the logging mode switches automatically to row-based in certain cases

实际上中继日志（ralay log）和binlog日志的格式是一样的

相应的工具：mysqlbinlog（可以实现日志重放）

binlog的记录时机：**Binary logging is done immediately after a statement or transaction completes but before any locks are released or any commit is done. This ensures that the log is logged in commit order.**（Updates to nontransactional tables are stored in the binary log immediately after execution.）

### 设置binlog 格式

设置后所有新连接生效: SET GLOBAL binlog_format = 'STATEMENT';
设置后自己会话的连接生效： SET SESSION binlog_format = 'STATEMENT';

row格式下还是有一些变动是 statement-based 格式的：例如所有的 DDL (data definition language) statements such as CREATE TABLE, ALTER TABLE, or DROP TABLE.