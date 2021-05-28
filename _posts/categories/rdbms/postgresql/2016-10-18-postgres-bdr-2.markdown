---
title:  "Running Postgres-BDR with Google Cloud Platform-2"
excerpt:   "PostgreSQL로 Multi-Master를 만들자"
toc: true
toc_sticky: true
header:
  teaser: /images/rdbms/postgresql/bdr.jpg

categories:
  - PostgreSQL
tags:
  - PostgreSQL
  - BDR
  - Multi-Master
last_modified_at: 2016-10-18T14:40:13+0900
comments: true
---

# 개요
>Open Source RDBMS인 PostgreSQL은 기본 구성으로 Master 서버를 1대만 구성할 수 있다.
>그러나 Postgres-BDR을 이용하면 Multi-Master 구성이 가능하며,
>GCP의 네트워크를 이용하여 Multi-Region 구성으로 PostgreSQL의 데이터를 동기화 할 수 있다.

# BDR을 위한 PostgreSQL 설정

## postgresql.conf 설정

```
[code lang=text]
# 모든 서버 요청에 대해서 받을수 있게 설정
listen_addresses = '*'
# BDR 이 매개 변수는 쉼표로 구분 된 값 중 하나 BDR을 포함한다.매개 변수는 서버 기동시 변경 될 수 있습니다.
shared_preload_libraries = 'bdr'
# BDR 둘다 이 변수는 logical 을 세팅해야 함
wal_level = 'logical'
# BDR을 사용하기 위해서는 이 변수가 true 세팅 되어야 하며, UDR을 사용할 경우 false, 문서와 실제 파일이 안 맞음 on 으로 세팅
track_commit_timestamp = on
max_connections = 100
# 접속가능한 슬레이브의 접속수를 설정( 슬레이브 수 + 2)인거 같은데.. backup용 확실치 않음
max_wal_senders = 10
# 노드 + 1
max_replication_slots = 10
# BDR 구성 데이터베이스 당 하나의 작업자 및 연결 당 하나의 작업자을 가지고 충분히 큰 값으로 설정해야합니다.
max_worker_processes = 10
[/code]
```

이 내용을 모두 반영하기위해 아래 내용을 스크립트로 만들어 sudo 명령어로 실행시키자. (ex. init_pgsql.sh)

```
[code lang=text]
#!/bin/bash
CONFIG=/etc/postgresql/9.4/main/postgresql.conf
#DATA=/var/lib/postgresql/9.4/main

sudo su -l postgres sh -c "sed -i -e \"s/#listen_addresses = 'localhost'/listen_addresses = '*'/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#shared_preload_libraries = ''/shared_preload_libraries = 'bdr'/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#wal_level = minimal/wal_level = 'logical'/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#track_commit_timestamp = off/track_commit_timestamp = on/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#max_wal_senders = 0/max_wal_senders = 10/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#max_replication_slots = 0/max_replication_slots = 10/g\" $CONFIG"
sudo su -l postgres sh -c "sed -i -e \"s/#max_worker_processes = 8/max_worker_processes = 10/g\" $CONFIG"
#sudo su -l postgres sh -c "sed -i -e \"s/#//g\" $CONFIG"

[/code]
```

## pg_hba.conf 설정

위 스크립트에 추가하여 함께 수행해도 된다.마지막 추가된 행은 중간 IPv4 부분으로 복사하고 주석을 풀어줘야 로컬에서도 접속이 된다.

```
[code lang=text]
HBACONFIG=/etc/postgresql/9.4/main/pg_hba.conf
sudo su -l postgres sh -c "echo 'host all all 10.128.0.0/9 md5' &gt;&gt; $HBACONFIG"
sudo su -l postgres sh -c "echo '' &gt;&gt; $HBACONFIG"
sudo su -l postgres sh -c "echo '# The standby server must connect with a user that has replication privileges.' &gt;&gt; $HBACONFIG"
sudo su -l postgres sh -c "echo '# for BDR setting' &gt;&gt; $HBACONFIG"
sudo su -l postgres sh -c "echo 'host replication postgres 10.128.0.0/9 trust' &gt;&gt; $HBACONFIG"
sudo su -l postgres sh -c "echo '# 중간 IPv4부분을 아래와 같이 10.128.0.0/9 해 줘야 로컬에서도 접속이 됨' &gt;&gt; $HBACONFIG"
sudo su -l postgres sh -c "echo '#host all all 10.128.0.0/9 trust' &gt;&gt; $HBACONFIG"
[/code]
```

## PostgreSQL 시작/중지/재시작

- 중지 : sudo service postgresql stop
- 시작 : sudo service postgresql start
- 재시작 : sudo service postgresql restart

---

# Postgres-BDR 설정

## Database 만들기

BDR 노드를 구성하기 전에 replication 할 database를 생성해야 한다.

```
[code lang=text]
$ sudo su - postgres
$ createdb testdb
[/code]
```

## 각 노드 구성

