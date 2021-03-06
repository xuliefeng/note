MySQL 的 Join 到底能不能用
经常听到 2 种观点：

join 性能低，尽量少用
多表 join 时，变为多个 SQL 进行多次查询
其实对于上面的观点一定程度上是正确的，但不是完全正确。但之所以流传这么广，主要还是没有搞清楚实际状态，而根据实际使用中总结出来的一些模糊规律。只有了解的 MySQL 的 Join 实际执行方式，就会知道上面 2 种观点是一种模糊的规律，这种规律并不能指导我们实际开发。下面就说说 MySQL 的实际 join 执行方式。

MySQL 的 Join 是如何执行的
join 可以说一种集合的运算，比如 left join,right join,inner join,full join,outer join，cross join 等，这些集合间的计算关系对应在高中数学集合里面的交集，并集，补集，全集等。但在实际的代码中，join 运算基本上是通过多层循环来实现的。

举一个例子，假设有 t1,t2 两张表。假设 t1 有 100 条数据，t2 表有 200 条数

查询 sql 为：

    select t1.*, t2.* from t1 left join t2 on (t1.username = t2.username)

那么这条 SQL 的执行步骤如下：

从表 t1 中取一行数据 r1
从 r1 中，取出字段 username 到表 t2 中查询
取出表 t2 中满足条件的行，跟 r1 组成一行，作为结果集的一部份
重复执行步骤 1,2,3,直到表 t1 的所以数据循环完毕
基本上先遍历 t，1,然后根据 t1 中的每行数据中的 username，去表 t2 中查找满足条件的记录。基本就是 2 层循环。

如何优化 join 查询
从上面可以看出，join 本质是循环，这里的开销如下：

遍历 t1 数据，读取数据为 t1 表的行数，假设行数为 n,则复杂度也为 n
根据 t1 的匹配字段 username 去 t2 中一行一行的查询数据
这个过程，因为 MySQL 的数据存储结构为二叉树，时间复杂度为 log2(m) m 为 t2 表的总行数
那么总复杂度近似为 n+n(2log2(m))
从上面的步骤可以看出，优化方向：

降低 t1 查询时的开销，主要是磁盘 io 开销，避免全表扫描，用索引
降低 t2 查询时的开销，也用索引
将数据量多的表做被驱动表，小表作驱动表，m 取了对数，大表数据量大对复杂度的影响没有线性增长
缓存 t1 表，不用每次去磁盘 load,比如一次缓存 100 条，那么能显著降低磁盘读数据次数，t2 每次与缓存中的 t1 数据进行比较
随机磁盘读比较耗费磁盘性能，转为顺序读，因为二叉树的存储结构，每次非主键查找，有一个回表的动作，即根据主键再次查询需要的数据

优化的基本方法：

减少循环次数，减少磁盘 IO 次数，变随机 IO 为顺序 IO

其实 MySQL 针对上面的优化方法有对应的算法：
Simple Nested Loop Join 最普通的循环，这个要避免
Block Nested Loop Join 主要是针对 t2 表上没有索引，在步骤 2 将 t2 中的每一行数据跟 join buffer 数据做对比，这样将磁盘操作转为内存操作进行比较，但是如果被驱动表的数据比较大的话，也影响性能，主要是 cache pool 被占满，导致 MySQL 性能下降
Index Nested Join 就是都通过主键进行查找关联，这种性能比较好
Batched Key Access Join 这个是 Index Nested Join 上做的优化，因为回表的存在，随机操作 io 也很耗费性能，这个算法的核心在于通过辅助索引去查找时，将得到的主键进行排序，然后按照主键递增的顺序进行查找，对磁盘的读接近顺序读，从而优化性能

到底要不用 Join
从上面的分析我们可以看到，用 Join 还是可行的，只要性能可控且在接受范围内，还是能减少代码复杂度的。需要避免的是 join 的表没有索引，不然这样的 SQL 发线上是灾难性的。

总结
Join 还是可以大胆的使用，只要把握好几个原则：

尽量让 join 的列是索引列，而且最好是类型相同,尽可能是主键索引

尽量将小表做驱动表（这一点 MySQL 在 5.6 某个版本后能自动完成）

养成将写好的 SQL 进行 explain 的好习惯，观察 SQL 的执行过程
