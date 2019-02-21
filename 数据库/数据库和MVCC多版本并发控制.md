## 事务特性

- Atomicity（原子性）

  一个事务必须被视为一个不可分割的最小工作单位，整个事务中的所有操作要么全部提交成功，要么全部失败回滚。

- Consistency（一致性）

  数据库总是从一个一致性状态转换到另一个一致性状态，事务执行之前和执行之后都必须处于一致性状态。

- Isolation（隔离性）

  通常来说，一个事务所做的修改在最终提交之前，对其它事务是不可见的。关于事务的隔离性，数据库提供了多种隔离级别。

- Durability（持久性）

  一旦事务提交，则其所做的修改就会永久保存到数据库中。即便是数据库系统遇到故障的情况下也不会丢失。

## 并发事务的问题

- 脏读

  一个事务正在对一条记录进行修改，在这个事务完成并提交前， 这条记录的数据就处于不一致状态。这时， 另一个事务也来读取同一条记录，如果不加控制，第二个事务读取了这些“脏”数据，并据此做进一步的处理，就会产生未提交的数据依赖关系。

  | 时间 | 事务A                      | 事务B                       |
  | ---- | -------------------------- | --------------------------- |
  | T1   | 开启事务                   | 开启事务                    |
  | T2   | 查询账户余额为1000         |                             |
  | T3   | 充值500，余额修改为1500    |                             |
  | T4   |                            | 查询余额为**1500**          |
  | T5   | 撤销事务，余额改回**1000** |                             |
  | T6   |                            | 汇入500，余额修改为**2000** |
  | T7   |                            | 提交事务                    |

- 不可重复读

  一个事务在读取某些数据后的某个时间，再次读取以前读过的数据，却发现其读出的数据已经发生了变更、或者某些记录已经被删除了。

  | 时间 | 事务A                                                     | 事务B                                         |
  | ---- | --------------------------------------------------------- | --------------------------------------------- |
  | T1   | 开启事务                                                  | 开启事务                                      |
  | T2   | select * from user where user_id=100 假设为小明的用户信息 |                                               |
  | T3   |                                                           | 将user_id=100的用户信息对应的年龄修改为**18** |
  | T4   |                                                           | 提交事务                                      |
  | T5   | 再次查询发现用户的**年龄变更**了                          |                                               |
  | T6   | ...                                                       |                                               |
  | T7   | 提交事务                                                  |                                               |

