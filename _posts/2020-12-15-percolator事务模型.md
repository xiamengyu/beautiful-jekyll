---
layout: post
# title: percolator
subtitle: Percolator事务模型总结笔记-Draft
# cover-img: /assets/img/path.jpg
# thumbnail-img: /assets/img/thumb.png
# share-img: /assets/img/path.jpg
tags: [percolator, 2pc, 分布式事务]
---

Percolator本质上是一种2PC，基于MVCC与全局递增的TSO实现Snapshot Isolation。

### Timestamp Oracle Server

全局递增ID分配器，需要高性能高可用，本身是一个高可用组，基于时间戳加自增ID生成全局唯一且严格递增的uint64的整数。自增ID分配的在内存中自增能保证高性能，能达到2Mil/s。

### 多版本

每个Put和Delete都是对操作的Key新增一个版本。Put覆盖的版本和Delete删标覆盖的版本，会由后续的GC做真正的删除。

### Snapshot读

Get的请求会先获取一个TSO作为start_ts，并且仅能读到小于这个TSO的最大已经提交的版本。 所有commit_ts < start_ts的数据都需要能被读到，但是当遇到Lock，并且Lock的start_ts < 读请求的start_ts时需要等待重试，等锁被resolve或锁对应的事务提交。

### 2PC

Percolator事务的两个阶段分别为Prewrite和Commit。

#### PreWrite

- 检查是否有已经 commit_ts > start_ts 的数据，如果有则返回写-写冲突错误
- 检查是否有Lock，如果有则返回Lock冲突
- 写入Data和Lock（是否需要Row-Based Transaction?）

#### Commit

- 检查是否有锁
- 检查是否有检查是否有已经 commit_ts > start_ts 的数据
- 写入commit列，删除锁（是否需要Row-Based Transaction?）

#### 异步提交Secondary

如果同时写多个key，会随机选择其中1个Key做为Primary Key，加锁类型为Primary Lock，其他Key为Secondary Key，加锁类型为Secondary Lock
Secondary Lock里面记录了Primary Lock的引用。

如果Primary Key提交成功则所有的Secondary可以异步提交，并最终全部提交成功。

### 异常处理

### 思考

**Snapshot读的正确性保证**

保证Snapshot-Read语义，即保证读请求能读到所有commit_ts < 读请求start_ts的数据即可。

如果Get请求的start_ts < Lock的start_ts，则一定也小于该Lock对应的commit请求的commit_ts，读直接返回即可，读的正确性没有影响。

如果Get请求start_ts > Lock的start_ts，此时按照Server收到请求的顺序，有两种情况：

1、先收到Lock请求后收到Get请求。这种情况下会等事务提交或者锁resolve。会等冲突的Key的最终结果，读的正确性没有问题。

2、先收到Get请求再收到Lock请求。由于Get请求没处理之前Lock请求不会处理，而且Lock请求处理完成之后，才会获取commit_ts发起Commit请求，所以这种情况下commit_ts一定大于Get时的start_ts，所以Get请求直接返回当前最新数据，没有正确性问题。

综上所述Percolator事务模型的Snapshot-Read语义正确性没有问题。
