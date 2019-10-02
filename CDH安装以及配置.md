# CDH 安装及配置

---

## 安装之前

### 1. 配置主机名

    sudo hostnamectl set-hostname tw-node1
    cat /etc/hostname

### 2. 注销回路 ip, **否则无法启动 manager server**

    sudo vim /etc/hosts
    #127.0.0.1   tw-node1
    添加
    218.25.161.235  tw-node1

### 3. 关闭防火墙

    sudo service ufw stop
    sudo ufw disable

### 4. 启动配置 NTP 服务

    vim /etc/ntp.conf
    ntpdate -u ntp1.aliyun.com
    hwclock --systohc

### 5. 配置 ssh,**之后选择秘钥认证方式**

<br><br>
**root 用户**

> ssh-keygen -t rsa<br>

    cat id_rsa.pub >> authorized_keys<br>
    无法远程访时修改 /etc/ssh/sshd_config<br>
    修改为**PermitRootLogin yes**<br>
    之后重启服务

### 6. 关闭 SELinux

> getenforce 如果为 Permissive or Disabled,跳过此步骤<br>
>
> > 否则 sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config<br>

    setenforce 0

---

## 正式安装(on ubuntu)

### 1. 设置存储库

> 从[cloudCloudera Manager 6 Version and Download Information](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_cm_6_version_download.html)<br>
> 选择你要下载的版本<br>
> wget https://archive.cloudera.com/cm6/6.3.0/ubuntu1804/apt/cloudera-manager.list -P /etc/apt/sources.list.d/

### 2. 导入 GPG 秘钥

> wget https://archive.cloudera.com/cm6/6.3.0/ubuntu1604/apt/archive.key<br>

    sudo apt-key add archive.key
     sudo apt-get update

### 3. 安装 jdk

> sudo apt-get install openjdk-8-jdk

### 4. 安装 Cloudera Manager

> sudo apt-get install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server

### 5. 安装数据库

此处为**mariadb**<br>

> sudo apt-get install mariadb-server

    将/var/lib/mysql/ib_logfile0 和 /var/lib/mysql/ib_logfile1 删除
    确认my.cnf(一般存在/etc/mysql/my.cnf下或者/etc/my.cnf)位置

推荐配置

    [mysqld]
    datadir=/var/lib/mysql
    socket=/var/lib/mysql/mysql.sock
    transaction-isolation = READ-COMMITTED
    # Disabling symbolic-links is recommended to prevent assorted security risks;
    # to do so, uncomment this line:
    symbolic-links = 0
    # Settings user and group are ignored when systemd is used.
    # If you need to run mysqld under a different user or group,
    # customize your systemd unit file for mariadb according to the
    # instructions in http://fedoraproject.org/wiki/Systemd

    key_buffer = 16M
    key_buffer_size = 32M
    max_allowed_packet = 32M
    thread_stack = 256K
    thread_cache_size = 64
    query_cache_limit = 8M
    query_cache_size = 64M
    query_cache_type = 1

    max_connections = 550
    #expire_logs_days = 10
    #max_binlog_size = 100M

    #log_bin should be on a disk with enough free space.
    #Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your
    #system and chown the specified folder to the mysql user.
    log_bin=/var/lib/mysql/mysql_binary_log

    #In later versions of MariaDB, if you enable the binary log and do not set
    #a server_id, MariaDB will not start. The server_id must be unique within
    #the replicating group.
    server_id=1

    binlog_format = mixed

    read_buffer_size = 2M
    read_rnd_buffer_size = 16M
    sort_buffer_size = 8M
    join_buffer_size = 8M

    # InnoDB settings
    innodb_file_per_table = 1
    innodb_flush_log_at_trx_commit  = 2
    innodb_log_buffer_size = 64M
    innodb_buffer_pool_size = 4G
    innodb_thread_concurrency = 8
    innodb_flush_method = O_DIRECT
    innodb_log_file_size = 512M

    [mysqld_safe]
    log-error=/var/log/mariadb/mariadb.log
    pid-file=/var/run/mariadb/mariadb.pid

    #
    # include all files from the config directory
    #
    !includedir /etc/my.cnf.d

**更改 mariadb 数据库密码模式并设置数据库远程访问权限**

    update mysql.user set authentication_string=PASSWORD('newPwd'), plugin='mysql_native_password' , host='%' where user='root';

    flush privileges;
    注释/etc/mysql/mariadb.conf.d中bind-address


    重启数据库
    sudo systemctl enable mariadb
    sudo systemctl restart mariadb

