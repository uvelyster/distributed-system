# Prerequisites

- docker 구성
- mysql:8.0.28 (latest) 이미지

### Prepare 4 node of mysql

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

plugin_dir=/usr/lib/mysql/plugin/
server_id=1
gtid_mode=ON
enforce_gtid_consistency=ON
binlog_checksum=NONE
transaction_write_set_extraction=XXHASH64
loose-group_replication_recovery_use_ssl=ON
loose-group_replication_group_name="a9d32ca8-a483-4f18-b705-ef02c8edb379"
loose-group_replication_start_on_boot=OFF
loose-group_replication_local_address="mysql1:13306"
loose-group_replication_group_seeds="mysql1:13306,mysql2:13306,mysql3:13306,mysql4:13306"
loose-group_replication_bootstrap_group=OFF
EOF
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

plugin_dir=/usr/lib/mysql/plugin/
server_id=2
gtid_mode=ON
enforce_gtid_consistency=ON
binlog_checksum=NONE
loose-group_replication_recovery_get_public_key=ON
loose-group_replication_recovery_use_ssl=ON
loose-group_replication_group_name="a9d32ca8-a483-4f18-b705-ef02c8edb379"
loose-group_replication_start_on_boot=OFF
loose-group_replication_local_address="mysql2:13306"
loose-group_replication_group_seeds="mysql1:13306,mysql2:13306,mysql3:13306,mysql4:13306"
loose-group_replication_bootstrap_group=OFF
EOF
exit

```

```bash
docker run --ip 172.30.0.103 \
 --network demo-net \
 --name mysql3 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker exec -it mysql3 bash

cat << EOF > /etc/mysql/my.cnf
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL

plugin_dir=/usr/lib/mysql/plugin/
server_id=3
gtid_mode=ON
enforce_gtid_consistency=ON
binlog_checksum=NONE
loose-group_replication_recovery_get_public_key=ON
loose-group_replication_recovery_use_ssl=ON
loose-group_replication_group_name="a9d32ca8-a483-4f18-b705-ef02c8edb379"
loose-group_replication_start_on_boot=OFF
loose-group_replication_local_address="mysql3:13306"
loose-group_replication_group_seeds="mysql1:13306,mysql2:13306,mysql3:13306,mysql4:13306"
loose-group_replication_bootstrap_group=OFF
EOF
exit

```

```bash
docker run --ip 172.30.0.104 \
 --network demo-net \
 --name mysql4 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker exec -it mysql4 bash

cat << EOF > /etc/mysql/my.cnf
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL

plugin_dir=/usr/lib/mysql/plugin/
server_id=4
gtid_mode=ON
enforce_gtid_consistency=ON
binlog_checksum=NONE
loose-group_replication_recovery_get_public_key=ON
loose-group_replication_recovery_use_ssl=ON
loose-group_replication_group_name="a9d32ca8-a483-4f18-b705-ef02c8edb379"
loose-group_replication_start_on_boot=OFF
loose-group_replication_local_address="mysql4:13306"
loose-group_replication_group_seeds="mysql1:13306,mysql2:13306,mysql3:13306,mysql4:13306"
loose-group_replication_bootstrap_group=OFF
EOF
exit

```

```bash
docker ps 
docker restart mysql1 mysql2 mysql3 mysql4

```

```bash

docker exec -it mysql1 bash 
mysql -uroot -proot
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
SELECT * FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME = 'group_replication' \G
exit
exit

```

```bash

docker exec -it mysql2 bash 
mysql -uroot -proot 
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
SELECT * FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME = 'group_replication' \G
exit
exit

```

```bash

docker exec -it mysql3 bash 
mysql -uroot -proot 
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
SELECT * FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME = 'group_replication' \G
exit
exit

```

```bash

docker exec -it mysql4 bash 
mysql -uroot -proot 
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
SELECT * FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME = 'group_replication' \G
exit
exit

```

```bash

cat << EOF > create_rpl_user.sql
SET SQL_LOG_BIN=0;
CREATE USER 'rpl_user'@'%' IDENTIFIED WITH mysql_native_password BY 'rpl_pass';
GRANT REPLICATION SLAVE ON *.* TO 'rpl_user'@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
EOF

docker cp create_rpl_user.sql mysql1:/
docker cp create_rpl_user.sql mysql2:/
docker cp create_rpl_user.sql mysql3:/
docker cp create_rpl_user.sql mysql4:/

```

```bash

docker exec mysql1 bash -c 'mysql -uroot -proot -e "source create_rpl_user.sql"'
docker exec mysql2 bash -c 'mysql -uroot -proot -e "source create_rpl_user.sql"'
docker exec mysql3 bash -c 'mysql -uroot -proot -e "source create_rpl_user.sql"'
docker exec mysql4 bash -c 'mysql -uroot -proot -e "source create_rpl_user.sql"'

```

```bash
docker exec -it mysql1 bash
mysql -uroot -proot
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
exit
exit

```

```bash
docker exec -it mysql2 bash
mysql -uroot -proot
CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_pass' FOR CHANNEL 'group_replication_recovery';
RESET MASTER;
START GROUP_REPLICATION;
exit
exit

```

```bash
docker exec -it mysql3 bash
mysql -uroot -proot
CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_pass' FOR CHANNEL 'group_replication_recovery';
RESET MASTER;
START GROUP_REPLICATION;
exit
exit

```

```bash
docker exec -it mysql4 bash
mysql -uroot -proot
CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_pass' FOR CHANNEL 'group_replication_recovery';
RESET MASTER;
START GROUP_REPLICATION;
exit
exit

```

```bash

docker exec -it mysql1 bash 
mysql -uroot -proot

SELECT * FROM performance_schema.replication_group_members \G;

SELECT MEMBER_HOST, MEMBER_PORT, MEMBER_STATE, MEMBER_ROLE FROM 
performance_schema.replication_group_members;
SELECT member_id, member_host, member_port \
FROM performance_schema.global_status \
JOIN performance_schema.replication_group_members \
ON VARIABLE_VALUE=member_id \
WHERE VARIABLE_NAME='group_replication_primary_member';

exit
exit

```

```bash
docker exec -it mysql2 bash
mysql -uroot -proot
CREATE DATABASE test;
exit
exit

```

```bash
docker exec -it mysql1 bash
mysql -uroot -proot
CREATE DATABASE test;
USE test;
CREATE TABLE test.t1 (id INT PRIMARY KEY, message TEXT NOT NULL);
INSERT INTO test.t1 VALUES (1, 'Chuck is here');
exit
exit

docker exec -it mysql2 bash
mysql -uroot -proot -e "SELECT * FROM test.t1"
```

```bash
docker stop mysql1 
docker exec -it mysql2 bash
mysql -uroot -proot
SELECT * FROM performance_schema.replication_group_members \G;
SELECT * FROM test.t1;
```

### Clean up

```bash
docker rm -fv mysql1 mysql2 mysql3 mysql4
```
