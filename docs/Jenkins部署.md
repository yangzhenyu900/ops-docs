# Jenkins部署:


## 部署信息:
- OS:          CentOS Linux release 7.4.1708 (Core)
- Kernel:      3.10.0-693.el7.x86_64
- Jenkins:     2.122-1.1
- IP:          10.11.26.200
- Port:        8080

## 1. 安装JDK1.8,自行下载jdk:
```
[root@jenkins ~]# ll /usr/local/src/jdk-8u172-linux-x64.tar.gz
-rw-r--r--. 1 root root 190921804 May 15 21:15 /usr/local/src/jdk-8u172-linux-x64.tar.gz
[root@jenkins ~]# tar xf /usr/local/src/jdk-8u172-linux-x64.tar.gz -C /usr/local/
[root@jenkins ~]# ln -s /usr/local/jdk1.8.0_172/ /usr/local/jdk
[root@jenkins ~]# cat <<'EOF'>>/etc/profile
###jdk
export JAVA_HOME=/usr/local/jdk
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH="$JAVA_HOME/bin:$PATH"
EOF

[root@jenkins ~]# source /etc/profile
[root@jenkins ~]# java -version
java version "1.8.0_172"
Java(TM) SE Runtime Environment (build 1.8.0_172-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.172-b11, mixed mode)
[root@jenkins ~]# 
```

## 2. 安装Jenkins:
```
[root@jenkins ~]# wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat/jenkins.repo
[root@jenkins ~]# rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key
[root@jenkins ~]# yum install -y jenkins
[root@jenkins ~]# sed -ri '/^candidates/a\/usr/local/jdk/bin/java' /etc/init.d/jenkins
[root@jenkins ~]# systemctl daemon-reload
[root@jenkins ~]# systemctl start jenkins
[root@jenkins ~]# systemctl status jenkins
[root@jenkins ~]# chkconfig jenkins on

访问Jenkins:
http://10.11.26.200:8080
```

## 3. Jenkins登录界面设置:
#### 根据界面提示,解锁Jenkins:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/set-1.png)

[root@jenkins ~]# cat /var/lib/jenkins/secrets/initialAdminPassword		\\默认密码存放位置,密码随机生成;
667432431e5c484fa5c2fd864e093075

![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/set-2.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/set-3.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/set-4.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/set-5.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/set-6.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/set-7.png)

## 4. jenkins插件代理设置:
#### 插件管理代理设置:
```
官方：https://updates.jenkins.io/current/update-center.json
清华大学：https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
日本：http://mirror.esuni.jp/jenkins/updates/update-center.json
俄罗斯：http://mirror.yandex.ru/mirrors/jenkins/updates/update-center.json
```
