
## 可插入数据库的概念

Oracle Multitenant Container Database(CDB)，即多租户容器数据库，是Oracle 12C引入的特性，指的是可以容纳一个或者多个可插拔数据库的数据库，这个特性允许在CDB容器数据库中创建并且维护多个数据库，在CDB中创建的数据库被称为PDB，每个PDB在CDB中是相互独立存在的，在单独使用PDB时，与普通数据库无任何区别。

## 多租户环境的组成

+ CDB Root

Root根容器数据库，一个CDB环境中只能有一个Root容器数据库，在CDB环境中被标识为CDB$ROOT。在根数据库中含有主数据字典视图，其中包含了与Root容器有关的元数据和CDB中所包含的所有的PDB信息。

+ CDB Seed

CDB Seed为PDB的种子，在CDB环境中被标识为PDB$SEED其中提供了数据文件，是创建新的 PDB的模板；你可以连接PDB$SEED，但是不能执行任何事物，因为PDB$SEED是只读的，不可进行修改。

+ PDBs

PDB数据库，在CDB环境中可以有多个PDB数据库且每个PDB都是独立存在的，与传统的Oracle数据库基本无差别，每个PDB拥有自己的数据文件和objects，唯一的区别在于PDB可以插入到CDB中，以及在CDB中拔出，并且在任何一个时间点之上PDB必须拔出或者插入到一个CDB中，当用户链接PDB时不会感觉到根容器和其他PDB的存在。
```bash

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 MP4CLOUD                       READ WRITE NO
SQL> 
```

PDB$SEED为CDB seed，为PDB


## CDB环境中的用户

+ 公用用户

公用用户是在root数据库中和所有的PDB数据库中都存在的用户，公用用户必须在根容器中创建，然后此用户会在所有的现存的PDB中自动创建（可见），公用用户标识必须以c##或者C##开头，sys和system用户是Oracle在CDB环境中自动创建的公用用户，所以在PDB中也可见。
```bash

grant dba to c##yyh container=all;
```
+ 本地用户

本地用户指的是在PDB中创建的普通用户，只有在创建它的PDB中才会存在该用户，并且PDB中只能创建本地用户。

## CDB中的各对象的基本知识整理

+ SYSTEM/SYSAUX表空间

在CDB的数据库环境中，SYSTEM/SYSAUX表空间并不是公用，CDB$ROOT以及每个PDB都拥有自己的SYSTEM和SYSAUX表空间。
```bash
SQL> select * from v$tablespace;

       TS# NAME                           INCLUDED_IN_DATABASE_BACKUP BIGFILE FLASHBACK_ON ENCRYPT_IN_BACKUP     CON_ID
---------- ------------------------------ --------------------------- ------- ------------ ----------------- ----------
         0 SYSTEM                         YES                         NO      YES                                     1
         0 SYSTEM                         YES                         NO      YES                                     2
         1 SYSAUX                         YES                         NO      YES                                     1
         1 SYSAUX                         YES                         NO      YES                                     2
         2 UNDOTBS1                       YES                         NO      YES                                     1
         2 UNDOTBS1                       YES                         NO      YES                                     2
         3 TEMP                           NO                          NO      YES                                     1
         3 TEMP                           NO                          NO      YES                                     2
         4 USERS                          YES                         NO      YES                                     1
         0 SYSTEM                         YES                         NO      YES                                     3
         1 SYSAUX                         YES                         NO      YES                                     3
         2 UNDOTBS1                       YES                         NO      YES                                     3
         3 TEMP                           NO                          NO      YES                                     3
         4 USERS                          YES                         NO      YES                                     3

14 rows selected

```

+ UNDO 表空间

