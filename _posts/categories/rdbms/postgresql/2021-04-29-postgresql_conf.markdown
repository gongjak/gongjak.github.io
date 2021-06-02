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

***`item = value`*** : 꼭 설정해야 할 항목들
***item = value*** : 설정하면 좋은 것들


recommend 값은 다음 시스템 사양을 기준으로 함.
([pgtune](https://pgtune.leopard.in.ua/#/) 사이트를 참조.)

> PostgreSQL Version : 13
> OS Type: linux
> DB Type: web
> Total Memory (RAM): 32 GB
> CPUs num: 16
> Connections num: 1000
> Data Storage: ssd


## Connection And Authentication
| conf 파일 기본값                | recommend                      | note                                  |
|:--------------------------------|:-------------------------------|:--------------------------------------|
| #listen_addresses = 'localhost' | ***`listen_addresses = '*'`*** | 외부로 부터 DB 접속을 모두 허용       |
| port = 5432                     | ***`port = 15432`***           | 원하는 포트로 변경하여 사용을 권고    |
| #max_connections = 100          | ***`max_connections = 1000`*** | DB 서버에 대한 최대 동시 연결 수 설정 |

## Resource Usage (except WAL)
| conf 파일 기본값                      | recommend                                    | note                                                                                                                                                                                                                                                                                                                                                                              |
|:--------------------------------------|:---------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #shared_buffers = 32GB                | ***`shared_buffers = 8GB`***                 | DB 서버가 공유 메모리 버퍼에 사용하는 메모리양 설정 (최소 128 kb). Disk IO 최소화 목적. 메모리가 1GB 이상의 서버라면 **시스템 메모리의 25% 권고**. PostgreSQL은 운영 체제 캐시에 의존 하기 때문에 지나치게 많이 할당하면 오히려 성능이 하락할 수 있음.                                                                                                                            |
| #temp_buffers = 8MB                   | temp_buffers = 10MB                          | min 800kB. 각 DB 세션에서 사용하는 임시 버퍼의 최대. 임시 테이블에 대한 엑세스에만 사용되는 세션 버퍼. 필요에 따라 최대 temp_buffers가 지정한 한계까지 할당.                                                                                                                                                                                                                      |
| #work_mem = 4MB                       | ***`work_mem = 2097kB`***                    | min 64KB. 임시 디스크 파일에 쓰기 전에 내부 정렬 작업 및 해시 테이블에서 사용할 메모리양 설정(sort, bitmap, hash join, merge join 작업). 복잡한 쿼리일수록 여러 세션이 동시에 작업할 수 있기 때문에 사용된 전체 메모리는 work_mem의 몇 배가 될 수 있음. 세션마다 다르게 설정 가능. **최대크기 = total RAM / max_connections / 4. 안전크기 = total RAM / max_connections / 16.** | |
| #maintenance_work_mem = 64MB          | ***`maintenance_work_mem = 2GB`***           | min 1MB. create index 혹은 alter table add 등 유지 보수 작업에서 사용할 최대 메모리 양 설정. work_mem 값보다 크게 설정하여 안전하게 운용. 데이터 제거 및 DB 덤프 복원 성능 향상. **1GB당 50MB 권고** |                                                                                                                                                                            |
| #max_worker_processes = 8             | ***`max_worker_processes = 16`***            | 백그라운드 프로세스의 최대 수                                                                                                                                                                                                                                                                                                                                                     |
| #effective_io_concurrency = 1         | ***`effective_io_concurrency = 200`***       |                                                                                                                                                                                                                                                                                                                                                                                   |
| #max_parallel_workers = 8             | ***`max_parallel_workers = 16`***            | 병렬 작업을 지원할 수 있는 최대 프로세스 수. max_worker_processes 보다 높게 설정해도 해당 설정 프로세스 풀에서 가져온 것이므로 아무런 영향을 미치지 않음. 이 값을 변경할때 max_parallel_maintenance_workers 및 max_parallel_workers_per_gather 값도 조정.                                                                                                                         |
| #max_parallel_workers_per_gather = 2  | ***`max_parallel_workers_per_gather = 4`***  |                                                                                                                                                                                                                                                                                                                                                                                   |
| #max_parallel_maintenance_workers = 2 | ***`max_parallel_maintenance_workers = 4`*** |                                                                                                                                                                                                                                                                                                                                                                                   |

### Write Ahead Log
| conf 파일 기본값                    | recommend                                  | note                                                                                                                                                                                                                                                                  |
|:------------------------------------|:-------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #synchronous_commit = on            | synchronous_commit = off                   | 쿼리가 클라이언트에 success 메시지를 반환하기 전에 WAL 레코드가 디스크에 기록되는 동안 트랜잭션 커밋이 대기할지 여부. 지연 시간이 최대 wal_writer_delay 값의 3배. off 로 해놓더라도 DB 상태는 트랜잭션이 정상 중단된 경우와 같기 때문에 성능이 중요하다면 off를 권장. |
| #wal_buffers = -1                   | ***`wal_buffers = 16MB`***                 | DB의 변경 사항을 임시 저장하는 버퍼. 디스크에 아직 기록되지 않은 WAL 데이터에 사용되는 공유 메모리 양. 값이 클 수록, 클라이언트가 한 번에 커밋하는 사용량이 많은 서버에서 쓰기 성능을 향상 시킴.                                                                      |
| #wal_writer_delay = 200ms           | wal_writer_delay = 200ms                   | WAL 작성기가 WAL을 플러시하는 빈도. 플러시를 한후 wal_writer_delay 값 동안 대기                                                                                                                                                                                       |
| #checkpoint_timeout = 5min          | checkpoint_timeout = 5min                  | 자동 WAL 체크 포인트 간 최대 시간 (초)                                                                                                                                                                                                                                |
| max_wal_size = 1GB                  | ***`max_wal_size = 4GB`***                 |                                                                                                                                                                                                                                                                       |
| min_wal_size = 80MB                 | ***`min_wal_size = 1GB`***                 |                                                                                                                                                                                                                                                                       |
| #checkpoint_completion_target = 0.5 | ***`checkpoint_completion_target = 0.7`*** |                                                                                                                                                                                                                                                                       |


### Replication
| conf 파일 기본값            | recommend                           | note                                                                                                                                                        |
|:----------------------------|:------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #effective_cache_size = 4GB | ***`effective_cache_size = 24GB`*** | 데이터 캐싱에 사용할 수 있는 메모리 양. 인덱스 사용 여부를 결정. shared_buffers 할당 메모리 + 사용 가능한 OS 캐시 메모리. **전체 메모리의 50% ~ 75% 권고.** |

### Query Tuning
| conf 파일 기본값                      | recommend                               | note                                                                                                                                                            |
|:--------------------------------------|:----------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #default_statistics_target = 100      | ***`default_statistics_target = 100`*** | ALTER TABLE SET STATISTICS 를 통해 열 특정 대상 세트가 없는 테이블 열의 기본 통계 대상을 설정. 값이 클수록 analyze를 수행하는 시간은 늘지만 예상 값은 향상 됨 | |
| #constraint_exclusion = partition     | constraint_exclusion = on               | 테이블 제약 조건 사용을 제어하여 쿼리를 최적화함. on : 모든 테이블 제약 조건 검사                                                                               |
| #random_page_cost = 4.0               | ***`random_page_cost = 1.1`***          | SSD 사용시 1.0 고정. 일반 HDD 사용시 2.5 사용.                                                                                                                  |
| #enable_bitmapscan = on               |                                         |                                                                                                                                                                 |
| #enable_hashagg = on                  | enable_hashagg = on                     | 일반적인 쿼리 Auto Plan                                                                                                                                         |
| #enable_hashjoin = on                 | enable_hashjoin = on                    | Hash Joint쿼리 Auto Plan                                                                                                                                        |
| #enable_indexscan = on                | enable_indexscan = on                   | Index Scan Auto Plan                                                                                                                                            |
| #enable_indexonlyscan = on            |                                         |                                                                                                                                                                 |
| #enable_material = on                 |                                         |                                                                                                                                                                 |
| #enable_mergejoin = on                | enable_mergejoin = on                   | Merge Join Auto Plan                                                                                                                                            |
| #enable_nestloop = on                 | enable_nestloop = on                    | Nest Loop Auto Plan                                                                                                                                             |
| #enable_seqscan = on                  | enable_seqscan = on                     | Sequence Scan Auto Plan                                                                                                                                         |
| #enable_sort = on                     | enable_sort = on                        | Sort Auto Plan                                                                                                                                                  |
| #enable_incremental_sort = on         |                                         |                                                                                                                                                                 |
| #enable_tidscan = on                  | enable_tidscan = on                     | TID Scan Auto Plan                                                                                                                                              |
| #enable_partitionwise_join = off      |                                         |                                                                                                                                                                 |
| #enable_partitionwise_aggregate = off |                                         |                                                                                                                                                                 |
| #enable_parallel_hash = on            |                                         |                                                                                                                                                                 |
| #enable_partition_pruning = on        |                                         |                                                                                                                                                                 |
seqscan과 nestloop의 경우 index scan보다 오히러 느려지는 경우가 있어 false로 두어야 할 떄가 있다. `explain analyse 쿼리;` 를 통해 어떤 쪽이 탐색에 더 유리한지에 따라 Default값을 변경한다.

### Reporting And Logging
| conf 파일 기본값                                 | recommend                                               | note                                                                                                   |
|:-------------------------------------------------|:--------------------------------------------------------|:-------------------------------------------------------------------------------------------------------|
| #log_destination = 'stderr'                      | ***`log_destination = 'stderr'`***                      | 서버 메시지를 stderr에 기록. **vim logging_collector = on** .stderr로 전송 된 로그를 파일로 리다이렉션 |
| #log_directory = 'log'                           | ***`log_directory = 'log'`***                           | 로그 파일 디렉토리 생성                                                                                |
| #log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' | ***`log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'`*** | 로그 파일 이름 설정                                                                                    |
| #log_truncate_on_rotation = off                  | ***`log_truncate_on_rotation = on`***                   | 설정한 시간만큼 파일에 추가됨.지나면 새로운 로그 파일 생성                                             |
| #log_rotation_age = 1d                           | ***`log_rotation_age = 1d`***                           | 로그 파일의 로테이션 기간                                                                              |
| #log_rotation_size = 10MB                        | ***`log_rotation_size = 0`***                           | 로그 파일 최대 사이즈. 크기를 기반으로 안두려면 0으로 지정                                             |
| log_line_prefix = '%m [%p] %q%u@%d '             | ***`log_line_prefix = '%m [%p] %q%u@%r/%d '`***         | 로그 라인 시작부분 문자열                                                                              |
| #log_min_duration_statement = -1                 | ***`log_min_duration_statement = 1000`***               | 단위 ms. 1초 이상 수행된 query를 남긴다.                                                               |
| #log_lock_waits = off                            | ***`log_lock_waits = on`***                             | Lock으로 인해 query가 지연되면 log를 남긴다. deadlock_timeout 설정 시간 기준.                          |

### AUTOVACUUM

| conf 파일 기본값                       | recommend                                | note                                                                                                                                                                                                                                                                                            |
|:---------------------------------------|:-----------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| #autovacuum = on                       | autovacuum = on                          | AUTOVACUUM 사용 설정                                                                                                                                                                                                                                                                            |
| #autovacuum_vacuum_threshold = 50      | autovacuum_vacuum_threshold = 100000     | vacuum 이 일어나기 위한 dead tuple 의 최소 갯수. 기본값 50. 테이블 별로 설정하면 dead tuple이 100,000개가 생성될 때마다 Autovacuum이 동작함. 운영중 적용 가능.                                                                                                                                  |
| #autovacuum_vacuum_scale_factor = 0.2  | autovacuum_vacuum_scale_factor = 0       | vacuum 이 일어나기 위한 live tuple 대비 dead tuple 의 최소 비율. 기본 값은 0.2. 이 값이 0 이면, autovacuum_vacuum_threshold 에 지정된 숫자만큼의 dead tuple 에 따라 Autovacuum 이 동작하게 되므로 훨씬 일관성 있는 성능을 확보할 수 있음. 운영중 적용 가능.                                     |
| #vacuum_cost_delay = 0                 | vacuum_cost_delay = 0                    |                                                                                                                                                                                                                                                                                                 |
| #vacuum_cost_page_hit = 1              | vacuum_cost_page_hit = 1                 | page_hit 영역(shared buffer 영역)에 있는 데이터를 vacuuming 할 때 마다 1 의 credit 을 소모.                                                                                                                                                                                                     |
| #vacuum_cost_page_miss = 10            | vacuum_cost_page_miss = 10               | page_miss 영역(디스크 영역)에 있는 데이터를 vacuuming 할 때 마다 10 의 credit 을 소모.                                                                                                                                                                                                          |
| #vacuum_cost_page_dirty = 20           | vacuum_cost_page_dirty = 20              | page_dirty 영역에 있는 데이터를 vacuuming 할 때 마다 20 의 credit 을 소모.                                                                                                                                                                                                                      |
| #vacuum_cost_limit = 200               | vacuum_cost_limit = 200                  | Autovacuum 이 한 번 실행될 때, 해당 프로세스는 200 의 credit 을 가지고, 200 의 credit 이 모두 소진되면 해당 Autovacuum 프로세스는 종료됨.                                                                                                                                                       |
| #autovacuum_vacuum_cost_limit = -1     | autovacuum_vacuum_cost_limit = 1000      | -1 일 경우, 해당 값은 vacuum_cost_limit 을 참조한다. 테이블 별로 설정이 가능. 1000으로 조정하면 기본보다 약 5배 많은 vacumming을 한 번에 처리함. 운영중 적용 가능.                                                                                                                              |
| #autovacuum_analyze_scale_factor = 0.1 | autovacuum_analyze_scale_factor = 100000 | 테이블 별로 autovacuum_vacuum_threshold 와 동일한 값으로 설정 권고. 운영중 적용 가능.                                                                                                                                                                                                           |
| #autovacuum_work_mem = -1              | autovacuum_work_mem = -1                 | Autovacuum 이 동작할 때, autovacuum_work_mem 에 설정된 메모리를 사용. -1은 maintenance_work_mem을 공유함.                                                                                                                                                                                       |
| #autovacuum_max_workers = 3            | autovacuum_max_workers = 3               | 동시에 동작 가능한 Autovacuum 의 프로세스 갯수. 관리해야 할 테이블이 많다면 값을 늘려야 함. 늘리지 않으면 XID Freeze 가 제때 실행이 되지 않을 수 있으며, XID 여유값이 100만 이하로 줄어들 경우, 해당 테이블의 모든 트랜잭션이 거부되며, 수동으로 Vacuuming을 해야 풀린다. 평소 모니터링이 중요. |

### LOCK MANAGEMENT
| conf 파일 기본값       | recommend                     | note                                                    |
|:-----------------------|:------------------------------|:--------------------------------------------------------|
| #deadlock_timeout = 1s | ***`deadlock_timeout = 1s`*** | deadlock 발생 시 log에 기록될 timeout 시간. 기본값 1초. |


### 도움받은 곳

[남기면 좋잖아](https://dbza.tistory.com/entry/PostgreSQL-설치-및-설정)
[아리수](https://arisu1000.tistory.com/1047)
[PostgreSQL 튜닝 - Autovacuum 최적화에 대하여](https://nrise.github.io/posts/postgresql-autovacuum/)
[PostgreSQL Archive(아카이브) 설정 및 복구](http://blog.naver.com/rladlaks123/221494788844)

---


---




Archiving -

**WAL에 대한 Archive 제어**

**archive_command = ''**

**archive_command = '/bin/cp -i %p /mnt/tape/%f > /var/log/wal-log-archiving.log'**

**%p**

**%f**




**ERROR REPORTING AND LOGGING**

**에러 리포팅과 로깅관련 제어**

Where to Log -

**log_destination = 'stderr'**

**redirect_stderr = true**

**log_directory = 'pg_log'**

**log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'**

**log_truncate_on_rotation = false**

**log_rotation_age = 1440**

**log_rotation_size = 10240**

**syslog_facility = 'LOCAL0'**

**syslog_ident = 'postgres'**

When to Log -

**로그 생성에 대한 제어**

에러 로그 레벨 설명 *

**`인용 또는 결과 :**DEBUG[1-5] : 실제적으로는 필요없으나 PostgreSQL 개발자들에게 필요한 정보라고 보면 됩니다.INFO : 유저레벨로 단순히 유저쪽에 알리는정보NOTICE : 유저에게 유용하게 알수 있거나 단순히 이렇게 하는게 좋겠어? 이건 주위해라는 정보WARNING : 유저 경고정도, 이건 어떻게든 처리해라던지 Transaction Start가 않되었는데 Commit을 한다던지하는 경고성ERROR : 현재의 업무수행을 중단할정도의 에러LOG : 관리자에게 필요한 정보류, 체크리스트 활동정보라던지 실질적인 DBA용 메세지FATAL : 현재의 쿼리 세션을 중단하게 하는 에러PANIC : 모든 세션에 대한 중단이 초래되는 원인에 대한 에러[root@good /root]$ *****`

client_min_messages = notice

debug5, debug4, debug3, debug2, debug1,
log, notice, warning, error

로그 기록상의 로그 Level의 설정하는 것으로 디폴트인 Notice를 두면

Notice,Warning,error 로그에 대해서 Client에서 전송합니다. 너무 높게 설정하면,
그 엄청난? 에러는 다박고 우엑하기 때문에 디폴트가 적당선인것 같습니다.

log_min_messages = notice

debug5, debug4, debug3, debug2, debug1,
info, notice, warning, error, log, fatal,
panic

위와 같으면 이는 서버에 로그 기록 레벨을 정하는 것입니다.

log_error_verbosity = default

terse, default, or verbose messages

이것은 로그 기록시의 해당 오류에 대한 정보에 대한것으로 terse로 설정하면 최대한 많은 에러로그에 대한
정보는 받을수 있으면 default와 verbose는 같은 옵션입니다.

서버 Transaction이 말고 terse 레벨로 로그를 남겨 상세히보고 싶으시면 Log Directory를 다른 하드디스크로
두시는 편이 좋습니다.

log_min_error_statement = panic

debug5, debug4, debug3, debug2, debug1,
info, notice, warning, error, panic(off)

사용자의 쿼리에 대한 에러로그로 레벨로 설정값 아래 수위일때 저장됩니다.
만약에 DB Application에서 쿼리 Fail에 대한 정보를 따로 수집않하시면
레벨을 올려 서버에 남겨서 하셔도 됩니다.

log_min_duration_statement = -1

이 옵션을 지정된 시간이상 Client의 질의 쿼리에 대해 작업할때 로그를 남깁니다.
이건 초기 개발이란 런칭한지 얼마 않되거나 Tunning시점에서 품질 기준시간을 설정해
너무 지체되는 쿼리에 대해 Tunning할때 필요합니다 -1은 Disable이면
밀리세컨드로 지정할수 있습니다.

silent_mode = false

slient 즉 조용하게 에러는 redirect로 Log Write하지 않을경우 주위 즉 기본옵션으로는 화면에 에러 출력을 하는데
이걸 그냥 true하면 에러나도 아무런 메세지를 볼수 없다. 주위를 좀 요합니다.(제가 그런적있거든요 쩝)

- What to Log -

로그 데이타 생성의 부분 제어

debug_print_parse = false
debug_print_rewritten = false
debug_print_plan = false

Debug 레벨 설정시에 문법Parsing, ReWrite,Plan에 대한 디버깅정보 출력 여부설정으로 레벨을 Debug[1-5]가 아니면
별 의미가 없습니다.

debug_pretty_print = false

Debug Level에서의 에러 로그시에 좀더 자세히 보기 쉽게 할때,즉 풀어서 보여준다는 걸로 이해하셔용

log_connections = false

접속 로그

log_disconnections = false

접속해제 로그

log_duration = false

???

log_line_prefix = ''

로그 기록시 여러분이 원하는 패턴을 정하면 로그의 각줄에 정한 패턴의 형식의 데이타가 앞에 붙어
좀더 라인단위 로그분석에 용의 합니다.

**`인용 또는 결과 :%u** : 접속유저명**%d** : 디비명**%r** : 리모트접속자의 HostName (Resolve)또는 IP와 Port**%p** : 프로세스 ID**%t** : Unix TimeStamp (일반적으로 사람이 보기 쉬운(흔한) 형태의 시간**%i** : command tag**%c** : session id**%l** : session line number**%s** : session start timestamp**%x** : transaction id**%q** : stop here in non-session processes**%%** : '%' <- 문자열 포맷팅 구분자로 %를 쓰니 쓰고 앂으시면 %%로 쓰셔야 만 합니다.[root@good /root]$ *****`

log_statement = 'none'

SQL쿼리 로그시에 원하는 부분만 남기고 싶을때 none은 아무것도 하지 않으면
mod(DML)(insert,update,delete,trunscate,copy from,prepare,explain등) , ddl (create,aler,drop),all은 ALL ^^

log_hostname = false

로그에서는 기본적으로 접속자IP를 기록하나 이걸 true화 하면 resolve또는 hostserver search(/etc/nsswaitch 설정기준으로)를 해서 해
당
IP에 대한 Host명을 찾는데 저로써는 하지 않으시길 바랍니다.

만약에 배포용 Visual Application에서 Danymic DNS 처엄 IP는 다르지만 Host명을 가지고 있는 경우 남기고 싶으시면 모르지만
일반적으로는 로그 하나 남길때마다 조회를 해야 하니 음... T.~; DNS Cache Server나 빵빵 하시면 해보심도. 암튼 비추

- RUNTIME STATISTICS -

실행시 통계정보 제어

- Statistics Monitoring -

log_parser_stats = false
log_planner_stats = false
log_executor_stats = false
log_statement_stats = false

위 옵션은 각각의 상황에 따라 로그를 남기느냐 입니다.

- Query/Index Statistics Collector -

stats_start_collector = true

통계정보 수집용 Process를 기동시킬건지에 대한 옵션

stats_command_string = false

각 접속된 세션의 Command String즉 쿼리문에 대한 통계정보 수집여부로 , 메인 관리자나 로그수집 유저가 아닌이상
다른이가 볼수없어 보안상 위험은 없다고 합니다.
시스템 카타로그인 pg_stat_activity 를 통해 감시감독이 가능 합니다.

stats_block_level = false

디비 활동에 대한 Block Level의 통계정보수집여부로 시스템 카라로그 테이블 pg_stat와pg_statio를 통해 감시가능

stats_row_level = false

위와 동일하면 row Level에 대해서

stats_reset_on_server_start = true

지금까지 설정한 정보를 서버 리스타트시 사제 할것인지 여부를 지정하는것

- CLIENT CONNECTION DEFAULTS -

유저 접속과 관련된 기본 설정

- Statement Behavior -

search_path = '$user,public'

???

default_tablespace = ''

사용자의 TableSpace 디폴트로 설정이 가능 합니다. 기본적으로는 pg_global즉 시스템의 PGDATA 폴더로 TableSpace로 사용하나
이를 따로 바꿀수도 있습니다.
즉 시스템과 직결과것은 PGDATA 환경변수나 맨위 data_directory로 잡고 , 유저들의 사용 Table Space는 create tablespace등으로
따로 잡지 않고 Config 파일 단위에서 처리 할수도 있습니다.

check_function_bodies = true

create function 내부 문법에 대한 오류체크선택,기본적으로는 true를 해야만합니다. 그래야 나중에 쌩뚱맞은에러에 대처하죵.~네~

default_transaction_isolation = 'read committed'

Transaction 처리 Mode에 대한것으로 트랜잭선 모드에 다른 접근자의 접근시의 허용여부로
옵션은 read uncommitted,read committed,repeatable read or serializable 을 지원합니다.

각가 레벨은 http://www.postgresql.org/docs/8.0/interactive/transaction-iso.html 을 참조 하십시요.

default_transaction_read_only = false

디폴트로 Transaction 수행중일때 해당 Table에 대한 Read Only설정 여부입니다.
set transaction 으로 각업무별로 주는것이 효율적인듯.

statement_timeout = 0

지정된 시간이상의 쿼리에 대해서는 모두 중단 시켜 버립니다. 0은 Disable이고 셋팅은 milliseconds로 하시면 됩니다.

- Locale and Formatting -

지역화와 각종 포캣팅관련 제어

datestyle = 'iso, mdy'

날짜 포맷에 대해 입력이나 출력시의 포맷에 대한 기본설정입니다.
다음은 지원되는 포맷들의 출력 예입니다.

ISO(ISO 8601) : 1997-12-17 07:37:16-08
SQL : 12/17/1997 07:37:16. 00 PST
POSTGRES : Wed Dec 17 07:37:16 1997 PST
German : 17.12. 1997 07:37:16. 00 PST
.
.

뒤의 mdy는 기본 출력 양식으로 Month Day Year로 월일년 출력순으로 변경은 가능 합니다.

timezone = unknown

시스템의 TimeZone기준으로 unknown으로 하면 System의 설정된 Time Zone기준입니다.

australian_timezones = false

australian TimeZone이 좀 일반적인 TimeZone과는 좀 달라 배려용 옵션인데 잘모르겠네용 -.-;
날 오스트레일리아로 보내준다면야~ ^^

extra_float_digits = 0

부동소숫점의 소숫점아래의 길이를 제한하는것으로 -15로 하면 0.000000000000000 까지 가능하다.
0으로 부터 Data Type에 맞추어 지므로 특별히 제한을 주고 싶으실때 정하시면 됩니다.

client_encoding = sql_ascii

Client Encoding Set을 정하는 것으로 기본 SQL Acscii입니다.
따로 shift-jis등 DB Encoding내의 Converting가능한 Encoding으로 하셔도 됩니다.
하지만 이는 디폴트이므로 기본으로 두고 DB Application단에서 PSQL API의 client_encoding으로 변환해 받는것이
좋습니다.

lc_messages = 'ko_KR.eucKR'

출처:

[https://arisu1000.tistory.com/1047](https://arisu1000.tistory.com/1047)

[아리수]

출처:

[https://arisu1000.tistory.com/1047](https://arisu1000.tistory.com/1047)

[아리수]

출처:

[https://arisu1000.tistory.com/1047](https://arisu1000.tistory.com/1047)

[아리수]
