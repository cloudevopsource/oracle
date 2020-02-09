
## 可插入数据库的概念

Oracle Multitenant Container Database(CDB)，即多租户容器数据库，是Oracle 12C引入的特性，指的是可以容纳一个或者多个可插拔数据库的数据库，这个特性允许在CDB容器数据库中创建并且维护多个数据库，在CDB中创建的数据库被称为PDB，每个PDB在CDB中是相互独立存在的，在单独使用PDB时，与普通数据库无任何区别。

# 内核升级

+ 升级软件包
``` bash
yum update
```

+ 安装内核源仓库
``` bash
#Import the public key:
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm 
```

+ 查看指定的内核
``` bash
yum --enablerepo=elrepo-kernel  list  |grep kernel*
```
+ 移除旧内核，并安装新内核