在12.2之前的版本中，所有的PDB共用CDB$ROOT中的UNDO文件，在12.2之后的版本中UNDO的使用模式有两种：SHARED UNDO MODE和LOCAL UNDO MODE；LOCAL UNDO MODE就是每个PDB使用自己的UNDO表空间，如果当PDB中没有自己的UNDO表空间时，会使用CDB$ROOT中的公共UNDO表空间;并且更改为local undo后CDB中的所有的PDB会自动创建自己的UNDO表空间。
```bash

SQL> select * from v$tablespace;

       TS# NAME                           INCLUDED_IN_DATABASE_BACKUP BIGFILE FLASHBACK_ON ENCRYPT_IN_BACKUP     CON_ID
---------- ------------------------------ --------------------------- ------- ------------ ----------------- ----------
         0 SYSTEM                         YES                         NO      YES                                     1
         0 SYSTEM                         YES                         NO      YES                                     2
         1 SYSAUX                         YES                         NO      YES                                     1
         1 SYSAUX                         YES                         NO      YES                                     2
         2 UNDOTBS1                       YES                         NO      YES                                     1
         2 UNDOTBS1                       YES                         NO      YES                                     2
         3 TEMP                           NO                          NO      YES                                     1
         3 TEMP                           NO                          NO      YES                                     2
         4 USERS                          YES                         NO      YES                                     1
         0 SYSTEM                         YES                         NO      YES                                     3
         1 SYSAUX                         YES                         NO      YES                                     3
         2 UNDOTBS1                       YES                         NO      YES                                     3
         3 TEMP                           NO                          NO      YES                                     3
         4 USERS                          YES                         NO      YES                                     3

14 rows selected

```
在创建CDB时使用了SHARED UNDO MODE方式，如果后续想更改为LOCAL UNDO MODE，我们可以使用如下命令更改UNDO MODE为LOCAL UNDO MODE:
```bash
startup upgrade
alter database local undo on;
shutdown immediate
startup
```

+ REDO文件

在CDB环境中所有的PDB共用CDB$ROOT中的REDO文件，REDO中的条目标识REDO来自那个PDB。在PDB中无法执行ALTERSYSTEM SWITCH LOGFILE命令，只有公用用户在ROOT容器中才可以执行该命令。

```bash

SQL> select * from V$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARCHIVED STATUS           FIRST_CHANGE# FIRST_TIME  NEXT_CHANGE# NEXT_TIME       CON_ID
---------- ---------- ---------- ---------- ---------- ---------- -------- ---------------- ------------- ----------- ------------ ----------- ----------
         1          1         52  209715200        512          2 NO       INACTIVE               2278626 2020/2/8 22      2322024 2020/2/9 6:          0
         2          1         53  209715200        512          2 NO       INACTIVE               2322024 2020/2/9 6:      2349302 2020/2/9 10          0
         3          1         54  209715200        512          2 NO       CURRENT                2349302 2020/2/9 10 1.8446744073                      0

SQL> select * from V$logfile;

    GROUP# STATUS  TYPE    MEMBER                                                                           IS_RECOVERY_DEST_FILE     CON_ID
---------- ------- ------- -------------------------------------------------------------------------------- --------------------- ----------
         1         ONLINE  +DATADG/TSCDB1/ONLINELOG/group_1.258.1030785513                                  NO                             0
         1         ONLINE  /orabak/fast_recovery_area/TSCDB1/onlinelog/o1_mf_1_h2wgm9vx_.log                YES                            0
         2         ONLINE  +DATADG/TSCDB1/ONLINELOG/group_2.259.1030785513                                  NO                             0
         2         ONLINE  /orabak/fast_recovery_area/TSCDB1/onlinelog/o1_mf_2_h2wgmbz5_.log                YES                            0
         3         ONLINE  +DATADG/TSCDB1/ONLINELOG/group_3.260.1030785515                                  NO                             0
         3         ONLINE  /orabak/fast_recovery_area/TSCDB1/onlinelog/o1_mf_3_h2wgmdst_.log                YES                            0

6 rows selected

```
+ archivelog

在CDB环境中所有的PDB共用CDB的归档模式，以及归档文件，不可以单独为PDB设置自己的归档模式，只有特权用户连接根容器之后才可以启动归档模式。

