Welcome to the greenplum diskquota Wiki!
This Wiki page includes the latest explanation of greenplum diskquota design

* [Postgresql Diskquota](#postgresql-diskquota)
* [What Is Diskquota](#what-is-diskquota)
* [Design Of Diskquota](#design-of-diskquota)
* [How to use Diskquota](#how-to-use-diskquota)
* [Known Issue](#known-issue)


# Postgresql Diskquota
Greenplum diskquota is based on the project [Postgresql Diskquota](https://github.com/greenplum-db/diskquota). Postgresql Diskquota is inspired by Heikki's [pg_quota](https://github.com/hlinnaka/pg_quota) project

The Wiki of Postgresql diskquota is at [Postgresql Diskquota Design](https://github.com/greenplum-db/diskquota/wiki/Postgresql-Diskquota-Design).

Greenplum diskquota follows the most of design of postgresql diskquota. At the same time, it supports diskquota extension on MPP architecture. The main difference is described as follows:
1. Using SPI to fetch active tables from each Segments instead of read active tables from shared memory directly.
2. Support AO/CO format tables.

# What Is Diskquota
Greenplum Diskquota is an extension to control the disk usage for database objects like schemas or roles. DBA could set the quota limit for a schema or a role on a database. Then diskquota worker process will monitor disk usage and maintain the diskquota model, which includes the current disk usage of each schema or role in the database and the blacklist of schema and role whose quota limit is reached. Loading data into tables, whose schema or role is in diskquota blacklist, will be cancelled.

Currently, Diskquota only supports soft limit of disk usage. On one hand it has some delay to detect the schemas or roles whose quota limit is exceeded. The delay also exists when you drop/truncate/vacuum full a table(delay could be adjusted by GUC `diskquota.naptime`). On the other hand, 'soft limit' is a kind of 'before running' enforcement: Query loading data into out-of-quota schema/role will be forbidden before query is running. It could not cancel the query whose quota limit is reached dynamically during the query is running. DBA could use show_fast_schema_quota_view and show_fast_role_quota_view to check which schema/role is out of limit and decide to grant more quota to that schema/role or not.

# Design Of Diskquota
To implement diskquota feature, we split the functionality into four parts.
1. Quota Status Checker. It maintains the diskquota model, calculate the schema or role whose quota limit is reached.
2. Quota Change Detector. It's responsible for detecting the change of disk usage, which is introduced by INSERT, COPY, DROP, VACUUM FULL etc. operations. It will report the active tables to Status Checker.
3. Quota Enforcement Operator. It's responsible for cancelling the queries which loading data into schema or role whose quota limit is exceeded.
4. Quota Setting Store. It's where the user defined quota limit of a schema or a role to be stored.

![GP_diskquota](images/GPdiskquota.png)
**Figure 1. High-Level Greenplum Diskquota Architecture**

## Quota Status Checker
Quota Status Checker is based on background worker framework. There are two kinds of background workers: diskquota launcher and diskquota worker.

Both diskquota launcher process and worker process are all run at Greenplum Master node. Diskquota worker processes will use SPI to communicate with Segment nodes which will be described later.

Launcher process is used to manage the worker processes for each databases. There is only one launcher process per database cluster(i.e. one launcher per postmaster).
Launcher process, as a background worker, is registered in _PG_init() of diskquota.so, which mean it will be started at the beginning of database start. During database startup, it will call RegisterDynamicBackgroundWorker() to create diskquota worker processes to monitor databases which are listed in table `database_list` in database `diskquota`. During database running, it also support to add/delete worker process dynamically by calling `create extension diskquota;`or `drop extension diskquota;` in the database separately.

Worker processes are the Quota Status Checkers for each monitored database. Worker processes are responsible for monitoring the disk usage of schemas and roles for the target database. It will periodically (can be set via diskquota.naptime) recalculate the table size of active tables, and update their corresponding schema or owner's disk usage. Then compare with user defined quota limit for those schemas or roles. If exceeds the limit, put the corresponding schemas or roles into the diskquota blacklist in shared memory. Schemas or roles in blacklist are used to do query enforcement to cancel queries which plan to load data into these schemas or roles.

Worker processes maintain the table size information in local memory(table_size_map). To support fast database size check function, we flush table_size_map/namespace_size_map to user table diskquota.table_size/diskquota.namespace_size periodically. The flush algorithm is described in diskquota model refreshing algorithm step 3.6.

UDF diskquota.init_table_size_table() is used to initialize the table diskquota.table_size when it is the first time to enable diskquota extension for this database. After initialization succeeds, flag is set to `ready` in table diskquota.state which is used by diskquota model initialization step in worker process described below.  Note that for the database with millions of files, the cost of UDF diskquota.init_table_size_table() is similar to pg_database_size() which would be extremely slow.

View diskquota.show_schema_quota_view is used to check the schema size/quota quickly based on table diskquota.schema_size. This could be treated as a fast version of pg_database_size().

The Algorithm to initialize diskquota model.
1. If diskquota.state is not set to `ready`, it will sleep the `naptime` and try again.
2. If diskquota.state is set to `ready`, then worker process is able to load the table size information from diskquota.table_size into table_size_map.
3. Follow other steps in diskquota model refreshing algorithm to finish the initialization work.

The Algorithm to refresh diskquota model in each refresh interval is as follows:
1. Fetch the latest user defined quota limit by reading table 'diskquota.quota_config'.
2. Call SPI function to fetch global active table lists from all the Segments. Each SPI QE will fetch local active table list(table_oid and size) from Active Tables shared memory on Segments. Table size on Segments is calculated by function pg_total_relation_size(table_oid). Diskquota worker on Master will sum the active table size from each segments to generate the global active table list(which includes table_oid and total_size). Note that this logic is implemented in gp_activetable.c which is the main difference between Greenplum diskquota and Postgresql diskquota.
3. Traverse user tables in pg_class:
    1. If table is in active table list, calculate the delta of table size change, update the corresponding table_size_map, namespace_size_map and role_size_map.
    2. If table is in active table list, mark table to be need_to_flush in table_size_map.
    3. If table's schema change, move the quota from old schema to new schema in namespace_size_map.
    4. If table's owner change, move the quota from old owner to new owner in role_size_map.
    5. Mark table is existed(not dropped) in table_size_map.
    6. If flush_to_disk_interval is reached, flush the `need_to_flush` tables and their size information into table: diskquota.table_size. Since `update` for each `need_to_flush` table is slow. We batch the `update` statements to one `delete` and one `insert` statement, which makes algorithm complexity from O(N) to O(1). To be specific, using `delete from diskquota.table_size where tableoid in (need_to_flush oid list)` and `insert into diskquota.table_size values(need_to_flush oid and size list)` instead.
4. Traverse table_size_map and detect 'dropped' tables in step 3.4. Reduce the quota from corresponding namespace_size_map and role_size_map.
5. Traverse namespace in pg_namespace:
    1. remove the dropped namespace from namespace_size_map.
    2. compare the quota usage and quota limit for each namespace, put the out-of-quota namespace into blacklist.
6. Traverse role in pg_role:
    1. remove the dropped role from role_size_map.
    2. compare the quota usage and quota limit for each role, put the out-of-quota role into blacklist.

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

# How to use Diskquota
## Install
1. Download diskquota from [diskquota repo](https://github.com/greenplum-db/diskquota/tree/gpdb) and install it.
```
cd $diskquota; 
make; 
make install;
```

2. Create database 'diskquota' to store the diskquota enabled database list.
```
create database diskquota;
```

3. Enable diskquota as preload shared library
```
# enable diskquota in preload shared library.
gpconfig -c shared_preload_libraries -v 'diskquota'
# restart database.
gpstop -ar
```

4. Config GUC of diskquota.
```
# set naptime (second) to refresh the disk quota stats periodically
gpconfig -c diskquota.naptime -v 2
```

5. Create diskquota extension in monitored database.
```
# suppose we are in database 'postgres'
create extension diskquota;
```

6. [optional] If create diskquota extension on an non-empty database. It is required to initialize the table size information at first. Note that depend on the existing data file numbers, initialization may take a long time to finish.
```
select diskquota.init_table_size_table();
```

7. Drop diskquota extension in database. Suppose we want to disable diskquota feature on 'postgres' database.
```
# login into 'postgres'
drop extension diskquota;
```

## Usage
1. Set/Update/Delete schema quota limit using diskquota.set_schema_quota
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

2. Set/Update/Delete role quota limit using diskquota.set_role_quota
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

3. Show schema quota limit and current usage
```
select * from diskquota.show_fast_schema_quota_view;
```

4. Show role quota limit and current usage
```
select * from diskquota.show_fast_role_quota_view;
```

## Test
Run regression tests.
```
cd $diskquota;
make installcheck
```
