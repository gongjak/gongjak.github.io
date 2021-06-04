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
    pg_settings
    ```sql
    SELECT name, setting, boot_val, reset_val, unit
    FROM pg_settings
    ORDER BY name;
    ```
    postgresql.conf
    ```bash
    $PGDATA/postgresql.conf
    ```

2. DBMS 정보 확인
    1. 테이블 리스트 확인
        ```sql
        SELECT table_name FROM information_schema.tables
        WHERE table_schema = 'public';
        ```
    2. 테이블 컬럼 정보 확인
        ```sql
        SELECT table_name, column_name, data_type, ordinal_position
        FROM information_schema.columns
        WHERE table_schema = 'public'
        ORDER BY table_name, ordinal_position;
        ```
    3. 인덱스 정보 확인
        ```sql
        SELECT a tablename, a.indexname, b.column_name
        FROM pg_catalog.pg_indexes a, information_schema.columns b
        WHERE a.tablename = b.table_name;
        ```
3. 트랜잭션 정보 확인
    1. 접속된 사용자 확인
        ```sql
        ```
    2. Active 세션 확인
    ```sql
    ```
    3. 현재 실행중인 SQL 상태 정보 확인
    ```sql
    ```
    4.
    ```sql
    ```