执行安全设置

    sudo /usr/bin/mysql_secure_installation

    [...]
    Enter current password for root (enter for none):
    OK, successfully used password, moving on...
    [...]
    Set root password? [Y/n] Y
    New password:
    Re-enter new password:
    [...]
    Remove anonymous users? [Y/n] Y
    [...]
    Disallow root login remotely? [Y/n] N
    [...]
    Remove test database and access to it [Y/n] Y
    [...]
    Reload privilege tables now? [Y/n] Y
    [...]
    All done!  If you've completed all of the above steps, your MariaDB
    installation should now be secure.

    Thanks for using MariaDB!

### 6. 安装 MySQL JDBC Driver for MariaDB

    >sudo apt-get install libmysql-java

### 7. 创建 Cloudera 数据库

| Service                            | Database  | User   |
| ---------------------------------- | :-------: | ------ |
| Cloudera Manager Server            |    scm    | scm    |
| Activity Monitor                   |   amon    | amon   |
| Reports Manager                    |   rman    | rman   |
| Hue                                |    hue    | hue    |
| Hive Metastore Server              | metastore | hive   |
| Sentry Server                      |  sentry   | sentry |
| Cloudera Navigator Audit Server    |    nav    | nav    |
| Cloudera Navigator Metadata Server |   navms   | navms  |
| Oozie                              |   oozie   | oozie  |

    CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE  utf8_general_ci;
    GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'wzlinux';
    CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE     utf8_general_ci;
    GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'wzlinux';
    CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE     utf8_general_ci;
    GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'wzlinux';
    CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'wzlinux';
    CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE    utf8_general_ci;
    GRANT ALL ON metastore.* TO 'hive'@'%' IDENTIFIED BY 'wzlinux';
    CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE   utf8_general_ci;
    GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'wzlinux';
    CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'wzlinux';
    CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE    utf8_general_ci;
    GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'wzlinux';
    CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE    utf8_general_ci;
    GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'wzlinux';

执行

    mysql -uroot -p<ROOT_PASSWORD> < ./cdh.sql

证实

    SHOW DATABASES;
    SHOW GRANTS FOR '<user>'@'%';

### 8. Cloudera Manager 数据库设置

> 格式为 sudo /opt/cloudera/cm/schema/scm_prepare_database.sh [options] \<databaseType> \<databaseName> \<databaseUser> \<password>  
>  如果存在`sudo rm /etc/cloudera-scm-server/db.mgmt.properties`

    否则`sudo /opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm scm`

```
 Enter SCM password:
 JAVA_HOME=/usr/java/jdk1.8.0_141-cloudera
 Verifying that we can write to /etc/cloudera-scm-server
 Creating SCM configuration file in /etc/cloudera-scm-server
 Executing:  /usr/java/jdk1.8.0_141-cloudera/bin/java -cp /usr/share/java/  mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/   java/postgresql-connector-java.jar:/opt/cloudera/cm/schema/../lib/*    com.cloudera.enterprise.dbutil.DbCommandExecutor /etc/cloudera-scm-server/ db.properties com.cloudera.cmf.db.
 [main] DbCommandExecutor INFO Successfully connected to database.
 All done, your SCM database is configured correctly!
```

### 9. 安装 CDH 和其他软件

> sudo systemctl start cloudera-scm-server  
> 查看日志  
> sudo tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log  
> 出现  
>  `INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.`

    访问
    http://<server_host>:7180

**默认账户密码 admin admin**  
<br>

## CDH 常见问题解决

---

### 1.当伪分布式搭建时提示 zookeeper 和 hdfs 节点数不足 3 果断抑制

### 2.节点少合理化设置资源

> 注意`NameNode 的 Java 堆栈大小（字节）`和`Secondary NameNode 的 Java 堆栈大小（字节`大小一致并适度调节使用内存

### 3.副本不足的块

> 查看本集群**datanode**数量,将复制因子(dfs.replication)修改为与**datanode**一致的数量 <br>
> 如果严重警告依然存在  
> 也可以直接主节点执行命令`sudo -u hdfs hadoop fs -setrep -R n /`,**n**为**datanode**数量

<!-- ### 3. -->

<br>

## kafka 常见问题解决

---

### 1.注意 kafka 备份因子 `replication-factor` 数量,不能小于设置并且与 broker 大小关系

### 2.如果设置`zookeeper.chroot`=`dir` 命令启动内 `kafka-topics --create --zookeeper zkinfo --replication-factor 1 --partitions 1 --topic test`中 `zkinfo`为`host:port/dir`

<br>

## spark 常见问题解决

---

### 1.
