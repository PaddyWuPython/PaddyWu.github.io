---
layout: post
title: 分布式事务和数据一致性的案例
categories: Transaction
description: 分布式事务和数据一致性的案例说明
keywords: distributed, Transaction, mysql
---

在海量数据的场景下，我需要对数据库做拆分，即分库分表。这种情况下，如何保证数据的一致性呢？

下面我们举一个下订单的业务场景。

我们先做如下的假设：总共有3个实体，用户、商品、订单，我们按照 user_id 来 sharding。相同 user_id 的用户和订单在同一个物理库下，而商品表在另一个物理库下。

该业务场景，主要涉及到两个事务操作，扣减库存和生成订单，因为两个操作涉及不同节点的数据库，所以无法保证强一致性。


## 本地消息表，实现最终一致性
我们可以通过本地消息表，来实现最终一致性，具体流程如下图：

![](/images/posts/transition/transaction-mq-caseone.png)

### 流程说明：

#### 1、生成全局唯一的交易流水号 trans_id。
通过这个唯一 id， 方便后续做关联查询。

#### 2、事务相关的处理

**事务一:有两个数据库操作。** 1) 扣减库存 2) 根据流水单号，生成对应商品的冻结记录。**
因为消息表和商品表在同一个物理库下，所以扣减库存以及对应商品的冻结记录表可以构成事务操作。

如果事务一成功，则进行事务二；如果事务一失败，则直接返回。

**事务二：根据交易流水号 trans_id 生成订单，订单的状态有三种：未支付、已支付、超时，订单的初始状态为未支付。若订单创建成功，则进行后续的支付流程。**


注意：

如果事务二失败，由于网络抖动超时等原因，不一定是真的生成订单失败，即 在事务二失败的情况下，可能生成了订单，也可能确实没有生成订单。


#### 3、定时任务的职责

*两个定时任务，他们的职责分别是：*

**定时任务一：**

设置一个每隔 n 分钟（比如15分钟）的定时任务（即一个订单必须在 n 分钟内完成支付），从订单表里捞出最近半小时内的所有订单，对每一个订单做如下处理。
- 若订单超时未支付，开启事务 SELECT FOR UPDATE 锁住该订单，即用悲观锁阻止用户对订单进行支付等操作
- 然后通过订单的 trans_id 去**冻结表**更新对应冻结记录的状态，置为释放未售出，并回滚商品数量，回滚商品的操作完成后，将订单状态置为超时。
- 若事务中调用的回滚商品数量的服务失败，则可以发出报警人工处理，或通过更长时间的定时任务去处理。
- 若订单为已支付，则将冻结表中记录的状态置为释放已售出。


**定时任务二：**

因为存在事务一成功，而事务二的订单确实没有创建成功的情况，这样会冻结一部分商品的数量，所以可以捞取出创建超过 10 分钟状态为已冻结的所有冻结记录，根据每个冻结记录的 trans_id 去订单表查询，若不存在对应的订单，则将冻结记录的状态更新为释放未售出，并回滚商品数量。