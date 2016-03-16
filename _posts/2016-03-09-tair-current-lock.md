---
layout: post
title: 记一次处理线上分布式环境下的并发问题
category : 技术分享
tagline: "Supporting tagline"
tags : [并发]
---

#### 业务处理逻辑
1. 应用接收淘客的notify消息，应用集群同时进行处理将notify消息保存到order_detail表中；

2. 其中一台应用服务器读取order_detail表记录发送到metaq中;

3. 应用集群接收metaq的消息进行订单的幂等性存储到cps_order表中，这里先根据订单select，如果不存在则insert，否则update订单状态（订单号唯一索引）

4. 应用是通过spring aop进行的事务处理，表达式：
````
<aop:config proxy-target-class="true">
    <aop:pointcut id="transactionPointcut"
      expression="execution(* com.taobao.wangcai..bo..*.*(..))" />
    <aop:advisor advice-ref="txAdvice" pointcut-ref="transactionPointcut" />
  </aop:config>
````
bo下的类都会被事务管理。

#### 代码中的实现逻辑
OrderDetailMessageListener 是consumer消息监听器调用代码
````
private boolean dealWithMessage(Message message) {
    OrderDetailDO orderDetail = null;
    try {
      if (TOPIC.equals(message.getTopic())) {
        String jsonStr = new String(message.getBody(), "UTF-8");
        // 获取消息并解析成orderDetail对象
        orderDetail = JSON.parseObject(jsonStr, new TypeReference<OrderDetailDO>() {
        });
        // 消息处理，将orderdetail对象转换成cpsorder对象
        orderProcessBO.process(orderDetail);
      }
    } catch (Exception e) {
      logger.error("order detail process fail", e);
      return false;
    }
    return true;
  }
````

OrderProcessBO的process方法，省略了部分代码只有存储订单的逻辑
````
public void process(OrderDetailDO orderDetailDO) {
    try {
      boolean isSuccess = persistCpsOrder(cpsOrderDO, appkey);
    } catch (Throwable t) {
      logger.error("order detail process error.");
      throw new RuntimeException(t);// 事务控制在bo中 try catch后需要抛出
    }
  }
````
persistCpsOrder 这个方法里是调用了GeneralCpsOrderPersist实现类，关键代码如下：
````
public boolean persist(CpsOrderDO cpsOrderDO) throws Exception {
    try {
      CpsOrderDO existedCpsOrder = cpsOrderBO.queryCpsOrderByOrderId(cpsOrderDO.getOutsideOrderId(), cpsOrderDO.getSplitCol());
      if (existedCpsOrder == null) {// 如果不存在cpsorder记录则直接插入
        cpsOrderBO.insert(cpsOrderDO);
        logger.info("insert cpsorder:{}", cpsOrderDO);
      } else {
      // 存在订单则比较订单状态，如果是后状态才进行更新，否则不处理
        if (!cpsOrderBO.updateCpsOrder(cpsOrderDO)) {// 如果更新失败，返回false标记重试
            logger.info("cps order update no record:{}", cpsOrderDO.getOutsideOrderId());
            return false;
          }
        }
      }
      return true;
    } catch (Exception e) {
      logger.error("cpsorder persist error orderid:{}", cpsOrderDO.getOutsideOrderId(), e);
      throw new Exception(e);
    }
  }
````

#### 出现的问题

1. 并发的时候会有两个线程同时到select 都没查询到订单，都去做insert操作，这个时候会有一个唯一约束失败，抛出异常标记Orderdetail重试并且返回metaq false重发消息；

2. 发现了两个update语句，和insert语句但是数据库里没有记录。

#### 问题排查

查了下数据库的事务
````
SELECT @@global.tx_isolation;
SELECT @@session.tx_isolation;
SELECT @@tx_isolation;
````
全部都是 READ-COMMITTED 提交读和mysql 默认的可重复读（REPEATABLE READ）低一个级别 就是有可能发生，这个类型会导致，不可重复读，幻读。
> 不重复读: 解决了脏读后，会遇到，同一个事务执行过程中，另外一个事务提交了新数据，因此本事务先后两次读到的数据结果会不一致。
> 幻读: 解决了不重复读，保证了同一个事务里，查询的结果都是事务开始时的状态（一致性）。但是，如果另一个事务同时提交了新数据，本事务再更新时，就会“惊奇的”发现了这些新数据，貌似之前读到的数据是“鬼影”一样的幻觉。