+ 临时表空间
每个PDB都有自己的临时表空间，如果PDB没有自己的临时表空间文件，那么，PDB可以使用CDB$ROOT中的临时表空间。

+ 参数文件

参数文件中只记录了根容器的参数信息，没有记录PDB级别的参数信息，在根容器中修改初始化参数，会被继承到所有的PDB中，在PDB中修改参数后，PDB的参数会覆盖CDB级别的参数，PDB级别的参数记录在根容器的pdb_spfile$视图中，但并不是所有的参数都可以在PDB中修改，可以通过v$system_parameter视图查看PDB中可修改的参数：

```bash
SQL> select * from pdb_spfile$;

SQL>SELECT name FROM v$system_parameter WHERE ispdb_modifiable = 'TRUE' ORDER BY name;

```

+ 控制文件

CDB环境中只有一组控制文件，所有的PDB共用这组公共的控制文件，从任何PDB中添加数据文件都会记录到公共控制文件当中，公用用户连接根容器时，可对控制文件进行管理。
```bash
#CDB:
SQL> show parameter control_files;
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
control_files                        string      +DATADG/TSCDB1/CONTROLFILE/current.257.1030785513, /orabak/fast_recovery_area/TSCDB1/controlfile/o1_mf_h2wgm8mf_.ctl
```

```bash
#PDB:
SQL> show parameter control_files;
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
control_files                        string      +DATADG/TSCDB1/CONTROLFILE/current.257.1030785513, /orabak/fast_recovery_area/TSCDB1/controlfile/o1_mf_h2wgm8mf_.ctl

```

+ 告警日志以及跟踪文件

在CDB中所有的PDB共用一个告警日志和一组跟踪文件，所有的PDB告警信息都会写入同一个告警日志中。

+ 字符集

在CDB中定义字符集也可以应用于它所含有的PDB中，每个PDB也可以有自己的字符集设置
```bash
SELECT a.value || '_' || b.value || '.'|| c.value NLS_LANG
FROM nls_database_parameters a,nls_database_parameters b, nls_database_parameters c
WHERE a.parameter = 'NLS_LANGUAGE' ANDb.parameter = 'NLS_TERRITORY' AND c.parameter = 'NLS_CHARACTERSET';

```
+ 数据字典视图与动态性能视图

在CDB环境中引入了CDB_级别的数据字典视图，它的级别高于DBA_/ALL_/USER_，CDB级别的数据字典视图含有所有PDB的元数据信息，其中增加了con_id列，con_id为CDB中所有容器唯一标识符，其中con_id为0的是CDB$ROOT，con_id为2的是PDB$SEED，每个PDB在CDB中都会分配一个唯一的con_id。如果要想查看CDB级别的数据字典视图，必须使用公用用户在跟容器中查看，并且要查看的PDB必须处于open状态，才可以看到PDB中的信息。
```bash
SQL> select con_id, pdb_id, pdb_name, dbid, status from cdb_pdbs;

    CON_ID     PDB_ID PDB_NAME                                                                               DBID STATUS
---------- ---------- -------------------------------------------------------------------------------- ---------- ----------
         2          2 PDB$SEED                                                                         1713339984 NORMAL
         3          3 MP4CLOUD                                                                         1881559932 NORMAL
```

## 多租户数据库的创建

+ 使用DBCA图形工具创建CDB

这里需要注意的是Oracle 12.2之后支持LOCAL UNDO，这里注意需要手动要勾选LOCAL UNDO选项。

+ NO-CDB转换为CDB
+ CREATE DATABASE语句创建CDB (补充。。。)

创建代码中”ENABLE PLUGGABLE DATABASE”之后部分与PDB有关，其他部分与创建传统的Oracle数据库语句均相同。
在使用脚本创建CDB时Oracle提供了两种方法，一种是使用OMF，另外一种是非OMF的方式。

参数文件中需要将ENABLE_PLUGGABLE_DATABASE设置为TRUE。
参数文件中需要将FILE_NAME_CONVERT 子句指定了使用创建PDBSEED文件名

