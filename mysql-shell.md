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
SELECT * FROM test.t1;
\q
exit
```

```bash
cd ~
yum install -y wget 
wget [https://downloads.mysql.com/docs/world_x-db.tar.gz](https://downloads.mysql.com/docs/world_x-db.tar.gz)
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

다음은 두가지 옵션에 대한 설명입니다.

- *복제* : 클러스터에 추가하려는 인스턴스에 기증자를 복제하고 인스턴스에 포함된 모든 트랜잭션을 삭제하려면 이 옵션을 선택합니다. MySQL Clone 플러그인이 자동으로 설치됩니다. 필요한 InnoDB 클러스터 복구 계정이 생성됩니다. `[BACKUP_ADMIN](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_backup-admin)` 특권. 비어 있거나(트랜잭션을 처리하지 않음) 유지하지 않으려는 트랜잭션이 포함된 인스턴스를 추가한다고 가정하고 복제 옵션을 선택합니다. 그런 다음 클러스터는 MySQL Clone을 사용하여 결합 인스턴스를 기증자 클러스터 구성원의 스냅샷으로 완전히 덮어씁니다. 기본적으로 이 방법을 사용하고 이 프롬프트를 비활성화하려면 클러스터의 `recoveryMethod`옵션을 로 설정하십시오 `clone`.

- *증분 복구 증분 복구* 를 사용하여 클러스터에서 처리한 모든 트랜잭션을 비동기 복제를 사용하여 조인 인스턴스로 복구하려면 이 옵션을 선택합니다. 증분 복구는 클러스터에서 처리한 모든 업데이트가 GTID가 활성화된 상태에서 완료되었으며 제거된 트랜잭션이 없으며 새 인스턴스에 클러스터 또는 클러스터의 하위 집합과 동일한 GTID 세트가 포함되어 있는 경우에 적합합니다. 기본적으로 이 방법을 사용하려면 `recoveryMethod`옵션을 로 `incremental`.

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
