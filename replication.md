

## MySQL replication types and formats
** Based on behavior, there are three types of MySQL replication, and they are synchronous, semisynchronous, and asynchronous. **

### Synchronous
In synchronous replication, the primary server waits for the replica servers to confirm that it has
received and applied the changes before committing. It ensures that all the changes made on the
primary server are immediately copied to the replica servers before committing. This ensures that
all the replicas are up to date, which provides strong data consistency. However, using this
method could lead to an impact on the performance and response time of the primary server and
limit the scalability of the system.


### Semi-synchronous
Semi-synchronous replication provides a compromise between synchronous and asynchronous
replication. It allows the primary server to commit without waiting for all the replica servers to
confirm the changes. However, it makes sure that at least one of the replica servers has
confirmed that it has received and applied the changes before it commits the transaction. This
reduces the risk of data loss that using asynchronous replication could cause and offers better
performance than synchronous. There is still a possibility of data loss if the replica server with
the data fails before other replica servers receive the data.


### Asynchronous
This is MySQL’s default and most used form of replication. Here, the primary server commits
the transaction immediately without waiting for confirmation from the replica servers. There is
no assurance that the data on the replica servers is always up to date with the primary server.
This can lead to the possibility of data loss and weaker data consistency. Asynchronous
replication is, however, faster and has increased scalability, as the primary server has no need to
wait for the replica servers



## Formats
** Formats are simply the different ways that events in the binary log are recorded given the type of event. The formats are statement-based, row-based, and mixed. **

### Statement-based
Statement-based replication copies the SQL statements executed on the primary server to the
replica server by recording each SQL statement that modifies data (INSERT, UPDATE,
DELETE) on the binary log of the primary server and then executing them on the replica servers.
This type of replication is the default mode in MySQL 5.7 and earlier versions. However, it has
some limitations, such as potential issues with non-deterministic functions and trigger-based
operations


### Row-based
Row-based replication copies the actual changes made to individual rows in the primary database
to the replica servers. Instead of replicating the SQL statements, the primary server records the
before and after values of each modified row in the binary log. The replica server then applies
the same changes to its own copy of the database. Row-based replication provides more precise
replication, especially when dealing with non-deterministic functions, triggers, and stored
procedures. This is the default mode in MySQL 8.0 and offers better compatibility and reliability
compared to statement-based replication.


### Mixed Mode
Mixed Mode replication is a combination of statement-based replication and row-based
replication. It allows the server to choose the appropriate replication method based on the nature
of the SQL statement. Most SQL statements are replicated using statement-based replication,
while some statements that cannot be safely logged using statement-based replication (e.g., nondeterministic functions) are replicated using row-based replication. Mixed Mode replication
provides flexibility and is the default mode in versions of MySQL between 5.7 and 8.0.


## Setting Up MySQL Replication – Step by Step:

### Configure Primary Server:
 - Edit the MySQL configuration file on the primary server (my.cnf or my.ini) to enable binary logging
 - Restart the MySQL service.
 - Log in to MySQL on the primary server and create a dedicated replication user with the necessary privileges.
 - 
 
 
## Advantages
There are many advantages of MySQL replication, but a few highlights include the enhancement
of scalability by allowing read-intensive workloads to be distributed across multiple replicas,
offloading a primary database server, and improving overall performance. Replication also
provides for data availability by enabling failover to a replica in case the primary server becomes
unavailable.
Lastly, in the event of a catastrophic scenario, data and databases can be quickly recovered, as
replication provides for geographically dispersed duplicates of data.


## Disadvantages
Despite all the benefits, there are some potential drawbacks associated with MySQL replication
one can face. One very common issue is data consistency, especially in setups with high write
activity. Replicas may lag behind the primary server, impacting any applications that depend on
real-time data. Another concern is the risk of single points of failure. If the primary server experiences a failure,
the entire replication process could be disrupted. Implementing the failover mechanisms
discussed earlier can mitigate this risk.
Regarding security, data transmitted between the primary and replica servers may be vulnerable
if encryption and access controls are not correctly configured.


#### Step 1 — Installing MySQL (Both Server)
```shell 
 #sudo apt update
 #sudo apt install mysql-server
 #sudo systemctl start mysql.service
 #sudo mysql_secure_installation
```
## Configure Master to Slave Asynchronous

