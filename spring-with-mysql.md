```bash
docker network create -d bridge --subnet 172.30.0.0/24 demo-net
```

```bash
yum install -y git wget 
git clone https://github.com/spring-petclinic/spring-framework-petclinic.git petclinic-tomcat

cd petclinic-tomcat

```

```bash
docker run -p 8080:8080 -it --rm  -v "$(pwd)":/usr/src/mymaven  -w /usr/src/mymaven  maven:3.3-jdk-8  mvn clean install -Dmaven.test.skip=true

docker run --net demo-net --name tomcat1 -d -p 8888:8080 tomcat:9
```

```bash
docker cp target/petclinic.war tomcat1:/usr/local/tomcat/webapps/

```

브라우저에서 192.168.56.100:8888 주소로 확인 합니다.

```bash
docker rm -fv tomcat1

```


# Deploy with jar

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

```

```bash
\c root:root@mysql1:33060
dba.configureInstance()

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

\sql
CREATE DATABASE petclinic;
CREATE USER 'petclinic'@'%' identified by 'petclinic';
GRANT ALL PRIVILEGES ON petclinic.* to 'petclinic'@'%';

```

```bash
docker run  --ip 172.30.0.200 \
 --name mysql-router \
 --network demo-net \
 --user root \
 -p 6446:6446 \
 -p 6447:6447 \
 -it mysql/mysql-router bash

mysqlrouter --bootstrap root@mysql1:3306 --user=root 
mysqlrouter -c /etc/mysqlrouter/mysqlrouter.conf &
```

```bash
git clone https://github.com/redhat-developer-demos/spring-petclinic petclinic-spring

cd petclinic-spring

docker run -p 8080:8080 -it --rm  -v "$(pwd)":/usr/src/mymaven  -w /usr/src/mymaven  maven  mvn clean install
```

```bash
# docker run --net demo-net --name mysql -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=petclinic -e MYSQL_DATABASE=petclinic -e MYSQL_USER=petclinic -e MYSQL_PASSWORD=petclinic mysql

docker run --net demo-net -e SPRING_PROFILES_ACTIVE=mysql -e MYSQL_URL=jdbc:mysql://mysql-router:6446/petclinic --name tomcat1 -it -p 8888:8080 openjdk:11 bash
```

```bash
docker cp target/spring-petclinic-2.3.0.BUILD-SNAPSHOT.jar tomcat1:/petclinic.jar
```

```bash
java -jar petclinic.jar
```
