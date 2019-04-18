# Greenplum 6.0 新功能介绍——磁盘配额管理工具diskquota

* [Diskquota是什么](#Diskquota是什么)
* [Diskquota架构](#Diskquota架构)
* [Diskquota快速上手](#Diskquota快速上手)

# Diskquota是什么
Diskquota extension是Greenplum6.0提供的磁盘配额管理工具,它支持控制数据库schema和role的磁盘使用量。当DBA为schema或者role设置磁盘配额上限后，diskquota工作进程会负责监控该schema和role的磁盘使用量，并维护包含超出配额上线的schema和role的黑名单。当用户试图往黑名单中的schema或者role中插入数据，该操作会被禁止。

Diskquota的典型应用场景是对于企业内部多个部门共享一个Greenplum集群，如何分配集群的磁盘资源给不同的部分。Greenplum的Resource Group模块支持对CPU，Memory等资源进行分配。Diskquota则是对磁盘资源的细粒度分配，支持在schema和role的层级进行磁盘用量的控制，这是传统基于的cron job的磁盘管理工具做不到的。企业可以选择为不同的部门分配专属schema，从而实现对部门的磁盘限额。

需要指出Diskquota是对磁盘用量的一种软限制，“软”体现在两个方面： 1. 计算schema和role的实时用量存在一定延时，因此schema和role的磁盘用量可能会超出限额。延时对应diskquota模型的最小刷新频率，可以通过GUC `diskquota.naptime` 调整其大小。2. 对插入语句，diskquota只做查询前检查。如果加载数据的语句在执行过程中动态地超过了磁盘配额上限，查询并不会被中止。DBA可以通过show_fast_schema_quota_view和show_fast_role_quota_view快速查询每个schema和role的配额和当前使用量，并对超出配额上限地schema和role进行相应处理。


# Diskquota架构
Diskquota设计伊始，主要考虑了如下几个问题：
1. 实现为native feature还是extension。
2. 使用单独进程管理diskquota模型，还是将数据库对象的实时用量存储在系统表中。
3. 对于单独进程管理diskquota模型的方案，如何实现Greenplum Master和Segment之间的通信。

最终diskquota extension的架构由以下四部分组成。

1. Quota Status Checker。负责维护diskquota模型，计算schema和role的实时磁盘用量，生成超出限额的schema和role的黑名单。
2. Quota Change Detector。负责监测磁盘变化。INSERT，COPY，DROP，VACUUM FULL等语句会改变数据表的大小，我们称被改变的数据表为活跃表，Quota Change Detector将活跃表存入共享内存，以供Quota Status Checker使用。
3. Quota Enforcement Operator。负责取消查询。当执行INSERT，UPDATE，COPY等操作时，如果被操作表所在schema或所属role超出了磁盘限额，查询会被取消。
4. Quota Setting Store。负责存储DBA定义的schema和role的磁盘配额。

![GP_diskquota](images/GPdiskquota.png)
**Figure 1. High-Level Greenplum Diskquota Architecture**

## Quota Status Checker
Quota Status Checker基于Greenplum background worker框架实现，包含两类bgworker: diskquota launcher和diskquota worker。diskquota worker通过SPI与Segment进行交互，因此diskquota launcher和diskquota worker都只运行在Master节点。

diskquota launcher负责管理diskquota worker。每个集群只有一个launcher，并且运行在Master节点。laucher进程在数据库启动时(具体时间是Postmaster加载diskquota链接库的时候)被注册并运行。

Laucher进程主要负责：
1. 当launcher启动时，基于数据库'diskquota'中的已启动diskquota extension的数据库列表，为每一个列表中数据库启动diskquota worker进程。
2. 当laucher运行中，监听数据库`Create Extension diskquota`和`Drop Extension diskquota`的请求，启动或中止相关worker进程，并更改数据库'diskquota'中的已启动diskquota extension的数据库列表。
3. 当launcher正常退出时，通知所有diskquota worker进程退出。

diskquota worker进程实际扮演Quota Status Checkers的角色。每个启动diskquota extension的数据库都有隶属于自己worker进程。没有采用一个worker进程监控多个数据库是由于Greenplum和Postgres存在一个进程只能访问一个数据库的限制。由于每个worker进程占用数据库连接和资源，我们为同时启动disk extension的数据库的个数设置了上限：10。

Worker进程主要负责：
1. 初始化diskquota模型，从表`diskquota.table_size`中读取所有table的磁盘用量，并计算schema和role的磁盘用量。对于非空数据库第一次启动diskquota extension，DBA需要调用UDF diskquota.init_table_size_table()对表`diskquota.table_size`进行初始化。该初始化需要计算数据库中的所有数据文件的大小，因此根据数据库大小，可能是一个耗时的操作。初始化完毕后，表`diskquota.table_size`将会由worker进程自动更新。
2. 以diskquota.naptime为最小刷新频率，周期性维护diskquota模型，包括所有table，schema，role的磁盘用量和超出diskquota限额的黑名单。

刷新diskquota模型的算法如下：
1. 获取schema和role的最新磁盘配额，配额记录在表'diskquota.quota_config'中.
2. 获取活跃表的磁盘使用量，首先调用SPI函数，从所有Segments的共享内存中获取全局活跃表的列表。之后，汇总计算全局活跃表在所有Segment上的磁盘使用量（通过调用pg_total_relation_size(table_oid)计算）。
3. 遍历pg_class系统表：
    1. 如果table在活跃表中，计算table磁盘使用量的变化值，并更新对应table_size_map, namespace_size_map和role_size_map。
    2. 如果table在活跃表中，标记该表的need_to_flush flag为true
    3. 如果table的schema发生变化, 将该表的磁盘使用量从旧schema移动到新schema。
    4. 如果table的owner(role)发生变化, 将该表的磁盘使用量从旧role移动到新role。
    5. 标识table的is_existed flag为true
    6. If flush_to_disk_interval is reached, flush the `need_to_flush` tables and their size information into table: diskquota.table_size. Since `update` for each `need_to_flush` table is slow. We batch the `update` statements to one `delete` and one `insert` statement, which makes algorithm complexity from O(N) to O(1). To be specific, using `delete from diskquota.table_size where tableoid in (need_to_flush oid list)` and `insert into diskquota.table_size values(need_to_flush oid and size list)` instead.
4. Traverse table_size_map and detect 'dropped' tables in step 3.4. Reduce the quota from corresponding namespace_size_map and role_size_map.
5. Traverse namespace in pg_namespace:
    1. remove the dropped namespace from namespace_size_map.
    2. compare the quota usage and quota limit for each namespace, put the out-of-quota namespace into blacklist.
6. Traverse role in pg_role:
    1. remove the dropped role from role_size_map.
    2. compare the quota usage and quota limit for each role, put the out-of-quota role into blacklist.


View diskquota.show_schema_quota_view is used to check the schema size/quota quickly based on table diskquota.schema_size. This could be treated as a fast version of pg_database_size().


The Algorithm to refresh diskquota model in each refresh interval is as follows:


Here are some optimization opportunities in the above algorithms. We leave them as future optimization.
1. Refetch table 'diskquota.quota_config' only when DBA modified the quota limit of namespaces or roles.
2. Avoid to traverse pg_class. We currently depend on traversing pg_class to detect dropped table, altered namespace and altered owner. We could use separate hooks to remove this cost.
3. Although we only calculate pg_total_relation_size() for active tables, whose cost is small in most cases. We could remove pg_total_relation_size() function call by replacing it with the delta of table size change. e.g. each smgr_extend()'s delta is exactly one block size. Side effect is that delta based method better to have calibration step to ensure quota model is correct.


## Quota Change Detector
Quota Change Detector is implemented as Hooks in smgr function and cdbbufferedappend function(For AO/CO tables). When table size changed, the corresponding hooks in function smgrcreate()/smgrextend()/smgrtruncate()/smgrdounlinkall() and BufferedAppendWrite() will be called on Segments. These hooks will write the changed relfilenode into shared memory called Active Tables on each Segments. Diskquota worker process on Master will periodically fetch global active table list from each Segments by calling SPI function. SPI function, which is running at Segment QE, will firstly convert relfilenode to relation oid, and then calculate Segment level table size by calling pg_total_relation_size(), which will sum the size of a table(including: base, vm, fsm, toast and index). Diskquota worker process will then sum the active table result from each Segments to generate global active table list, which is described in the Quota Status Checker section. 

Here is code fragment of smgr hook implementation.
```
void report_active_table_helper(const RelFileNodeBackend *relFileNode)
{
    DiskQuotaActiveTableFileEntry item;
    MemSet(&item, 0, sizeof(DiskQuotaActiveTableFileEntry));
    item.dbid = relFileNode->node.dbNode;
    item.relfilenode = relFileNode->node.relNode;
    item.tablespaceoid = relFileNode->node.spcNode;

    LWLockAcquire(active_table_shm_lock->lock, LW_EXCLUSIVE);
    entry = hash_search(active_tables_map, &item, HASH_ENTER_NULL, &found);
    if (entry && !found)
        *entry = item;
    LWLockRelease(active_table_shm_lock->lock);
}
```
Quota Change Detector enable us to reduce the cost of refreshing diskquota model.

## Quota Enforcement Operator
Quota Enforcement Operator is also implemented as hooks. Hook functions will check whether the corresponding schema or role is in diskquota blacklist. For soft limit diskquota, it only support one kind of enforcement hooks: enforcement hook before query is running.

The corresponding 'before query run' hook is implemented at ExecutorCheckPerms_hook in function ExecCheckRTPerms().



## Quota Setting Store
Quota limit of a schema or a role is stored in table 'quota_config' in schema 'diskquota' in monitored database. So each database stores and manages its own disk quota configuration. Note that although role is a db object in cluster level, we limit the diskquota of a role to be database specific. That is to say, a role may has different quota limit on different databases and role's disk usages are also isolated between databases. Quota Setting Store is defined as the following table.
```
create table diskquota.quota_config (targetOid oid, quotatype int, quotalimitMB int8, PRIMARY KEY(targetOid, quotatype));
```

## Diskquota HA
Diskquota support HA feature based on background worker framework. Postmaster on standby master will not start diskquota launcher process when it's in standby mode. Diskquota launcher process only exists on master node. When master is down and DBA run `activatestandy` command, standby master change its role to master and diskquota launcher process will be forked automatically. Based on the diskquota enabled database list, corresponding diskquota worker processes will be created by diskquota launcher process to do the real diskquota job.

# Diskquota快速上手
## 安装
1. 开源版diskquota下载地址：[diskquota repo](https://github.com/greenplum-db/diskquota/tree/gpdb)，安装步骤如下
```
# source greenplum_path.sh
cd $diskquota; 
make; 
make install;
```

2. 创建数据库'diskquota' 用来持久化启动diskquota的数据库列表。
```
create database diskquota;
```

3. 将diskquota添加到shared_preload_libraries列表
```
# enable diskquota in preload shared library.
gpconfig -c shared_preload_libraries -v 'diskquota'
# restart database.
gpstop -ar
```

4. 配置diskquota的刷新频率
```
# set naptime (second) to refresh the disk quota stats periodically
gpconfig -c diskquota.naptime -v 2
```

5. 创建diskquota extension，例如希望在'postgres'数据库启用diskqota extension。
```
# suppose we are in database 'postgres'
create extension diskquota;
```

6. 如果DBA在非空数据库创建了dikquota extension，会收到提示信息，需要手动执行UDF初始化表`table_size`。根据数据库中已经存在的文件数，该操作可能耗时。 
```
# after create extension diskquota on non empty database
select diskquota.init_table_size_table();
```

7. 删除diskquota extension，例如希望在'postgres'数据库禁用diskqota extension。
```
# login into 'postgres'
drop extension diskquota;
```

## 使用
1. 设置schema的磁盘配额
```
create schema s1;
select diskquota.set_schema_quota('s1', '1 MB');
set search_path to s1;
create table a(i int);
# insert small data succeeded
insert into a select generate_series(1,100);
# insert large data succeeded, since diskquota is soft limit
insert into a select generate_series(1,10000000);
# insert small data failed
insert into a select generate_series(1,100);
# delete quota configuration
select diskquota.set_schema_quota('s1', '-1');
# insert small data succeeded
select pg_sleep(5);
insert into a select generate_series(1,100);
reset search_path;
```

2. 设置role的磁盘配额
```
create role u1 nologin;
create table b (i int);
alter table b owner to u1;
select diskquota.set_role_quota('u1', '1 MB');
# insert small data succeeded
insert into b select generate_series(1,100);
# insert large data succeeded, since diskquota is soft limit
insert into b select generate_series(1,10000000);
# insert small data failed
insert into b select generate_series(1,100);
# delete quota configuration
select diskquota.set_role_quota('u1', '-1');
# insert small data succeed
select pg_sleep(5);
insert into a select generate_series(1,100);
reset search_path;
```

3. 查询schema的磁盘使用量和限额
```
select * from diskquota.show_fast_schema_quota_view;
```

4. 查询role的磁盘使用量和限额
```
select * from diskquota.show_fast_role_quota_view;
```
