---
layout: post
title:  "raft与zab和paxos的区别"
date:   2020-05-01 19:49:45 +0200
categories: 一致性 算法 分布式 raft zab paxos
---
英文论文地址：
[raft](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf) [zab](https://marcoserafini.github.io/papers/zab.pdf)
[paxos](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf)


之前写过一篇关于raft的文章，主要记录了raft里面一些值得注意的点，但是后来发现单看上述的论文，只是对raft算法一知半解（上述论文并没有提及很多实现上的优化），于是花了点时间拜读raft作者的[博士论文](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf),感觉对raft的理解又加深了很多。

本篇文章主要关注raft与zab和paxos(指multi paxos)之间的区别，由于算法的实现是在不断变化的，比如zab一开始有多个选主算法，但随着版本的变化，最后就只变成fastLeaderElection算法了，所以很多目前网上的区别，仅适用于某些版本，故本篇文章只打算做一些提取工作，来看看raft作者眼中这些算法的区别（我个人认为会比很多网上的一知半解权威很多），读者也可以直接看raft作者的[博士论文](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)的第11章。


- 领导人选举
  - 发现并消除失败的领导人
    - raft: 使用心跳及超时机制来发现无效的领导人
    - zab和paxos并没有详细提及
  - 选择新的领导人并确保它有所有已经提交的日志
    - raft: 只有拥有最新日志的服务器会被选择为新的领导人，领导人已拥有最新日志，无需更新
    - paxos: 任何服务器都可以被选择为新的领导人，在当选领导人之后，需要进行数据同步，同步方法为对每一条日志进行一轮paxos来获取正确的值，知道没有任何服务器知道当前index的提案，这种方法会导致相当大的延迟。
    - zab: 任何服务器都可以被选择为新的领导人，在当选领导人之后，需要进行数据同步，同步方法为非领导者发送自己的日志到领导者那边，由领导者来筛选形成最新日志，当日志很大时会是一个负担。
- 日志复制
  ![png1]({{ site.baseurl }}/assets/img/raft/raft-zab-paxos.png){: .center-image }
  如上所示，日志在索引为5的地方开始不一致(注意已知提交到索引3)
  - raft: 发送索引5及以后的日志(5-8)到跟随者，跟随者直接用领导者的覆盖自己，属于一次性覆盖，比较而言raft发现最少的数据
  - paxos: 从索引为4的日志条目开始(因为4还没有被提交)，对每一条目运行paxos,属于多次性覆盖，效率低下
  - zab:  发送领导者所有日志(1-8)到跟随者，跟随者用领导者的覆盖自己，属于一次性覆盖，但日志大了会有问题（实现时肯定需要一些优化）
- 日志提交
  - raft: 有序
  - paxos: 支持乱序
  - zab:  有序
- 集群配置更改
  - raft:
    - 推荐模式：一次只增加一个服务器或者删除一个服务器
    - 其他模式：也可以一次增加/删除多个服务器，需要两步，第一步先转到一个中间状态，使得老的配置和新的配置都满足中间状态（领导人同时拥有新旧配置的大部分选票），然后再转到最终状态（只有新的配置），但这种模式是不推荐的
  - paxos: 通过基础的paxos决议出新的集群配置，落在日志里，同时应用窗口延缓变更失效，防止大量领导人请求同一时间全都失败
  - zab: 和raft类似的两阶段，首先添加一条日志被新旧集群配置大部分认可，然后旧配置的领导人发送请求到新配置的所有服务器，选择出一个新的领导人。
- 日志压缩
  - raft和zab均使用快照模式，但略有不同，raft使用copy on write技术，快照是一致的，而zab使用的是fuzzy snapshot(模糊快照)，只是部分一致
- 状态机还是主从拷贝
![png2]({{ site.baseurl }}/assets/img/raft/sm-pca.png){: .center-image }
状态机是将操作利用一致性算法拷贝到不同的服务器，而主从拷贝是先将操作应用到领导者的状态机，再把状态变化通过一致性算法传到跟随者服务器
  - raft: 状态机
  - paxos: 状态机
  - zab: 主从拷贝
- 可理解性
  - raft: 最容易理解
  - zab: 一般容易理解
  - paxos: 有点难（对我来说是理解了又忘）
  - multi paxos: 相当难，各有各的理解
