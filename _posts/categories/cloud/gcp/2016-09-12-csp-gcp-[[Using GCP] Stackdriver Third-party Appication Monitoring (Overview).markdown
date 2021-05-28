---
title:  "[Using GCP] Stackdriver Third-party Appication Monitoring (Overview)"
excerpt: "Stackdriver Third-party Appication Monitoring (Overview)"
toc: true
toc_sticky: true
header:
  teaser: /images/csp/gcp/gcp-book1.png

categories:
  - GCP
tags:
  - GCP(Google Cloud Platform)
  - Monitoring
last_modified_at: 2016-09-12T13:18:13+0900
comments: true
---
> 이 내용은 "빠르게 훓어보는 구글 클라우드 플랫폼(한빛미디어. 2016-08-23)"을 집필한 뒤, 부족한 부분에 대해 추가 작성한 내용이다.
> 출판된 책은 PDF 파일로 무료로 [다운로드](https://www.hanbit.co.kr/store/books/look.php?p_code=E5359426070) 받을 수 있다.

작성자: [윤성재](https://gongjak.github.io/)

# 개요

Stackdriver에 대한 내용과 설치 방법은 [Stackdriver Monitoring Documentation](https://cloud.google.com/monitoring/docs/)을 참조하기 바란다.여기에서는 Stackdriver Agent가 설치되어 Monitoring 이 되고 있는 인스턴스에 설치된 Third-party Application 에 대한 추가 설정에 대해 설명하도록 하겠다.


# 준비할 것

당연하겠지만 구글 콘솔에 접속할 수 있는 브라우저가 있는 개인 PCGCP(Google Cloud Platform) 을 사용할 수 있는 AccountGCP에 생성된 Project. 만약 없다면 처음 사용자를 위한 준비를 따라하도록 한다.개인 신용카드 (본인 확인을 위해 $1.00이 승인되나 청구되지는 않는다.)

# 모니터링을 위한 GCP 준비

우선 프로젝트를 만들고 빌링계정을 생성하자.

New project 생성 : stacktest빌링 계정 생성 : 신용카드를 등록해준다.

처음 GCP에 가입하면 60일동안 테스트 용으로 사용하라고 $300 크레딧을 무료로 넣어준다.이는 프로젝트를 생성하고 들어가면 Billing 메뉴에서 확인할 수 있다.

![2016-09-12-csp-gcp-stackdriver-overview-1.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-1.jpg)

## 테스트 인스턴스 생성

이제 모니터링을 위한 인스턴스를 생성하자.

> GCP 콘솔 -> Compute Engine -> VM instance -> CREATE INSTANCE

test-stackdriver-001

- Name : test-stackdriver-001
- Zone : asia-east1-a
- Machine type : 1 vCPU , 3.75 GB memory
- Boot disk : Debian GNU/Linux 8 (jessie)
- 나머지는 모두 default

test-stackdriver-002

- Name : test-stackdriver-002
- Zone : asia-east1-a
- Machine type : 1 vCPU , 3.75 GB memory
- Boot disk : CentOS 7
- 나머지는 모두 default

GCP에서 권고하는 Debian Linux 와 많은 사람들이 사용하는 CentOS 7을 기준으로 설명하도록 하겠다.

## Stackdriver Agent 설치

잠시 기다린 후에 인스턴스 생성이 완료 되면 SSH로 접속하여 stackdriver agent를 설치하도록 한다. 설치 방법은 OS에 상관없이 동일하다. (https://cloud.google.com/monitoring/agent/install-agent)

```bash
curl -O "https://repo.stackdriver.com/stack-install.sh"
sudo bash stack-install.sh --write-gcm
```

## Stackdriver Monitoring 초기 설정

이제 Monitoring에 제대로 나오는지 확인해보자.

> GCP 콘솔 -> Monitoring

Login with google 이라고 나오면 Google ID로 로그인하면 된다.이 후는 아래 그림을 보면서 진행한다.

Create a new Stackdriver account 를 선택하고 Continue 클릭.

![2016-09-12-csp-gcp-stackdriver-overview-2.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-2.jpg)

stacktest 를 선택하고 Create Account 클릭.

![2016-09-12-csp-gcp-stackdriver-overview-3.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-3.jpg)


더 추가할 프로젝트는 없으므로 Continue 클릭.

![2016-09-12-csp-gcp-stackdriver-overview-4.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-4.jpg)


AWS 계정은 생략하고 Done 클릭.

![2016-09-12-csp-gcp-stackdriver-overview-5.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-5.jpg)

잠시 기다리면 다음과 같이 완료되었다고 나온다.‘Launch monitoring’을 클릭하여 모니터링을 시작해보자.

![2016-09-12-csp-gcp-stackdriver-overview-6.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-6.jpg)

모니터링 리포트를 날마다 받을지, 주마다 받을지 안받을지를 정하면 된다.여기서는 테스트만 할 것이기에 ‘No reports’를 선택.

![2016-09-12-csp-gcp-stackdriver-overview-7.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-7.jpg)

모니터링 화면이 나오면 인스턴스가 제대로 등록되었는지 확인해본다.

> Stackdriver Monitoring -> Resource -> Instances

![2016-09-12-csp-gcp-stackdriver-overview-8.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-8.jpg)

다음과 같이 생성한 인스턴스 2개가 모두 나오는지 확인해본다.

![2016-09-12-csp-gcp-stackdriver-overview-9.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-9.jpg)

인스턴스 Name을 클릭하여 시스템 기본 정보를 잘 수집해 오는지도 확인해보자.Agent가 수집하여 화면에 그래프로 보여주는 기본 정보는 다음과 같다.

- CPU Usage
- CPU Load
- CPU Steal
- Memory Usage
- Disk Usage
- Disk I/O
- Network Traffic
- Open TCP Connections
- Processes

## Monitoring Group 생성

> Stackdriver Monitoring -> Groups -> Create…

![2016-09-12-csp-gcp-stackdriver-overview-10.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-10.jpg)

화면이 나오면 다음과 같이 생성하자.‘test-stackdriver-’를 포함하는 인스턴스들을 이 그룹에 포함하겠다는 의미이다.즉, "test-stackdriver-003”을 추가로 생성하고, agent를 설치하면 자동으로 이 그룹에 포함되게 된다.

- Group Name : test-group
- Filter criteria match : Any
- Name
- Contains
- test-stackdriver-

![2016-09-12-csp-gcp-stackdriver-overview-11.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-overview-11.jpg)

그룹을 생성하게 되면 보여지는 그래프도 조금은 바뀌게 된다.

- KEY METRICS
- Instance (GCE) - CPU (agent)
- Block Storage Volumes - Total Volume Capacity
- Block Storage Volumes - Total volumes
- RUNNING RESOURCES
- Running Instances

의미는 어렵지 않으니 설명하지 않도록 하겠다. KEY METRICS는 필요하다면 챠트를 더 추가할 수도 있다. 이제 Stackdriver agent에 plugin을 설치하여 모니터링을 추가해보도록 하자.

---

# 모니터링 가능한 Thrid-party Application 목록

추가 가능한 plugin 목록이다. 가장 많이 사용하는 Application 들을 중심으로 실제 설치하고 셋팅하면서 Stackdriver Monitoring에 어떻게 보여지는지 확인해 보도록 하겠다.(https://cloud.google.com/monitoring/agent/plugins/)

- Apache web server
- Cassandra
- CouchDB
- Elasticsearch
- HBase
- JVM Monitoring
- Kafka
- Memcached
- MongoDB
- MySQL
- Nginx
- PostgreSQL
- RabbitMQ
- Redis
- Riak
- Tomcat
- Varnish
- ZooKeeper