查看CDB数据库是否创建成功
```bash
#CDB：
SQL> SELECT dbid, name, open_mode, cdb, con_id FROM v$database;

      DBID NAME      OPEN_MODE            CDB     CON_ID
---------- --------- -------------------- --- ----------
2817260453 TSCDB1    READ WRITE           YES          0
```
```bash
#PDB：
SQL> SELECT dbid, name, open_mode, cdb, con_id FROM v$database;

      DBID NAME      OPEN_MODE            CDB     CON_ID
---------- --------- -------------------- --- ----------
2817260453 TSCDB1    READ WRITE           YES          0
```
查看所有数据库的状态
```bash
SQL> 
SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 MP4CLOUD                       READ WRITE NO
SQL> 
```
如果是新创建的CDB中只会包含有两个容器：根容器CDB$ROOT和种子容器PDB$SEED
```bash
#CDB:
SQL> SELECT con_id, dbid, con_uid, guid, name FROM v$containers;

    CON_ID       DBID    CON_UID GUID                             NAME
---------- ---------- ---------- -------------------------------- --------------------------------------------------------------------------------
         1 2817260453          1 9D15E34210D75E05E05306E0A8C0C761 CDB$ROOT
         2 1713339984 1713339984 9D15E34210D85E05E05306E0A8C0C761 PDB$SEED
         3 1881559932 1881559932 9D164E5302EE764EE05306E0A8C0BB61 MP4CLOUD
```
#pdb:
```bash
SQL> SELECT con_id, dbid, con_uid, guid, name FROM v$containers;

    CON_ID       DBID    CON_UID GUID                             NAME
---------- ---------- ---------- -------------------------------- --------------------------------------------------------------------------------
         3 1881559932 1881559932 9D164E5302EE764EE05306E0A8C0BB61 MP4CLOUD

```
## 多租户数据库的管理
管理CDB时，通常需要使用sys用户连接根容器数据库，在操作方式上与非CDB数据库同样。

+ 当前连接容器的信息
```bash

SQL> show con_id con_name user;

CON_ID
------------------------------
1

CON_NAME
------------------------------
CDB$ROOT
USER is "SYS"
SQL> 
```
+ 启动和停止CDB
```bash
SQL>startup--默认情况下启动CDB时不会自动启动PDBs，后续我们可以使用手工的方式启动PDB：

SQL>ALTER PLUGGABLE DATABASE [pdb_name] OPEN;
or
SQL>ALTER PLUGGABLE DATABASE ALL OPEN;
SQL>shutdown immediate

```

+ 查看CDB数据库表空间使用情况
```bash
with generator0 as
(select cf.con_id,cf.tablespace_name, sum(cf.bytes) / 1024 / 1024 frm
from cdb_free_space cf
group by cf.con_id,cf.tablespace_name),
generator1 as
(select cd.con_id,cd.tablespace_name, sum(cd.bytes) / 1024 / 1024 usm
from cdb_data_files cd
group by cd.con_id,cd.tablespace_name),
generator2 as(
select g0.con_id, c.name con_name, g0.tablespace_name, g0.frm, g1.usm
from generator0 g0, generator1 g1,v$containers c
where g0.con_id = g1.con_id
and g0.tablespace_name =g1.tablespace_name
and c.con_id = g1.con_id
union
select c.con_id,
   c.name,
   ct.tablespace_name,
   null,
   sum(ct.bytes) / 1024 / 1024
from v$containers c,cdb_temp_files ct
where c.con_id = ct.con_id
group by c.con_id, c.name,ct.tablespace_name)
select con_id,
case when con_name = LAG(con_name, 1) OVER(PARTITION BY con_name ORDER BY tablespace_name) THEN null ELSE con_name END
con_name, tablespace_name, frm freemb, usm usemb
from generator2
order by con_id;

    CON_ID CON_NAME                                                                         TABLESPACE_NAME                    FREEMB      USEMB
---------- -------------------------------------------------------------------------------- ------------------------------ ---------- ----------
         1 CDB$ROOT                                                                         SYSAUX                            43.9375        730
         1                                                                                  SYSTEM                           233.4375        700
         1                                                                                  USERS                                   4          5
         1                                                                                  UNDOTBS1                          300.875        320
         1                                                                                  TEMP                                              20
         3 MP4CLOUD                                                                         SYSAUX                            18.5625        165
         3                                                                                  SYSTEM                            19.8125        210
         3                                                                                  USERS                                   4          5
         3                                                                                  UNDOTBS1                         217.8125        235
         3                                                                                  TEMP                                              20

10 rows selected
```
+ 切换容器

