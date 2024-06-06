# 微服务架构中你必须了解的 CAP 原理

发布于 2020-02-24 14:24:32

3K0

举报

文章被收录于专栏：[Java后端技术栈cwnait](https://cloud.tencent.com/developer/column/78716)

> **摘要**：什么是分布式 CAP 原理，什么是分区容错性，zookeeper 和 eureka 的 CAP 区别是什么，还有作为公司的架构师你们是怎么做的，那些分布式系统设计成了 CP、AP，为什么这样设计等等一系列问题。

###### **分布式系统 CAP 到底指什么**

- `C（Consistency）`：一致性，即数据一致性，特指分布式系统中的数据一致性。
- `A（Availability）`：可用性，即服务的高可用，特指分布式系统中服务的高可用，某个服务瘫痪不影响整个分布式系统的正常运行。
- `P（Partition Tolerance）`：分区容错性（也有的叫分区耐受性），即网络故障，特指分布式系统中服务之间出现了网络故障，整个分布式系统仍然保持可用性和一致性。

![img](https://ask.qcloudimg.com/http-save/yehe-4143945/qgnufc0cf4.png)

一句话概括 `CAP`：**在分布式系统中，网络故障，服务瘫痪，整个系统的数据仍然保持一致性**。

上面的表述可能不容易理解，举一个通俗的例子讲讲什么是分布式 CAP 原理。

![img](https://ask.qcloudimg.com/http-save/yehe-4143945/qld3iwnjkk.jpeg)

###### **大白话描述案例：**

如上图所示，小张在京东商城准备购买几本微服务实战相关的书籍，订单总金额共 200 元，他刚好有一张满 200 减 100 元的优惠劵。 

这时分 3 种情况说明分布式系统的 CAP 如下：

1. ```
   数据一致性体现
   ```

   ：

   - 使用 100 元优惠劵扣减，实付 100 元，优惠劵用完。
   - 使用 100 元优惠劵失败，实付 200 元，优惠劵还在。

2. ```
   系统可用性体现
   ```

   ：

   - 小张下订单时，订单服务或是 PLUS 会员服务挂了，此时不应该影响小张下单。

3. ```
   系统分区容错性体现
   ```

   ：

   - 小张下订单时，订单服务和 PLUS 会员服务之间网络不通也不能影响小张下单。

###### **什么是分区容错性**

CAP 定理中最难理解的概念是 P，分区容错性，画个图大家就理解了。

![img](https://ask.qcloudimg.com/http-save/yehe-4143945/8u5r2lvyrp.png)

在断网的情况下，2 台服务器变成了独立网络，彼此无法通信，这个情况就是分区。在分区的情况下，分布式系统如果要保证数据一致性和可用性的话，那就满足分区容错性了。

###### **CAP 技术实现的难度，下面一一解惑**

目前大部分互联网企业都是微服务架构，即分布式系统。  现在某电商微服务架构，假设出现网络故障（P），服务挂掉（A），整个系统的数据仍然保持一致。这是无法做到的。

相对可以实现的方案：  业界的做法是 **CAP 三选其二**，即

- `CP`
- `AP`
- `CA`（请思考？后面叙述）

当服务之间出现网络故障的情况下：

![img](https://ask.qcloudimg.com/http-save/yehe-4143945/4id408rad6.jpeg)

问题：

1. 如何保证订单服务和 PLUS 会员服务高可用？
2. 下订单同时扣除 100 元优惠劵如何实现？

分布式系统的解决方案：

1. ```
   CAP 牺牲一致性（AP）
   ```

   ：保证高可用，即保证订单服务可以正常访问，保证 PLUS 会员服务可以正常访问，牺牲了数据的一致性。 小张去京东商城下订单（但是没有扣除 100 元优惠劵），这种情况下，小张订单提交成功后，再去查看 100 元优惠劵还在，居然没有扣除成功，但是实付金额是 100 元，很纳闷（心里窃喜）。

   - 怎么办呢？如何解决这种问题？  一般的做法是，当网络恢复正常的情况下，订单服务重试请求 PLUS 会员服务，再扣除 100 元优惠劵。

2. ```
   CAP 牺牲可用性（CP）
   ```

   ：保证数据一致性。 当小张去京东商城下单时，提示：“网络异常，请稍后再试”。

   - 怎么办呢？如何解决这种问题？  只能等网络恢复正常后，小张才可以成功下单。

3. `CAP 牺牲分区容错性（CA）`：不要P分区，即不允许出现网络故障，这是**不可能实现**的。  所以在分布式系统中，是不存在 CA 的。即使传统单体系统也做不到CA，因为单体系统也会出现单一故障。

通过小张在京东商城下订单的案例，小伙伴应该都弄明白了吧。

###### **下面分析两个微服务架构中常用的服务注册中心 —— zookeeper & eureka 的 CAP 原理**

**1）图解 zookeeper 的 CAP 原理**  *注：此处不介绍 zookeeper 底层原理和实现*  zookeeper 作为[微服务注册中心](https://cloud.tencent.com/product/src?from_column=20065&from=20065)是 CP 原理，即保证了数据的一致性，牺牲了可用性。如下图所示：

![img](https://ask.qcloudimg.com/http-save/yehe-4143945/tcarqsh4fg.png)

- zookeeper 的数据同步原理：  Client1 客户端向 zk Server1 注册，zk Server1 同步信息给 zk Server2，zk Server2 是[注册中心](https://cloud.tencent.com/product/tse?from_column=20065&from=20065)的 leader 节点，该节点负责把消息广播同步给其他 follower 的 zk 节点，为了保证数据的一致性，只有等整个注册中心信息同步完成，Client1客户端才能收到注册成功的消息。

![img](https://ask.qcloudimg.com/http-save/yehe-4143945/6btg2nsyh0.png)

- 如上图所示，当注册中心 leader 重启或是出现网络故障的情况下，整个 zk 集群重新选举 leader 节点，**在选举期间，Client 客户端无法注册**，即此时 zookeeper 服务不可用，所以牺牲了系统的可用性。

![img](https://ask.qcloudimg.com/http-save/yehe-4143945/wdt34ubeht.png)

- 如上图所示，只有等到整个系统选举出 leader 节点，系统才能恢复注册，故 zookeeper 为了保证数据一致性，牺牲了系统的可用性。
- **CP 有一个致命的缺点，就是在大型分布式系统中，网络非常复杂，leader 节点出现故障的频率特别高，而且很容易引起雪崩。所以这是很多大型分布式系统都不选择 zookeeper 作为注册中心的原因**。

**2）图解 Eureka 的 CAP 原理**  如下图所示：

![img](https://ask.qcloudimg.com/http-save/yehe-4143945/4wc5e6ejbv.png)

- eureka 是 AP 原理，即保证了系统的可用性，却牺牲了系统的一致性。
- eureka 的数据同步原理：  第一步，Client1 客户端注册到 eureka Server1 服务中；  第二步，eureka Server1 直接告诉 Client1 注册成功。  第三步，eureka Server1 把 Client1 的注册信息同步给 Server2，为了保证服务的可用性，eureka Server 之间是异步同步的。

###### **小结**

通过以上的案例描述和图形解读，相信大家对于微服务（分布式系统）架构中 CAP 原理有了一定的了解。比如知道了什么是 CAP 原理，什么是分区容错性，zookeeper 和 eureka 作为注册中心的 CAP 区别是什么。同时希望对今后你们公司的系统架构设计有所帮助，系统设计是遵循 CP 还是 AP。

本文参与 [腾讯云自媒体同步曝光计划](https://cloud.tencent.com/developer/support-plan)，分享自微信公众号。

原始发表：2020-02-19，如有侵权请联系 [cloudcommunity@tencent.com](mailto:cloudcommunity@tencent.com) 删除