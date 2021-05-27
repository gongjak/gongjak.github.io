---
layout: post
title:  "[Using GCP] Stackdriver Third-party Appication Monitoring (2.Cassandra)"
subtitle:   "Stackdriver Third-party Appication Monitoring (2.Cassandra)"
categories: csp
tags: gcp google cloud platform stackdriver cassandra
date: 2016-09-12 15:18:13 +0900
background: '/img/csp/gcp/gcp-book1.png'
comments: true
---
## 개요
> 이 내용은 "빠르게 훓어보는 구글 클라우드 플랫폼(한빛미디어. 2016-08-23)"을 집필한 뒤, 부족한 부분에 대해 추가 작성한 내용이다.
> 출판된 책은 PDF 파일로 무료로 [다운로드](https://www.hanbit.co.kr/store/books/look.php?p_code=E5359426070) 받을 수 있다.
>
>Stackdriver의 Cassandra Plugin에 대한 포스팅이다.

- 목차
    - [Cassandra 설치와 시작](#cassandra-설치와-시작)
    - [사전 준비](#사전-준비)
    - [plugin 설치](#plugin-설치)
    - [Stackdriver Agent 재시작](#stackdriver-agent-재시작)
    - [Monitoring 확인](#monitoring-확인)
    - [현재 모니터링이 가능한 항목](#현재-모니터링-가능한-항목)


---

## Cassandra Plugin

### Cassandra 설치와 시작

< Debian 8 >

```bash
sudo sh -c "echo 'deb http://www.apache.org/dist/cassandra/debian 37x main' &gt; /etc/apt/sources.list.d/cassandra.list"
sudo sh -c "echo 'deb-src http://www.apache.org/dist/cassandra/debian 37x main' &gt;&gt; /etc/apt/sources.list.d/cassandra.list"
gpg --keyserver pgp.mit.edu --recv-keys 749D6EEC0353B12C
gpg --export --armor 749D6EEC0353B12C | sudo apt-key add -
sudo apt-get update
sudo apt-get install cassandra
sudo service cassandra start
```

참조 : http://wiki.apache.org/cassandra/DebianPackaging

< CentOS 7 >

cassandra 3.x Repository 를 추가해준다.

```bash
/etc/yum.repos.d/datastax.repo:
[datastax-ddc]
name = DataStax Repo for Apache Cassandra
baseurl = http://rpm.datastax.com/datastax-ddc/3.7
enabled = 1
gpgcheck = 0

```

설치하고 시작해준다.

```bash
sudo yum install java-1.8.0-openjdk datastax-ddc
sudo /etc/init.d/cassandra start
```

참조 : http://docs.datastax.com/en/cassandra/3.x/cassandra/install/installRHEL.html

### 사전 준비

노드가 정상 작동중인지 확인한다.

```bash
nodetool status
```

![2016-09-12-csp-gcp-stackdriver-cassandra-1.jpg](/img/csp/gcp/2016-09-12-csp-gcp-stackdriver-cassandra-1.jpg)

Cassandra 는 JMX 를 통해서 모니터링을 하기에, Cassandra에 JMX 설정이 되어야만 하는데, 다행히 localhost 호스트에서의 JMX 설정은 되어 있는 상태이며 기본 포트는 7199이므로 다음 명령어로 확인할 수 있다.

```bash
netstat -na| grep 7199
```
![2016-09-12-csp-gcp-stackdriver-cassandra-2.jpg](/img/csp/gcp/2016-09-12-csp-gcp-stackdriver-cassandra-2.jpg)

### Plugin 설치

```bash
cd /opt/stackdriver/collectd/etc/collectd.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/collectd.d/cassandra-22.conf

```

### Stackdriver Agent 재시작

< Debian 8 >

```bash
sudo service stackdriver-agent restart
```

< CentOS 7 >

```bash
sudo systemctl restart stackdriver-agent
```

### Monitoring 확인

> Monitoring -> Resources -> Instances

각 서버를 클릭해보자. stackdriver agent가 plugin이 정상으로 동작하여 정보를 제대로 가져온다면 Monitoring의 오른쪽 그래프 윗부분에서 다음과 같은 화면을 볼 수 있다.

![2016-09-12-csp-gcp-stackdriver-cassandra-3.jpg](/img/csp/gcp/2016-09-12-csp-gcp-stackdriver-cassandra-3.jpg)

Cassandra는 JVM 기반으로 동작하기에 친절한 Stackdriver는 JVM 모니터링까지 함께 할 수 있게 자료를 수집해서 보여준다.

![2016-09-12-csp-gcp-stackdriver-cassandra-4.jpg](/img/csp/gcp/2016-09-12-csp-gcp-stackdriver-cassandra-4.jpg)

### 현재 모니터링 가능한 항목

JVM

- Active JVM Threads
- JVM Heap memory usage
- JVM Non-Heap memory usage
- JVM Open File Descriptors
- JVM Garbage Collection Count

Cassandra

- Storage Load: The amount of data stored on each Cassandra node.
- Pending Tasks: The number of basic task stages waiting to run.
- Active Tasks: The number of basic task stages currently running.
- Blocked Tasks: The number of basic task stages blocked from running.
- Pending Internal Tasks: The number of internal task stages waiting to run.
- Active Internal Tasks: The number of internal task stages currently running.
- Cassandra Pending Compactions
- Cassandra Pending CommitLog
