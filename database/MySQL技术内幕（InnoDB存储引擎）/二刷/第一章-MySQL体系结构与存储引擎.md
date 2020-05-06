# 第一章 MySQL体系结构与存储引擎

存储引擎是基于表的，而不是数据库。

InnoDB存储引擎的一些重点了解的地方：

- 逻辑表空间
- 使用多版本并发控制（MVCC）来获得高并发性，并且实现了SQL标准的4种隔离级别
- 使用next-key locking 的策略来避免幻读
- 插入缓冲（insert buffer）
- 二次写（double write）
- 自适应哈希索引（adaptive hash index）
- 预读（read ahead）