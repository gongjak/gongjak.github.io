---
title:  "PostgreSQL 용량 산정하기"
excerpt:   "PostgreSQL을 운영하기 위한 용량 산정 (Capacity calculation)"
toc: true
toc_sticky: true
header:
  teaser: /assets/img/postgresql-log.png

categories:
  - PostgreSQL
tags:
  - PostgreSQL
last_modified_at: 2021-03-31T11:51:13+0900
comments: true
---
# 개요
DB 서버를 운영하기 위해

# PostgreSQL을 운영하기 위한 용량 산정 (Capacity calculation)

## PostgreSQL의 최대 메모리 용량

서버의 메모리를 고려하여 max_connections는 스왑하지 않을 정도로 설정한다.
DB에 연결시 소비하는 최대 메모리 견적은 다음과 같이 산출한다.

```bash
postmaster 크기 * max_connections
```

## 디스크 구성 용량
아래의 권장 용량은 절대값이 아닌 최소 용량이며, 서비스와 디스크 전체 용량에 따라 적절히 조절하길 바란다.

| 마운트 파티션 | 권장 용량 | 설명                                                                                                                                                                                                                                            |
|:--------------|:----------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| /postgresql   | 5~10 GB   | PostgreSQL DBMS Engine                                                                                                                                                                                                                          |
| /pg_xlog      | 50 GB     | 트랜잭션의 양에 맞게. </br> max_wal_size * 20, checkpoint_segments * 16 * 60.</br> 아카이브 디스크 여유 공간 부족으로 pg_xlog 쪽 공간을 계속 쓸 것을 대비해 그 장애 발생 대책을 처리할 수 있는 소요 기간 동안 견딜 수 있는 여유공간을 잡아둔다. |
| /data         | 150 GB    | 서비스와 디스크 용량에 맞게.</br> 물리적인 디스크 공간의 3배 이상.                                                                                                                                                                              |
| /archive      | 90 GB     | 통상 1~2일 보관후 백업 서버로 이동. </br>또는 보관 주기에 맞게.                                                                                                                                                                                 |

## DB 용량 산정
용량 산정 방식
```
테이블 row 당 Byte 길이 합 * 일 발생 건수 예상치 / 1024 / 1024 = 용량 mb
```



## Database 크기 구하기
1. Database 전체 크기
    ```sql
    select pg_size_pretty(pg_database_size('DBName'));
    ```
    ```sql
    postgres=$ select pg_size_pretty(pg_database_size('postgres'));
     pg_size_pretty
    ----------------
     7877 kB
    (1 row)
    ```
2. Database별 사용량
    ```sql
    select datname, pg_size_pretty(pg_database_size(datname)) from pg_database;
    ```
    ```sql
    postgres=$ select datname, pg_size_pretty(pg_database_size(datname)) from pg_database;
      datname  | pg_size_pretty
    -----------+----------------
     postgres  | 7877 kB
     test      | 7877 kB
     template1 | 7877 kB
     template0 | 7729 kB
    (4 rows)
    ```
3. 테이블 스페이스별 사용량
    ```sql
    select spcname, pg_size_pretty(pg_tablespace_size(spcname)) from pg_tablespace;
    ```
    ```sql
    postgres=$ select spcname, pg_size_pretty(pg_tablespace_size(spcname)) from pg_tablespace;
      spcname   | pg_size_pretty
    ------------+----------------
     pg_default | 31 MB
     pg_global  | 559 kB
    (2 rows)
    ```

## Table 크기 구하기
1. 테이블 크기 (index 미포함)
    ```sql
    select pg_size_pretty(pg_relation_size('TableName'));
    ```
    ```sql
    postgres=$ select pg_size_pretty(pg_relation_size('pg_stat_activity'));
     pg_size_pretty
    ----------------
     0 bytes
    (1 row)
    ```

2. 테이블 총 크기 (index 포함)

    ```sql
    select pg_size_pretty(pg_total_relation_size('TableName'));
    ```
    ```sql
    postgres=$ select pg_size_pretty(pg_total_relation_size('pg_stat_activity'));
     pg_total_relation_size
    ------------------------
                          0
    (1 row)
    ```

3. 테이블의 Index 크기
    ```sql
    select pg_size_pretty(pg_relation_size('IndexName'));
    ```
---

