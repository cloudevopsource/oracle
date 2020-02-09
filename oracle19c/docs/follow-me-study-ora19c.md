
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

+ REDO文件

在CDB环境中所有的PDB共用CDB$ROOT中的REDO文件，REDO中的条目标识REDO来自那个PDB。

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
