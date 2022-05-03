# Prerequisites

- InnoDB Cluster + Spring + Tomcat 실습 완료

```bash
docker run --net demo-net -e SPRING_PROFILES_ACTIVE=mysql -e MYSQL_URL=jdbc:mysql://mysql-router:6446/petclinic --name tomcat1 -it  openjdk:11 bash

# Escape from container with key 'Ctrl+p , Ctrl+q'

docker run --net demo-net -e SPRING_PROFILES_ACTIVE=mysql -e MYSQL_URL=jdbc:mysql://mysql-router:6446/petclinic --name tomcat2 -it  openjdk:11 bash

# Escape from container with key 'Ctrl+p , Ctrl+q'

```

새로운 터미널에서 실행

```bash
docker cp target/spring-petclinic-2.3.0.BUILD-SNAPSHOT.jar tomcat1:/petclinic.jar
docker cp target/spring-petclinic-2.3.0.BUILD-SNAPSHOT.jar tomcat2:/petclinic.jar

docker attach tomcat1 
java -jar petclinic.jar 

# Escape from container with key 'Ctrl+p , Ctrl+q'

docker attach tomcat2 
java -jar petclinic.jar 

# Escape from container with key 'Ctrl+p , Ctrl+q'

```

```bash
docker run -d --net demo-net --name web-server -p 80:80 nginx

docker cp web-server:/etc/nginx/conf.d/default.conf .
vi default.conf
```

```bash

server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        proxy_pass  http://petclinic-app;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}

upstream petclinic-app {
  least_conn;
  # ip_hash;

  server tomcat1:8080;
  server tomcat2:8080;
}
```

```bash
docker cp default.conf web-server:/etc/nginx/conf.d/default.conf
docker restart web-server
```
