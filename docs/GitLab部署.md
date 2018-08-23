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
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/GitLab/login.png)

