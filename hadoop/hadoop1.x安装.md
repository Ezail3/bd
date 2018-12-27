**环境**

CentOS 7.5(三台，一主两从)
```
172.16.0.4  master
172.16.0.17 slave1
172.16.0.6  slave2
```
**安装包**
```
[root@master src]# pwd
/usr/local/src
[root@master src]# ll
total 107604
-rw-r--r-- 1 root root  38096663 Dec 19 23:23 hadoop-1.2.1-bin.tar.gz
-rwxr-xr-x 1 root root  72087592 Dec 27 16:33 jdk-6u45-linux-x64.bin
```
## Ⅰ、关闭防火墙
```
systemctl stop firewalld.service
systemctl disable firewalld.service
setenforce 0
sed -i '/^SELINUX=/cSELINUX=disabled' /etc/selinux/config
```
## Ⅱ、配置host
```
cat >> /etc/hosts << EOF
172.16.0.4 	master
172.16.0.17 slave1
172.16.0.6 	slave2
EOF
```
**master**
```
echo 'HOSTNAME=master' >> /etc/sysconfig/network
hostname master
```
**slave1**
```
echo 'HOSTNAME=slave1' >> /etc/sysconfig/network
hostname slave1
```
**slave2**
```
echo 'HOSTNAME=slave2' >> /etc/sysconfig/network
hostname slave2
```

## Ⅲ、SSH免密码通信
三台机器互ping host review机器是否通

**master**
```
ssh-keygen
ssh-copy-id master
ssh-copy-id slave1
ssh-copy-id slave2
```
**slave1**
```
ssh-keygen
ssh-copy-id master
ssh-copy-id slave1
ssh-copy-id slave2
```
**slave2**
```
ssh-keygen
ssh-copy-id master
ssh-copy-id slave1
ssh-copy-id slave2
```
三台机器互相ssh review

**tips:**
不要手动做auth文件再scp到每个机器，踩到坑了，没找到原因，用ssh-copy-id没问题

## Ⅳ、安装jdk
**master**
```
mv /usr/local/src/jdk-6u45-linux-x64.bin /usr/local/
cd /usr/local/ && ./jdk-6u45-linux-x64
分发至slave
scp -pr /usr/local/jdk1.6.0_45 slave1:/usr/local/
scp -pr /usr/local/jdk1.6.0_45 slave2:/usr/local/
```
**配置环境变量(3台机器)**
```
cat >> /etc/profile << EOF
export JAVA_HOME=/usr/local/jdk1.6.0_45/
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin
EOF
source /etc/profile
```
执行命令java 看输出 review

## ⅴ、安装hadoop
**master**
```
tar zxvf /usr/local/src/hadoop-1.2.1-bin.tar.gz -C /usr/local/
cd /usr/local/hadoop-1.2.1 && mkdir tmp
cd conf
```
**修改6个配置文件**
- masters
```
master
```
- slaves
```
slave1
slave2
```
- core-site.xml
```
<configuration>
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/usr/local/hadoop-1.2.1/tmp/</value>
	</property>
	<property>
		<name>fs.default.name</name>
		<value>hdfs://172.16.0.4:9000</value>
	</property>
</configuration>
```
- mapred-site.xml
```
<configuration>
	<property>
		<name>mapred.job.tracker</name>
		<value>http://172.16.0.4:9001</value>
	</property>
</configuration>
```
- hdfs-site.xml
```
<configuration>
	<property>
		<name>dfs.replication</name>
		<value>3</value>
	</property>
</configuration>
```
- hadoop-env.sh
```
export JAVA_HOME=/usr/local/jdk1.6.0_45/
```
**将程序目录分发到slave**
```
scp -pr /usr/local/hadoop-1.2.1 slave1:/usr/local/
scp -pr /usr/local/hadoop-1.2.2 slave1:/usr/local/
```
**配置环境变量**
```
cat >> /etc/profile << EOF
HADOOP_HOME=/usr/local/hadoop-1.2.1
export PATH=$PATH:$HADOOP_HOME/bin
EOF
source /etc/profile
```

## Ⅵ、玩起来
**初始化NameNode**
```
hadoop namenode -format
```
**启动hadoop集群**
```
start-all.sh
```
**查看集群状态(三个机器都可以查看)**
```
[root@master hadoop-1.2.1]# jps
11596 SecondaryNameNode
12804 Jps
11415 NameNode
11692 JobTracker
```
**验证**
```
[root@master hadoop-1.2.1]# hadoop fs -ls /
Found 1 items
drwxr-xr-x   - root supergroup          0 2018-12-27 17:38 /usr
[root@master hadoop-1.2.1]# hadoop fs -put /etc/passwd /
[root@master hadoop-1.2.1]# hadoop fs -ls /
Found 2 items
-rw-r--r--   3 root supergroup       1245 2018-12-27 17:40 /passwd
drwxr-xr-x   - root supergroup          0 2018-12-27 17:38 /usr
[root@master hadoop-1.2.1]# hadoop fs -cat /passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
...
```
**关闭hadoop集群**
```
stop-all.sh
```
