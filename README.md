# IdempotencyTree
分布式系统的幂等性设计

<pre>
分布式系统的接口幂等性
       单体架构转成微服务架构之后带来的幂等性问题，当然不是说单体架构下没有这些问题，在
   单体架构下同样要避免重复请求，但是出现问题要比分布式环境下少得多。
       接口的幂等性实际上就是接口可以重复调用，在调用方多次调用的情况下，接口最终得到的结
   果是一致的。有些接口可以天然的实现幂等性，比如查询接口，对于查询来说，查询一次，两次对于
   系统来说，没有任何影响，查出的结果也是一样。
       除了查询功能具有天然的幂等性之外，增加，更新，删除等都要保证幂等性。
</pre>

<pre>
全局唯一ID
       根据业务的操作和内容生成一个全局ID，在执行操作前先根据这个全局唯一ID是否存在，来
   判断这个操作是否已经被执行。如果不存在则把全局ID存储到存储系统中，比如数据库，redis等
   如果存在则表示该方法已经执行。 
       从工程的角度来说，使用全局ID做幂等性可以作为一个业务的基础的服务存在，在很多的
   微服务中都会用到这样的服务，在每个微服务中都完成这样的功能，会存在工作量重复，另外打造
   一个高可靠的幂等性服务还需要考虑很多问题，比如一台机器虽然把全局ID先写入了存储，但是在
   写入后挂了，这就需要引入全局ID的超时机制。
       使用全局ID是一个通用方案，可以支持插入，更新，删除等业务。但是这个方案看起来很美，
   但是实现起来比较麻烦。 
</pre>

<pre>
去重表
      这种方法适用于在业务中有唯一标的的插入场景，比如在以上的支付场景中，如果一个订单只会
  支付一次，所以订单ID可以作为唯一标识。这时，我们就可以建一张去重表，并且把唯一标识作为索引。在我们的实现中，把创建支付单据和写入去重表放在一个事务中，如果重复创建，数据库会抛出
  唯一约束异常，操作就会回滚。
</pre>

<pre>
插入或更新
      这种方法插入并且有唯一索引的情况，比如我们要关联商品品类，其中商品的ID和品类的ID
  可以构成唯一索引，并且哎数据表中也增加了唯一索引。这时就可以使用InsertOrUpdate操作。

      insert into goods_category (goods_id,category_id,create_time,update_time)
      values(#{goodsId},#{categoryId},now(),now())
      on DUPLICATE KEY UPDATE
      update_time=now()
</pre>

<pre>
多版本控制
      这种方法适合在更新的场景中，比如我们需要更新商品的名字，这时我们就可以在更新的接口
  中增加一个版本号来做幂等性。
      update goods set name=#{newName},version=#{version} where id=#{id} and version<${version}
</pre>

<pre>
状态机控制
      这种方法适合在有状态机流转的情况下，比如订单的创建和付款，订单的付款肯定是在前，这时
  我们可以通过在设计状态字段时，使用int类型，并且通过值类型的大小来做幂等性，比如订单的
  创建为0，付款成功为100，付款失败为99

     update `order` set status=#{status} where id=#{id} and status<#{status}
</pre>
