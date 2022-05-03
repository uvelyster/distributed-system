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
dba.checkInstanceConfiguration()
dba.configureInstance()

# other terminal
docker restart mysql1

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

router 기동

```bash
docker run  --ip 172.30.0.200 \
 --name mysql-router \
 --network demo-net \
 --user root \
 -v `pwd`:/etc/mysqlrouter \
 -it mysql/mysql-router bash

mysqlrouter --bootstrap root@mysql1:3306 --user=root 
mysqlrouter -c /etc/mysqlrouter/mysqlrouter.conf &
```

test 

```python
docker run --net demo-net --ip 172.30.0.201 --rm -it -p 5000:5000 --name=python-server  python:3.5 bash
pip install mysql-connector-python==8.0.22
```

```python
cat << EOF > test.py
import mysql.connector

def show_results(cur_obj):
  for row in cur:
    print(row)
my_cfg = {
  'user':'root',
  'passwd':'root',
  'host':'mysql-router',
  'port':6446
}

# Connecting
conn = mysql.connector.connect(**my_cfg)

print("Listing the databases on the server.")
query = "SHOW DATABASES"
cur = conn.cursor()
cur.execute(query)
show_results(cur)

print("\nRetrieve the port for the server to which we're connecting.")
query = "SELECT @@port"
cur = conn.cursor()
cur.execute(query)
show_results(cur)

# Close connection
cur.close()
conn.close()
EOF
```

```bash
python test.py
```

### Clean Up

```bash
docker rm -fv mysql1 mysql2 mysql3 mysql4 client mysql-router
```
