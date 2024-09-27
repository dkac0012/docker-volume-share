# docker-volume-share

### 🐣 docker의 volume이 어떤식으로 구현하고 활용할 수 있는지 공부해보고 싶어 docker의 volume을 생성해보았습니다.

## 개요
docker volume은 docker 내부에서 관리하는 DB이다.
container와의 연결이 끊어지거나 container가 삭제되어도 지속성을 유지한다.

## docker volume 생성 파이프라인

### docker volume

docker volume 생성
```bash
docker volume create mysql-data
```
docker volume 확인
```bash
docker volume ls
```
```bash
username@servername:~/Dockercompose$ docker volume ls
DRIVER    VOLUME NAME
local     data
local     mysql-data
```

### docker container

docker container 생성
```bash
docker run --name mysqldb2 -d -p 3306:3306 -v mysql-data:/var/lib/mysql --network springboot-mysql-net -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=dmount mysql:latest
```
docker container 확인
```bash
username@servername:~$ docker ps  -> -a 옵션시 종료된 파일도 확인 가능
CONTAINER ID   IMAGE                       COMMAND                  CREATED         STATUS        PORTS                                                    NAMES
8e1bbbe65d92   mysql                       "docker-entrypoint.s…"   2 seconds ago   Up 1 second   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp     mysqldb2
```

### mount data 
mysql data 생성
```bash
mysql> select * from dept;
+--------+------------+----------+
| deptno | dname      | loc      |
+--------+------------+----------+
|     10 | ACCOUNTING | NEW YORK |
|     20 | RESEARCH   | DALLAS   |
|     30 | SALES      | CHICAGO  |
|     40 | OPERATIONS | BOSTON   |
+--------+------------+----------+
```

다른 컨테이너에서의 접속
```bash
username@servername:~$ docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED          STATUS         PORTS                                                    NAMES
37437a80844f   mysql                       "docker-entrypoint.s…"   40 seconds ago   Up 2 seconds   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp     mysqldb3
```
데이터 공유 
```bash
mysql> select * from dept;
+--------+------------+----------+
| deptno | dname      | loc      |
+--------+------------+----------+
|     10 | ACCOUNTING | NEW YORK |
|     20 | RESEARCH   | DALLAS   |
|     30 | SALES      | CHICAGO  |
|     40 | OPERATIONS | BOSTON   |
+--------+------------+----------+
```

### docker compose

docker compose 작성
```bash
services:
  db:
    container_name: mysqldb
    image: mysql:latest
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: dmount
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    networks:
      - spring-mysql-net
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ['CMD-SHELL', 'mysqladmin ping -h 127.0.0.1 -u root --password=$$MYSQL_ROOT_PASSWORD']
      interval: 10s
      timeout: 2s
      retries: 100

  app:
    container_name: springbootapp
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "8081:8081" 
    environment:
      MYSQL_HOST: db  # DB 호스트 설정
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

networks:
  spring-mysql-net:
    driver: bridge  # 네트워크 드라이버 정의

volumes :
  mysql-data :
```

docker volume 접속 후 확인
```bash
root@servername:/var/lib/docker/volumes/mysql-data/_data# ls -> mount가 되지 않음
```
성공시 화면
```bash
root@servername:/var/lib/docker/volumes/mysql-data/_data# ls
 auto.cnf        binlog.000004   ca-key.pem        dmount                 ibdata1         mysql                   performance_schema   server-key.pem
 binlog.000001   binlog.000005   ca.pem           '#ib_16384_0.dblwr'   ibtmp1          mysql.ibd               private_key.pem      sys
 binlog.000002   binlog.000006   client-cert.pem  '#ib_16384_1.dblwr'  '#innodb_redo'   mysql.sock              public_key.pem       undo_001
 binlog.000003   binlog.index    client-key.pem    ib_buffer_pool      '#innodb_temp'   mysql_upgrade_history   server-cert.pem      undo_002
```

### 결론

docker의 volume mount기능을 활용하여 여러대의 서버가 데이터를 공유하여 관리가 용이하며 접근성이 좋다는 것을 알 수 있었습니다.
docker-compose환경에서는 volume mount를 구현하지 못했지만 추후에 구현하도록하겠습니다.


