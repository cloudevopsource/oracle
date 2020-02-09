
## 设置所有节点的主机名

+ hostnamectl修改主机名 （所有root用戶節點執行）

``` bash
hostnamectl set-hostname k8scloud[1-5].frcloud.io
```


+ 编辑hosts文件 （所有root用戶節點執行）

``` bash
cat <<EOF > /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.224.1   k8scloud1.frcloud.io k8scloud1
192.168.224.2   k8scloud2.frcloud.io k8scloud2
192.168.224.3   k8scloud3.frcloud.io k8scloud3
192.168.224.4   k8scloud4.frcloud.io k8scloud4
192.168.224.5   k8scloud5.frcloud.io k8scloud5
127.0.0.1       k8scloudapi.frcloud.io k8scloudapi
EOF

```

## 内核升级

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
