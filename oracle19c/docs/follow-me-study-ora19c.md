
## 可插入数据库的概念

Oracle Multitenant Container Database(CDB)，即多租户容器数据库，是Oracle 12C引入的特性，指的是可以容纳一个或者多个可插拔数据库的数据库，这个特性允许在CDB容器数据库中创建并且维护多个数据库，在CDB中创建的数据库被称为PDB，每个PDB在CDB中是相互独立存在的，在单独使用PDB时，与普通数据库无任何区别。

## 多租户环境的组成

+ CDB Root
Root根容器数据库，一个CDB环境中只能有一个Root容器数据库，在CDB环境中被标识为CDB$ROOT。在根数据库中含有主数据字典视图，其中包含了与Root容器有关的元数据和CDB中所包含的所有的PDB信息。

+ CDB Seed
CDB Seed为PDB的种子，在CDB环境中被标识为PDB$SEED其中提供了数据文件，是创建新的 PDB的模板；你可以连接PDB$SEED，但是不能执行任何事物，因为PDB$SEED是只读的，不可进行修改。

+ PDBs
PDB数据库，在CDB环境中可以有多个PDB数据库且每个PDB都是独立存在的，与传统的Oracle数据库基本无差别，每个PDB拥有自己的数据文件和objects，唯一的区别在于PDB可以插入到CDB中，以及在CDB中拔出，并且在任何一个时间点之上PDB必须拔出或者插入到一个CDB中，当用户链接PDB时不会感觉到根容器和其他PDB的存在。
