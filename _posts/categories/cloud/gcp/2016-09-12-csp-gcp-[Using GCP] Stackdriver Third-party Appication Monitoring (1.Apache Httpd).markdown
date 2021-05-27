---
layout: post
title:  "[Using GCP] Stackdriver Third-party Appication Monitoring (1.Apache Httpd)"
subtitle:   "Stackdriver Third-party Appication Monitoring (1.Apache Httpd)"
categories: csp
tags: gcp google cloud platform stackdriver apache httpd
date: 2016-09-12 15:18:13 +0900
background: '/img/csp/gcp/gcp-book1.png'
comments: true
---
## 개요
> 이 내용은 "빠르게 훓어보는 구글 클라우드 플랫폼(한빛미디어. 2016-08-23)"을 집필한 뒤, 부족한 부분에 대해 추가 작성한 내용이다.
> 출판된 책은 PDF 파일로 무료로 [다운로드](https://www.hanbit.co.kr/store/books/look.php?p_code=E5359426070) 받을 수 있다.
>
>Stackdriver의 Apache Httpd Plugin에 대한 포스팅이다.


- 목차
    - [Apache Httpd 설치와 시작](#apache-httpd-설치와-시작)
    - [사전 준비](#사전-준비)
    - [plugin 설치](#plugin-설치)
    - [Stackdriver Agent 재시작](#stackdriver-agent-재시작)
    - [Monitoring 확인](#monitoring-확인)
    - [현재 모니터링이 가능한 항목](#현재-모니터링이-가능한-항목)


---

## Apache Httpd Plugin

### Apache Httpd 설치와 시작

< Debian 8 >

```bash
sudo apt-get install apache2
sudo service apache2 start
sudo service apache2 status
```

< CentOS 7 >

```bash
sudo yum install httpd
sudo systemctl start httpd
sudo systemctl status httpd
```

웹서버로 직접 접속하여 확인하고자 한다면 방화벽을 열어줘야만 한다.

> GCP 콘솔 -> Compute Engine -> VM instance -> 인스턴스 클릭 -> EDIT -> Firewalls -> Allow HTTP traffic 클릭 -> Save

이제 각 인스턴스의 “External IP”로 웹서버에 접속할 수 있다.

### 사전 준비

Apache plugin 중에 mod_status 가 설치되어 있는지 확인한다.

```bash
curl http://localhost:80/server-status?auto
```

![2016-09-12-csp-gcp-stackdriver-httpd-1.jpg](/img/csp/gcp/2016-09-12-csp-gcp-stackdriver-httpd-1.jpg)

< Debian 8 >

GCP 에서는 status.conf 가 기본으로 설치되므로 잘 된다.

< CentOS 7 >

server-status 가 없어서 404 Not Found 가 나온다. server-status 를 설치해주자.

```bash
cd /etc/httpd/conf.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/httpd/conf.d/status.conf
sudo sed -i s/127.0.0.1:/localhost:/g /etc/httpd/conf.d/status.conf
sudo sed -i s/127.0.0.1/"127.0.0.1 localhost"/g /etc/httpd/conf.d/status.conf
sudo su -c "echo 'LoadModule status_module modules/mod_status.so' > /etc/httpd/conf.modules.d/00-status.conf"
sudo systemctl restart httpd
```

### plugin 설치

```bash
cd /opt/stackdriver/collectd/etc/collectd.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/collectd.d/apache.conf
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

![2016-09-12-csp-gcp-stackdriver-httpd-2.jpg](/img/csp/gcp/2016-09-12-csp-gcp-stackdriver-httpd-2.jpg)

### 현재 모니터링이 가능한 항목

- Active Connections (count): The number of active connections currently attached to Apache.
- Idle Workers (count): The number of idle workers currently attached to Apache.
- Requests (count/s): The number of requests per second serviced by Apache.
- Scoreboard
- Traffic
