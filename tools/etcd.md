---
date: 2022-07-12T09:44:28+08:00
author: "Rustle Karl"

title: "Etcd 架构与实现解析"
url:  "posts/k8s/tools/etcd"  # 永久链接
tags: [ "K8S", "README" ]  # 标签
categories: [ "K8S 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

**Etcd** 按照官方介绍

> Etcd is a distributed, consistent **key-value** store for **shared configuration** and **service discovery**

是一个分布式的，一致的 key-value 存储，主要用途是共享配置和服务发现。Etcd 已经在很多分布式系统中得到广泛的使用，本文的架构与实现部分主要解答以下问题：

1. Etcd是如何实现一致性的？
2. Etcd的存储是如何实现的？
3. Etcd的watch机制是如何实现的？
4. Etcd的key过期机制是如何实现的？

## 为什么需要 Etcd ？

所有的分布式系统，都面临的一个问题是多个节点之间的数据共享问题，这个和团队协作的道理是一样的，成员可以分头干活，但总是需要共享一些必须的信息，比如谁是 leader， 都有哪些成员，依赖任务之间的顺序协调等。所以分布式系统要么自己实现一个可靠的共享存储来同步信息（比如 Elasticsearch ），要么依赖一个可靠的共享存储服务，而 Etcd 就是这样一个服务。

## Etcd 提供什么能力？

Etcd 主要提供以下能力。

1. 提供存储以及获取数据的接口，它通过协议保证 Etcd 集群中的多个节点数据的强一致性。用于存储元信息以及共享配置。
2. 提供监听机制，客户端可以监听某个 key 或者某些 key 的变更（v2 和 v3 的机制不同，参看后面文章）。用于监听和推送变更。
3. 提供 key 的过期以及续约机制，客户端通过定时刷新来实现续约（v2 和 v3 的实现机制也不一样）。用于集群监控以及服务注册发现。
4. 提供原子的 CAS（Compare-and-Swap）和 CAD（Compare-and-Delete）支持（v2 通过接口参数实现，v3 通过批量事务实现）。用于分布式锁以及 leader 选举。

## Etcd 如何实现一致性的？

1. raft 通过对不同的场景（选主，日志复制）设计不同的机制，虽然降低了通用性（相对 paxos），但同时也降低了复杂度，便于理解和实现。
2. raft 内置的选主协议是给自己用的，用于选出主节点，理解 raft 的选主机制的关键在于理解 raft 的时钟周期以及超时机制。
3. 理解 Etcd 的数据同步的关键在于理解 raft 的日志同步机制。

Etcd 实现 raft 的时候，充分利用了 go 语言 CSP 并发模型和 chan 的魔法，想更进行一步了解的可以去看源码，这里只简单分析下它的 wal 日志。

![etcdv3](http://dd-static.jd.com/ddimg/jfs/t1/48897/7/19823/2019/62ccd22cE43559e1d/a56fc9f4bd8f3924.png)

wal 日志是二进制的，解析出来后是以上数据结构 LogEntry。其中第一个字段 type，只有两种，一种是 0 表示 Normal，1 表示 ConfChange（ConfChange 表示 Etcd 本身的配置变更同步，比如有新的节点加入等）。第二个字段是 term，每个 term 代表一个主节点的任期，每次主节点变更 term 就会变化。第三个字段是 index，这个序号是严格有序递增的，代表变更序号。第四个字段是二进制的 data，将 raft request 对象的 pb 结构整个保存下。Etcd 源码下有个 tools/etcd-dump-logs，可以将 wal 日志 dump 成文本查看，可以协助分析 raft 协议。

raft 协议本身不关心应用数据，也就是 data 中的部分，一致性都通过同步 wal 日志来实现，每个节点将从主节点收到的 data apply 到本地的存储，raft 只关心日志的同步状态，如果本地存储实现的有 bug，比如没有正确的将 data apply 到本地，也可能会导致数据不一致。

### Etcd v2 与 v3

Etcd v2 和 v3 本质上是共享同一套 raft 协议代码的两个独立的应用，接口不一样，存储不一样，数据互相隔离。也就是说如果从 Etcd v2 升级到 Etcd v3，原来 v2 的数据还是只能用 v2 的接口访问，v3 的接口创建的数据也只能访问通过 v3 的接口访问。所以我们按照 v2 和 v3 分别分析。

## Etcd v2 存储，Watch以及过期机制

![etcdv2](http://dd-static.jd.com/ddimg/jfs/t1/20348/29/16761/25204/62ccd22dE39e78b05/e39019e16516cd87.png)

Etcd v2 是个纯内存的实现，并未实时将数据写入到磁盘，持久化机制很简单，就是将 store 整合序列化成 json 写入文件。数据在内存中是一个简单的树结构。比如以下数据存储到 Etcd 中的结构就如图所示。

```
/nodes/1/name  node1  
/nodes/1/ip    192.168.1.1 
```

store 中有一个全局的 currentIndex，每次变更，index 会加 1. 然后每个 event 都会关联到 currentIndex.

当客户端调用 watch 接口（参数中增加 wait 参数）时，如果请求参数中有 waitIndex，并且 waitIndex 小于 currentIndex，则从 EventHistroy 表中查询 index 小于等于 waitIndex，并且和 watch key 匹配的 event，如果有数据，则直接返回。如果历史表中没有或者请求没有带 waitIndex，则放入 WatchHub 中，每个 key 会关联一个 watcher 列表。当有变更操作时，变更生成的 event 会放入 EventHistroy 表中，同时通知和该 key 相关的 watcher。

这里有几个影响使用的细节问题：

1. EventHistroy 是有长度限制的，最长 1000。也就是说，如果你的客户端停了许久，然后重新 watch 的时候，可能和该 waitIndex 相关的 event 已经被淘汰了，这种情况下会丢失变更。
2. 如果通知 watch 的时候，出现了阻塞（每个 watch 的 channel 有 100 个缓冲空间），Etcd 会直接把 watcher 删除，也就是会导致 wait 请求的连接中断，客户端需要重新连接。
3. Etcd store 的每个 node 中都保存了过期时间，通过定时机制进行清理。

从而可以看出，Etcd v2 的一些限制：

1. 过期时间只能设置到每个 key 上，如果多个 key 要保证生命周期一致则比较困难。
2. watch 只能 watch 某一个 key 以及其子节点（通过参数 recursive),不能进行多个 watch。
3. 很难通过 watch 机制来实现完整的数据同步（有丢失变更的风险），所以当前的大多数使用方式是通过 watch 得知变更，然后通过 get 重新获取数据，并不完全依赖于 watch 的变更 event。

## Etcd v3 存储，Watch 以及过期机制

![etcdv3](http://dd-static.jd.com/ddimg/jfs/t1/151226/15/25372/23497/62ccd22dE98b7d8e1/18ca4cac711a3961.png)

Etcd v3 将 watch 和 store 拆开实现，我们先分析下 store 的实现。

Etcd v3 store 分为两部分，一部分是内存中的索引，kvindex，是基于 google 开源的一个 golang 的 btree 实现的，另外一部分是后端存储。按照它的设计，backend 可以对接多种存储，当前使用的 boltdb。boltdb 是一个单机的支持事务的 kv 存储，Etcd 的事务是基于 boltdb 的事务实现的。Etcd 在 boltdb 中存储的 key 是 reversion，value 是 Etcd 自己的 key-value 组合，也就是说 Etcd 会在 boltdb 中把每个版本都保存下，从而实现了多版本机制。

举个例子： 用 etcdctl 通过批量接口写入两条记录：

```
etcdctl txn <<<' 
put key1 "v1" 
put key2 "v2" 

' 
```

再通过批量接口更新这两条记录：

```
etcdctl txn <<<' 
put key1 "v12" 
put key2 "v22" 

' 
```

boltdb中其实有了4条数据：

```
rev={3 0}, key=key1, value="v1" 
rev={3 1}, key=key2, value="v2" 
rev={4 0}, key=key1, value="v12" 
rev={4 1}, key=key2, value="v22" 
```

reversion 主要由两部分组成，第一部分 main rev，每次事务进行加一，第二部分 sub rev，同一个事务中的每次操作加一。如上示例，第一次操作的 main rev 是 3，第二次是 4。当然这种机制大家想到的第一个问题就是空间问题，所以 Etcd 提供了命令和设置选项来控制 compact，同时支持 put 操作的参数来精确控制某个 key 的历史版本数。

了解了 Etcd 的磁盘存储，可以看出如果要从 boltdb 中查询数据，必须通过 reversion，但客户端都是通过 key 来查询 value，所以 Etcd 的内存 kvindex 保存的就是 key 和 reversion 之前的映射关系，用来加速查询。

然后我们再分析下 watch 机制的实现。Etcd v3 的 watch 机制支持 watch 某个固定的 key，也支持 watch 一个范围（可以用于模拟目录的结构的 watch），所以 watchGroup 包含两种 watcher，一种是 key watchers，数据结构是每个 key 对应一组 watcher，另外一种是 range watchers, 数据结构是一个 IntervalTree（不熟悉的参看文文末链接），方便通过区间查找到对应的 watcher。

同时，每个 WatchableStore 包含两种 watcherGroup，一种是 synced，一种是 unsynced，前者表示该 group 的 watcher 数据都已经同步完毕，在等待新的变更，后者表示该 group 的 watcher 数据同步落后于当前最新变更，还在追赶。

当 Etcd 收到客户端的 watch 请求，如果请求携带了 revision 参数，则比较请求的 revision 和 store 当前的 revision，如果大于当前 revision，则放入 synced 组中，否则放入 unsynced 组。同时 Etcd 会启动一个后台的 goroutine 持续同步 unsynced 的 watcher，然后将其迁移到 synced 组。也就是这种机制下，Etcd v3 支持从任意版本开始 watch，没有 v2 的 1000 条历史 event 表限制的问题（当然这是指没有 compact 的情况下）。

另外我们前面提到的，Etcd v2 在通知客户端时，如果网络不好或者客户端读取比较慢，发生了阻塞，则会直接关闭当前连接，客户端需要重新发起请求。Etcd v3 为了解决这个问题，专门维护了一个推送时阻塞的 watcher 队列，在另外的 goroutine 里进行重试。

Etcd v3 对过期机制也做了改进，过期时间设置在 lease 上，然后 key 和 lease 关联。这样可以实现多个 key 关联同一个 lease id，方便设置统一的过期时间，以及实现批量续约。

相比 Etcd v2, Etcd v3 的一些主要变化：

1. 接口通过 grpc 提供 rpc 接口，放弃了 v2 的 http 接口。优势是长连接效率提升明显，缺点是使用不如以前方便，尤其对不方便维护长连接的场景。
2. 废弃了原来的目录结构，变成了纯粹的 kv，用户可以通过前缀匹配模式模拟目录。
3. 内存中不再保存 value，同样的内存可以支持存储更多的 key。
4. watch 机制更稳定，基本上可以通过 watch 机制实现数据的完全同步。
5. 提供了批量操作以及事务机制，用户可以通过批量事务请求来实现 Etcd v2 的 CAS 机制（批量事务支持 if 条件判断）。

## Etcd，Zookeeper，Consul 比较

这三个产品是经常被人拿来做选型比较的。 Etcd 和 Zookeeper 提供的能力非常相似，都是通用的一致性元信息存储，都提供 watch 机制用于变更通知和分发，也都被分布式系统用来作为共享信息存储，在软件生态中所处的位置也几乎是一样的，可以互相替代的。二者除了实现细节，语言，一致性协议上的区别，最大的区别在周边生态圈。

Zookeeper 是 apache 下的，用 java 写的，提供 rpc 接口，最早从 hadoop 项目中孵化出来，在分布式系统中得到广泛使用（hadoop, solr, kafka, mesos 等）。

Etcd 是 coreos 公司旗下的开源产品，比较新，以其简单好用的 rest 接口以及活跃的社区俘获了一批用户，在新的一些集群中得到使用（比如 kubernetes）。虽然 v3 为了性能也改成二进制 rpc 接口了，但其易用性上比 Zookeeper 还是好一些。

而 Consul 的目标则更为具体一些，Etcd 和 Zookeeper 提供的是分布式一致性存储能力，具体的业务场景需要用户自己实现，比如服务发现，比如配置变更。而 Consul 则以服务发现和配置变更为主要目标，同时附带了 kv 存储。在软件生态中，越抽象的组件适用范围越广，但同时对具体业务场景需求的满足上肯定有不足之处。

```shell

```

```shell

```
