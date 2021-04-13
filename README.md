ba

实现高可靠、高可拓展、高性能、高自动化等功能，并最终存储用户数据。RADOS系统主要由两部分组成，分别是OSD和Monitor

> 2、 RADOS之上是LIBRADOS，LIBRADOS是一个库，它允许应用程序通过访问该库来与RADOS系统进行交互，支持多种编程语言，比如C、C++、Python等。
> 3 基于LIBRADOS层开发的有三种接口，分别是RADOSGW、librbd和MDS。
> 4、RADOSGW是一套基于当前流行的RESTFUL协议的网关，支持对象存储，兼容S3和Swift。
> 5、 librbd提供分布式的块存储设备接口，支持块存储。
> 6、MDS提供兼容POSIX的文件系统，支持文件存储。
>
> * Ceph的核心组件包括Client客户端、MON监控服务、MDS元数据服务、OSD存储服务，各组件功能如下：
>   <1> Client客户端：负责存储协议的接入，节点负载均衡。
>   <2> MON监控服务：负责监控整个集群，维护集群的健康状态，维护展示集群状态的各种图表，如OSD Map、Monitor Map、PG Map和CRUSH Map。
>   <3> MDS元数据服务：负责保存文件系统的元数据，管理目录结构。
>   <4> OSD存储服务：主要功能是存储数据、复制数据、平衡数据、恢复数据，以及与其它OSD间进行心跳检查等。一般情况下一块硬盘对应一个OSD。

# ceph集群部署

| IP           | 节点名 | 节点上的服务   |
| :----------- | ------ | -------------- |
| 192.168.0.20 | node1  | admin,osd, mon |
| 192.168.0.21 | node2  | osd, mon       |
| 192.168.0.22 | node3  | osd, mon       |

# 开始部署

> * 初始化服务器

```bash
cat /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.80.20 node1
192.168.80.21 node2
192.168.80.22 node3
```



```bash
cat /etc/yum.repos.d/ceph.repo
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
```

```
所有节点安装软件：
yum install ceph ceph-radosgw -y
```

```bash
随意挑选一个节点为部署节点，这里选择的node1节点上执行：

yum install ceph-deploy -y
```

```bash
部署mon服务：
cd /etc/ceph/

ceph-deploy new --cluster-network 192.168.0.0/24 --public-network 192.168.0.0/24 node-1
# ls -l
total 16
-rw-r--r-- 1 root root 200 Jan 20 15:08 ceph.conf
-rw-r--r-- 1 root root 2986 Jan 20 15:08 ceph-deploy-ceph.log
-rw------- 1 root root 73 Jan 20 15:08 ceph.mon.keyring
-rw-r--r-- 1 root root 92 Dec 17 02:08 rbdmap
```

```bash
cat /etc/ceph/ceph.conf 
[global]
fsid = 564dd817-f13f-4f49-bb3d-27f3dd788da3
public_network = 192.168.0.0/24
cluster_network = 192.168.0.0/24
mon_initial_members = node-1,node-2,node-3
mon_host = 192.168.0.20,192.168.0.21,192.168.0.22
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```

```bash
ceph-deploy mon create-initial
```

# 查看ceph 状态

```bash
ceph -s
cluster:
id: sdfesdreasasdfewe
health: HEALTH_OK
```



# 部署mgr服务：

```bash
ceph-deploy mgr create node-1 node-2 node-3
```

# 部署osd服务,查看各节点上的存储磁盘

```bash
  122  ceph-deploy disk list node-1
  123  ceph-deploy disk list node-2
  124  ceph-deploy disk list node-3
```

#  销毁磁盘中原来的数据

```bash
  125   ceph-deploy disk zap node-1 /dev/vdb
  127   ceph-deploy disk zap node-1 /dev/vdb
  130   ceph-deploy disk zap node-1 /dev/vdb
  134   ceph-deploy disk zap node-1 /dev/vdb
  如果有错如下可以解决
  lsblk
NAME                                                                                                  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sr0                                                                                                    11:0    1  1024M  0 rom  
vda                                                                                                   252:0    0   300G  0 disk 
├─vda1                                                                                                252:1    0   200M  0 part /boot
└─vda2                                                                                                252:2    0 299.8G  0 part 
  ├─centos-root                                                                                       253:0    0 291.9G  0 lvm  /
  └─centos-swap                                                                                       253:1    0   7.9G  0 lvm  [SWAP]
vdb                                                                                                   252:16   0    20G  0 disk 
└─ceph--086b17ff--b5ff--4406--92ca--af75ccb38c21-osd--block--f4dcfc46--0948--4a9a--9092--7d2c833ec98e 253:2    0    20G  0 lvm  
wipefs -af /dev/vdb
dmsetup remove ceph--37b66e4d--49d5--41b3--872b--826fb614c9ec-osd--block--6fe90792--5605--43e8--a45f--c72a3094bc44


```

#  创建osd服务：

```bash
  156  ceph-deploy osd create 172.18.12.88 --data /dev/vdb
  157  ceph-deploy osd create 172.18.12.89 --data /dev/vdb
  158  ceph-deploy osd create 172.18.12.90 --data /dev/vdb
  检查osd是否正常
  ceph osd tree
  检查集群状态
  ceph -s
```

# 创建块存储在，部署节点上执行，这里是node-1上，将ceph.conf及其密钥文件拷贝到client上

```bash
任意客户端可以使用存储，这里我们在node2上
 yum install ceph -y
 ceph-deploy admin node-2
 ceph-deploy admin node-3
```



### 创建存储池以及镜像文件

````bash
ceph osd pool create --pool pool01 128
ceph osd pool ls
 rbd create -p pool01 --image volume01 --size 5120M
 rbd info -p pool01 --image volume01
 rbd feature disable pool01/volume01 object-map fast-diff deep-flatten
 rbd map pool01/volume01
 
````

### 挂在和使用

```
mkfs.xfs /dev/rbd0
mkdir /data
mount /dev/rbd0 /data
```




