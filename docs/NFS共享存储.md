# NFS共享存储部署:


## 部署信息:
- OS:          CentOS Linux release 7.4.1708 (Core)
- Kernel:      3.10.0-693.el7.x86_64
- rpcbind:     0.2.0-44
- nfs-utils:   1.3.0-0.54
- IP:          10.11.40.54


## 1. 部署NFS共享存储 (NFS服务端和客户端操作):
- 备份中心: 10.11.40.54
### PS: NFS客户端不启动服务;
```
yum install nfs-utils rpcbind -y
```

## 2. 部署NFS服务端 (NFS服务端操作):

### PS: 服务启动顺序: rpcbind服务在nfs服务之前,先起端口注册服务再进行服务端口注册;
### 2.1 启动rpcbind服务端口注册:
```
[root@kvm-node4 ~]# systemctl enable rpcbind
[root@kvm-node4 ~]# systemctl start rpcbind
[root@kvm-node4 ~]# systemctl status rpcbind

[root@kvm-node4 ~]# rpcinfo -p localhost    \\查看下状态确定服务运行没问题;

[root@kvm-node4 ~]# lsof -i:111
COMMAND   PID USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
rpcbind 33185  rpc    6u  IPv4 7881399      0t0  UDP *:sunrpc 
rpcbind 33185  rpc    8u  IPv4 7881401      0t0  TCP *:sunrpc (LISTEN)
rpcbind 33185  rpc    9u  IPv6 7881402      0t0  UDP *:sunrpc 
rpcbind 33185  rpc   11u  IPv6 7881404      0t0  TCP *:sunrpc (LISTEN)
[root@kvm-node4 ~]# 
```

### 2.2 启动NFS服务端:
```
[root@kvm-node4 ~]# systemctl enable nfs
[root@kvm-node4 ~]# systemctl start nfs
[root@kvm-node4 ~]# systemctl status nfs

[root@kvm-node4 ~]# netstat -lntup|grep 2049
tcp        0      0 0.0.0.0:2049            0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::2049                 :::*                    LISTEN      -                   
udp        0      0 0.0.0.0:2049            0.0.0.0:*                           -                   
udp6       0      0 :::2049                 :::*                                -                   
[root@kvm-node4 ~]# 

[root@kvm-node4 ~]# rpcinfo -p localhost    \\查看是否有端口映射关系,nfs服务端启动,所以显示端口映射关系;
```

## 3. 配置NFS服务 (NFS服务端操作):

### 3.1 准备环境:
- 备份目录: /data/backups
```
[root@kvm-node4 ~]# mkdir -p /data/backups/
[root@kvm-node4 ~]# chown -R nfsnobody.nfsnobody /data/backups
[root@kvm-node4 ~]# ls -ld /data/backups/
drwxr-xr-x 7 nfsnobody nfsnobody 101 Jul 30 10:34 /data/backups/
[root@kvm-node4 ~]# tree -d -L 1 /data/backups/
/data/backups/
├── buildmachine            \\Jenkins备份目录;
├── codebackup              \\GitLab备份目录;
├── confluencehome          \\Wiki备份目录;
├── jirahome                \\Jira备份目录;
└── mysqlbackup             \\MySQL备份目录;

5 directories
[root@kvm-node4 ~]# 
```

### 3.2 修改配置文件:
```
[root@kvm-node4 ~]# cat /etc/exports
##Backup to GitLab
/data/backups/codebackup		10.11.40.0/24(rw,sync,no_root_squash)
##Backup to MySQL wiki/jira
/data/backups/mysqlbackup		10.11.40.0/24(rw,sync,all_squash)
##Backup to Jenkins
/data/backups/buildmachine		10.11.40.0/24(rw,sync,all_squash)
##Backup to wiki Home
/data/backups/confluencehome	10.11.40.0/24(rw,sync,all_squash)
##Backup to Jira Home
/data/backups/jirahome			10.11.40.0/24(rw,sync,all_squash)
[root@kvm-node4 ~]# 

[root@kvm-node4 ~]# exportfs -r
```

### 3.3 访问测试:
```
[root@git ~]# showmount -e 10.11.40.54
Export list for 10.11.40.54:
/data/backups/jirahome       10.11.40.0/24
/data/backups/confluencehome 10.11.40.0/24
/data/backups/buildmachine   10.11.40.0/24
/data/backups/mysqlbackup    10.11.40.0/24
/data/backups/codebackup     10.11.40.0/24
You have new mail in /var/spool/mail/root
[root@git ~]# 
```

### 3.4 测试挂载:

