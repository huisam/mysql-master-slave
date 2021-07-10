## Mysql Master Slave 구축하기

간단하게 Docker compose를 활용한 구축 방법에 대해 알아보도록 하겠습니다. 💿  

해당 Repository 에서 항목들을 모두 로컬로 가져온 다음에,  

```bash
docker-compose up -d
```

를 해주시면 정상적으로 booting이 됩니다!!  

모든 부팅 작업이 끝나면, `master-slave` 간의 연결작업이 필요한데요.  

```bash
docker network ls

NETWORK ID     NAME              DRIVER    SCOPE
a6260b4d5998   bridge            bridge    local
f5c621aee97f   host              host      local
f997b87515bb   mysql_net-mysql   bridge    local
ae7e075418cd   none              null      local
```

mysql bridge 에 해당하는 network 에 대해서 inspect 하는 과정을 거쳐서,  
master의 internal ip를 알아와야 합니다.!!  

```bash
docker inspect f997b87515bb | jq
[
  {
    "Name": "mysql_net-mysql",
    "Containers": {
      "913e258e271eca08249611c813b7efd84756ac1d4b3c5254c42e9dbfcab8e9d3": {
        "Name": "mysql_db-master_1",
        "EndpointID": "7951e87e0ab78728a8db387c180375e19b0aa8136593fbe7793ce932350de162",
        "MacAddress": "02:42:ac:16:00:03",
        "IPv4Address": "172.22.0.3/16", // 이 정보 기억하기
        "IPv6Address": ""
      },
      "c64a8e5ec49e748f899c1b2158ea19a4ae527e0cc8422a4d854501e65d253c74": {
        "Name": "mysql_db-slave_1",
        "EndpointID": "c1608ce02b2e834721b2902257e7e7a65fc5bfce75f4fb447df702f905ccf300",
        "MacAddress": "02:42:ac:16:00:02",
        "IPv4Address": "172.22.0.2/16",
        "IPv6Address": ""
      }
    },
    "Options": {},
    "Labels": {
      "com.docker.compose.network": "net-mysql",
      "com.docker.compose.project": "mysql",
      "com.docker.compose.version": "1.29.0"
    }
  }
]
```

master의 `ip주소를 알아옵니다`  

자 그러면 모든 준비는 끝났습니다.  


### slave에서 master 연결하기

slave mysql 에 들어가봅니다!  

```bash
docker exec -it mysql_db-slave_1 bash
mysql -u root -p
```

이제 master를 향해서 연결해야 되는데요   

```bash
CHANGE MASTER TO MASTER_HOST='172.22.0.3', MASTER_USER='root', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000013', MASTER_LOG_POS=0, GET_MASTER_PUBLIC_KEY=1;

start slave;

show slave status\G;

```

위와 같이 연결 해주고 상태값을 보게 되면.!! 

```bash

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.22.0.3
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000013
          Read_Master_Log_Pos: 350
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 565
        Relay_Master_Log_File: mysql-bin.000013
             Slave_IO_Running: Yes // Yes 가 나오면 성공
            Slave_SQL_Running: Yes // Yes 가 나오면 성공
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
```

**Slave_IO** 와 **Slave_SQL** 이 정상적으로 모두 활성화되면 성공입니다!!  

### 테스트 database 만들기

정상적으로 모든 설정이 끝났으니 slave에서 반영이 되나 한번 봐야겠죠.?  

```bash
docker exec -it mysql_db-master_1 bash
mysql -u root -p
create database test_db;
```

마스터에서 데이터베이스를 생성하고, slave에 들어가서 확인하면?  

```bash
docker exec -it mysql_db-slave_1 bash
mysql -u root -p
show databases;
```

정상적으로 뜬 것이 확인되면 모든 설정은 끝났습니다! 😄

### 참고 링크

* [docker-compsoe로 띄워보기](https://www.programmersought.com/article/40905161651/#dockercomposeyml_33)
* [caching_sha2_password 이슈](https://kogle.tistory.com/87)