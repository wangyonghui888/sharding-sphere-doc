+++
toc = true
title = "路由引擎"
weight = 2
+++

根据解析上下文匹配数据库和表的分片策略，并生成路由路径。
对于携带分片键的SQL，根据分片键的不同可以划分为单片路由(分片键的操作符是等号)、多片路由(分片键的操作符是IN)和范围路由(分片键的操作符是BETWEEN)。
不携带分片键的SQL则采用广播路由。

分片策略通常可以采用由数据库内置或由用户方配置。
数据库内置的方案较为简单，内置的分片策略大致可分为尾数取模、哈希、范围、标签、时间等。
由用户方配置的分片策略则更加灵活，可以根据使用方需求定制复合分片策略。
如果配合数据自动迁移来使用，可以做到无需用户关注分片策略，自动由数据库中间层分片和平衡数据即可，进而做到使分布式数据库具有的弹性伸缩的能力。
在Sharding-Sphere的线路规划中，弹性伸缩将于4.x开启。

## 分片路由

用于根据分片键进行路由的场景，又细分为直接路由、标准路由和笛卡尔积路由这3种类型。

满足直接路由的条件相对苛刻，它需要通过Hint（使用HintAPI直接指定路由至库表）方式分片，并且是只分库不分表的前提下，则可以避免SQL解析和之后的结果归并。
因此它的兼容性最好，可以执行包括子查询、自定义函数等复杂情况的任意SQL。直接路由还可以用于分片键不在SQL中的场景。

标准路由是Sharding-Sphere最为推荐使用的分片方式，它的适用范围是不包含关联查询或仅包含绑定表之间关联查询的SQL。
当分片运算符是等于号时，路由结果将落入单库（表），当分片运算符是BETWEEN或IN时，则路由结果不一定落入唯一的库（表），因此一条逻辑SQL最终可能被拆分为多条用于执行的真实SQL。
举例说明，如果按照order_id的奇数和偶数进行数据分片，一个单表查询的SQL如下：

```sql
SELECT * FROM t_order WHERE order_id IN (1, 2);
```

那么路由的结果应为：

```sql
SELECT * FROM t_order_0 WHERE order_id IN (1, 2);
SELECT * FROM t_order_1 WHERE order_id IN (1, 2);
```

绑定表的关联查询与单表查询复杂度和性能相当。举例说明，如果一个包含绑定表的关联查询的SQL如下：

```sql
SELECT * FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id  WHERE order_id IN (1, 2);
```

那么路由的结果应为：

```sql
SELECT * FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id  WHERE order_id IN (1, 2);
SELECT * FROM t_order_1 o JOIN t_order_item_1 i ON o.order_id=i.order_id  WHERE order_id IN (1, 2);
```

可以看到，SQL拆分的数目与单表是一致的。

笛卡尔积路由是最复杂的情况，它无法根据绑定表的关系定位分片规则，因此非绑定表之间的关联查询需要拆解为笛卡尔积组合执行。
如果上个示例中的SQL并未配置绑定表关系，那么路由的结果应为：

```sql
SELECT * FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id  WHERE order_id IN (1, 2);
SELECT * FROM t_order_0 o JOIN t_order_item_1 i ON o.order_id=i.order_id  WHERE order_id IN (1, 2);
SELECT * FROM t_order_1 o JOIN t_order_item_0 i ON o.order_id=i.order_id  WHERE order_id IN (1, 2);
SELECT * FROM t_order_1 o JOIN t_order_item_1 i ON o.order_id=i.order_id  WHERE order_id IN (1, 2);
```

笛卡尔积路由查询性能较低，需谨慎使用。

## 广播路由

对于不携带分片键的SQL，则采取广播路由的方式。根据SQL类型又可以划分为全库广播路由、全库表广播路由、全实例广播路由、单播路由和阻断路由这5种类型。

全库广播路由用于处理对数据库的操作，包括用于库设置的SET类型的数据库管理命令，以及TCL这样的事务控制语句。

全库表广播路由用于处理对数据库中与其逻辑表相关的所有真实表的操作，主要包括不带分片键的DQL和DML，以及DDL等。

全实例广播路由用于DCL操作，授权语句针对的是数据库的实例。无论一个实例中包含多少个Schema，每个数据库的实例只执行一次。

单播路由用于获取某一真实表信息的场景，它仅需要从任意库中的任意真实表中获取数据即可。例如：

```sql
DESCRIBE t_order;
```

阻断路由用于屏蔽SQL对数据库的操作，例如：

```sql
USE order_db;
```

因为Sharding-Sphere采用的是逻辑Schema的方式，无需将切换数据库Schema的命令发送至数据库中。

路由引擎的整体结构划分如下图。

![路由引擎结构](http://shardingsphere.jd.com/document/current/img/sharding/route_architecture.png)