#### 3.4.1 Jenkins服务备份:
```
[root@jenkins ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Mon May 14 13:31:39 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=07dd8945-e440-47fd-a5e3-61d092ee06d2 /boot                   xfs     defaults        0 0
##Backup to Jenkins NFS
10.11.40.54:/data/backups/buildmachine	/var/lib/jenkins	nfs4	defaults	0	0	
[root@jenkins ~]# 

[root@jenkins ~]# mount -a

[root@jenkins ~]# df -hT
Filesystem                             Type      Size  Used Avail Use% Mounted on
/dev/mapper/centos-root                xfs       150G   30G  121G  20% /
devtmpfs                               devtmpfs  7.5G     0  7.5G   0% /dev
tmpfs                                  tmpfs     7.5G     0  7.5G   0% /dev/shm
tmpfs                                  tmpfs     7.5G  217M  7.3G   3% /run
tmpfs                                  tmpfs     7.5G     0  7.5G   0% /sys/fs/cgroup
/dev/vda1                              xfs       197M  121M   77M  62% /boot
tmpfs                                  tmpfs     1.3G     0  1.3G   0% /run/user/1000
10.11.40.54:/data/backups/buildmachine nfs4      1.6T  330G  1.3T  21% /var/lib/jenkins
tmpfs                                  tmpfs     1.3G     0  1.3G   0% /run/user/997
tmpfs                                  tmpfs     1.3G     0  1.3G   0% /run/user/0
[root@jenkins ~]# 
```

#### 3.4.2 GitLab备份:
```
[root@git ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Mon May 14 13:58:37 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=e71f0f8f-14da-4116-ba54-9dcb0ad3ac35 /boot                   xfs     defaults        0 0
##Backup to GitLab NFS
10.11.40.54:/data/backups/codebackup	/var/opt/gitlab/backups	nfs4	defaults	0	0
[root@git ~]# 

[root@git ~]# mount -a

[root@git ~]# df -hT
Filesystem                           Type      Size  Used Avail Use% Mounted on
/dev/mapper/centos-root              xfs       150G   17G  133G  12% /
devtmpfs                             devtmpfs  3.8G     0  3.8G   0% /dev
tmpfs                                tmpfs     3.8G  4.0K  3.8G   1% /dev/shm
tmpfs                                tmpfs     3.8G  377M  3.4G  10% /run
tmpfs                                tmpfs     3.8G     0  3.8G   0% /sys/fs/cgroup
/dev/vda1                            xfs       197M  121M   77M  62% /boot
tmpfs                                tmpfs     582M     0  582M   0% /run/user/1000
10.11.40.54:/data/bakcups/codebackup nfs4      1.6T  330G  1.3T  21% /var/opt/gitlab/backups
[root@git ~]# 
```

#### 3.4.3 Wiki备份:
```
[root@wiki ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Mon May 14 14:01:18 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=c4feafb8-1a8b-4b79-89bd-f96f97c150cf /boot                   xfs     defaults        0 0
##Backup confluence-home to NFS
10.11.40.54:/data/backups/confluencehome	/usr/local/confluence-home	nfs4	defaults    0   0
[root@wiki ~]# 

[root@wiki ~]# mount -a

[root@wiki ~]# df -hT
Filesystem                               Type      Size  Used Avail Use% Mounted on
/dev/mapper/centos-root                  xfs       150G   43G  108G  29% /
devtmpfs                                 devtmpfs   31G     0   31G   0% /dev
tmpfs                                    tmpfs      31G     0   31G   0% /dev/shm
tmpfs                                    tmpfs      31G  1.2G   29G   4% /run
tmpfs                                    tmpfs      31G     0   31G   0% /sys/fs/cgroup
/dev/vda1                                xfs       197M  121M   77M  62% /boot
tmpfs                                    tmpfs     5.8G     0  5.8G   0% /run/user/1000
10.11.40.54:/data/backups/confluencehome nfs4      1.6T  330G  1.3T  21% /usr/local/confluence-home
[root@wiki ~]# 
```

#### 3.4.4 Jira和MySQL备份:
```
[root@jira ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Mon May 14 13:59:44 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=8bcd24a7-4978-488d-b1a6-528367ca14aa /boot                   xfs     defaults        0 0
##Backup MySQL wiki/jira to NFS
10.11.40.54:/data/backups/mysqlbackup	/mysqlbackup	nfs4	defaults	0	0
##Backup jira-home to NFS
10.11.40.54:/data/backups/jirahome      /usr/local/jira-home	nfs4	defaults    0   0
[root@jira ~]# 

[root@jira ~]# mount -a

[root@jira ~]# df -hT
Filesystem                            Type      Size  Used Avail Use% Mounted on
/dev/mapper/centos-root               xfs       150G   22G  129G  15% /
devtmpfs                              devtmpfs   31G     0   31G   0% /dev
tmpfs                                 tmpfs      31G     0   31G   0% /dev/shm
tmpfs                                 tmpfs      31G  1.1G   29G   4% /run
tmpfs                                 tmpfs      31G     0   31G   0% /sys/fs/cgroup
/dev/vda1                             xfs       197M  121M   77M  62% /boot
tmpfs                                 tmpfs     5.8G     0  5.8G   0% /run/user/1000
10.11.40.54:/data/backups/jirahome    nfs4      1.6T  330G  1.3T  21% /usr/local/jira-home
10.11.40.54:/data/backups/mysqlbackup nfs4      1.6T  330G  1.3T  21% /mysqlbackup
tmpfs                                 tmpfs     5.8G     0  5.8G   0% /run/user/0
[root@jira ~]# 
```