- 幻读

  一个事务按相同的查询条件重新读取以前检索过的数据，却发现其它事务插入了满足其查询条件的新数据。

  | 时间 | 事务A                                            | 事务B                          |
  | ---- | ------------------------------------------------ | ------------------------------ |
  | T1   | 开启事务                                         | 开启事务                       |
  | T2   | select * from user where age=18 假设得到两条记录 |                                |
  | T3   |                                                  | 向user表插入一条age=18的新记录 |
  | T4   |                                                  | 提交事务                       |
  | T5   | **再次查询得到三条记录**                         |                                |
  | T6   | ..                                               |                                |
  | T7   | 提交事务                                         |                                |

  ### 幻读和不可重复读的区别

  - 不可重复读的重点是修改：在同一事务中，相同的条件，第一次和第二次读到的数据不一致（中间有其它事务提交了修改）。
  - 幻读的重点是新增或者删除：在同一事务中，相同的条件，第一次和第二次读到的记录数不一样（中间有其它事务提交了新增或者删除）。

  ## 事务隔离级别

  SQL标准定义了4类隔离级别，每一种级别都规定了一个事务中所做的修改，哪些在事务内和事务间是可见的，哪些是不可见的。

  - Read Uncommited

  所有事务都可以看到其它未提交事务的执行结果，该隔离级别一般不会使用。

  - Read Committed（RC）

  一个事务只能看到已经提交的事务所做的变更。

  - Repeatable Read（RR）

  确保同一事务的多个实例在并发读取数据时会看到相同的数据行。

  - Serializable

  完全串行化读，每次读都需要获得表级共享锁，读写相互阻塞。

  | 隔离级别        | 脏读 | 不可重复读 | 幻读 |
  | --------------- | ---- | ---------- | ---- |
  | Read Uncommited | Yes  | Yes        | Yes  |
  | Read Committed  | No   | Yes        | Yes  |
  | Repeatable Read | No   | No         | Yes  |
  | Serializable    | No   | No         | No   |

  ## 并发事务解决方案

  脏读、不可重复读和幻读都是数据库读一致性问题，需要由数据库提供一定的事务隔离机制来解决。

  （1）**锁机制**

  解决**写-写**冲突问题。在读取数据前，对其加锁，防止其它事务对该数据进行修改。

  - 悲观锁

    往往依靠数据库提供的锁机制。

  - 乐观锁

    大多是基于数据版本记录机制来实现。

  （2）**MVCC多版本并发控制**

  解决**读-写**冲突问题。不用加锁，通过一定机制生成一个数据请求时间点时的一致性数据快照， 并用这个快照来提供一定级别 （语句级或事务级） 的一致性读取。这样在读操作的时候不需要阻塞写操作，写操作时不需要阻塞读操作。

  ## MVCC多版本并发控制

  Mysql的大多数事务型存储引擎实现都不是简单的行级锁，基于并发性能考虑，一般都实现了MVCC多版本并发控制。MVCC是通过保存数据在某个时间点的快照来实现的。不管事务执行多长时间，事务看到的数据都是一致的。

  ### 读操作

  读操作分成两类：快照读和当前读。

  **快照读**：简单的select操作属于快照读，不加锁。

  - select * from table where ? ;

  **当前读**：特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。

  - select * from table where ? lock in share mode ;
  - select * from table where ? for update ;
  - update table set ? where ? ;
  - delete from table where ? ;

  ### 数据存储

  innodb存储引擎中，每行数据都包含了一些隐藏字段：DB_ROW_ID、DB_TRX_ID、DB_ROLL_PTR和DELETE_BIT。

  ![数据库记录](/Users/qiuweisen/Downloads/数据库记录.png)

  - DB_TRX_ID：用来标识最近一次对本行记录做修改的事务的标识符，即最后一次修改本行记录的事务id。delete操作在内部来看是一次update操作，更新行中的删除标识位DELELE_BIT。
  - DB_ROLL_PTR：指向当前数据的**undo log**记录，回滚数据通过这个指针来寻找记录被更新之前的内容信息。
  - DB_ROW_ID：包含一个随着新行插入而单调递增的行ID, 当由innodb自动产生聚集索引时，聚集索引会包括这个行ID的值，否则这个行ID不会出现在任何索引中。
  - DELELE_BIT：用于标识该记录是否被删除。

  ### 数据操作

  - insert

    创建一条记录，DB_TRX_ID为当前事务ID，DB_ROLL_PTR为NULL。

  - delete

    将当前行的DB_TRX_ID设置为当前事务ID，DELELE_BIT设置为1。

  - update
    复制一行，新行的DB_TRX_ID为当前事务ID，DB_ROLL_PTR指向上个版本的记录，事务提交后DB_ROLL_PTR设置为NULL。

  - select

    1、只查找创建早于当前事务ID的记录，确保当前事务读取到的行都是事务之前就已经存在的，或者是由当前事务创建或修改的；

    2、行的DELETE BIT为1时，查找删除晚于当前事务ID的记录，确保当前事务开始之前，行没有被删除。

  ### 一致性读

  Mysql的一致性读是通过read view结构来实现。

  read view主要是用来做可见性判断的，它维护的是**本事务不可见的当前其他活跃事务**。其中最早的事务ID为`up_limit_id`，最迟的事务ID为`low_limit_id`。

  ```c++
  	trx_id_t	low_limit_id;
  				/*!< The read should not see any transaction
  				with trx id >= this value. In other words,
  				this is the "high water mark". */
  	trx_id_t	up_limit_id;
  				/*!< The read should see all trx ids which
  				are strictly smaller (<) than this value.
  				In other words,
  				this is the "low water mark". */
  ```

  #### 如何理解low_limit_id

  可以参考知乎这个答案来理解。low_limit_id应该是**当前系统尚未分配的下一个事务ID**，也就是**目前已经出现过的事务ID的最大值+1**。

  ![image-20190203151446779](/Users/qiuweisen/Library/Application Support/typora-user-images/image-20190203151446779.png)

  #### 可见性判断

  假设要读取的行的最后提交事务id(即当前数据行的稳定事务id)为 trx_id，可见性比较过程如下：

  1. trx_id < up_limit_id **=>** 此记录的最后一次修改在read view创建之前，跳转到步骤5；

  2. trx_id > low_limit_id **=>** 此记录的最后一次修改在read view创建之后，跳转到步骤4；
  3. up_limit_id <= trx_id <= low_limit_id **=>** 从up_limit_id到low_limit_id进行遍历，如果trx_id等于他们之中的某个事务id的话，表示该记录的最后一次修改尚未保存，跳转到步骤4。否则跳转到步骤5；
  4. 从此记录的DB_ROLL_PTR指针所指向的undo log（此记录的上一次修改），将undo log的DB_TRX_ID赋值给trx_id，跳转到步骤1重新开始计算可见性；
  5. 如果此记录的DELELE_BIT为false，说明该记录未被删除，可以返回，否则不返回。

  #### RR和RC隔离级别

  Repeatable Read和Read Committed隔离级别都是基于read view来实现，不同之处在于：

  - Repeatable Read

    read view是在执行事务中第一条select语句的瞬间创建，后续所有的select都是复用这个对象，所以能保证每次读取的一致性。（**可重复读的语义**）

  - Read Committed

    事务中每条select语句都会创建read view，这样就可以读取到其它事务已经提交的内容。

  > 对于InnoDB来说，Repeatable Read虽然比Read Committed隔离级别高，开销反而相对较小。

  

  ### 参考资料

  [数据库事务与MySQL事务总结](https://zhuanlan.zhihu.com/p/29166694)

  [MySQL-InnoDB-MVCC多版本并发控制](https://segmentfault.com/a/1190000012650596)

  [MySQL InnoDB MVCC深度分析](https://www.cnblogs.com/stevenczp/p/8018986.html)

  [乐观锁和 MVCC 的区别？](https://www.zhihu.com/question/27876575)

  [MySQL 在 RC 隔离级别下是如何实现读不阻塞的？](https://www.zhihu.com/question/66320138)

  

  

  

  