#### Check 3306 port enable or not 
```shell 
sudo ufw allow from any to any port 3306;
```

#### Check network for master to slave connected or not 
```shell 
telnet 192.168.25.134 3306
```

#### If you make slave clone from master.
if master and server uuid() is same:
```shell 
sudo systemctl stop mysql
sudo rm /var/lib/mysql/auto.cnf
sudo systemctl start mysql
```


### Master Configure
```shell
sudo vim /etc/mysql/mysql.conf.d/mysql.cnf

#bind-address = 127.0.0.1

server-id = 1
log_bin = /var/log/mysql/mysql-bin.log 
binlog_do_db = chat //database name for replication
```
**Note: Restart your MySQL Server**

#### Create User For replication
```sql 
create user 'replica'@'slave host ip address' identified with mysql_native_password by 'replica';
grant replication slave on *.* to 'replica'@'slave host ip address';
flush privileges;
```

#### Create User For replication
```sql 
SHOW MASTER STATUS \G;
```


### Slave Configure
```shell 
sudo vim /etc/mysql/mysql.conf.d/mysql.cnf

#bind-address = 127.0.0.1

server-id = 2
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = chat //database name for replication
relay-log = /var/log/mysql/relay.log
```
**Note: Restart your MySQL Server**

### Slave Source change 
```sql
change replication source to 
source_user='replication', 
source_password='password', 
source_host='master ip address',
source_log_file='mysql-bin.00001',
source_log_pos=867; //867 and  mysql-bin.00001 from master status; 
```

### Start Slave 
```sql 
START SLAVE;

SHOW SLAVE STATUS \G; //for check status
```

### If Has Any Error in Slave 
```text
Last_IO_Error: error connecting to master ‘<Replica Username>@<IP>:3306’ — retry-time: 60
retries: 1 message: Authentication plugin ‘caching_sha2_password’ reported error: Authentication
requires secure connection. 
```

**In this case, you have to reset the slave and need to reconfigure it.**
```sql
STOP SLAVE;
RESET SLAVE;
RESET SLAVE ALL;
CHANGE MASTER TO GET_MASTER_PUBLIC_KEY=1;
CHANGE MASTER TO MASTER_HOST=’<Master IP>’,MASTER_USER=’<Replica
Username>’,MASTER_PASSWORD=<Replica User Password>’,MASTER_LOG_FILE=’mysqlbin.000001',MASTER_LOG_POS=891;
START SLAVE;

SHOW SLAVE STATUS \G;
```



## How to Change Replication Format /Statement-based /Row based
**Show replication format**
```sql 
select @@binlog_format
```

**Change format**
```sql 
set global binlog_format='ROW'
set global binlog_format="STATEMENT"
```

**Check Binlog files**
```sql 
cd /var/lib/mysql ls -lrth
```

**Check Binlog file Data**
```sql 
mysqlbinlog --base64-output=decode-rows --vv mysql-bin.000003 | less
```



## Configure Master to Slave Semi-synchronous

**Dependency**
```sql 
select @@have_dynamic_loading;
```
**Note: If yes that is okay for semi-synchronous**


**If you want to plugin list**
```sql 
show plugin;
```

### Master Server Configure

Install plugin for semi-synchronous
```sql 
install plugin rpl_semi_sync_master SONAME 'semisync_master.so';
```

**Enable the plugin**
```sql 
SET GLOBAL rpl_semi_sync_master_enabled=1;
```

**Show the plugin enabled/disabled**
```sql 
select @@rpl_semi_sync_master_enabled;
select @@rpl_semi_sync_master_timeout;
```


### Slave Server Configure

Install plugin for semi-synchronous
```sql 
install plugin rpl_semi_sync_slave SONAME 'semisync_slave.so';
```

**Enable the plugin**
```sql 
SET GLOBAL rpl_semi_sync_slave_enabled=1;
```

**Restart IO_THREAD**
```sql 
stop slave IO_THREAD;
start slave IO_THREAD;
```

**Show Slave Commands**
```sql 
show slave status \G;
system clear;
```


**for clear terminal**
```sql 
system clear;
```