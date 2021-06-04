---
title:  "postgresql.conf 설정"
excerpt:   "PostgreSQL의 기본 설정 권고값"
toc: true
toc_sticky: true
header:
  teaser: /assets/img/postgresql-logo.png

categories:
  - PostgreSQL
tags:
  - PostgreSQL
last_modified_at: 2021-04-29T17:24:13+0900
comments: true
---

# 개요

PostgreSQL을 설치하고 운영함에 있어 꼭 설정해야 하는 항목들과 설정하는 좋은 것들을 중심으로 정리해보았다.
버전에 따라 없는 것들도 있을 수 있고, 빠진 항목들도 있으며, 절대적인 값이 아니기에 본인의 운영 환경에 맞춰서 값을 변경하길 바란다.

# postgresql.conf 설정 권고값

recommend 값은 다음 시스템 사양을 기준으로 함. ([pgtune](https://pgtune.leopard.in.ua/#/) 사이트를 참조.)

* PostgreSQL Version : 13
* OS Type: linux
* DB Type: web
* Total Memory (RAM): 32 GB
* CPUs num: 16
* Connections num: 1000
* Data Storage: ssd

`value`\** : 꼭 설정해야 할 항목들

---

## Connection And Authentication

| default                         | recommend                      | note                                  |
|:--------------------------------|:-------------------------------|:--------------------------------------|
| #listen_addresses = 'localhost' | #listen_addresses = `'*'`\**   | 외부로 부터 DB 접속을 모두 허용       |
| port              = 5432        | port              = `15432`\** | 원하는 포트로 변경하여 사용을 권고    |
| #max_connections  = 100         | #max_connections  = `1000`\**  | DB 서버에 대한 최대 동시 연결 수 설정 |

## Resource Usage (except WAL)

| default                                  | recommend                                      | note                                                                                                                                                                                                                                                                                                                                                                                                |
|:-----------------------------------------|:-----------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #shared_buffers                   = 32GB | shared_buffers                   = `8GB`\**    | DB 서버가 공유 메모리 버퍼에 사용하는 메모리양 설정 (최소 128 kb). <br>Disk IO 최소화 목적.<br> 메모리가 1GB 이상의 서버라면 **시스템 메모리의 25% 권고**. <br>PostgreSQL은 운영 체제 캐시에 의존 하기 때문에 지나치게 많이 할당하면 오히려 성능이 하락할 수 있음.                                                                                                                                  |
| #temp_buffers                     = 8MB  | temp_buffers                     = 10MB        | min 800kB. <br>각 DB 세션에서 사용하는 임시 버퍼의 최대. <br>임시 테이블에 대한 엑세스에만 사용되는 세션 버퍼. <br>필요에 따라 최대 temp_buffers가 지정한 한계까지 할당.                                                                                                                                                                                                                            |
| #work_mem                         = 4MB  | work_mem                         = `2097kB`\** | min 64KB. <br>임시 디스크 파일에 쓰기 전에 내부 정렬 작업 및 해시 테이블에서 사용할 메모리양 설정(sort, bitmap, hash join, merge join 작업). <br>복잡한 쿼리일수록 여러 세션이 동시에 작업할 수 있기 때문에 사용된 전체 메모리는 work_mem의 몇 배가 될 수 있음. <br>세션마다 다르게 설정 가능. <br>**최대크기 = total RAM / max_connections / 4.<br> 안전크기 = total RAM / max_connections / 16.** |
| #maintenance_work_mem             = 64MB | maintenance_work_mem             = `2GB`\**    | min 1MB. <br>create index 혹은 alter table add 등 유지 보수 작업에서 사용할 최대 메모리 양 설정. <br>work_mem 값보다 크게 설정하여 안전하게 운용. <br>데이터 제거 및 DB 덤프 복원 성능 향상. <br>**1GB당 50MB 권고**                                                                                                                                                                                |
| #max_worker_processes             = 8    | max_worker_processes             = `16`\**     | 백그라운드 프로세스의 최대 수                                                                                                                                                                                                                                                                                                                                                                       |
| #effective_io_concurrency         = 1    | effective_io_concurrency         = `200`\**    |                                                                                                                                                                                                                                                                                                                                                                                                     |
| #max_parallel_workers             = 8    | max_parallel_workers             = `16`\**     | 병렬 작업을 지원할 수 있는 최대 프로세스 수. <br>max_worker_processes 보다 높게 설정해도 해당 설정 프로세스 풀에서 가져온 것이므로 아무런 영향을 미치지 않음. <br>이 값을 변경할때 max_parallel_maintenance_workers 및 max_parallel_workers_per_gather 값도 조정.                                                                                                                                   |
| #max_parallel_workers_per_gather  = 2    | max_parallel_workers_per_gather  = `4`\**      |                                                                                                                                                                                                                                                                                                                                                                                                     |
| #max_parallel_maintenance_workers = 2    | max_parallel_maintenance_workers = `4`\**      |                                                                                                                                                                                                                                                                                                                                                                                                     |

## Write Ahead Log

| default                               | recommend                                | note                                                                                                                                                                                                                                                                          |
|:--------------------------------------|:-----------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #synchronous_commit           = on    | synchronous_commit           = off       | 쿼리가 클라이언트에 success 메시지를 반환하기 전에 WAL 레코드가 디스크에 기록되는 동안 트랜잭션 커밋이 대기할지 여부. <br>지연 시간이 최대 wal_writer_delay 값의 3배. <br>off 로 해놓더라도 DB 상태는 트랜잭션이 정상 중단된 경우와 같기 때문에 성능이 중요하다면 off를 권장. |
| #wal_buffers                  = -1    | wal_buffers                  = `16MB`\** | DB의 변경 사항을 임시 저장하는 버퍼.<br> 디스크에 아직 기록되지 않은 WAL 데이터에 사용되는 공유 메모리 양. <br>값이 클 수록, 클라이언트가 한 번에 커밋하는 사용량이 많은 서버에서 쓰기 성능을 향상 시킴.                                                                      |
| #wal_writer_delay             = 200ms | wal_writer_delay             = 200ms     | WAL 작성기가 WAL을 플러시하는 빈도.<br> 플러시를 한후 wal_writer_delay 값 동안 대기                                                                                                                                                                                           |
| #checkpoint_timeout           = 5min  | checkpoint_timeout           = 5min      | 자동 WAL 체크 포인트 간 최대 시간 (초)                                                                                                                                                                                                                                        |
| max_wal_size                  = 1GB   | max_wal_size                  = `4GB`\** |                                                                                                                                                                                                                                                                               |
| min_wal_size                  = 80MB  | min_wal_size                  = `1GB`\** |                                                                                                                                                                                                                                                                               |
| #checkpoint_completion_target = 0.5   | checkpoint_completion_target = `0.7`\**  |                                                                                                                                                                                                                                                                               |


## Replication

| default                     | recommend                        | note                                                                                                                                                                    |
|:----------------------------|:---------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #effective_cache_size = 4GB | `effective_cache_size = 24GB`\** | 데이터 캐싱에 사용할 수 있는 메모리 양. <br>인덱스 사용 여부를 결정.<br> shared_buffers 할당 메모리 + 사용 가능한 OS 캐시 메모리.<br> **전체 메모리의 50% ~ 75% 권고.** |

## Query Tuning

| default                                     | recommend                                | note                                                                                                                                                              |
|:--------------------------------------------|:-----------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #default_statistics_target      = 100       | `default_statistics_target     = 100`\** | ALTER TABLE SET STATISTICS 를 통해 열 특정 대상 세트가 없는 테이블 열의 기본 통계 대상을 설정.<br> 값이 클수록 analyze를 수행하는 시간은 늘지만 예상 값은 향상 됨 |
| #constraint_exclusion           = partition | constraint_exclusion          = on       | 테이블 제약 조건 사용을 제어하여 쿼리를 최적화함.<br> on : 모든 테이블 제약 조건 검사                                                                             |
| #random_page_cost               = 4.0       | `random_page_cost              = 1.1`\** | SSD 사용시 1.0 고정. 일반 HDD 사용시 2.5 사용.                                                                                                                    |
| #enable_bitmapscan              = on        |                                          |                                                                                                                                                                   |
| #enable_hashagg                 = on        | enable_hashagg                = on       | 일반적인 쿼리 Auto Plan                                                                                                                                           |
| #enable_hashjoin                = on        | enable_hashjoin               = on       | Hash Joint쿼리 Auto Plan                                                                                                                                          |
| #enable_indexscan               = on        | enable_indexscan              = on       | Index Scan Auto Plan                                                                                                                                              |
| #enable_indexonlyscan           = on        |                                          |                                                                                                                                                                   |
| #enable_material                = on        |                                          |                                                                                                                                                                   |
| #enable_mergejoin               = on        | enable_mergejoin              = on       | Merge Join Auto Plan                                                                                                                                              |
| #enable_nestloop                = on        | enable_nestloop               = on       | Nest Loop Auto Plan                                                                                                                                               |
| #enable_seqscan                 = on        | enable_seqscan                = on       | Sequence Scan Auto Plan                                                                                                                                           |
| #enable_sort                    = on        | enable_sort                   = on       | Sort Auto Plan                                                                                                                                                    |
| #enable_incremental_sort        = on        |                                          |                                                                                                                                                                   |
| #enable_tidscan                 = on        | enable_tidscan                = on       | TID Scan Auto Plan                                                                                                                                                |
| #enable_partitionwise_join      = off       |                                          |                                                                                                                                                                   |
| #enable_partitionwise_aggregate = off       |                                          |                                                                                                                                                                   |
| #enable_parallel_hash           = on        |                                          |                                                                                                                                                                   |
| #enable_partition_pruning       = on        |                                          |                                                                                                                                                                   |

seqscan과 nestloop의 경우 index scan보다 오히러 느려지는 경우가 있어 false로 두어야 할 떄가 있다. `explain analyse 쿼리;` 를 통해 어떤 쪽이 탐색에 더 유리한지에 따라 Default값을 변경한다.

## Reporting And Logging

| default                                          | recommend                                                          | note                                                                                                   |
|:-------------------------------------------------|:-------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------------|
| #log_destination = 'stderr'                      | log_destination            = `'stderr'`\**                         | 서버 메시지를 stderr에 기록. **vim logging_collector = on** .stderr로 전송 된 로그를 파일로 리다이렉션 |
| #log_directory = 'log'                           | log_directory              = `'log'`\**                            | 로그 파일 디렉토리 생성                                                                                |
| #log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' | log_filename               = `'postgresql-%Y-%m-%d_%H%M%S.log'`\** | 로그 파일 이름 설정                                                                                    |
| #log_truncate_on_rotation = off                  | log_truncate_on_rotation   = `on`\**                               | 설정한 시간만큼 파일에 추가됨.지나면 새로운 로그 파일 생성                                             |
| #log_rotation_age = 1d                           | log_rotation_age           = `1d`\**                               | 로그 파일의 로테이션 기간                                                                              |
| #log_rotation_size = 10MB                        | log_rotation_size          = `0`\**                                | 로그 파일 최대 사이즈. 크기를 기반으로 안두려면 0으로 지정                                             |
| log_line_prefix = '%m [%p] %q%u@%d '             | log_line_prefix            = `'%m [%p] %q%u@%r/%d '`\**            | 로그 라인 시작부분 문자열                                                                              |
| #log_min_duration_statement = -1                 | log_min_duration_statement = `1000`\**                             | 단위 ms. 1초 이상 수행된 query를 남긴다.                                                               |
| #log_lock_waits = off                            | log_lock_waits             = `on`\**                               | Lock으로 인해 query가 지연되면 log를 남긴다. deadlock_timeout 설정 시간 기준.                          |

## AUTOVACUUM

| default                                | recommend                                | note                                                                                                                                                                                                                                                                                                        |
|:---------------------------------------|:-----------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #autovacuum = on                       | autovacuum                      = on     | AUTOVACUUM 사용 설정                                                                                                                                                                                                                                                                                        |
| #autovacuum_vacuum_threshold = 50      | autovacuum_vacuum_threshold     = 100000 | vacuum 이 일어나기 위한 dead tuple 의 최소 갯수. <br>테이블 별로 설정하면 dead tuple이 100,000개가 생성될 때마다 Autovacuum이 동작함. 운영중 적용 가능.                                                                                                                                                     |
| #autovacuum_vacuum_scale_factor = 0.2  | autovacuum_vacuum_scale_factor  = 0      | vacuum 이 일어나기 위한 live tuple 대비 dead tuple 의 최소 비율. <br>이 값이 0 이면, autovacuum_vacuum_threshold 에 지정된 숫자만큼의 dead tuple 에 따라 Autovacuum 이 동작하게 되므로 훨씬 일관성 있는 성능을 확보할 수 있음. 운영중 적용 가능.                                                            |
| #vacuum_cost_delay = 0                 | vacuum_cost_delay               = 0      |                                                                                                                                                                                                                                                                                                             |
| #vacuum_cost_page_hit = 1              | vacuum_cost_page_hit            = 1      | page_hit 영역(shared buffer 영역)에 있는 데이터를 vacuuming 할 때 마다 1 의 credit 을 소모.                                                                                                                                                                                                                 |
| #vacuum_cost_page_miss = 10            | vacuum_cost_page_miss           = 10     | page_miss 영역(디스크 영역)에 있는 데이터를 vacuuming 할 때 마다 10 의 credit 을 소모.                                                                                                                                                                                                                      |
| #vacuum_cost_page_dirty = 20           | vacuum_cost_page_dirty          = 20     | page_dirty 영역에 있는 데이터를 vacuuming 할 때 마다 20 의 credit 을 소모.                                                                                                                                                                                                                                  |
| #vacuum_cost_limit = 200               | vacuum_cost_limit               = 200    | Autovacuum 이 한 번 실행될 때, 해당 프로세스는 200 의 credit 을 가지고, 200 의 credit 이 모두 소진되면 해당 Autovacuum 프로세스는 종료됨.                                                                                                                                                                   |
| #autovacuum_vacuum_cost_limit = -1     | autovacuum_vacuum_cost_limit    = 1000   | -1 일 경우, 해당 값은 vacuum_cost_limit 을 참조한다. 테이블 별로 설정이 가능.<br> 1000으로 조정하면 기본보다 약 5배 많은 vacumming을 한 번에 처리함. 운영중 적용 가능.                                                                                                                                      |
| #autovacuum_analyze_scale_factor = 0.1 | autovacuum_analyze_scale_factor = 100000 | 테이블 별로 autovacuum_vacuum_threshold 와 동일한 값으로 설정 권고. 운영중 적용 가능.                                                                                                                                                                                                                       |
| #autovacuum_work_mem = -1              | autovacuum_work_mem             = -1     | Autovacuum 이 동작할 때, autovacuum_work_mem 에 설정된 메모리를 사용. -1은 maintenance_work_mem을 공유함.                                                                                                                                                                                                   |
| #autovacuum_max_workers = 3            | autovacuum_max_workers          = 3      | 동시에 동작 가능한 Autovacuum 의 프로세스 갯수. <br>관리해야 할 테이블이 많다면 값을 늘려야 함. <br>늘리지 않으면 XID Freeze 가 제때 실행이 되지 않을 수 있으며, XID 여유값이 100만 이하로 줄어들 경우, 해당 테이블의 모든 트랜잭션이 거부되며, 수동으로 Vacuuming을 해야 풀린다.<br> 평소 모니터링이 중요. |

## LOCK MANAGEMENT

| default                | recommend                  | note                                                    |
|:-----------------------|:---------------------------|:--------------------------------------------------------|
| #deadlock_timeout = 1s | `deadlock_timeout = 1s`\** | deadlock 발생 시 log에 기록될 timeout 시간. 기본값 1초. |

---

# 도움받은 곳

- [남기면 좋잖아](https://dbza.tistory.com/entry/PostgreSQL-설치-및-설정)
- [아리수](https://arisu1000.tistory.com/1047)
- [PostgreSQL 튜닝 - Autovacuum 최적화에 대하여](https://nrise.github.io/posts/postgresql-autovacuum/)
- [PostgreSQL Archive(아카이브) 설정 및 복구](http://blog.naver.com/rladlaks123/221494788844)
