---
title:  "PostgreSQL 모니터링 Query"
excerpt:   "PostgreSQL에서 자주 사용하는 모니터링 Query 모음"
toc: true
toc_sticky: true
header:
  teaser: /assets/img/postgresql-logo.png

categories:
  - PostgreSQL
tags:
  - PostgreSQL
  - Monitoring
last_modified_at: 2021-04-02T18:47:13+0900
comments: true
---
# 개요
PostgreSQL에서 자주 사용하는 모니터링 Query 모음

# PostgreSQL 모니터링 query 모음

1. DB 설정값 확인
    1. pg_settings 확인
        ```sql
        SELECT name, setting, boot_val, reset_val, unit
        FROM pg_settings
        ORDER BY name;
        ```
    2. postgresql.conf
        ```bash
        $PGDATA/postgresql.conf
        ```

1. 데이터베이스 조회
    1. 모든 database 보기
        ```sql
        \list
        /* or */
        \l
        ```
    2. database 이름만 보기
        ```sql
        SELECT datname FROM pg_database;
        ```

1. 스키마 조회
    1. 스키마 불러오기
        ```sql
        \dn
        ```
    2. pg_catalog를 통해서 스키마 불러오기
        ```sql
        SELECT * FROM pg_catalog.pg_namespace;
        ```

1. 테이블 조회
    1. public 테이블 조회
        ```sql
        \dt
        /* or */
        SELECT * FROM pg_catalog.pg_tables;
        ```
    2. 특정 스키마의 테이블 조회
        ```sql
        \dt schema_name.*
        ```
    3. |로 여러 스키마와 테이블을 필터링
        ```sql
        \dt (public|schema_name).(a_table|b_table)
        ```
    4. 모든 테이블 보기
        ```sql
        SELECT table_schema, table_name FROM information_schema.tables
        ORDER BY table_schema, table_name;
        ```
    5. public 테이블 리스트 확인
        ```sql
        SELECT table_name FROM information_schema.tables
        WHERE table_schema = 'public';
        ```
    6. 테이블 컬럼 정보 확인
        ```sql
        SELECT table_name, column_name, data_type, ordinal_position
        FROM information_schema.columns
        WHERE table_schema = 'public'
        ORDER BY table_name, ordinal_position;
        ```
    7. 인덱스 정보 확인
        ```sql
        SELECT a tablename, a.indexname, b.column_name
        FROM pg_catalog.pg_indexes a, information_schema.columns b
        WHERE a.schemaname = 'public'
        AND a.tablename = b.table_name;
        ```

1. 트랜잭션 정보 확인
    1. 접속된 사용자 확인
        ```sql
        SELECT pid, datname, usename, query FROM pg_stat_activity;
        ```
    1. Active 세션 확인
        ```sql
        SELECT datname, usename, state, query FROM pg_stat_activity
        WHERE state = 'active';
        ```
    1. 현재 실행중인 SQL 상태 정보 확인
        ```sql
        SELECT current_timestamp - query_start AS runtime,
            datname, usename, query
        FROM pg_stat_activity
        WHERE state = 'active' ORDER BY 1 DESC;
        ```
    1. 1분 이상 실행되는 쿼리 확인
        ```sql
        SELECT current_timestamp - query_start AS runtime,
            datname, usename, query
        FROM pg_stat_activity
        WHERE state = 'active'
            AND current_timestamp - query_start > '1min'
        ORDER BY 1 DESC;
        ```
        Query를 process title에 보이도록 설정하기
        ```bash
        update_process_title = on
        ```
    1. wait 또는 blocking 되는 세션 확인
        ```sql
        SELECT datname, usename, query
        FROM pg_stat_activity
        WHERE waiting = true;
        ```
    1. query block user 찾기
        ```sql
        SELECT w.query AS waiting_query,
            w.pid AS waiting_pid,
            w.usename AS waiting_user,
            l.query AS locking_query,
            l.pid AS locking_pid,
            l.usename AS locking_user,
            t.schemaname ||'.' t.relname AS tablename

        ```
    1. 현재 실행중인 SQL 상태 정보 확인
        ```sql
        ```

    1. asdfasdf
        ```sql
        ```

1. 트랜잭션 정보 확인
    1. 접속된 사용자 확인
        ```sql
        ```
    1. Active 세션 확인
        ```sql
        ```
    1. 현재 실행중인 SQL 상태 정보 확인
        ```sql
        ```
    1.
        ```sql
        ```
