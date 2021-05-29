---
title:  "Running Postgres-BDR with GCP - 4"
excerpt:   "PostgreSQL로 Multi-Master를 만들자"
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
  - GCP
last_modified_at: 2016-10-18T14:50:13+0900
comments: true
---

# 개요
>Open Source RDBMS인 PostgreSQL은 기본 구성으로 Master 서버를 1대만 구성할 수 있다.
>그러나 Postgres-BDR을 이용하면 Multi-Master 구성이 가능하며,
>GCP의 네트워크를 이용하여 Multi-Region 구성으로 PostgreSQL의 데이터를 동기화 할 수 있다.

# pgbench 를 이용한 Performance Test

- pgbench 테스트를 위한 테이블을 생성한다.

```bash
$ psql testdb
testdb=# create table tbl_bdr(c1 int);
CREATE TABLE
testdb=# \q
```

- Performance Test 를 위해 준비해준다.

```bash
postgres@bdr-asia-2:~$ pgbench -U postgres -P 5432 -i testdb
NOTICE: table "pgbench_history" does not exist, skipping
NOTICE: table "pgbench_tellers" does not exist, skipping
NOTICE: table "pgbench_accounts" does not exist, skipping
NOTICE: table "pgbench_branches" does not exist, skipping
creating tables...
100000 of 100000 tuples (100%) done (elapsed 0.15 s, remaining 0.00 s).
vacuum...
set primary keys...
done.
```

-U postgres : postgres 유저로 접속
-P 5432 : 5432 포트로 접속
-i : 테스트준비
testdb : 디비로 testdb 를 이용하겠다.

```bash
postgres@bdr-asia-2:~$ pgbench -U postgres -p 5432 -S -c 10 -t 10000 testdb
starting vacuum...end.
transaction type: SELECT only
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
number of transactions per client: 10000
number of transactions actually processed: 100000/100000
latency average: 0.000 ms
tps = 11496.597122 (including connections establishing)
tps = 11520.436044 (excluding connections establishing)
```

-U postgres 유저로 로그인
-P 5432번 포트를 통해 접속
-S TCP-B 기준으로 테스트해서 결과를 얻겠다.
-c 동시연결 Client 는 10명
-t 트랜잭션을 10,000개로 하겠다.

부하를 주고 싶은 SQL 문을 만들어서 테스트 할 수도 있다.

```bash
postgres@bdr-asia-2:~$ cat test.sql
insert into tbl_bdr values (3);
```

test.sql 에는 부하를 주고 싶은 하나의 SQL 문만을 적어 놓는다.
반드시 한줄로 적어야 한다. 여러 줄 일때는 에러가 발생할 수 있다.

```bash
$ pgbench -U postgres -n -S -T 60 -c 10 -f test.sql testdb
```

10 동시 유저로 60초 동안 test.sql 의 SQL 문을 실행하라는 의미이다.

```bash
postgres@bdr-asia-2:~$ pgbench -U postgres -n -S -T 60 -c 10 -f test.sql testdb
transaction type: Custom query
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
duration: 60 s
number of transactions actually processed: 379920
latency average: 1.579 ms
tps = 6331.344284 (including connections establishing)
tps = 6332.955115 (excluding connections establishing)
```

실행하는 동안 vmstat 이나 sar, nmon 등으로 시스템 리소스를 모니터링 한다.

---

# 주의 사항 (0.9 에서 해당됨 / 1.0 에서는 확인이 필요함)

## 1. 노드명 삭제 불가

- bdr.bdr_group_join 에 참여한 노드는 서버 이름을 변경하지 않으면 다시 등록이 안됨.

bdr.bdr_part_by_node_names()로 group에서 제외시켰음에도 불구하고,bdr.bdr_nodes 목록에 여전히 등록되어 있으며, 같은 서버 이름으로는 이미 등록되어 있다고 등록이 안됨

- 한 번 그룹에서 빠진 서버는 서버명을 변경해서 join하면 될거 같으나 클라우드 서버의 특성상 서버명 변경이 불가하기에 다른 서버를 생성하여 추가해야 함.
- bdr.bdr_part_by_node_names() 함수의 버그로 보이며, 함수 실행 후 bdr.bdr_nodes 와 bdr.bdr_connections 의 정보를 확인하여 삭제해 주고, bdr_init_copy 명령어로 노드 추가를 할 수 있다.
- 삭제는 노드 중 한 곳에서만 실행해도 된다.

