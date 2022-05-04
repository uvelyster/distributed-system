# Prerequisites

- docker 구성
- mysql:8.0.28 (latest) 이미지

# Traditional Replication

### Prepare 2 node of mysql

```bash
docker network create -d bridge --subnet 172.30.0.0/24 demo-net

docker run --ip 172.30.0.101 \
 --network demo-net \
 --name mysql1 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker exec -it mysql1 bash

cat << EOF > /etc/mysql/my.cnf
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL

server_id       = 1
log_bin         = master_binlog
binlog_format   = row
EOF

cat << EOF > /create_rpl_user.sql
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'%' IDENTIFIED WITH mysql_native_password BY 'rpl_pass';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
EOF

mysql -uroot -proot -e "source create_rpl_user.sql"

mysql -uroot -proot -e "show grants for 'rpl_user'@'172.30.0.102';"

exit
```

```bash
docker run --ip 172.30.0.102 \
 --network demo-net \
 --name mysql2 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker exec -it mysql2 bash

cat << EOF > /etc/mysql/my.cnf
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL

server_id       = 2
log_bin         = slave1_binlog
binlog_format   = row
EOF

cat << EOF > /create_rpl_user.sql
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'172.30.0.102' IDENTIFIED WITH mysql_native_password BY 'rpl_pass';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'172.30.0.102';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
EOF

mysql -uroot -proot -e "source create_rpl_user.sql"

mysql -uroot -proot -e "show grants for 'rpl_user'@'172.30.0.102';"

exit

docker restart mysql1 mysql2
```

```bash
docker exec  mysql1 mysql -uroot -proot -e 'show master status;'

docker exec -it mysql2 bash
mysql -uroot -proot

CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_pass', \
MASTER_HOST='172.30.0.101', MASTER_PORT=3306, \
MASTER_LOG_FILE='master_binlog.000002', MASTER_LOG_POS=157;

start slave;
show slave status \G
exit
```

### Clean up

```bash
docker rm -fv mysql1 mysql2
```

# GTID Replication

```bash
docker network create -d bridge --subnet 172.30.0.0/24 db-net

docker run --ip 172.30.0.101 \
 --network db-net \
 -p 13306:3306 \
 --name mysql1 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker exec -it mysql1 bash

cat << EOF > /etc/mysql/my.cnf
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL

server_id       = 1
log_bin         = master_binlog
binlog_format   = row

# GTID VARIABLES
gtid_mode=on
enforce_gtid_consistency=on
log_slave_updates=on

EOF

cat << EOF > /create_rpl_user.sql
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'172.30.0.102' IDENTIFIED WITH mysql_native_password BY 'rpl_pass';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'172.30.0.102';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
EOF

mysql -uroot -proot -e "source create_rpl_user.sql"

mysql -uroot -proot -e "show grants for 'rpl_user'@'172.30.0.102';"

exit
```

```bash
docker run --ip 172.30.0.102 \
 --network db-net \
 -p 23306:3306 \
 --name mysql2 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker exec -it mysql2 bash

cat << EOF > /etc/mysql/my.cnf
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL

server_id       = 2
log_bin         = slave1_binlog
binlog_format   = row

# GTID VARIABLES
gtid_mode=on
enforce_gtid_consistency=on
log_slave_updates=on

EOF

cat << EOF > /create_rpl_user.sql
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'172.30.0.102' IDENTIFIED WITH mysql_native_password BY 'rpl_pass';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'172.30.0.102';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
EOF

mysql -uroot -proot -e "source create_rpl_user.sql"

mysql -uroot -proot -e "show grants for 'rpl_user'@'172.30.0.102';"

exit

docker restart mysql1 mysql2
```

```bash
docker exec  mysql1 mysql -uroot -proot -e 'show master status;'

docker exec -it mysql2 bash
mysql -uroot -proot

CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_pass', \
MASTER_HOST='172.30.0.101', MASTER_PORT=3306, \
MASTER_AUTO_POSITION = 1;

start slave;
show slave status \G
exit
exit
```

### Verify by Create Sample Data

```bash
docker exec -it mysql1 bash
mysql -uroot -proot 
```

```sql
CREATE DATABASE test;
USE test;
CREATE TABLE test.t1 (c1 INT PRIMARY KEY, c2 TEXT NOT NULL);
INSERT INTO test.t1 VALUES (1, 'Chuck');
SELECT * FROM test.t1;
exit
exit
```

```sql
docker exec -it mysql2 mysql -uroot -proot -e "SELECT * FROM test.t1"
```

### Clean up

```bash
docker rm -fv mysql1 mysql2
```
