Redis为什么能作为消息队列使用？
redis有五种基本类型 分别为

String
List
Hash
Set
ZSet
其中List类型是一个双向链表，自然就可以当作普通的队列Queue来使用了，再加之Redis本身就是一个中间件，所以就可以使用list类型完成一个简易的消息队列了。

Redis与RabbitMq或者其他MQ有什么异同
RabbitMq Kafa等一系列都是专门的消息队列中间件，相比Redis 提供了更丰富的功能，如交换机、绑定、死信队列、生产或消费时的确认机制。
而Redis除了可以用lua脚本实现“事务”、以及设置过期时间以外没有什么其他扩展功能，也没有RabbitMq中的Confirm消息确认机制 难保消息不丢失。

那RabbitMQ等功能如此齐全为什么还要用Redis来做消息队列呢
一个字 快 。
Redis每秒十万读写这是其他消息队列中间件很难匹敌的，而且对与一些简单的业务需求，使用RabbitMQ等一些复杂的功能也降低了整体架构的可用性。