```sql
select * from bdr.bdr_nodes;
delete from bdr.bdr_nodes where node_name='asia-node-003';
select * from bdr.bdr_connections;
delete from bdr.bdr_connections where conn_sysid='6189506087542501647';
```

## 2. Global 시퀀스를 사용해야 함

http://bdr-project.org/docs/stable/global-sequences.html
* 시퀀스를 사용하는 경우 글로벌 시퀀스를 사용해야 함. 그렇지 않으면 fail over가 발생했을 경우 시퀀스 번호가 다른 노드에 자동 증가가 되지 않아서 오류가 발생함.
* 문법적으로 약간 다름. 예) create sequence access_log_seq USING bdr;
* create sequence xxxx increment 1 minvalue 1 maxvalue 999999999999 start 1 cache 1;
    구문과 같이 minvalue, maxvalue를 지정할수 없고cache 도 사용불가. minvalue는 1로 고정됨* 데이터 타입이 전부 bigserial(bigint) 이어야 함.
* 시퀀스 번호가 순차적으로 생성 되지 않음.예를 들어 노드1번이 1번부터 시작한다면 노드 2번은 100000번부터 노드3번은10000000부터 시작함. 내부적으로 트랜잭션 동기화 때문인거 같음

## 3. ddl 문 실행시 제약 조건

http://bdr-project.org/docs/stable/ddl-replication.html* 다른 제약도 많지만 alter 문 실행시 default 값이 있을 경우 실행이 안됨* 자세한 내용은 메뉴얼 참조

## 4. 로그

- fail over 발생시 로그가 계속 출력됨. log rotate를 하지 않을 경우 문제가 될거 같음
- (09/01 추가) 서버 fail 이 발생했을 경우에는 재빨리 replication group에서 제거해줘야 함.
- 노드 제거 시에도 발생하게 되는데, bdr.bdr_part_by_node_names() 함수의 버그로 보이며 함수 실행 후, 제거된 노드에서 bdr.bdr_nodes 와 bdr.bdr_connections 의 정보를 확인하여 삭제하면 더 이상 추가 로그가 발생하지 않는다.
- 삭제는 노드 중 한 곳에서만 실행해도 된다.

```sql
select * from bdr.bdr_nodes;
delete from bdr.bdr_nodes where node_name='asia-node-003';
select * from bdr.bdr_connections;
delete from bdr.bdr_connections where conn_sysid='6189506087542501647';
```

## 5. table 선택적 replication 사용 가능 ==> 추가 테스트 필요

http://bdr-project.org/docs/0.9.0/functions-replication-sets.html

- 특정 table을 replicaion 하고 싶지 않을 경우

```sql
select bdr.table_set_replication_sets('user_info', '{}');
```

를 실행하면 user_info table 이 복제가 되지 않는다 table 속성을 보면 default 속성이 빠진걸 볼수 있음

- 복원은 bdr.table_get_replication_sets() 사용

---

# 테스트 마무리

Cloud 서비스가 생기기 전에는 Global 서비스를 생각하는 기업은 별로 없었다. Global 서비스라 하더라도 그 내용에 따라 각 지역별로 서버를 구성하고 독립적으로 운영하는 방식이 대부분이다. 하나의 단일 DB를 가지고 서비스를 한다는 것은 결코 쉽지 않은 일인데, Postgres-BDR 은 이걸 가능하게 해준다.

물론 장점만 있는 것은 아니다.
Postgres-BDR 은 Asynchronous 방식의 동기화를 지원하고 있다. Asnync 방식은 상황에 따라 Query 의 결과가 다를 수도 있음을 의미한다. 이런 동기화의 지연은 서비스에 따라 매우 치명적일 수 있다.MySQL 에 비하면 PostgreSQL 은 아직 국내에선 낯설고 사용자도 많지 않은 상황인데다가 아직 국내에는 2ndQuadrant 의 협력사가 없어서 기술 지원도 문제가 될 수 있다. 그럼에도 불구하고 Postgres-BDR 이란 솔루션은 많은 장점을 가지고 있다고 생각한다.

개인적으로도 PostgreSQL 이 궁금해져서 [PostgreSQL Korea User Group](https://www.facebook.com/groups/postgres.kr/) 의 Study 모임에도 참여하고 있다. 아직도 봐야할 것들이 많은 PostgreSQL 이지만 Global 서비스를 생각한다면 감히 사용해보길 추천한다.
