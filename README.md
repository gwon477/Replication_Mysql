서비스 아키텍처를 구축하는 과정에서 CQRS패턴을 고려해야 했습니다. 그러면서 자연스럽게 DB Replication에 대해 공부한 내용입니다.

먼저, docker를 사용해 mysql container를 실행하고 직접 컨테이너 파일을 수정하려했지만 sudo 권한을 얻지 못해서 탈락 다른 방법을 고려했습니다.
> 원인은 아직 찾는 중입니다......

그래서 아래처럼 파일구조를 만들고 docker-compose를 작성해 실행했습니다.

# 파일 구조
```
C:\USERS\YANGK\DESKTOP\WORKOUT\REPLICATION
│  docker-compose.yml
│
├─master
│      Dockerfile
│      my.cnf
│
└─slave
        Dockerfile
        my.cnf
```

# Docker-compose.yml
```
version: "3"
services:
  db-write:
    build: 
      context: ./
      dockerfile: master/Dockerfile
    restart: always
    environment:
      MYSQL_DATABASE: 'reservation'
      MYSQL_USER: 'user'
      MYSQL_PASSWORD: 'password'
      MYSQL_ROOT_PASSWORD: 'password'
    ports:
      - '3321:3306'
    # Where our data will be persisted
    volumes:
      - master:/var/lib/mysql
      - master:/var/lib/mysql-files
    networks:
      - net-mysql
  
  db-read:
    build: 
      context: ./
      dockerfile: slave/Dockerfile
    restart: always
    environment:
      MYSQL_DATABASE: 'reservation'
      MYSQL_USER: 'user'
      MYSQL_PASSWORD: 'password'
      MYSQL_ROOT_PASSWORD: 'password'
    ports:
      - '3322:3306'
    # Where our data will be persisted
    volumes:
      - slave:/var/lib/mysql
      - slave:/var/lib/mysql-files
    networks:
      - net-mysql
  
# Names our volume
volumes:
  master:
  slave: 

networks: 
  net-mysql:
    driver: bridge
```

# 확인할 부분
>
- mac에서 실행하는 경우 ` --platform: linux/x86_64 `가 필요합니다.
- 아래 부분 DB내용을 Repication하기 때문에 서비스 내에서 필요한 DB를 선정해줘야합니다.
```
enviroment:
	MYSQL_DATABASE: {'사용할 데이터베이스명'}
```



# Master 폴더 Dockerfile
```
FROM mysql:8.0.33
ADD ./master/my.cnf /etc/mysql/my.cnf
```

# Master 폴더 my.cnf
```
[mysqld]
# 이진 로그를 사용해서 MySQL 서버에서 발행하는 변경 사항을 기록
log_bin = mysql-bin 
# MySQL 서버의 고유한 식별자를 설정
server_id = 1
binlog_do_db=reservation 
default_authentication_plugin=mysql_native_password 
```


# Slave 폴더 Dockerfile
```
FROM mysql:8.0.33
ADD ./slave/my.cnf /etc/mysql/my.cnf
```

# Master 폴더 my.cnf
```
[mysqld]
# 이진 로그를 사용해서 MySQL 서버에서 발행하는 변경 사항을 기록
log_bin = mysql-bin
# MySQL 서버의 고유한 식별자를 설정
server_id = 2
relay_log = /var/lib/mysql/mysql-relay-bin
default_authentication_plugin=mysql_native_password
```

# Replication 설정

**다음 과정을 진행합니다.**
>docker-compse 실행
네트워크 목록 확인
masterDB ip 주소 확인
masterDB 정보 확인
slaveDB 접속
slaveDB master정보 변경


**docker-compose 실행**
```
docker-compose up -d
```

**네트워크 목록 확인**
```
> docker network ls

f93e64795c7a   bridge                  bridge    local
4653aed10313   host                    host      local
435376f94b8a   none                    null      local
1ad8c7d739f9   replication_net-mysql   bridge    local
```

**masterDB ip 주소 확인 : ipv4 확인**
```
> docker inspect replication_net-mysql

"Containers": {
			...
            "64760ff4548f0935a43699f485b63e8abe7053c87a56f1e225ae4421a69ba9ea": {
                "Name": "replication-db-write-1",
                "EndpointID": "a8e373f9b74cd755bc523284c2a7a28e9b573721ca3a40640ed001678878a29f",
                "MacAddress": "02:42:ac:16:00:03",
                "IPv4Address": "172.22.0.3/16",
                "IPv6Address": ""
            }
        },

```

**masterDB 정보 확인 : File, Position 확인**
```
> docker exec -it replication-db-write-1 bash
> mysql -u root -p

> show master status;

+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000008 |      684 | reservation  |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

**slaveDB 접속**
```
> docker exec -it replication-db-read-1 bash
> mysql -u root -p

```

**slaveDB master 정보 변경**
```
// 아래에 맞는 정보를 확인 후 실행
> CHANGE MASTER TO MASTER_HOST={'masterDB ip주소'},
    -> MASTER_USER='root',
    -> MASTER_PASSWORD={'비밀번호'},
    -> MASTER_LOG_FILE={'masterDB FILE 정보'},
    -> MASTER_LOG_POS={masterDB Position 정보};

```

이후 MasterDB로 접속해 해당 DB에 작업을 진행하고 SlavdDB에 접근해 확인하면 log를 반영하여 변경된 것을 확인할 수 있습니다!