
# Prerequisites

- docker 구성
- InnoDB Cluster 구성

```bash
docker exec -it client sh -c mysqlsh
\c root:root@mysql-router:6446
\sql 
CREATE DATABASE shopping;

# DROP TABLE IF EXISTS shopping.list;

CREATE TABLE shopping.list (
rowid int AUTO_INCREMENT PRIMARY KEY,
description char(200),
note char(100),
purchased int DEFAULT '0' 
);

INSERT INTO shopping.list VALUES 
(NULL, "우유", "2L 저지방", NULL);

INSERT INTO shopping.list VALUES
(NULL, "기저귀", "3단계.", NULL);

```

```bash
mysqldump -p -h mysql-router -P 6447 \
 --all-databases \
 --routines \
 --events \
 --single-transaction \
 --flush-privileges \
 --hex-blob \
 --log-error=/var/log/mysqldump.log \
 --result-file=./backup.sql

docker exec -it client sh -c mysqlsh

\c root:root@mysql-router:6446
\sql
 
STOP GROUP_REPLICATION;
SET SQL_LOG_BIN=0;
RESET MASTER; 
RESET REPLICA;
SET GLOBAL super_read_only=0;
source backup.sql
SET SQL_LOG_BIN=1;

# check
var cls=dba.getCluster()
cls.status()
cls.rejoinInstance("mysqlX:3306")
cls.status()
```
