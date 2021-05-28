---
title:  "[Using GCP] Stackdriver Third-party Appication Monitoring (4.MySQL)"
excerpt: "Stackdriver Third-party Appication Monitoring (4.MySQL)"
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


# MySQL Plugin

## MySQL 설치와 시작

< Debian 8 >

```bash
sudo apt-get install mysql-server
```

< CentOS 7 >

```bash
wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
sudo rpm -ivh mysql-community-release-el7-5.noarch.rpm
sudo yum update
sudo yum install mysql-server
sudo systemctl start mysqld
```

## 사전 준비

모니터링을 위해 DB 사용자를 추가해줘야 한다. ID와 Password는 반드시 수정해서 입력하길 바란다.

< CentOS 7 >

```bash
mysql -u root -p
```

< CentOS 7 >

```bash
mysql -u root

mysql> GRANT SELECT, SHOW DATABASES
ON *.*
TO 'stackdriver'@'localhost'
IDENTIFIED BY 'password';
```

## Plugin 설치

```bash
cd /opt/stackdriver/collectd/etc/collectd.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/collectd.d/mysql.conf
```

다은 받은 파일을 열어서 "STATS_USER”, “STATS_PASS” 를 모니터링을 위해 추가한 DB 사용자 정보로 바꿔준다.

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

![2016-09-12-csp-gcp-stackdriver-4-1.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-4-1.jpg)


## 현재 모니터링 가능한 항목

- Connections (count): The number of active connections to MySQL.
- Select Queries (count): The number of select queries being run.
- Insert Queries (count): The number of insert queries being run.
- Update Queries (count): The number of update queries being run.
- Slave replication lag