使用公用用户连接CDB后可以使用alter session的方式切换不同的容器,在切换容器时无需运行监听器和密码文件。只要公用用户拥有相关权限就可以切换到另外的容器中
```bash
alter session set container=mp4cloud;
alter session set container = cdb$root;
```
## PDB的创建

PDB数据库的创建可以从现存的数据库中复制数据文件，包括种子容器、可插拔数据库、non-CDB数据库，创建时可以使用CREATE PLUGGABLE、RMAN、DBCA以及EM等。

在12.1版本中在创建PDB时，Source PDB必须处于read only状态，在12.2版本中，因为undo local mode新特性的推出，在创建PDB时，Source PDB在read write状态，依然可以创建。另外在12.2版本中Oracle推出了refresh PDB特性，具有对Source PDB进行增量同步的功能。


+ CDB seed (PDB$SEED)
```bash

SQL> conn / as sysdba
Connected.
SQL>  create pluggable database mp4cloud admin user mp4cloudadmin identified by undead ROLES=(CONNECT);

Pluggable database created.

```

语句执行完毕之后查看创建完成的PDB：
```bash

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 MP4CLOUD                       MOUNTED
SQL> alter pluggable database mp4cloud open;

Pluggable database altered.

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 MP4CLOUD                       READ WRITE NO
SQL> 

```
+ 克隆本地已经存在的PDB
```bash
SQL> create pluggable database fzpu from mp4cloud;

Pluggable database created.

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 MP4CLOUD                       READ WRITE NO
         4 FZPU                           MOUNTED

SQL> alter pluggable database fzpu open; 

Pluggable database altered.

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 MP4CLOUD                       READ WRITE NO
         4 FZPU                           READ WRITE NO

```

 检查service_name
 ```bash
 SQL> SELECT service_id, name, network_name, enabled, pdb, con_id FROM cdb_services;

SERVICE_ID NAME                                                             NETWORK_NAME                                                                     ENABLED PDB                                                                                  CON_ID
---------- ---------------------------------------------------------------- -------------------------------------------------------------------------------- ------- -------------------------------------------------------------------------------- ----------
         1 SYS$BACKGROUND                                                                                                                                    NO      CDB$ROOT                                                                                  1
         2 SYS$USERS                                                                                                                                         NO      CDB$ROOT                                                                                  1
         3 cdbXDB                                                           cdbXDB                                                                           NO      CDB$ROOT                                                                                  1
         4 cdb                                                              cdb                                                                              NO      CDB$ROOT                                                                                  1
         8 MP4CLOUD                                                         MP4CLOUD                                                                         NO      MP4CLOUD                                                                                  3
        10 FZPU                                                             FZPU                                                                             NO      FZPU                                                                                      4

6 rows selected
 
 
 
 ```
 
 源PDB中的service_name(mp4cloud)已经被更改指定的service_name(fzpu)


+ Creating a PDB by Cloning a Remote PDB
+ non-CDB数据库

+ 拔下的PDB


## 使用DBCA可以使用以下资源创建PDB
+ CDB seed (PDB$SEED)

+ RMAN备份

拔下的PDB

## PDB Refresh

PDB Refresh是12C推出的特性，具有对源端PDB进行增量同步的功能，每次刷新会将源端PDB中的任何更改同步到目标PDB(在此环境中目标PDB被称作Refreshable PDB)中，目前增量同步方式有两种：手动方式与自动方式。
