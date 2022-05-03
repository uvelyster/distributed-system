# Prerequisites

- docker 구성
- mysql:8.0.28 (latest) 이미지
- mysql/mysql-server 이미지
- mysql/mysql-router 이미지

### Prepare 4 node of InnoDB Cluster

```bash
docker run --ip 172.30.0.101 \
 --network demo-net \
 --name mysql1 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker run --ip 172.30.0.102 \
 --network demo-net \
 --name mysql2 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker run --ip 172.30.0.103 \
 --network demo-net \
 --name mysql3 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker run --ip 172.30.0.104 \
 --network demo-net \
 --name mysql4 -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql

docker run --ip 172.30.0.105 \
 --network demo-net \
 --name client -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql/mysql-server

```

```bash

docker exec -it client sh -c mysqlsh

\c root:root@mysql1:33060

```

```bash

dba.checkInstanceConfiguration()
dba.configureInstance()

# other terminal
docker restart mysql1

dba.checkInstanceConfiguration()
dba.createCluster('mycluster')

```

```bash

\c root:root@mysql2:33060
dba.configureInstance()

\c root:root@mysql3:33060
dba.configureInstance()

\c root:root@mysql4:33060
dba.configureInstance()

# docker restart mysql1 mysql2 mysql3 mysql4

\c  root:mysecret@mysql1
dba.checkInstanceConfiguration()
dba.createCluster('mycluster')
var mycls = dba.getCluster()

mycls.addInstance('root@mysql2:3306')
mycls.addInstance('root@mysql3:3306')
mycls.addInstance('root@mysql4:3306')
mycls.status()

```

```sql

CREATE DATABASE shopping;

# DROP TABLE IF EXISTS shopping.list;

CREATE TABLE shopping.list (
rowid int AUTO_INCREMENT PRIMARY KEY,
description char(200),
note char(100),
purchased int DEFAULT '0' 
);
```

router 기동

```bash
docker run  --ip 172.30.0.200 \
 --name mysql-router \
 --network demo-net \
 --user root \
 -it mysql/mysql-router bash

mysqlrouter --bootstrap root@mysql1:3306 --user=root 
mysqlrouter -c /etc/mysqlrouter/mysqlrouter.conf &
```

test 

```python
docker run --net demo-net \
 --ip 172.30.0.201 \
 --rm -it -p 5000:5000 \
 --name=app-server  \
 python:3.5 bash

git clone https://github.com/uvelyster/shopping-python.git

cd shopping-python/shopping
pip install -r requirement.txt

```

```bash
python /shopping-python/shopping/shopping_list.py runserver -h 172.30.0.201
```

### Clean Up

```bash
docker rm -fv mysql1 mysql2 mysql3 mysql4 client mysql-router app-server
```
