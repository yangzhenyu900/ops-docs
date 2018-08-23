# Nexus部署:


## 部署信息:
- OS:          CentOS Linux release 7.4.1708 (Core)
- Kernel:      3.10.0-693.el7.x86_64
- nexus:       3.11.0-01
- IP:          10.11.26.60
- Port:        8081

## 1. Nexus服务部署:

### 1.1 安装JDK1.8,自行下载jdk:
```
[root@nexus ~]# ll /usr/local/src/jdk-8u172-linux-x64.tar.gz
-rw-r--r--. 1 root root 190921804 May 15 21:15 /usr/local/src/jdk-8u172-linux-x64.tar.gz
[root@nexus ~]# tar xf /usr/local/src/jdk-8u172-linux-x64.tar.gz -C /usr/local/
[root@nexus ~]# ln -s /usr/local/jdk1.8.0_172/ /usr/local/jdk
[root@nexus ~]# cat <<'EOF'>>/etc/profile
###jdk
export JAVA_HOME=/usr/local/jdk
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH="$JAVA_HOME/bin:$PATH"
EOF

[root@nexus ~]# source /etc/profile
[root@nexus ~]# java -version
java version "1.8.0_172"
Java(TM) SE Runtime Environment (build 1.8.0_172-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.172-b11, mixed mode)
[root@nexus ~]# 
```

### 1.2 安装Nexus:
```
[root@nexus ~]# ll /usr/local/src/nexus-3.12.1-01-unix.tar.gz 
-rw-r--r-- 1 root root 120563681 Jun 14 14:39 /usr/local/src/nexus-3.12.1-01-unix.tar.gz
[root@nexus ~]# tar xf /usr/local/src/nexus-3.12.1-01-unix.tar.gz -C /usr/local/
[root@nexus ~]# ln -s /usr/local/nexus-3.12.1-01/ /usr/local/nexus
[root@nexus ~]# useradd nexus
[root@nexus ~]# chown -R nexus.nexus /usr/local/nexus-3.11.0-01/ /usr/local/sonatype-work/

[root@nexus ~]# cat <<'EOF'>>/etc/security/limits.conf
*       soft    nproc   819200
*       hard    nproc   819200
*       soft    nofile  819200
*       hard    nofile  819200
EOF

[root@nexus ~]# su - nexus
[nexus@nexus ~]$ cd /usr/local/nexus
[nexus@nexus nexus]$ ./bin/nexus start
[nexus@nexus nexus]$ ./bin/nexus status
[nexus@nexus nexus]$ ps -ef|grep java
[nexus@nexus nexus]$ lsof -i:8081

PS: 等待一会才能看到,Java启动比较慢;
```

### 1.3 访问Nexus:
```
http://10.11.26.60:8081

默认用户密码:
admin
admin123

```
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/set-1.png)

## 2. Nexus登录界面设置:

### 2.1 登录Nexus:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/set-2.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/set-3.png)

### 2.2 创建私有仓库:

- 创建流程: proxy + hosted = group

#### 2.2.1 Maven私有仓库:
- 流程: maven-snapshots + maven-releases + maven-central = maven-public
- Proxy_URL: http://maven.aliyun.com/nexus/content/repositories/central/
- URL: http://10.11.26.60:8081/repository/maven-public/

![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/maven-set-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/maven-set-2.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/maven-set-3.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/maven-set-4.png)

#### 2.2.2 Node.Js私有仓库:
- 流程: npm-proxy + npm-hosted = npm
- Proxy_URL: https://registry.npm.taobao.org
- URL: http://10.11.26.60:8081/repository/npm/

![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/init-set.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/npm-set-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/npm-set-2.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/npm-set-3.png)

#### 2.2.3 Docker私有仓库:
- 流程: docker-hosted
- Docker加速器: https://7caofi77.mirror.aliyuncs.com

![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/init-set.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/docker-set.png)

#### 2.2.4 PyPi私有仓库:
- 流程: pypi-proxy + pypi-hosted = pypi
- Proxy_URL: http://mirrors.aliyun.com/pypi
- URL: http://10.11.26.60:8081/repository/pypi/

![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/init-set.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/pypi-set-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/pypi-set-2.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/pypi-set-3.png)

#### 2.2.5 RubyGems私有仓库:
- 流程: rubygems-proxy + rubygems-hosted = rubygems
- Proxy_URL: https://ruby.taobao.org/
- URL: http://10.11.26.60:8081/repository/rubygems/

![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/init-set.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/rubygems-set-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/rubygems-set-2.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Nexus/rubygems-set-3.png)

