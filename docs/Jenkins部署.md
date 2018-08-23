# Jenkins部署:


## 部署信息:
- OS:          CentOS Linux release 7.4.1708 (Core)
- Kernel:      3.10.0-693.el7.x86_64
- Jenkins:     2.122-1.1
- IP:          10.11.26.200
- Port:        8080

## 1. Jenkins服务部署:

### 1.1 安装JDK1.8,自行下载jdk:
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

### 1.2 安装Jenkins:
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

### 1.3 Jenkins登录界面设置:
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

### 1.4 jenkins插件代理设置:
#### 插件管理代理设置:
```
官方：https://updates.jenkins.io/current/update-center.json
清华大学：https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
日本：http://mirror.esuni.jp/jenkins/updates/update-center.json
俄罗斯：http://mirror.yandex.ru/mirrors/jenkins/updates/update-center.json
```

## 2. Jenkins邮箱配置:

### 2.1 部署postfix:
```
[root@jenkins ~]# yum install -y postfix
[root@jenkins ~]# cat <<'EOF'>>/etc/main.rc
# ADD POSTFIX Config
set from="shejiyun" smtp="mail.shejiyun.corp.rs.com" smtp-auth-user="shejiyun" smtp-auth-password="shejiyun" smtp-auth=login
EOF

163邮箱示例:
set from=xxx@163.com smtp=smtp.163.com smtp-auth-user=xxx smtp-auth-password="xxxx" smtp-auth=login

提示: 163邮箱密码可采用远程密码,在邮箱界面可设置,只可远程访问不可登录其帐号;

PS: smtp="mail.shejiyun.corp.rs.com" smtp-auth-user="shejiyun" smtp-auth-password="shejiyun"
   以上信息为公司内部IT配置的Exchange邮箱转发服务器;

[root@jenkins ~]# sed -i 's@ inet_interfaces = localhost@ inet_interfaces = all@g' /etc/postfix/main.cf
[root@jenkins ~]# systemctl enable postfix
[root@jenkins ~]# systemctl start postfix
[root@jenkins ~]# systemctl status postfix

```

### 2.2 测试邮箱设置:
```
[root@jenkins ~]# echo "This is a test mail."|mail -s "Test" li.zhiqiang@chinaredstar.com
```
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/msg-test.png)

### 2.3 设置Jenkins:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/msg-set-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/msg-set-2.png)

### 2.4 多人邮箱通知设置:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/msg-set-3.png)

```
PS: 图1和2,两点邮箱地址必须一样,下面是构建成功通知信息;
```

#### 构建通知信息如下:
- 标题:
```
【构建通知】：$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!
```

- 正文:
```
<html>    
<head>    
<meta charset="UTF-8">    
<title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>    
</head>    
    
<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"    
    offset="0">    
    <table width="95%" cellpadding="0" cellspacing="0"  style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">    
        <tr>    
            <td>各位同事，大家好，以下是：${PROJECT_NAME } 项目构建信息</td>    
        </tr>    
        <tr>    
            <td><br />    
            <b><font color="#0B610B">构建信息</font></b>    
            <hr size="2" width="100%" align="center" /></td>    
        </tr>    
        <tr>    
            <td>    
                <ul>    
                    <li>项目名称 ： ${PROJECT_NAME}</li>    
                    <li>构建编号 ： 第${BUILD_NUMBER}次构建</li>    
                    <li>触发原因： ${CAUSE}</li>    
                    <li>构建状态： ${BUILD_STATUS}</li>    
                    <li>构建日志： <a href="${BUILD_URL}console">${BUILD_URL}console</a></li>    
                    <li>构建  Url ： <a href="${BUILD_URL}">${BUILD_URL}</a></li>    
                    <li>工作目录 ： <a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>    
                    <li>项目  Url ： <a href="${PROJECT_URL}">${PROJECT_URL}</a></li>    
                </ul>    
            </td>    
        </tr>    
    </table>    
</body>    
</html>
```

效果如下:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/msg-set-4.png)

### 2.5 单人邮箱通知设置:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/msg-set-5.png)


## 3. Jenkins服务发布配置:

### 3.1 创建项目:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/job-set-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/job-set-2.png)

### 3.2 创建项目用户分配项目权限:
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/adduser-set-1.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/adduser-set-2.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/adduser-set-3.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/adduser-set-4.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/adduser-set-5.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/adduser-set-6.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/adduser-set-7.png)
![](https://github.com/DevOps-m/ops-docs/blob/master/docs/images/Jenkins/adduser-set-8.png)
