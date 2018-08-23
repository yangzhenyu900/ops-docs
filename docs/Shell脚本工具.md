# Shell脚本工具:


## 环境信息:
- OS:          CentOS Linux release 7.4.1708 (Core)
- Kernel:      3.10.0-693.el7.x86_64

## 1. 服务检测脚本:

### 1.1 Java服务检测脚本:
- 检测机: 10.11.40.55
```
[root@scripts ~]# cat /server/scritps/check_alpha_api_status.sh 
#!/bin/bash
array=(
	"http://apis-alpha.myunsheji.com/pms/version"
	"http://pms-alpha.myunsheji.com/version.txt"
)

check_url() {
	for ((i=0;i<${#array[@]};i++))
	do
		sleep 30;
		Check=$(curl -I -s --connect-timeout 10 -w "%{http_code}" ${array[i]} -o /dev/null)
		if [ "${Check[0]}" != "200" ];then
			echo "${array[i]} is Failure."|mail -s "${array[i]}" sjy-alarm@chinaredstar.com
		fi
	done
}

main() {
	check_url
}
main
[root@scripts ~]# 
```

### 1.2 检测计划任务:
```
#API Check Status
*/5 * * * * /bin/bash /server/scritps/check_alpha_api_status.sh >/dev/null 2&>1
```

### 1.3 服务检测脚本:
- 检测机: 10.11.40.60

#### 1.3.1 MySQL服务HA检测脚本:
- 此脚本交给keepalived服务进行触发检测;
```
[root@jira ~]# cat /server/scripts/ha_check.sh 
#!/bin/bash
# MySQL Master <--> Slave HA Check Scripts

Counter=$(ps -C mysqld --no-heading|wc -l)
if [ "${Counter}" = "0" ];then
    systemctl stop keepalived
	RETVAL=$?
	if [ $RETVAL -eq 0 ];then
      echo "MySQL is Down , VIP is Switch."|mail -s "MySQL Master <--> Slave HA Check" li.zhiqiang@chinaredstar.com
    fi
fi
[root@jira ~]#
```

#### 1.3.2 keepalived服务裂脑检测脚本:
- 检测思路: 作为备机,如果master节点还存活,vip就应该存活在master节点上,如果master节点存活且vip在备节点上,则裂脑;
```
[root@jira ~]# cat /server/scripts/check_split_brain.sh 
#!/bin/bash

Check1() {
	Master_IP="10.11.40.58"
	ping -W 3 -c 2 $Master_IP &>/dev/null
	RETVAL1=$?
}

Check2() {
	Vip="10.11.40.200"
	/sbin/ip a|grep $Vip &>/dev/null
	RETVAL2=$?
}

Judge() {
	if [ $RETVAL1 -eq 0 -a $RETVAL2 -eq 0 ];then
		echo "Error,HA is split brain."|mail -s "HA Check." li.zhiqiang@chinaredstar.com
	fi
}

main() {
	while true
	do
		Check1
		Check2
		Judge
		sleep 5
	done
}

main
[root@jira ~]# 
```

#### 1.3.3 wiki和jira服务检测脚本:
- 检测思路: 如果服务宕机则进行启动;
```
[root@jira ~]# cat /server/scripts/check_service_status.sh 
#!/bin/bash 
echo " "
echo ">>>>>>>>>>>> Check wiki/jira Process Status >>>>>>>>>>>>"
echo " "

# ENV HostName
HostName=$(/bin/hostname|awk -F'.' '{print $1}')

# Date Time
Date=$(date "+%Y-%m-%d %H:%M:%S")

# Service Start
Start_WIKI() {
	WIKI_PID=$(cat /usr/local/confluence/work/catalina.pid)
    if [ -z "$WIKI_PID" ];then
        echo "$Date ---> Wiki is DOWN,Wiki is Starting..." | tee >> /tmp/wiki_out.log
        su - confluence -c "/usr/local/confluence/bin/start-confluence.sh"
    fi
}

Start_JIRA() {
	JIRA_PID=$(cat /usr/local/jira/work/catalina.pid)
    if [ -z "$JIRA_PID" ];then
        echo "$Date ---> Jira is DOWN,Jira is Starting..." | tee >> /tmp/jira_out.log
        su - jira -c "/usr/local/jira/bin/start-jira.sh"
    fi
}


main() {
    if [ "$HostName" == "wiki" ];then
        Start_WIKI;
    else
        Start_JIRA;
    fi
}
main
[root@jira ~]# 
```

## 2. 备份脚本:

### 2.1.1 数据库备份脚本:
- 备份机: 10.11.40.60
#### 2.1.1 备份脚本:
```
[root@jira ~]# cat /var/job/tools-mysql-backup.sh
#!/bin/bash

export MYSQL_HOME="/usr/local/mysql"
export PATH=$MYSQL_HOME/bin:$PATH

HOST="10.11.40.60"
PORT="3306"
MySQL_USER="root"
MySQL_PASS="xxxxxx"
Defaults_File="/etc/my.cnf"
MySQL_Backup_Dir="/mysqlbackup/"
MySQL_Socket="/data/mysql/data/mysql.sock"

TimeStart=$(date '+%Y%m%d%H%M%S')
LogFile="/var/log/cronjob/full-${TimeStart}.log"

NoticeUser="sjy-alarm@chinaredstar.com"

cd $MySQL_Backup_Dir && \
innobackupex --defaults-file=$Defaults_File --host=$HOST --port=$PORT --use-memory=4G --user=$MySQL_USER --password=$MySQL_PASS $MySQL_Backup_Dir &> "$LogFile"

#Mail Notice
if [ -f "$LogFile" ];then
    mail -s "tools-mysql-backup is SUCCESS." "$NoticeUser" < "$LogFile"
else
    echo "tools-mysql-backup is FAILURE." | mail -s "tools-mysql-backup" "$NoticeUser"
fi

#save Last 3 days
cd $MySQL_Backup_Dir && \
ls -t|awk '{if(NR>3) print $0}'|xargs rm -rf
[root@jira ~]#
```

#### 2.1.2 备份计划任务:
```
#Ansible: tools mysql wiki/jira backup
0 2 * * * /var/job/tools-mysql-backup.sh
```

### 2.2 GitLab备份脚本:
- 备份机: 10.11.40.56
#### 2.2.1 备份脚本:
```
[root@git ~]# cat /var/job/tools-gitlab-backup.sh 
#!/bin/bash

TimeStart=$(date '+%Y%m%d%H%M%S')
LogFile="/var/log/cronjob/tools-gitlab-backup.${TimeStart}.log"

NoticeUser="sjy-alarm@chinaredstar.com"


/usr/bin/gitlab-rake gitlab:backup:create &> "$LogFile"

#Mail Notice
if [ -f "$LogFile" ];then
    mail -s "tools-gitlab-backup is SUCCESS." "$NoticeUser" < "$LogFile"
else
    echo "tools-gitlab-backup is FAILURE." | mail -s "tools-gitlab-backup" "$NoticeUser"
fi
[root@git ~]# 
```

#### 2.1.2 备份计划任务:
```
#Ansible: tools gitlab backup
0 2 * * * /var/job/tools-gitlab-backup.sh
```
