# **redis 分布式锁的 5个坑**

> 参考: http://www.chengxy-nds.top/2020/01/26/redis%20%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E7%9A%84%205%E4%B8%AA%E5%9D%91%EF%BC%8C%E7%9C%9F%E6%98%AF%E5%8F%88%E5%A4%A7%E5%8F%88%E6%B7%B1/

## **I. 锁未被释放**

这种情况是一种低级错误，由于当前线程获取到 redis 锁，处理完业务后未及时释放锁，导致其它线程会一直尝试获取锁阻塞，例如：用 Jedis 客户端会报如下的错误信息：

```java
redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource from the pool
```

redis 线程池已经没有空闲线程来处理客户端命令。

解决的方法也很简单，只要我们细心一点，拿到锁的线程处理完业务及时释放锁，如果是重入锁未拿到锁后，线程可以释放当前连接并且 sleep 一段时间。

```java
public void lock() {
      while (true) {
          boolean flag = this.getLock(key);
          if (flag) {
                TODO .........
          } else {
                // 释放当前redis连接
                redis.close();
                // 休眠1000毫秒
                sleep(1000);
          }
        }
    }
```

## **II. B 的锁被A 给释放了**

我们知道 Redis 实现锁的原理在于 SETNX 命令。当 key 不存在时将 key 的值设为 value ，返回值为 1；若给定的 key 已经存在，则 SETNX 不做任何动作，返回值为 0。

```bash
SETNX key value
```

我们来设想一下这个场景：A、B 两个线程来尝试给 key myLock 加锁，A 线程先拿到锁（假如锁 3 秒后过期），B 线程就在等待尝试获取锁，到这一点毛病没有。

那如果此时业务逻辑比较耗时，执行时间已经超过 redis 锁过期时间，这时A线程的锁自动释放（删除 key），B 线程检测到 myLock 这个 key 不存在，执行 SETNX 命令也拿到了锁。

但是，此时 A 线程执行完业务逻辑之后，还是会去释放锁（删除 key），这就导致 B 线程的锁被 A 线程给释放了。

为避免上边的情况，一般我们在每个线程加锁时要带上自己独有的 value 值来标识，只释放指定 value 的 key，否则就会出现释放锁混乱的场景。

## **III. 数据库事务超时**

```java
@Transaction
   public void lock() {

    while (true) {
        boolean flag = this.getLock(key);
        if (flag) {
            insert();
        }
    }
}
```

给这个方法添加一个 ```@Transaction``` 注解开启事务，如代码中抛出异常进行回滚，要知道数据库事务可是有超时时间限制的，并不会无条件的一直等一个耗时的数据库操作。

比如：我们解析一个大文件，再将数据存入到数据库，如果执行时间太长，就会导致事务超时自动回滚。

一旦你的 key 长时间获取不到锁，获取锁等待的时间远超过数据库事务超时时间，程序就会报异常。

一般为解决这种问题，我们就需要将数据库事务改为手动提交、回滚事务。

```java
@Transaction
public void lock() {
    //手动开启事务
    TransactionStatus transactionStatus = dataSourceTransactionManager.getTransaction(transactionDefinition);
    try {
        while (true) {
            boolean flag = this.getLock(key);
            if (flag) {
                insert();
                //手动提交事务
                dataSourceTransactionManager.commit(transactionStatus);
            }
        }
    } catch (Exception e) {
        //手动回滚事务
        dataSourceTransactionManager.rollback(transactionStatus);
    }
}
```

## **IV. 锁过期了，业务还没执行完**

这种情况和我们上边提到的第二种比较类似，但解决思路上略有不同。

同样是 redis 分布式锁过期，而业务逻辑没执行完的场景，不过，这里换一种思路想问题，把 redis 锁的过期时间再弄长点不就解决了吗？

那还是有问题，我们可以在加锁的时候，手动调长 redis 锁的过期时间，可这个时间多长合适？业务逻辑的执行时间是不可控的，调的过长又会影响操作性能。

## **V. redis主从复制的坑**

redis 高可用最常见的方案就是**主从复制**（master-slave），这种模式也给 redis 分布式锁挖了一坑。

redis cluster 集群环境下，假如现在A 客户端想要加锁，它会根据路由规则选择一台 master 节点写入 key mylock，在加锁成功后，master 节点会把 key 异步复制给对应的 slave 节点。

如果此时 redis master 节点宕机，为保证集群可用性，会进行主备切换，slave 变为了 redis master。B 客户端在新的 master 节点上加锁成功，而 A 客户端也以为自己还是成功加了锁的。

此时就会导致同一时间内多个客户端对一个分布式锁完成了加锁，导致各种脏数据的产生。

至于解决办法嘛，目前看还没有什么根治的方法，只能尽量保证机器的稳定性，减少发生此事件的概率。