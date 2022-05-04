# Prerequisites

- docker 구성
- mysql:8.0.28 (latest) 이미지
- mysql/mysql-server 이미지

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

docker run --ip 172.30.0.105 \
 --network demo-net \
 --name client -d \
 -e MYSQL_ROOT_PASSWORD=root \
 mysql/mysql-server

docker exec -it client sh -c mysqlsh
mysqlsh
```

```bash
\c root:root@mysql1:33060
\disconnect
\c root:root@mysql2:33060
\sql
CREATE DATABASE test;
USE test;
CREATE TABLE test.t1 (c1 INT PRIMARY KEY, c2 TEXT NOT NULL);
INSERT INTO test.t1 VALUES (1, 'test');
SELECT * FROM test.t1;

\c root:root@mysql1:33060
SELECT * FROM test.t1;
\q
exit
```

```bash
cd ~
yum install -y wget 
wget https://downloads.mysql.com/docs/world_x-db.tar.gz
tar xvzf world_x-db.tar.gz
docker cp world_x-db client:/

```

```bash

docker exec -it client bash
mysqlsh --sql --uri root:root@mysql1:33060 
\source /world_x-db/world_x.sql
SHOW DATABASES;
USE world_x;
EXPLAIN city;
\q
exit
```

Create get_record.js file

```jsx
var mySession = mysqlx.getSession('root:root@mysql1:33060');
var result = mySession.getSchema('world_x').getCollection('countryinfo').find().execute();
var record = result.fetchOne();
while(record){
  print(record);
  record = result.fetchOne();
}
```

```bash
docker cp get_record.js client:/
docker exec -it client sh -c 'mysqlsh --uri root:root@mysql1:33060 -f get_record.js'
```

# Deploy Sandbox of InnoDB Cluster

Deploy Sandbox Instance of The InnoDB Cluster

```bash
# mysqlsh -- dba deploy-sandbox-instance 1111 --password=password
dba.deploySandboxInstance(1111, {password: 'mysecret'})
\c  root:mysecret@localhost:1111
dba.checkInstanceConfiguration()
```

Create Cluster using sandbox

```bash
# mysqlsh root@localhost:1234 -- dba create-cluster mycluster
dba.createCluster('mycluster')
```

check status

```bash
# mysqlsh root@localhost:1234 -- cluster status
var cluster = dba.getCluster()
cluster.status()
```

add instance

```bash
# mysqlsh root@localhost:1111 -- cluster status
\c root:root@mysql1:33060 
dba.deploySandboxInstance(2222, {password: 'mysecret'})
\c  root:mysecret@localhost:1111
var cluster = dba.getCluster()
cluster.addInstance('root@localhost:2222')
cluster.status()
```

| 명령 | 동작 |
| --- | --- |
| delete_sandbox_instance (int port, dict options) | 샌드박스 삭제 |
| deploy_sandbox_instance (int port, dict options) | 샌드박스 구성 |
| kill_sandbox_instance (int port, dict options) | 샌드박스 강제종료 |
| start_sandbox_instance (int port, dict options) | 샌드박스 시작 |
| stop_sandbox_instance (int port, dict options) | 샌드박스 종료 |

```bash
docker rm -fv mysql1 mysql2 client 
```
