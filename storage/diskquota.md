# Greenplum 6.0 新功能介绍——磁盘配额管理工具Diskquota

* [Diskquota是什么](#Diskquota是什么)
* [Diskquota架构](#Diskquota架构)
* [Diskquota快速上手](#Diskquota快速上手)

# Diskquota是什么
Diskquota extension是Greenplum6.0提供的磁盘配额管理工具,它支持控制数据库schema和role的磁盘使用量。当DBA为schema或者role设置磁盘配额上限后，diskquota工作进程负责监控该schema和role的磁盘使用量，并维护超出配额上限的schema和role的黑名单。当用户试图往黑名单中的schema或者role中插入数据时，操作会被禁止。

Diskquota的典型应用场景是对于企业内部多个部门共享一个Greenplum集群，如何分配集群的磁盘资源给不同的部门。Greenplum的Resource Group功能支持对CPU，Memory等资源进行分配。而Diskquota则是对磁盘资源的细粒度分配，支持在schema和role的层级进行磁盘用量的控制，支持秒级延时的磁盘实时使用量检测，这是传统基于的cron job的磁盘管理工具做不到的。企业可以选择为不同的部门分配专属schema，从而实现对各个部门的磁盘配额分配。

需要指出Diskquota是对磁盘用量的一种软限制，“软”体现在两个方面： 1. 计算schema和role的实时用量存在一定延时（秒级），因此schema和role的磁盘用量可能会超出配额。延时对应diskquota模型的最小刷新频率，可以通过GUC `diskquota.naptime` 调整其大小。2. 对插入语句，diskquota只做查询前检查。如果加载数据的语句在执行过程中动态地超过了磁盘配额上限，查询并不会被中止。DBA可以通过show_fast_schema_quota_view和show_fast_role_quota_view快速查询每个schema和role的配额和当前使用量，并对超出配额上限的schema和role进行相应处理。


# Diskquota架构
Diskquota设计伊始，主要考虑了如下几个问题：
1. 使用单独进程管理diskquota模型，还是将数据库对象的实时磁盘使用量存储在系统表中。
系统表pg_class存储了数据表的元数据信息，比如relpages记录了analyze后数据表的页面数。我们没有采用此方式管理diskquota模型，一方面，如果让每次数据加载、删除操作都实时更新relpages等信息，会降低数据库性能；另一方面，diskquota基于extension框架，不希望改动DB内核，因而选择使用单独的监控进程来管理diskquota模型(在diskquot工作进程中维护所有数据表的实时磁盘使用量)。

2. 对于单独进程管理diskquota模型的方案，如何实现Greenplum Master和Segment之间的通信。
对于diskquota而言，Master和Segment之间的通信内容主要是每个Segment上实时的磁盘使用量数据。由于diskquota是集群级别的磁盘配额管理工具，我们只关心Master节点维护的数据表的全局磁盘使用量。因此，基于Greenplum的SPI框架，Master上的diskquota工作进程可以定期通过SPI查询，实时计算和汇总所有Segment上活跃表的磁盘使用量，Segment不需要常驻diskquota工作进程，而只需要一块存储活跃表的共享内存。

3. Diskquota性能及其对于Greenplum数据库的影响。
Diskquota工作进程在每个查询周期只对活跃的数据表计算其磁盘使用量，因此其时间复杂度正比于活跃表的个数。当数据库没有数据加载操作时，diskquota没有对IO资源的占用。Greenplum的只读操作不会与diskquota产生交互影响；写入和修改等操作，每次数据落盘（新的页面申请）会存在将数据表的relfilenode信息写入共享内存的额外开销。

最终diskquota extension的架构由以下四部分组成。

1. Quota Status Checker。负责维护diskquota模型，计算schema和role的实时磁盘用量，生成超出配额的schema和role的黑名单。
2. Quota Change Detector。负责监测磁盘变化。INSERT，COPY，DROP，VACUUM FULL等语句会改变数据表的大小，我们称被改变的数据表为活跃表，Quota Change Detector将活跃表存入共享内存，以供Quota Status Checker使用。
3. Quota Enforcement Operator。负责取消查询。当执行INSERT，UPDATE，COPY等操作时，如果被操作表所在schema或所属role超出了磁盘配额，查询会被取消。
4. Quota Setting Store。负责存储DBA定义的schema和role的磁盘配额。

![GP_diskquota](images/GPdiskquota.png)
**图1. Greenplum Diskquota架构**

## Quota Status Checker
Quota Status Checker基于Greenplum background worker框架实现，包含两类bgworker: diskquota launcher和diskquota worker。

Diskquota launcher负责管理diskquota worker。每个集群只有一个launcher，并且运行在Master节点。laucher进程在数据库启动时(具体时间是Postmaster加载diskquota链接库的时候)被注册并运行。

Laucher进程主要负责：
1. 当launcher启动时，基于数据库`diskquota`中已启动diskquota extension的数据库列表，为每一个列表中数据库启动diskquota worker进程。
2. 当laucher运行中，监听数据库`Create Extension diskquota`和`Drop Extension diskquota`请求，启动或中止相关worker进程，并更改数据库`diskquota`中的已启动diskquota extension的数据库列表。
3. 当launcher正常退出时，通知所有diskquota worker进程退出。

Diskquota worker进程实际扮演Quota Status Checkers的角色。每个启动diskquota extension的数据库都有隶属于自己worker进程。没有采用一个worker进程监控多个数据库是由于Greenplum和Postgres存在一个进程只能访问一个数据库的限制。由于每个worker进程占用数据库连接和资源，我们为同时启动disk extension的数据库个数设置了上限：10。diskquota worker通过SPI与Segment进行交互，因此，diskquota worker同样只运行在Master节点。

Worker进程主要负责：
1. 初始化diskquota模型，从表`diskquota.table_size`中读取所有table的磁盘用量，并计算schema和role的磁盘用量。对于非空数据库第一次启动diskquota extension，DBA需要调用UDF diskquota.init_table_size_table()对表`diskquota.table_size`进行初始化。该初始化需要计算数据库中的所有数据文件的大小，因此根据数据库大小，可能是一个耗时的操作。初始化完毕后，表`diskquota.table_size`将会由worker进程自动更新。
2. 以diskquota.naptime为最小刷新频率，周期性维护diskquota模型，包括所有table，schema，role的磁盘用量和超出diskquota配额的黑名单。

刷新diskquota模型的算法如下：
1. 获取schema和role的最新磁盘配额，配额记录在表'diskquota.quota_config'中.
2. 获取活跃表的磁盘使用量，首先调用SPI函数，从所有Segment的共享内存中获取全局活跃表的列表。之后，汇总计算全局活跃表在所有Segment上的磁盘使用量（通过调用pg_total_relation_size(table_oid)计算，pg_total_relation_size会自动计算数据主表，索引表，toast表，fsm表等数据表的总磁盘用量）。
3. 遍历pg_class系统表：
    1. 如果table在活跃表中，计算table磁盘使用量的变化值，并更新对应table_size_map, namespace_size_map和role_size_map。
    2. 如果table在活跃表中，标记该表的need_to_flush flag为true
    3. 如果table的schema发生变化, 将该表的磁盘使用量从旧schema移动到新schema。
    4. 如果table的owner(role)发生变化, 将该表的磁盘使用量从旧role移动到新role。
    5. 标识table的is_existed flag为true
4. 遍历table_size_map，基于is_existed flag识别被删除的表，并减少相关schema和role的磁盘使用量。
5. 遍历pg_namespace系统表:
    1. 从namespace_size_map删除对应schema。
    2. 比较每个schema的磁盘使用量和配额，将超出配额的schema放入diskquota黑名单。
6. 遍历pg_role系统表:
    1. 从role_size_map删除对应role。
    2. 比较每个role的磁盘使用量和配额，将超出配额的role放入diskquota黑名单。
7. 遍历table_size_map，基于need_to_flush flag将表的磁盘使用量写入数据表`diskquota.table_size`。Update操作需要针对每条数据执行一条SQL语句，为了加速操作,使用批量Delete+批量Insert的方式代替逐条Update。具体来说通过以下两条SQL语句处理所有大小发生变化的表：`delete from diskquota.table_size where tableoid in (need_to_flush oid list)`和`insert into diskquota.table_size values(need_to_flush oid and size list)`。



## Quota Change Detector
Quota Change Detector通过一系列Hook函数实现。对于Heap表，在smgrcreate()/smgrextend()/smgrtruncate()等位置设置Hook函数，记录活跃表信息；对于AO表和CO表，在BufferedAppendWrite/copy_append_only_data/TruncateAOSegmentFile等位置设置Hook函数，记录活跃表信息。活跃表被存储在每个Segment的共享内存中，等待Quota Status Checker周期性查询。由于活跃表只是一个子集，显著减少了每次刷新diskquota模型的代价。

## Quota Enforcement Operator
Quota Enforcement Operator同样通过Hook函数实现。通过重用Greenplum的Hook函数ExecutorCheckPerms_hook，实现在每次插入和更新数据前，检查目标schema或role是否在diskquota的黑名单中，并中止击中黑名单的查询。

## Quota Setting Store
diskquota的磁盘配额分为schema和role两类，存储在数据表'diskquota.quota_config'中。每一个启动diskquota的数据库存储和管理自己的磁盘配额。需要指出的是，尽管role不隶属于数据库，而是一个数据库集群的对象，在diskquota中将role的磁盘配额限定为数据库特定。即role会在不同的数据库有不同的配额，role的磁盘使用量也是不同数据库独立计算。Quota Setting Store被定义为如下数据表。
```
create table diskquota.quota_config (targetOid oid, quotatype int, quotalimitMB int8, PRIMARY KEY(targetOid, quotatype));
```

# Diskquota快速上手
## 安装和配置
1. 开源版diskquota下载地址：[diskquota repo](https://github.com/greenplum-db/diskquota/tree/gpdb)，安装步骤如下
```
# source greenplum_path.sh
cd $diskquota; 
make; 
make install;
```

2. 创建数据库`diskquota` 用来持久化启动diskquota extension的数据库列表。
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

4. 配置diskquota extension的刷新频率
```
# set naptime (second) to refresh the disk quota stats periodically
gpconfig -c diskquota.naptime -v 2
```

5. 创建diskquota extension，例如希望在`postgres`数据库启用diskqota extension, 执行如下语句。
```
# suppose we are in database 'postgres'
create extension diskquota;
```

6. 如果DBA在非空数据库创建了dikquota extension，会收到提示信息，需要手动执行UDF初始化表`table_size`。根据数据库中已经存在的文件数，该操作可能耗时。 
```
# after create extension diskquota on non empty database
select diskquota.init_table_size_table();
```

7. 删除diskquota extension，例如希望在'postgres'数据库禁用diskqota extension，执行如下语句。
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

3. 查询schema的磁盘使用量和配额
```
select * from diskquota.show_fast_schema_quota_view;
```

4. 查询role的磁盘使用量和配额
```
select * from diskquota.show_fast_role_quota_view;
```
