title: Mysql Online DDL 详解
tags: [mysql]
categories:
  - mysql
date: 2022-07-12 23:32:56
---
<script src="/js/mermaid.full.min.js"></script>

# 背景
在和DBA沟通前对alter table 不熟悉，导致沟通不顺畅。从本文可以了解到DDL的执行过程，评估执行DDL对线上的影响。
# `ALTER TABLE`-修改表结构
能够修改shechma的内容如下：
* 添加或删除列(columns)
* 创建或销毁索引(index)
* 改变列的类型(type)
* 重命名列或表名
* 修改表的存储引擎(storage engine)或者备注(comment)
详细：https://dev.mysql.com/doc/refman/5.7/en/alter-table.html

# 新增列的DDL会阻塞DML
列操作的DDL支持的一些行为，其中新增列「Adding a column」是支持In place， rebuild table，「某些条件下」允许并发DML，特别注意新增自增列并不知支持并发DML。因此，add column操作在某些条件下是会阻塞DML。[详细的材料](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html#online-ddl-column-operations)
![功能表格](https://github.com/maidongcao/img/blob/master/online_ddl/online_ddl_allow_table.png?raw=true)

* 下边做个实验，观察新增列对DML的影响。实验参考：https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-performance.html

    1. session1: 创建表，进行事务查后不提交事务，加MDL读锁
    ```
    mysql> CREATE TABLE t1 (c1 INT) ENGINE=InnoDB;
    mysql> START TRANSACTION;
    mysql> SELECT * FROM t1;
    ```
    
    2. session2 ：给表格t1新增列x， 此时DDL会被阻塞，加MDL写锁
    ```
    mysql> ALTER TABLE t1 ADD COLUMN x INT, ALGORITHM=INPLACE, LOCK=NONE;
    ```
    
    3. session3 ：查询表格的数据，语句也会被阻塞，加MDL读锁
    ```
    mysql> SELECT * FROM t1;
    ```
    
    4. session4: 查询当前执行的线程；会看到阻塞的原因
    ```
    mysql> SHOW FULL PROCESSLIST;
    ```
    结果如图：
![执行结果](https://github.com/maidongcao/img/blob/master/online_ddl/show_process_result.png?raw=true)
* 结论
    1. MDL写锁触发锁表，导致DDL和DML都会阻塞
    2. DDL过程中会持有MDL写锁，导致锁表
    3. DDL的执行时长决定了DML的阻塞时长，如果DDL执行时长，会阻塞后续的DML 

# 新增列DDL的执行过程
在mysql 5.5 之前新增列「**copy方式：在服务器层执行**」需要锁表并且时间比较长，mysql5.6后 新增列支持**INPLACE方法「存储引擎层执行**」，大幅度缩短了锁表的时
## COPY和INPLACE
从mysql5.6开始使用online DDL对DDL的执行过程进行了优化，解决了早期版本MysqlDDL操作锁表的问题，能够保证DDL执行过程报纸读写，不影响数据库对外服务。
* 5.7之前版本 Mysql执行DDL的主要算法
间。
1. COPY：
* 主要步骤：
    1. 创建与原表结构定义一致的临时表
    2. 对原表加锁，不允许执行DML，但允许快照读
    3. 在临时表执行DDL语句
    4. 逐行拷贝原表数据到临时表
    5. 原表和临时表进行RENAME操作，此时升级原表上的锁，不允许读写，直至完成RENAME。
    
* 总结：
    1. 需要将数据从原表拷贝到临时表，这个过程颇为耗时
    2. 整个过程大部分时间会锁住原表，导致DDL过程数据表无法提供服务
2. INPLACE
INPLACE「InnoDB fast index creation」5.5之前只限于二级索引的创建和删除。
* 主要步骤：
    1. 创建临时的frm文件；
    2. 对原表加锁，不允许执行DML，但允许查询；
    3. 根据聚集索引的顺序，构造新的索引项，按照顺序插入新索引页；
    4. 升级原表上的锁，不允许读写操作；
    5. 进行RENAME操作，替换原表的frm文件，完成DDL操作；
* 总结：
    1. DDL过程加锁的粒度比较小
    2. 直接操作底层frm文件，不需要新建临时表拷贝数据

3. COPY和INPLACE的区别
     * COPY需要建立临时表，逐行拷贝数据到临时表，而INPLACE是通过修改底层文件来实现DDL
     * INPLACE的持有锁时间更短，对DML的并发性更好些

## online DDL 执行新增列过程
**由于目前线上使用的mysql 5.7，存储引擎为InnoDb，以下内容都基于这个版本讨论。**
参考文档：https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl.html

上一节，我们看到了INPLACE的核心思想是通过操作mysql文件，降低IO操作和锁的粒度来降低MDL对DML的影响。mysql 5.7 的新增列操作也是使用INPLACE实现类似的操作。

### INPLACE 分类

| INPLACE执行方式 |  描述|
| --- | --- |
| Rebuilds Table | 行记录格格式的修改，如字段的增、删、类型修改 |
| No-Rebuilds Table | 不涉及行记录格式的修改，如索引删除、字段名修改 |

### 新增列的Online DDL过程 
Online DDL主要有PREPARE(准备)、EXECUTE(执行)和COMMIT(提交)三个阶段，如下：

1. PREPARE：此过程都是轻量操作，新增frm和ibd文件和一些初始化操作
    *     **创建新的临时frm文件「表结构文件」**；
    *     **持有EXCLUSIVE-MDL锁，禁止读写操作**；
    *     根据ALTER类型，确定执行方式(copy,Online-Rebuilds,Online-No-Rebuilds)；
    *      更新数据字典的内存对象；
    *      分配row_log对象记录增量(Rebuilds需要)；
    *      **生成新的临时ibd文件「数据文件」(Rebuilds需要)**。
    
2. EXECUTE：
    *     **降级EXCLUSIVE-MDL锁，允许读写**；
    *     **记录执行期间产生的DML增量到row_log中(Rebuilds需要)**；
    *     扫描old_table的聚集索引中每一条记录record；
    *     根据构造新表对应的索引项；
    *     将构造的索引项插入sort_buffer块中；
    *     将sort_buffer块插入到新的索引中；
    *     **将row_log中的记录应用到新临时表中，应用到最后一个block**；

3. COMMIT：
    * **升级到EXECLUSIVE-MDL锁，禁止事务读写**；
    * 重做row_log中最后一部分的增量；
    * 提交事务，写InnoDB redo日志；
    * 更新InnoDB的数据字典表；
    * 修改统计信息；
    * **RENAME临时的ibd和frm文件**；
    * 执行变更完成。

* 概括
row_log记录了DDL执行期间产生的DML操作，这保证了变更期间表的并发性，通过以上过程可以看出在EXECUTE(执行)阶段表允许读写操作，操作记录在row_log中，在最后阶段应用到新表当中，保证了数据的完整性。
## 原生Online DDL的局限
原生Online DDL 实际上是在存储引擎层执行的，对存储引擎的类型有一定的要求，并且目前并不是所有的DDL都执行oneline DDL。某些DDL不支持online DDL，会出现较大的代价。
实际生产环境中，更多的是希望有一种方式能够适配所有存储引擎和DML和所有的DDL能够并发执行，所有业界一般是更多使用例如gh-host等中间件来实现服务器层的并发DDL。

# gh-host执行DDL
  由于原生online DDL支持的DDL有限「限制了存储引擎，DDL类型等」，所以业界一般使用的是gh-host工具「实现服务器层的DML和DDL并发，使用所有的DDL和存储引擎」执行DDL
  ## 执行过程
  ![gh-host](https://github.com/maidongcao/img/blob/master/online_ddl/gh-ost-general-flow.png?raw=true)
      1. Gh-ost 在主库和从库建立临时表「ghost table」
      2. Gh-ost 消费从库的binlog「公司内部消费的是主库的binlog」
      3. 把binlog回放临时表，在此过程中也同时拷贝原表到从表「原生online dll是先全量拷贝再同步增量」
      4. cut-over：临时表和原表切换「此才会有阻塞」
  ## cut-over 阻塞过程
  

|  | clinet session 1 | gh-ost session 1 | clinet session 2 | gh-ost session 2 | clinet session 3 |
| --- | --- | --- | --- | --- | --- |
| Time 1 |  正常 DML |  |  |  |  |
| Time 2 |  | 创建del表「回滚使用」锁原表锁del表| SELECT & DML 等待gh-ost session 1  |  |  |
| Time 3 |  |  |  | 执行rename表交换 | SELECT & DML 等待gh-ost session 1 锁  |
| Time 4 |  | 检测到gh-ost session 2 的rename sql，释放锁 | 等待锁 | 执行rename，完成表交换  | 等待锁 |
| Time 5 |  |  | 正常执行 |  |  正常执行 |

1. client session 1 执行DML，此时gh-ost尝试加表锁；如果client session 1的DML是大事务，会加锁等待，3s后未加锁成功，会DDL执行失败。
2. 创建del表、锁原表和锁del表，这个步骤会加表锁
3. 提交rename表操作和正常流量的DML都会等待步骤2的表锁
4. 步骤2释放锁的时机是检测到gh-ost有提交rename sql。此时释放表锁，rename sql优先持有表锁执行rename 操作，完成表交换「执行时间很短」。
5. 在步骤4「rename」执行过程中，其他的DML都在等待表锁，一旦rename完成，会继续获取锁继续执行DML操作

  ## 总结
  1. cut-over过程中，主要的耗时在gh-ost锁原表。此时如果正常的DML有大事务，会导致无法获取到表锁，导致失败
  2. gh-ost获取表锁前有大事务，不会影响正常的DML。因为此时gh-ost未获取锁。
  3. 一旦gh-ost获取表锁，后续的DML都会锁等待，直到rename「获取锁优先级最高」完成才执行
# 总结
1.  online DDL下进行新增列会存在短暂的锁表。对于大表的表记录修改，建议在低峰期操作。
2.  执行DDL前，先评估下DDL可能对业务产生的影响，确认影响方案，和DBA确认阻塞的影响范围。
3. 由于执行alter table 是需要拷贝数据的，建议多个alter table 合并成一条。
4. 执行DDL前需要保证有足够的磁盘空间