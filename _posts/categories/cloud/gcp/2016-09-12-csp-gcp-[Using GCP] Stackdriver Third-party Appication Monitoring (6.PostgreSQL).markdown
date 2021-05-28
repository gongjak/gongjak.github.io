---
title:  "[Using GCP] Stackdriver Third-party Appication Monitoring (6.PostgreSQL)"
excerpt: "Stackdriver Third-party Appication Monitoring (6.PostgreSQL)"
toc: true
toc_sticky: true
header:
  teaser: /images/csp/gcp/gcp-book1.png

categories:
  - GCP
tags:
  - GCP(Google Cloud Platform)
  - Monitoring
last_modified_at: 2016-09-12T15:18:13+0900
comments: true
---
> 이 내용은 "빠르게 훓어보는 구글 클라우드 플랫폼(한빛미디어. 2016-08-23)"을 집필하면서 작성했던 원고이다.
> 출판된 책은 PDF 파일로 무료로 [다운로드](https://www.hanbit.co.kr/store/books/look.php?p_code=E5359426070) 받을 수 있다.

작성자: [윤성재](https://gongjak.github.io/)

# 개요

Stackdriver의 PostgreSQL Plugin에 대한 포스팅이다.


# PostgreSQL Plugin

## PostgreSQL 설치와 시작

< Debian 8 >

```bash
sudo apt-get install postgresql
```

< CentOS 7 >

```bash
sudo yum install postgresql-server postgresql-contrib
sudo postgresql-setup initdb
sudo systemctl start postgresql
```

## 사전 준비

모니터링을 위해 DB 사용자를 추가해줘야 한다. ID와 Password는 반드시 수정해서 입력하길 바란다.

```bash
sudo su - postgres
psql
postgres=# CREATE ROLE stackuser LOGIN PASSWORD ‘stack1234’ SUPERUSER;
postgres=# \q
```

pg_hba.conf 파일에 앞에 생성한 stackdriver 유저로 접근할 수 있도록 접근 권한을 줘야 한다. 다음 내용을 추가해준다.

< Debian 8 >

/etc/postgresql/9.4/main/pg_hba.conf

```bash
host all stackuser 127.0.0.1/32 md5
```

< CentOS 7 >

/var/lib/pgsql/data/pg_hba.conf

```bash
host all stackuser ::1/128 md5
host all stackuser 127.0.0.1/32 md5
```

postgresql 서버를 재기동 한 후, 잘 접속 되는지 확인해본다.

```bash
psql -U stackuser -W -h localhost -d postgres
```

## Plugin 설치

```bash
cd /opt/stackdriver/collectd/etc/collectd.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/collectd.d/postgresql.conf
```

postgresql.conf 파일을 열어서 다음 내용으로 수정해준다.

```bash
DATABASE_NAME ==> postgres
#Host "localhost" ==> Host "localhost"
STATS_USER ==> stackuser
STATS_PASS ==> stack1234
```

## Stackdriver Agent 재시작

< Debian 8 >

```bash
sudo service stackdriver-agent restart
```

< CentOS 7 >

```bash
sudo systemctl restart stackdriver-agent
```

## Monitoring 확인

> Monitoring -> Resources -> Instances

각 서버를 클릭해보자. stackdriver agent가 plugin이 정상으로 동작하여 정보를 제대로 가져온다면 Monitoring의 오른쪽 그래프 윗부분에서 다음과 같은 화면을 볼 수 있다.

![2016-09-12-csp-gcp-stackdriver-6-1.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-6-1.jpg)



## 현재 모니터링 가능한 항목

- Connections (count): Number of connections to PostgreSQL.
- Disk Usage (byte): Number of bytes currently being used on disk.
- Commits (count/s): Number of commits per second.
- Rollbacks (count/s): Number of rollbacks per second.
- Heap Blocks Read Rate (count/s): Number of blocks read from the heap.
- Heap Cache Hit Rate (count/s): Number of blocks read directly out of the cache.
- Index Blocks Read Rate (count/s): Number of blocks read from the index.
- Index Cache Hit Rate (count/s): Number of index blocks read directly out of the cache.
- Toast Blocks Read Rate (count/s): Number of reads from the toast blocks.
- Toast Cache Hit Rate (count/s): Number of toast blocks read directly out of the cache.
- Toast Index Blocks Read Rate (count/s): Number of blocks read from the toast index.
- Toast Index Cache Hit Rate (count/s): Number of toast index blocks read directly out of the cache.
- Operations [delete, insert, update, heap only update] (count/s): Number of rows [deleted, inserted, updated, heap only updated] in the db.
- Dead Tuples (count): Number of tuples that are dead in the db.
- Live Tuples (count): Number of tuples that are live in the db.