- Server Name : bdr-asia-1, bdr-asia-2, bdr-us-1, bdr-us-2, bdr-eu-1, bdr-eu-2
- bdr-asia-1 을 1번 노드로 먼저 구성하고, 나머지를 하나씩 추가 한다.
- 모든 작업은 해당 database 에 하나씩 connect 하여 실행해야 한다.
- 확인하는 쿼리(bdr.bdr_node_join_wait_for_ready)를 이용한 확인에는 시간이 걸릴수도 있다. 모든 설정이 정상이라면 기다리면 결과가 나온다.만약 기다려도 답이 안올 경우, postgresql.conf 파일 중 max_wal_senders 와 max_replication_slots 값과 전체 노드 수가 맞는지 확인한다.
- 모든 노드에 아래 순서대로 설정을 한다.

## 모든 서버에서의 설정

```
[code lang=text]
$ sudo su - postgres
$ psql
postgres=# \connect testdb
You are now connected to database "testdb" as user "postgres".

testdb=# CREATE EXTENSION btree_gist;
CREATE EXTENSION

testdb=# CREATE EXTENSION bdr;
CREATE EXTENSION
[/code]
```

## 1. bdr-asia-1 서버에서의 설정

최초의 노드이므로 node_external_dsn 값만을 가지고 올라온다.

```
[code lang=text]
# 메인 노드 그룹 생성
testdb=# SELECT bdr.bdr_group_create(
local_node_name := 'asia-node-001',
node_external_dsn := 'host=bdr-asia-1 port=5432 dbname=testdb'
);
bdr_group_join
----------------

(1 row)

# 확인 하는 쿼리
testdb=# SELECT bdr.bdr_node_join_wait_for_ready();
bdr_node_join_wait_for_ready
------------------------------

(1 row)
[/code]
```

## 2. bdr-asia-2 서버에서의 설정

bdr-asia-2 는 bdr-asia-1 로 join한다.

```
[code lang=text]
# 메인 그룹 참여
testdb=# SELECT bdr.bdr_group_join(
local_node_name := 'asia-node-002',
node_external_dsn := 'host=bdr-asia-2 port=5432 dbname=testdb',
join_using_dsn := 'host=bdr-asia-1 port=5432 dbname=testdb'
);
bdr_group_join
----------------

(1 row)

# 확인 하는 쿼리
testdb=# SELECT bdr.bdr_node_join_wait_for_ready();
bdr_node_join_wait_for_ready
------------------------------

(1 row)
[/code]
```

## 3. bdr-us-1 서버에서의 설정

bdr-us-1 은 bdr-asia-1 로 join 한다.

```
[code lang=text]
# 메인 그룹 참여
testdb=# SELECT bdr.bdr_group_join(
local_node_name := 'us-node-001',
node_external_dsn := 'host=bdr-us-1 port=5432 dbname=testdb',
join_using_dsn := 'host=bdr-asia-1 port=5432 dbname=testdb'
);
bdr_group_join
----------------

(1 row)

# 확인 하는 쿼리
testdb=# SELECT bdr.bdr_node_join_wait_for_ready();
bdr_node_join_wait_for_ready
------------------------------

(1 row)
[/code]
```

## 4. bdr-us-2 서버에서의 설정

bdr-us-2 는 bdr-us-1 로 join한다.

```
[code lang=text]
# 메인 그룹 참여
testdb=# SELECT bdr.bdr_group_join(
local_node_name := 'us-node-002',
node_external_dsn := 'host=bdr-us-2 port=5432 dbname=testdb',
join_using_dsn := 'host=bdr-us-1 port=5432 dbname=testdb'
);
bdr_group_join
----------------

(1 row)

# 확인 하는 쿼리
testdb=# SELECT bdr.bdr_node_join_wait_for_ready();
bdr_node_join_wait_for_ready
------------------------------

(1 row)
[/code]
```

## 5. bdr-eu-1 서버에서의 설정

bdr-eu-1 은 asia-1 로 join 한다.

```
[code lang=text]
# 메인 그룹 참여
testdb=# SELECT bdr.bdr_group_join(
local_node_name := 'europe-node-001',
node_external_dsn := 'host=bdr-eu-1 port=5432 dbname=testdb',
join_using_dsn := 'host=bdr-asia-1 port=5432 dbname=testdb'
);
bdr_group_join
----------------

(1 row)

# 확인 하는 쿼리
testdb=# SELECT bdr.bdr_node_join_wait_for_ready();
bdr_node_join_wait_for_ready
------------------------------

(1 row)
[/code]
```

## 6. bdr-eu-2 서버에서의 설정

bdr-eu-2는 bdr-eu-1로 join한다.

```
[code lang=text]
# 메인 그룹 참여
testdb=# SELECT bdr.bdr_group_join(
local_node_name := 'europe-node-003',
node_external_dsn := 'host=bdr-eu-2 port=5432 dbname=testdb',
join_using_dsn := 'host=bdr-eu-1 port=5432 dbname=testdb'
);
bdr_group_join
----------------

(1 row)

# 확인 하는 쿼리
testdb=# SELECT bdr.bdr_node_join_wait_for_ready();
bdr_node_join_wait_for_ready
------------------------------

(1 row)
[/code]
```

---

To be continue ...
