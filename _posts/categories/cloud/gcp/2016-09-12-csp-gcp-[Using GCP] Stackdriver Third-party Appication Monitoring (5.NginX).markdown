---
title:  "[Using GCP] Stackdriver Third-party Appication Monitoring (5.NginX)"
excerpt: "Stackdriver Third-party Appication Monitoring (5.NginX)"
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
Stackdriver의 NginX Plugin에 대한 포스팅이다.


# Nginx Plugin

## Nginx 설치와 시작

< Debian 8 >

```bash
sudo apt-get install nginx
```

< CentOS 7 >

```bash
sudo yum install epel-release
sudo yum install nginx
sudo systemctl start nginx
```

## 사전 준비

```bash
cd /etc/nginx/conf.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/nginx/conf.d/status.conf
```

## Plugin 설치

```bash
cd /opt/stackdriver/collectd/etc/collectd.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/collectd.d/nginx.conf
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

![2016-09-12-csp-gcp-stackdriver-5-1.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-5-1.jpg)


## 현재 모니터링 가능한 항목

- Active Connections (count): The number of active connections currently attached to Nginx.
- Reading Connections (count): The number of reading connections currently attached to Nginx.
- Writing Connections (count): The number of writing connections currently attached to Nginx.
- Waiting Connections (count): The number of waiting connections currently attached to Nginx.
- Requests (count/s): The number of requests per second Nginx is servicing.
