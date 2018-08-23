# GitLab部署:


## 部署信息:
- OS:          CentOS Linux release 7.4.1708 (Core)
- Kernel:      3.10.0-693.el7.x86_64
- Gitlab:      10.7.3
- gitlab-rake: 12.3.0
- IP:          10.11.26.56
- Port:        80(对外访问),其余会占用很多端口,注意不要占用对应端口,否则服务会失败;


## 1. 部署Gitlab:
```
[root@gitlab ~]# yum install -y curl policycoreutils openssh-server openssh-clients postfix mailx
[root@gitlab ~]# systemctl enable postfix
[root@gitlab ~]# systemctl start postfix
[root@gitlab ~]# systemctl status postfix
[root@gitlab ~]# cat <<EOF>/etc/yum.repos.d/gitlab-ce.repo
[gitlab-ce]
name=gitlab-ce
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/
repo_gpgcheck=0
gpgcheck=0
enabled=1
gpgkey=https://packages.gitlab.com/gpg.key
EOF

[root@gitlab ~]# yum install -y gitlab-ce-10.7.3-ce.0.el7.x86_64
```

## 2. 配置GitLab:
#### 修改GitLab配置文件:

##### 2.1 GitLab对外访问域名:
```
[root@gitlab ~]# sed -i "s@^external_url 'http://gitlab.example.com'@external_url 'http://code.myunsheji.com'@g" /etc/gitlab/gitlab.rb
```

##### 2.2 备份设置:
```
[root@git ~]# egrep -rn "backup_path|backup_keep_time" /etc/gitlab/gitlab.rb
292:# gitlab_rails['manage_backup_path'] = true
293:gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"		\\默认备份路径;
301:gitlab_rails['backup_keep_time'] = 604800					\\保留7天备份;
```

##### 2.3 GitLab邮箱通知设置:
```
[root@gitlab ~]# cat <<EOF>>/etc/gitlab/gitlab.rb
##email setting
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "mail.shejiyun.corp.rs.com"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "shejiyun"
gitlab_rails['smtp_password'] = "shejiyun"
gitlab_rails['smtp_domain'] = "@shejiyun.corp.rs.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
EOF
```

##### 2.4 启动GitLab服务:
```
[root@gitlab ~]# gitlab-ctl reconfigure
[root@gitlab ~]# gitlab-ctl status
```

##### 2.5 浏览器访问:
```
http://10.11.26.56

user:root
pass:登录强制设置密码

```
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/GitLab/login-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/GitLab/login-2.png)

## 3. GitLab界面设置:
#### 关闭用户注册功能,GitLab对内服务,不需要提供注册,管理员开户:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/GitLab/close-registration-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/GitLab/close-registration-2.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/GitLab/close-registration-3.png)

## 4. GitLab备份恢复:
### 4.1 备份:
#### 默认备份位置: /var/opt/gitlab/backups
```
[root@gitlab ~]# cat <<EOF>>/var/job/tools-gitlab-backup.sh
#!/bin/bash

/usr/bin/gitlab-rake gitlab:backup:create
EOF

Crontab设置:
#tools gitlab backup
0 2 * * * /var/job/tools-gitlab-backup.sh

[root@git ~]# ll /var/opt/gitlab/backups/
total 21167444
-rw------- 1 git git 3089346560 Aug 17 02:02 1534442496_2018_08_17_10.7.3_gitlab_backup.tar
-rw------- 1 git git 3096524800 Aug 18 02:02 1534528897_2018_08_18_10.7.3_gitlab_backup.tar
-rw------- 1 git git 3097497600 Aug 19 02:02 1534615300_2018_08_19_10.7.3_gitlab_backup.tar
-rw------- 1 git git 3097395200 Aug 20 02:02 1534701699_2018_08_20_10.7.3_gitlab_backup.tar
-rw------- 1 git git 3097774080 Aug 21 02:02 1534788096_2018_08_21_10.7.3_gitlab_backup.tar
-rw------- 1 git git 3098337280 Aug 22 02:02 1534874498_2018_08_22_10.7.3_gitlab_backup.tar
-rw------- 1 git git 3098572800 Aug 23 02:02 1534960890_2018_08_23_10.7.3_gitlab_backup.tar
[root@git ~]# 
```

### 4.2 恢复:
#### PS: 要注意版本问题,版本不一致会导致恢复失败;
```
[root@gitlab ~]# cd /var/opt/gitlab/backups
[root@gitlab ~]# /usr/bin/gitlab-rake gitlab:backup:restore BACKUP=1532328982_2018_07_23_10.7.3
```

### 4.3 异机灾备:
#### 4.3.1 异机灾备方案:
```
方案: 使用NFS共享存储挂载备份路径,GitLab备份脚本备份的时候就直接备份到异机;
配置:
- 配置NFS共享存储,NFS服务器:10.11.40.54;
- 挂载NFS共享目录到备份路径;

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
/dev/mapper/centos-root              xfs       150G   17G  134G  12% /
devtmpfs                             devtmpfs  3.8G     0  3.8G   0% /dev
tmpfs                                tmpfs     3.8G  4.0K  3.8G   1% /dev/shm
tmpfs                                tmpfs     3.8G  361M  3.4G  10% /run
tmpfs                                tmpfs     3.8G     0  3.8G   0% /sys/fs/cgroup
/dev/vda1                            xfs       197M  121M   77M  62% /boot
tmpfs                                tmpfs     582M     0  582M   0% /run/user/1000
10.11.40.54:/data/bakcups/codebackup nfs4      1.6T  330G  1.3T  21% /var/opt/gitlab/backups
[root@git ~]# 
```

#### 4.3.2 NFS共享存储:
##### 部署移步:
- [NFS共享存储部署文档](NFS共享存储.md);

## 备份完成后登录进行验证即可;
