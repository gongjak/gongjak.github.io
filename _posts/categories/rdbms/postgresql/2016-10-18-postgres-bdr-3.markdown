---
title:  "Running Postgres-BDR with Google Cloud Platform-3"
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
last_modified_at: 2016-10-18T14:50:13+0900
comments: true
---

# 개요
>Open Source RDBMS인 PostgreSQL은 기본 구성으로 Master 서버를 1대만 구성할 수 있다.
>그러나 Postgres-BDR을 이용하면 Multi-Master 구성이 가능하며,
>GCP의 네트워크를 이용하여 Multi-Region 구성으로 PostgreSQL의 데이터를 동기화 할 수 있다.

# Postgres-BDR 운영

[Node Management Manual](http://bdr-project.org/docs/stable/node-management.html)

## 노드 추가

- 노드 추가는 pg_basebackup 으로 SOURCE로부터 전체 백업을 받아온 후, bdr_init_copy 라는 명령어를 이용한다.
- 데이터가 계속 인입되는 runtime 환경에서 테스트한 결과, 노드 추가 후 group 에 join되어 곧 바로 동기화가 진행되었다.
- 노드 추가에 영향을 주는 환경변수는 다음과 같다.

```bash
# 접속가능한 슬레이브의 접속수를 설정( 슬레이브 수 + 2)인거 같은데.. backup용 확실치 않음
max_wal_senders = 8
# 노드 + 1
max_replication_slots = 7
# BDR과 UDR 모두이 구성 데이터베이스 당 하나의 작업자 및 연결 당 하나의 작업자을 가지고 충분히 큰 값으로 설정해야합니다.
max_worker_processes = 10
```

- 환경변수를 꼭 확인하고 신규로 추가할 서버에서 Postgres-BDR 을 설치하고 아래의 스크립트를 그대로 이용하면 된다.

```bash
# Created by Peter Yun. 2015-09-01.
#!/bin/bash
SOURCE="bdr-asia-2"
TARGET="bdr-asia-3"
NODENAME="asia-node-003"

sudo service postgresql stop
sudo rm -rf /var/lib/postgresql/9.4/main/*
sudo su -l postgres -c "pg_basebackup -h $SOURCE -U postgres -D /var/lib/postgresql/9.4/main -xlog -c fast -P"

sudo su -l postgres -c "/usr/lib/postgresql/9.4/bin/bdr_init_copy -d testdb -h $SOURCE -p 5432 \
--local-dbname=testdb \
--local-host=$TARGET --local-port=5432 \
--pgdata=/var/lib/postgresql/9.4/main \
--postgresql-conf=/etc/postgresql/9.4/main/postgresql.conf \
--hba-conf=/etc/postgresql/9.4/main/pg_hba.conf \
--node-name=$NODENAME "

sleep 10
sudo su -l postgres -c "tail -20 ~/bdr_init_copy_postgres.log"

# bdr 노드 추가 후 서버 중지
echo ""
echo ""
echo "bdr_init_copy_postgres.log 를 확인하여 이상없으면, bdr_init_copy 로 실행된 postgresql을 중지 하고 service 로 다시 띄운다."
echo "----------"
echo "sudo su -l postgres -c \"tail -20 ~/bdr_init_copy_postgres.log\""
echo "sudo su - postgres"
echo "/usr/lib/postgresql/9.4/bin/pg_ctl --pgdata=/var/lib/postgresql/9.4/main stop -m fast"
echo "exit"
echo "sudo service postgresql start"
echo "----------"
```

## 노드 제거

- 함수로 쉽게 노드를 제거 할 수 있다고 하나, 실제로는 함수 실행 후 확인해보면 노드 정보가 남아있다. (select * from bdr.bdr_nodes;)

```sql
# node 하나만 제거
SELECT bdr.bdr_part_by_node_names(ARRAY['node-1']);
# 여러개의 노드를 한 번에 제거
SELECT bdr.bdr_part_by_node_names(ARRAY['node-1', 'node-2', 'node-3']);
# asia-node-003 제거
SELECT bdr.bdr_part_by_node_names(ARRAY['asia-node-003']);
```

- 때문에 함수 실행 후 다음을 확인하여 남아있는 정보를 함께 지워준다.

```sql
# bdr_nodes 를 확인해서 제거한 노드가 남아 있으면 삭제해준다.
select * from bdr.bdr_nodes;
delete from bdr.bdr_nodes where node_name='asia-node-003';
# bdr_connections 를 확인해서 제거한 노드의 정보가 남아 있으면 삭제해준다.
select * from bdr.bdr_connections;
delete from bdr.bdr_connections where conn_sysid='6189502444603979480';

SELECT * FROM pg_replication_slots;
SELECT pg_drop_replication_slot('bdr_16386_6319248612493171876_2_16386__');
```

---

# Postgres-BDR 모니터링

[Monitoring manual](http://bdr-project.org/docs/stable/monitoring.html)

- 노드 확인

```sql
SELECT * FROM bdr.bdr_nodes;
```

- Monitoring connected peers using pg_stat_replication

```sql
SELECT * FROM pg_stat_replication;

SELECT
pg_xlog_location_diff(pg_current_xlog_insert_location(), flush_location) AS lag_bytes,
pid, application_name
FROM pg_stat_replication;
```

- Monitoring replication slots

```sql
SELECT * FROM pg_replication_slots;

SELECT
slot_name, database, active,
pg_xlog_location_diff(pg_current_xlog_insert_location(), restart_lsn) AS retained_bytes
FROM pg_replication_slots
WHERE plugin = 'bdr';
```

- Monitoring global DDL locks

```sql
select * from bdr.bdr_global_locks ;
```

---

# 백업 관련

- 증분 백업
- barman을 사용하여 증분 백업을 하고 싶지만… bdr를 사용할 경우, wal_level = 'logical' 옵션 때문에 증분 백업을 사용할수 없음
- 증분 백업을 위해서는 wal_level 값이 'archive', 'hot_standby' 값을 가져야만 함.
- 노드 추가가 어렵지 않은 만큼 데이터 백업은 snapshot 또는 pg_dump, pg_basebackup 만 해도 괜찮을 듯 하다.

---

# 파티션 테이블 관리

- 월별 range 파티션 사용 시 자동 파티션 생성 procedure 및 job 스케줄러 등록하여 관리가 가능한지 추가 확인 필요

---

To be continue ...
