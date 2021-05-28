---
title:  "[Using GCP] Stackdriver Third-party Appication Monitoring (3.MongoDB)"
excerpt: "Stackdriver Third-party Appication Monitoring (3.MongoDB)"
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
> 이 내용은 "빠르게 훓어보는 구글 클라우드 플랫폼(한빛미디어. 2016-08-23)"을 집필한 뒤, 부족한 부분에 대해 추가 작성한 내용이다.
> 출판된 책은 PDF 파일로 무료로 [다운로드](https://www.hanbit.co.kr/store/books/look.php?p_code=E5359426070) 받을 수 있다.

작성자: [윤성재](https://gongjak.github.io/)

# 개요
Stackdriver의 MongoDB Plugin에 대한 포스팅이다.


# MongoDB Plugin

## MongoDB 설치와 시작

< Debian 8 >

```bash
sudo apt-get install mongodb
```

< CentOS 7 >

```bash
sudo yum install mongodb-server mongodb
sudo systemctl start mongod
```

## 사전 준비

모니터링을 위해 clusterMonitor Role를 가진 유저를 admin database에 생성해준다. db.auth 명령어에서 "1"이 나오면 정상으로 인증에 성공한 것이다.

< Debian 8 >

```bash
mongo
> use admin
> show users;
> db.addUser("stackuser","stack1234","clusterMonitor");
> db.auth("stackuser","stack1234");
1
> exit
```
< CentOS 7 >

```bash
mongo
> use admin
> show users;
> db.createUser({ user: "stackuser", pwd: "stack1234", roles: ["clusterMonitor"] });
> db.auth("stackuser","stack1234");
1
> exit
```

## Plugin 설치

```bash
cd /opt/stackdriver/collectd/etc/collectd.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/collectd.d/mongodb.conf
```
mongodb.conf 파일을 열어 STATS_USER, STATS_PASS 의 주석처리("#")을 지워주고, 앞에서 생성한 유저의 ID와 Password로 바꿔준다.


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

**Monitoring -> Resources -> Instances**

각 서버를 클릭해보자. stackdriver agent가 plugin이 정상으로 동작하여 정보를 제대로 가져온다면 Monitoring의 오른쪽 그래프 윗부분에서 다음과 같은 화면을 볼 수 있다.

![2016-09-12-csp-gcp-stackdriver-3-1.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-3-1.jpg)



## 현재 모니터링 가능한 항목

- Current Connections (count): The number of active connections to MongoDB.
- Global Lock Hold Time (ms): How long the global lock has been held.
- Mapped Memory (bytes): The amount of mapped memory used by MongoDB. This is roughly equivalent to the total size of your database, due to the use of memory mapped files.
- Virtual Memory (bytes): The amount of virtual memory used by MongoDB. If the virtual memory size is significantly larger than mapped memory size (e.g. 3 or more times), this may indicate a memory leak.
- Resident Memory (bytes): The amount of resident memory that is used by MongoDB. This is the amount of RAM being physically used by the database.
- Operations [command, delete, getmore, insert, query, update] (count/s): The number of [command, delete, getmore, insert, query, update] operations executed per second.
- Database [Collection, Index, Object, Extents] Count (count): The number of [collections, indices, objects, extents] currently in the database.
- Database Data Size (bytes): The size of the data currently in the database.
- Database Storage Size (bytes): The size of the storage currently allocated to the database.
- Database Index Size (bytes): The size of the index for the database.
