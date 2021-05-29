---
title:  "[PostgreSQL] Install PostgreSQL 9.4 + BDR 0.9.2"
excerpt:   "PostgreSQL 9.4 + BDR 0.9.2 설치하기"
toc: true
toc_sticky: true
header:
  teaser: /images/rdbms/postgresql/postgresql_bdr.jpg

categories:
  - PostgreSQL
tags:
  - PostgreSQL
  - BDR
  - Multi-Master
last_modified_at: 2016-09-1T21:48:13+0900
comments: true
---

# 개요
> 2ndquadrant사의 PostgreSQL BDR 솔루션의 설치 및 환경구성에 관한 포스트입니다.
> - BDR 홈페이지 : [http://2ndquadrant.com/en/resources/bdr/](http://2ndquadrant.com/en/resources/bdr/)
> - 0.8 버전에 비해 0.9 버전은 function 기반으로 설정이 쉬워짐.
> - 최신 안정 버전 Documentation : [http://bdr-project.org/docs/stable/](http://bdr-project.org/docs/stable/)


---

# postgresql 설치

- BDR 을 지원하는 postgresql 9.4 패치 버전의 패키지를 설치 ([http://bdr-project.org/docs/stable/installation-packages.html](http://bdr-project.org/docs/stable/installation-packages.html))
    - RedHat 계열 : [http://bdr-project.org/docs/stable/installation-packages.html#INSTALLATION-PACKAGES-REDHAT](http://bdr-project.org/docs/stable/installation-packages.html#INSTALLATION-PACKAGES-REDHAT)
    - Debian 계열 : [http://packages.2ndquadrant.com/bdr/apt/](http://packages.2ndquadrant.com/bdr/apt/)
- BDR extension 이 반드시 설치되어야 한다. (package의 경우, postgresql-bdr-contrib)
- pg_createcluster 를 이용해서 원하는 위치에 data directory를 생성postgresql 인스턴스를 시작
- Google Cloud Platform에서 asia 2대, us 2대, europe 2대 씩 설치 구성함.

## 패키지 설치 작업
```bash
sudo echo "deb http://ftp.tw.debian.org/debian unstable main" >> /etc/apt/sources.list
sudo echo "deb http://packages.2ndquadrant.com/bdr/apt/ jessie-2ndquadrant main" >> /etc/apt/sources.list.d/2ndquadrant.list
wget --quiet -O - http://packages.2ndquadrant.com/bdr/apt/AA7A6805.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get -y install postgresql-bdr-9.4 postgresql-bdr-client-9.4 postgresql-bdr-9.4-bdr-plugin postgresql-bdr-contrib-9.4 postgresql-bdr-plpython-9.4 postgresql-bdr-server-dev-9.4
```

### 소스 다운로드
[http://bdr-project.org/docs/0.9.0/installation-source.html](http://bdr-project.org/docs/0.9.0/installation-source.html)

패키지 설치를 원하지 않는다면 소스로도 설치할 수 있다.

방법은 알아서 ~~~

## database 초기화
```bash
SERVICE_MNT=/service
sudo mkdir -p $SERVICE_MNT/db/pgsql/9.4
sudo mkdir -p $SERVICE_MNT/db/pgsql/pgbackup
sudo mkdir -p $SERVICE_MNT/db/pgsql/ARCHIVE
sudo chown -R postgres.postgres $SERVICE_MNT/db/pgsql

sudo service postgresql stop
sudo rm -rf /var/lib/postgresql/9.4
sudo rm -rf /etc/postgresql/9.4/main
sudo rm -rf /service/db/pgsql/9.4/main
sudo chown -R postgres.postgres /var/lib/postgresql
sudo pg_createcluster -d /service/db/pgsql/9.4/main 9.4 main
sudo service postgresql start
```

## Database 서비스 계정 추가
```bash
sudo su -l postgres sh -c "psql -dpostgres -c \"CREATE ROLE dbadmin LOGIN PASSWORD 'dbadmin1234' superuser;\""
```

## postgresql config 설정

```bash
# 모든 서버 요청에 대해서 받을수 있게 설정
listen_addresses = '*'
# BDR과 DDR 모두에 대해이 매개 변수는 쉼표로 구분 된 값 중 하나 BDR을 포함한다.매개 변수는 서버 기동시 변경 될 수 있습니다.
shared_preload_libraries = 'bdr'
# BDR, UDR 둘다 이 변수는 logical 을 세팅해야 함
wal_level = 'logical'
# BDR을 사용하기 위해서는 이 변수가 true 세팅 되어야 하며, UDR을 사용할 경우 false, 문서와 실제 파일이 안 맞음 on 으로 세팅
track_commit_timestamp = on
max_connections = 100
# 접속가능한 슬레이브의 접속수를 설정( 슬레이브 수 + 2)인거 같은데.. backup용 확실치 않음
max_wal_senders = 8
# 노드 + 1
max_replication_slots = 7
# BDR과 UDR 모두이 구성 데이터베이스 당 하나의 작업자 및 연결 당 하나의 작업자을 가지고 충분히 큰 값으로 설정해야합니다.
max_worker_processes = 10
```

이 내용을 모두 반영하기
```bash
CONFIG=/etc/postgresql/9.4/main/postgresql.conf
DATA=/service/db/pgsql/9.4/main

sudo su -l postgres sh -c "sed -i -e \"s/#listen_addresses = 'localhost'/listen_addresses = '*'/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#shared_preload_libraries = ''/shared_preload_libraries = 'bdr'/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#wal_level = minimal/wal_level = 'logical'/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#track_commit_timestamp = off/track_commit_timestamp = on/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#max_wal_senders = 0/max_wal_senders = 8/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#max_replication_slots = 0/max_replication_slots = 7/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#max_worker_processes = 8/max_worker_processes = 10/g\" $CONFIG"
#sudo su -l postgres sh -c "sed -i -e \"s/#//g\" $CONFIG"
```

## pg_hba config 설정
```bash
HBACONFIG=/etc/postgresql/9.4/main/pg_hba.conf
sudo su -l postgres sh -c "echo 'host       all             all             10.240.0.0/16           md5' >> $HBACONFIG"
sudo su -l postgres sh -c "echo '' >> $HBACONFIG"
sudo su -l postgres sh -c "echo '# The standby server must connect with a user that has replication privileges.' >> $HBACONFIG"
sudo su -l postgres sh -c "echo '# for BDR setting' >> $HBACONFIG"
sudo su -l postgres sh -c "echo 'host       replication     postgres        10.240.0.0/16           trust' >> $HBACONFIG"
sudo su -l postgres sh -c "echo '# 중간 IPv4부분을 아래와 같이 10.240.0.0/16 해 줘야 로컬에서도 접속이 됨' >> $HBACONFIG"
sudo su -l postgres sh -c "echo '#host       all             all             10.240.0.0/16           trust' >> $HBACONFIG"
```

## PostgreSQL 기동 절차

- 중지  : sudo service postgresql stop
- 시작  : sudo service postgresql start
- 재시작 : sudo service postgresql restart

## Database 생성

- BDR 노드를 구성하기 전에 replication 할 database를 생성해야 한다.

    ```sql
    createdb testdb
    ```

---

# 각 노드 구성

- 서버 네임 : asia-psql-bdr-001, asia-psql-bdr-002, us-psql-bdr-001, us-psql-bdr-002, europe-psql-bdr-001, europe-psql-bdr-002
- asia-psql-bdr-001 을 1번 노드로 구성하고, 나머지를 하나씩 추가.
- 모든 작업은 해당 database 에 connect 하여 실행해야 한다.
- 확인하는 쿼리(bdr.bdr_node_join_wait_for_ready)를 이용한 확인에는 시간이 걸릴수도 있다. 모든 설정이 정상이라면 기다리면 결과가 나온다.만약 기다려도 답이 안올 경우, postgresql.conf 파일 중 max_wal_senders 와 max_replication_slots 값과 전체 노드 수가 맞는지 확인한다.

## 1) asia 1번 노드에 BDR 적용
사실 모든 서버에 이걸 해 줘야 BDR 사용이 가능함

- asia-psql-bdr-001 서버에서 ...
    ```sql
    psql
    \connect testdb
    CREATE EXTENSION btree_gist;
    CREATE EXTENSION bdr;
    ```

- 메인 노드 그룹 생성
    ```sql
	SELECT
	    bdr.bdr_group_create (local_node_name := 'asia-node-001',
	        node_external_dsn := 'host=asia-psql-bdr-001 port=5432 dbname=testdb');

    ```

- 확인 하는 쿼리
    ```sql
    SELECT bdr.bdr_node_join_wait_for_ready();
    ```

## 2) asia 다른 node에 BDR 적용

- asia-psql-bdr-002 서버에서 ...
    ```sql
    psql
    \connect testdb
    CREATE EXTENSION btree_gist;
    CREATE EXTENSION bdr;
    ```

- 메인 그룹 참여
    ```sql
	SELECT
	    bdr.bdr_group_join (local_node_name := 'asia-node-002',
	        node_external_dsn := 'host=asia-psql-bdr-002 port=5432 dbname=testdb',
	        join_using_dsn := 'host=asia-psql-bdr-001 port=5432 dbname=testdb');

	```

- 확인 하는 쿼리
    ```sql
    SELECT bdr.bdr_node_join_wait_for_ready();
    ```

## 3) US node에 BDR 적용

- us-psql-bdr-001 은 asia-psql-bdr-001 로 join하고, us-psql-bdr-002는 us-psql-bdr-001로 join한다.
- us-psql-bdr-001 서버에서 ...
    ```sql
    psql
    \connect testdb
    CREATE EXTENSION btree_gist;
    CREATE EXTENSION bdr;
    ```

- 메인 그룹 참여
    ```sql
	SELECT
	    bdr.bdr_group_join (local_node_name := 'us-node-001',
	        node_external_dsn := 'host=us-psql-bdr-001 port=5432 dbname=testdb',
	        join_using_dsn := 'host=asia-psql-bdr-001 port=5432 dbname=testdb');

    ```

- 확인 하는 쿼리
    ```sql
    SELECT bdr.bdr_node_join_wait_for_ready();
    ```

- us-psql-bdr-002 서버에서 ...
    ```sql
    psql\connect testdbCREATE EXTENSION btree_gist;CREATE EXTENSION bdr;
    ```

- 메인 그룹 참여
    ```sql
	SELECT
	    bdr.bdr_group_join (local_node_name := 'us-node-002',
	        node_external_dsn := 'host=us-psql-bdr-002 port=5432 dbname=testdb',
	        join_using_dsn := 'host=us-psql-bdr-001 port=5432 dbname=testdb');

    ```

- 확인 하는 쿼리
    ```sql
    SELECT bdr.bdr_node_join_wait_for_ready();
    ```

## 4) europe node에 BDR 적용

- europe-psql-bdr-001 은 asia-psql-bdr-001 로 join하고, europe-psql-bdr-002는 europe-psql-bdr-001로 join한다.
- europe-psql-bdr-001 서버에서 ...
    ```sql
    psql\connect testdbCREATE EXTENSION btree_gist;CREATE EXTENSION bdr;
    ```

- 메인 그룹 참여
    ```sql
	SELECT
	    bdr.bdr_group_join (local_node_name := 'europe-node-001',
	        node_external_dsn := 'host=europe-psql-bdr-001 port=5432 dbname=testdb',
	        join_using_dsn := 'host=asia-psql-bdr-001 port=5432 dbname=testdb');

    ```

- 확인 하는 쿼리
    ```sql
    SELECT bdr.bdr_node_join_wait_for_ready();
    ```

- europe-psql-bdr-002 서버에서 ...
    ```sql
    psql\connect testdbCREATE EXTENSION btree_gist;CREATE EXTENSION bdr;
    ```

- 메인 그룹 참여
    ```sql
	SELECT
	    bdr.bdr_group_join (local_node_name := 'europe-node-003',
	        node_external_dsn := 'host=europe-psql-bdr-002 port=5432 dbname=testdb',
	        join_using_dsn := 'host=europe-psql-bdr-001 port=5432 dbname=testdb');

    ```

- 확인 하는 쿼리
    ```sql
	SELECT
	    bdr.bdr_node_join_wait_for_ready ();

    ```

---

# 노드 제거

- 함수로 쉽게 노드를 제거 할 수 있다고 하나, 실제로는 함수 실행 후 확인해보면 노드 정보가 남아있다. (select * from bdr.bdr_nodes)
    ```sql
	SELECT
	    bdr.bdr_part_by_node_names (ARRAY [ 'node-1' ]);

	SELECT
	    bdr.bdr_part_by_node_names (ARRAY [ 'node-1',
	        'node-2',
	        'node-3' ]);

	SELECT
	    bdr.bdr_part_by_node_names (ARRAY [ 'asia-node-003' ]);
    ```

- 때문에 함수 실행 후 다음을 확인하여 남아있는 정보를 함께 지워준다.
    ```sql
	select * from
	    bdr.bdr_nodes;

	delete from bdr.bdr_nodes
	where node_name = 'asia-node-003';

	select * from
	    bdr.bdr_connections;

	delete from bdr.bdr_connections
	where conn_sysid = '6189502444603979480';
    ```
---

# 노드 추가

- 노드 추가는 pg_basebackup 으로 SOURCE로부터 전체 백업을 받아온 후, bdr_init_copy 라는 명령어를 이용한다.
- 데이터가 계속 인입되는 runtime 환경에서 테스트한 결과, 노드 추가 후 group 에 join되어 곧 바로 동기화가 진행되었다.노드 추가에 영향을 주는 환경변수는 다음과 같다.
    - 접속가능한 슬레이브의 접속수를 설정( 슬레이브 수 + 2)인거 같은데.. backup용 확실치 않음

        > max_wal_senders = 8

    - 노드 + 1

        > max_replication_slots = 7

    - BDR과 UDR 모두 구성 데이터베이스 당 하나의 작업자 및 연결 당 하나의 작업자을 가지고 충분히 큰 값으로 설정해야합니다.

        > max_worker_processes = 10

    - 환경변수를 꼭 확인하고 아래의 스크립트를 그대로 이용하면 된다.
        ```bash
        # Created by Peter Yun. 2015-09-01.
        #!/bin/sh
        SOURCE="asia-psql-bdr-002"
        TARGET="asia-psql-bdr-003"
        NODENAME="asia-node-003"

        sudo service postgresql stop
        sudo rm -rf /service/db/pgsql/9.4/main/*
        sudo su -l postgres -c "pg_basebackup -h $SOURCE -U postgres -D /service/db/pgsql/9.4/main -xlog -c fast -P"

        sudo su -l postgres -c "/usr/lib/postgresql/9.4/bin/bdr_init_copy -d testdb -h $SOURCE -p 5432 \
        --local-dbname=testdb \
        --local-host=$TARGET --local-port=5432 \
        --pgdata=/service/db/pgsql/9.4/main \
        --postgresql-conf=/etc/postgresql/9.4/main/postgresql.conf \
        --hba-conf=/etc/postgresql/9.4/main/pg_hba.conf \
        --node-name=$NODENAME "

        sleep 10
        sudo su -l postgres -c "tail -20 ~/bdr_init_copy_postgres.log"
        ```

    - bdr 노드 추가 후 서버 중지

        echo "bdr_init_copy_postgres.log 를 확인하여 이상없으면, bdr_init_copy 로 실행된 postgresql을 중지 하고 service 로 다시 띄운다.
        ```bash
        echo "----------"
        echo "sudo su -l postgres -c \"tail -20 ~/bdr_init_copy_postgres.log\""
        echo "sudo su - postgres"
        echo "/usr/lib/postgresql/9.4/bin/pg_ctl  --pgdata=/service/db/pgsql/9.4/main  stop -m fast"
        echo "exit"
        echo "sudo service postgresql start"
        echo "----------"
        ```
---

# bdr 모니터링

## 1) 노드 확인
```sql
SELECT * FROM bdr.bdr_nodes;

```

## 2) Monitoring connected peers using pg_stat_replication
```sql
SELECT * FROM pg_stat_replication;

SELECT
    pg_xlog_location_diff(pg_current_xlog_insert_location(), flush_location) AS lag_bytes,
    pid,
    application_name
FROM
    pg_stat_replication
WHERE
    application_name LIKE 'bdr%';


```

## 3) Monitoring replication slots
```sql
SELECT * FROM pg_replication_slots;

SELECT
    slot_name,
    database,
    active,
    pg_xlog_location_diff(pg_current_xlog_insert_location(), restart_lsn) AS retained_bytes
FROM
    pg_replication_slots
WHERE
    plugin = 'bdr';

```

---

# 백업 관련

1. 증분 백업
    - barman을 사용하여 증분 백업을 하고 싶지만…
    - bdr를 사용할 경우, wal_level = 'logical' 옵션 때문에 증분 백업을 사용할수 없음
    - 증분 백업을 위해서는 wal_level 값이 'archive', 'hot_standby' 값을 가져야만 함.
2. 노드 추가가 어렵지 않은 만큼 데이터 백업은 snapshot 또는 pg_dump, pg_basebackup 만 해도 괜찮을 듯 하다.

# 파티션 테이블 관리

- 월별 range 파티션 사용 시 자동 파티션 생성 procedure 및 job 스케줄러 등록하여 관리가 가능한지 추가 확인 필요

---

# 주의 사항

1. 노드명 삭제 불가
    - bdr.bdr_group_join 에 참여한 노드는 서버 이름을 변경하지 않으면 다시 등록이 안됨.

        bdr.bdr_part_by_node_names()로 group에서 제외시켰음에도 불구하고,bdr.bdr_nodes 목록에 여전히 등록되어 있으며, 같은 서버 이름으로는 이미 등록되어 있다고 등록이 안됨

    - 한 번 그룹에서 빠진 서버는 서버명을 변경해서 join하면 될거 같으나 클라우드 서버의 특성상 서버명 변경이 불가하기에 다른 서버를 생성하여 추가해야 함. (2021/04/29 추가 - GCP CLI를 이용하면 인스턴스 이름 변경이 가능함)
    - (09/01 추가) bdr.bdr_part_by_node_names() 함수의 버그로 보이며, 함수 실행 후bdr.bdr_nodes 와 bdr.bdr_connections 의 정보를 확인하여 삭제해 주고, bdr_init_copy 명령어로 노드 추가를 할 수 있다.
    - 삭제는 노드 중 한 곳에서만 실행해도 된다.
        ```sql
		select * from bdr.bdr_nodes;

		delete from bdr.bdr_nodes
			where node_name = 'asia-node-003';

		select * from bdr.bdr_connections;

		delete from bdr.bdr_connections
			where conn_sysid = '6189506087542501647';

        ```


2. Global 시퀀스를 사용해야 함

    [http://bdr-project.org/docs/0.9.0/global-sequences.html](http://bdr-project.org/docs/0.9.0/global-sequences.html)

    - 시퀀스를 사용하는 경우 글로벌 시퀀스를 사용해야 함. 그렇지 않으면fail over가 발생했을 경우시퀀스 번호가 다른 노드에 자동 증가가 되지 않아서 오류가 발생함.
    - 문법적으로 약간 다름

        예) create sequence access_log_seq USING bdr;

    - 아래 구문과 같이 minvalue, maxvalue를 지정할수 없고 cache 도 사용불가minvalue는 1로 고정됨- 데이터 타입이 전부 bigserial(bigint) 이어야 함.
        ```sql
		create sequence xxxx
		    increment 1
		    minvalue 1
		    maxvalue 999999999999 start 1
		    cache 1;

        ```

    - 시퀀스 번호가 순차적으로 생성 되지 않음.

        예를 들어 노드1번이 1번부터 시작한다면 노드 2번은 100000번부터 노드3번은10000000부터시작함. 내부적으로 트랜잭션 동기화 때문인거 같음


3. ddl 문 실행시 제약 조건

[ http: / / bdr - project.org / docs / 0.9.0 / ddl - replication.html ] (http: / / bdr - project.org / docs / 0.9.0 / ddl - replication.html) - alter default -


4. 로그
    - fail over 발생시 로그가 계속 출력됨. log rotate를 하지 않을 경우 문제가 될거 같음
    - (09/01 추가) 서버 fail 이 발생했을 경우에는 재빨리 replication group에서 제거해줘야 함.
    - 노드 제거 시에도 발생하게 되는데, bdr.bdr_part_by_node_names() 함수의 버그로 보이며 함수 실행 후 ,제거된 노드에서 bdr.bdr_nodes 와 bdr.bdr_connections 의 정보를 확인하여 삭제하면 더 이상 추가 로그가 발생하지 않는다.
    - 삭제는 노드 중 한 곳에서만 실행해도 된다.
        ```sql
        select * from bdr.bdr_nodes;

        delete from bdr.bdr_nodes
			where node_name='asia-node-003';

        select * from bdr.bdr_connections;

        delete from bdr.bdr_connections
			where conn_sysid='6189506087542501647';

		```


5. table 선택적 replication 사용 가능 ==> 추가 테스트 필요

    [http://bdr-project.org/docs/0.9.0/functions-replication-sets.html](http://bdr-project.org/docs/0.9.0/functions-replication-sets.html)

    - 특정 table을 replication 하고 싶지 않을 경우
        ```sql
        select bdr.table_set_replication_sets('user_info', '{}');
        ```

        를 실행하면 user_info table 이 복제가 되지 않는다.

        table 속성을 보면 default 속성이 빠진걸 볼수 있음

    - 복원은 bdr.table_get_replication_sets() 사용