# 참고 1) PostgreSQL DB 용량 확인하기
## /* 테이블별 용량 보기 */
```sql
SELECT nspname || '.' || relname AS "relation",
    pg_size_pretty(pg_total_relation_size(C.oid)) AS "total_size"
  FROM pg_tables A, pg_class C
  LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
  WHERE nspname NOT IN ('pg_catalog', 'information_schema')
    AND C.relkind <> 'i'
    AND A.tablename = C.relname
    AND A.tableowner = 'tacs'
    AND nspname !~ '^pg_toast'
  ORDER BY pg_total_relation_size(C.oid) DESC;
```

## /* 테이블 전체 용량 보기 */
```sql
SELECT pg_size_pretty(sum(cast(total_size as bigint))) FROM (
SELECT nspname || '.' || relname AS "relation",
    pg_total_relation_size(C.oid) AS "total_size"
  FROM pg_tables A, pg_class C
  LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
  WHERE nspname NOT IN ('pg_catalog', 'information_schema')
    AND C.relkind <> 'i'
    AND A.tablename = C.relname
    AND A.tableowner = '테이블 오너'
    AND nspname !~ '^pg_toast'
  ORDER BY pg_total_relation_size(C.oid) DESC) as foo;
```

> 출처: [Internet play room](https://interwater.tistory.com/entry/PostgreSQL-db-용량-확인-하기)

---
# 참고 2) 데이타베이스 용량 설계
## 1.  데이타베이스 용량 분석의 목적
1. 정확한 데이타 용량을 산정하여 디스크 사용의 효율을 높인다.
2. 업무량이 집중되어 있는 디스크를 분리,설계하여 디스크에 대한 입출력 부하를 분산시킬 수 있다.
3. 디스크 입출력 경합을 최소화할 수 있다.
4. 데이타베이스 오브젝트의 익스텐트(범위,Extent) 발생을 줄인다 - 데이타가 증가하면서 데이타베이스 내의 각종 오브젝트들도 추가적인 스페이스 확보 작업이 일어나는데 이를 최소하 해야 한다.


## 2. 데이타베이스 용량 분석 절차
###    1. 용량분석을 위한 기초 데이타를 수집한다
1) 로우길이:해당 테이블의 칼럼길이를 모두 합하여 기록한다.
2) 보존기간:테이블을 디스크에 어느 정도의 기간동안 보관할 것인지 기록
3) 초기건수:기존 시스템에서 신시스템으로 마이그레이션 한 데이타 건수
4) 발생건수:일정주기별 발생한 내용
5) 발생주기:
6) 년 증가율:


###    2. 기초 데이타를 이용하여 DBMS에 이용하는 오브젝트별로 용량을 산정한다
1) 오브젝트설계:저장공간을 주로 차지하는 오브젝트에는 테이블스페이스,테이블,인덱스등이다.
2) 테이블스페이스 용량산정
    - 분산위치:테이블이 분산되는 위치
    - 테이블명:
    - 테이블용량:
    - 테이블스페이스명:
    - 테이블스페이스 용량:테이블 스페이스내에 생성되는 테이블의 용량을 더한 값의 약 40%를 더한 값을 기입한다. 40%는 절대적인 값은 아니며 경험과 해당 업무의 확장성을 고려하여 적절히 더한다.
3) 데이타파일 용량 산정
    - 디스크:데이타 파일이 물리적으로 생성될 디스크의 이름
    - 데이타파일 디렉토리:
    - 데이타파일명:
    - 데이타파일 크기:저장될 테이블 스페이스 용량을 합하여 지정한다.
    - 테이블 스페이스:
    - 테이블 스페이스 용량:
    - 비고:테이블 스페이스가 여러 개의 데이타 파일을 이용할 경우 참조자료로 기입한다.

    (?) 디스크의 종류에 따라 데이타파일 크기를 지정해야 한다? ->로우디스크,파일시스템 (페이지 315)

4) 디스크 용량 산정
    - 디스크:데이타파일이 물리적으로 생성될 디스크 이름
    - 테이타파일 디렉토리:
    - 디스크용량:하나의 디스크가 가진 원초적인 용량을 기록한다.
    - 사용된 디스크 용량:디스크에 데이타파일이 생성된 총 용량을 기록한다.
    - 디스크 사용비율 : (디스크용량/사용된 디스크 용량)*100 디스크 구성 방법에 따라 사용비율을 조절해야 한다.
    - 데이타파일명:
    - 데이타파일크기:저장될 테이블 스페이스 용량을 합하여 지정한다.
    - 데이타파일용량

> 출처 : [꿈꾸는 개발자, DBA 커뮤니티 구루비](http://www.gurubee.net/lecture/4244)