备注：mysql 默认的事务隔离级别是无法消除幻读的

因此两个update和insert 但是数据库没有该记录，就有可能发生了幻读的问题。

#### 解决问题
由于是分布式环境，同一个订单多个消息（重复的|不同类型的）都可能落到两个机器上并发执行才会造成上面的问题。这里想到的是用tair实现一个并发锁来解决，还可以用metaq的消息对列队订单hash，然后同一个订单到可以实现串行的效果，这个还没试验。

第一次是这样实现的
````
public boolean persist(CpsOrderDO cpsOrderDO) throws Exception {
    if (tariClient.tryLock(cpsOrderDO.getOutsideOrderId())){
        try {
          CpsOrderDO existedCpsOrder = cpsOrderBO.queryCpsOrderByOrderId(cpsOrderDO.getOutsideOrderId(), cpsOrderDO.getSplitCol());
          if (existedCpsOrder == null) {// 如果不存在cpsorder记录则直接插入
            cpsOrderBO.insert(cpsOrderDO);
            logger.info("insert cpsorder:{}", cpsOrderDO);
          } else {
          // 存在订单则比较订单状态，如果是后状态才进行更新，否则不处理
            if (!cpsOrderBO.updateCpsOrder(cpsOrderDO)) {// 如果更新失败，返回false标记重试
                logger.info("cps order update no record:{}", cpsOrderDO.getOutsideOrderId());
                return false;
              }
            }
          }
          return true;
        } catch (Exception e) {
          logger.error("cpsorder persist error orderid:{}", cpsOrderDO.getOutsideOrderId(), e);
          throw new Exception(e);
        }
    }
    tariClient.releaseLock(cpsOrderDO.getOutsideOrderId());
}
````
问题依旧发生，因为事务控制在OrderProcessBO 类里 而锁是加在了OrderProcessBO他的某个调用类的某个方法上，比如一个订单的两个消息过来，如果是并发的线程其中一个获取锁失败就会重试这个是没有问题的，问题出现在两个线程过来的间歇非常小的情况比如

1. 线程1、线程2 线程1首先获取锁执行查询没有该订单，则执行insert这个时候他跳出这个方法，但还没提交事务，cpu调度切换到线程2，线程2也回去取查询没有该笔订单进入if (existedCpsOrder == null) 这个语句块后，cpu调度切换到线程1，他开始提交事务，这个时候线程2继续执行insert 就会产生唯一索引冲突的异常信息；
2. 幻读的情况也是一样的由于这个事务比较长，而锁的力度是在事务的某个环境所以出现这样的问题。

#### 继续解决问题
最后在风闲的帮助下解决了这个问题
就是在OrderDetailMessageListener 这个类里调用事务的时候来加订单锁进行控制
````
private boolean dealWithMessage(Message message) {
    OrderDetailDO orderDetail = null;
    try {
      if (TOPIC.equals(message.getTopic())) {
        String jsonStr = new String(message.getBody(), "UTF-8");
        // 获取消息并解析成orderDetail对象
        orderDetail = JSON.parseObject(jsonStr, new TypeReference<OrderDetailDO>() {
        });
        // 消息处理，将orderdetail对象转换成cpsorder对象
        if(tairClient.tryLock(orderDetail.getOutsideOrderId())){
          orderProcessBO.process(orderDetail);
        }
        tariClient.releaseLock(cpsOrderDO.getOutsideOrderId());
      }
    } catch (Exception e) {
      logger.error("order detail process fail", e);
      return false;
    }
    return true;
  }
````

#### 收获经验
1. 这种并发的问题需要仔细思考，不是简简单单的加个锁就可以解决的，以前遇到的情况加锁解决了也许是运气锁加的地方一下就蒙对了，对这个没有过多的思考，遇到并发和锁的问题一定要细心分析问题；

2. 遇到线上问题一定要冷静思考，更不要陷入局部，考虑周全，这一点说的容易做起来比较难啊，来阿里半年了，是因为平台大，出现线上问题就会比较严重的原因吗，为什么遇到问题就紧张起来了呢，自己一下就乱了。做了开发8年了遇到线上问题也应该是无数了，以前没发现还有这毛病 真愁人；

3. 最近总是精神恍惚，有时候别人说一个事我好想都没听见完全走神了，遇到问题老是纠缠不清，大脑反应迟钝无比，在别人说话的时候这种感受就好像是个傻X一样，遇到这个大姨妈期间我觉得最好是放松一下出去透透风；
