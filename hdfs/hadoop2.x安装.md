由于hadoop1.x不支持yarn，咱们部署一个2.x版本，步骤和1.x大差不差，配置文件稍微有些许区别

**环境**

CentOS 7.5(3台，一主两从)
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
修改hosts配置(3台机器)
```
cat >> /etc/hosts << EOF
172.16.0.4 	master
172.16.0.17     slave1
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

## Ⅳ、安装jdk 1.8
**master**
```
tar zxvf /usr/local/src/jdk-8u201-linux-x64.tar.gz -C /usr/local

分发至slave
scp -pr /usr/local/jdk1.8.0_201 slave1:/usr/local/
scp -pr /usr/local/jdk1.8.0_201 slave2:/usr/local/
```
**配置环境变量(3台机器)**
```
cat >> /etc/profile << EOF
export JAVA_HOME=/usr/local/jdk1.8.0_201/
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin
EOF
source /etc/profile
```
执行命令java 看输出 review


## ⅴ、安装hadoop
**master**
```
tar zxvf /usr/local/src/hadoop-2.6.1.tar.gz -C /usr/local
cd /usr/local/hadoop-2.6.1 && mkdir -p tmp dfs/name dfs/data
cd /usr/local/hadoop-2.6.1/etc/hadoop/
```
**修改7个配置文件**
- slaves
```
slave1
slave2
```
- core-site.xml
```
<configuration>
	<property>
		<name>fs.defaultFS</name>
		<value>hdfs://172.16.0.4:9000</value>
	</property>
	<property>
		<name>hadoop.tmp.dir</name>
		<value>file:/usr/local/hadoop-2.6.1/tmp/</value>
	</property>
</configuration>
```
- mapred-site.xml
```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```
- hdfs-site.xml
```
<configuration>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>master:9001</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/hadoop-2.6.1/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/hadoop-2.6.1/dfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
</configuration>
```
- yarn-site.xml
```
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>master:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>master:8030</value>
    </property>
        <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>master:8035</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address</name>
        <value>master:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>master:8088</value>
    </property>
    <!-- 关闭虚拟内存检查-->
    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
</configuration>
```
- hadoop-env.sh
```
export JAVA_HOME=/usr/local/jdk1.8.0_201/
```
- yarn-env.sh
```
export JAVA_HOME=/usr/local/jdk1.8.0_201/
```
**将程序目录分发到slave**
```
scp -pr /usr/local/hadoop-2.6.1/ slave1:/usr/local/
scp -pr /usr/local/hadoop-2.6.1/ slave2:/usr/local/
```
**配置环境变量(3台机器)**
```
cat >> /etc/profile << EOF
HADOOP_HOME=/usr/local/hadoop-2.6.1
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
/usr/local/hadoop-2.6.1/sbin/start-all.sh
```
**查看集群状态(三个机器都可以查看)**
```
master
[root@master ~]# jps
16707 SecondaryNameNode
16854 ResourceManager
16524 NameNode
21102 Jps

slave
[root@slave1 ~]# jps
32432 NodeManager
32323 DataNode
4122 Jps
```
**验证**
```
[root@slave1 ~]# hadoop fs -ls /
[root@slave1 ~]# hadoop fs -put /etc/passwd /
[root@slave1 ~]# hadoop fs -cat /passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
...
```
也可以在浏览器访问8088端口查看监控web页面

**关闭hadoop集群**
```
/usr/local/hadoop-2.6.1/sbin/stop-all.sh
```

**tips:**
2版本的启动关闭脚本默认已经不在bin目录中，所以需要用绝对目录操作
