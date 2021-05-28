---
title:  "Running Postgres-BDR with Google Cloud Platform-1"
excerpt:   "PostgreSQL로 Multi-Master를 만들자"
toc: true
toc_sticky: true
header:
  teaser: /images/rdbms/postgresql/bdr.jpg

categories:
  - PostgreSQL
tags:
  - PostgreSQL
  - BDR
  - Multi-Master
last_modified_at: 2016-10-18T14:30:13+0900
comments: true
---

# 개요
>Open Source RDBMS인 PostgreSQL은 기본 구성으로 Master 서버를 1대만 구성할 수 있다.
>그러나 Postgres-BDR을 이용하면 Multi-Master 구성이 가능하며,
>GCP의 네트워크를 이용하여 Multi-Region 구성으로 PostgreSQL의 데이터를 동기화 할 수 있다.

# 2ndQuadrant 란?

아마도 생소한 기업일거라 생각되어 간단히 소개 하겠다.PostgreSQL 을 이용하여 기술 지원 사업을 하는 기업으로는 [EDB(EnterpriseDB)](http://kr.enterprisedb.com/) 가 유명하다.EDB는 EDB Postgre Advanced Server 라는 제품을 주력으로 하며 Oracle 제품과의 호환성을 장점으로 내세우고 있다.EDB가 미국의 PostgreSQL 기술지원 기업이라면, 2ndQuadrant는 영국의 기업이다.PostgreSQL에 대한 여러 책도 썼고, 기술지원, 교육, 개발, migration 과 컨설팅을 해주는 기업이다.

[2ndQuadrant.com Homepage](https://2ndquadrant.com/)

---

# Postgres-BDR 사용에 최적의 환경을 제공해주는 Google Cloud Platform

Cloud 서비스를 제공하는 업체는 한둘이 아니다. AWS, MS Azure, Google Cloud Platform(GCP), IBM 등 여러 업체들이 서비스를 제공하고 있다.

이 중에서 제가 사용해본 AWS 와 GCP 를 비교해본다면, AWS 의 서비스는 Region 중심으로 되어 있다.만약 서울 region 에 DB 를 두고 Global 서비스를 하기 위해 미국, 유럽 등에 front 서버를 둔다면 Network latency 로 인해 서비스는 원활하지 않을 것이다.미국 서비스를 위해 버지니아 region 에, 한국은 서울 region 에 셋팅하고, DB 동기화를 위해 연결시키려면 두 region 간의 VPN 터널을 열어줘야 한다.각 Region 이 서로 다른 네트워크인 AWS 는 각 region 간의 서버들끼리 내부 망으로 연결이 안된다.글로벌 서비스를 하기 위해 10개 region 을 셋팅했다면 이를 VPN 으로 모두 연결해줘야 하는데... ㅠㅠ;생각만해도 너무 복잡하다.(다른 해결 방법이 있다면 알려주세요.)

GCP는 어떨까?AWS 와 마찬가지로 Region 이라는 개념이 있기는 하지만, 모든 Region 이 하나의 네트워크로 연결되어 있다.모든 region 들의 서버가 하나의 네트워크에 존재하기에 그냥 ssh 로 연결하면 된다.

Postgres-BDR 은 48대까지 master 서버를 만들수 있는 것도 장점 중의 하나이고, Region 간의 latency 때문에 Delay 가 발생하기는 하지만, 수 초 내에 데이타 손실없이 제대로 동기화를 완료 해주었다.하나의 Region 안에서는 매우 낮은 latency 를 보장해주기에 Raplicatoin 서버를 만들어도 빠른 데이타 복제가 가능한데, 다른 Region 으로의 데이타 복제는 높은 latency 로 인해 성능이 반토막 나기 일쑤였다.Riak 이 그러했고, 테스트해봤던 Bucardo 의 경우에는 지속적인 데이타 인입에 대해 결국 동기화가 안되는 상황까지 발생했다.

현재 Region 간의 복제를 지원하는 솔루션은 몇 가지 안되는 걸로 알고 있다.처음 BDR을 접했던 1년전에 비하면 지금은 많이 늘었는데, 혹시 더 있다면 알려주세요.

- Couchbase
- Cassandra
- Crate
- Postgres-BDR
- Amazon RDS for MySQL (only Read replica)
- Amazon Aurora (release 2016.6.1)

이 중에서 AWS 를 제외한 다른 Cloud 에서도 사용할 수 있으면서 RDBMS 이고, 클라우드 서비스를 사용하면서 벤더에 Lock-in 되는 것을 걱정하는 사람이라면 Postgres-BDR 외에 다른 대안은 없어 보인다.GCP 에서는 복잡한 네트워크 작업을 할 필요 없이 그냥 인스턴스를 생성하고, Postgres-BDR 을 설치하고, 사용하면 Global 로 Multi-Region 간의 동기화가 되는 RDBMS를 사용할 수 있게 되기에 그야말로 최적의 환경이라 할 수 있다.

---

# Postgres-BDR 소개

- [BDR 홈페이지](https://2ndquadrant.com/en/resources/bdr/)
- [BDR Performance](https://2ndquadrant.com/en/resources/bdr/bdr-performance/)
- [Installation Instructions](https://2ndquadrant.com/en/resources/bdr/bdr-installation-instructions/)
- [안정 버전 Documentation](http://bdr-project.org/docs/stable/)

Bi-Directional Replication (BDR) 은 PostgreSQL 을 이용한 asynchronous multi-master replication system 이며, 특히 지리적으로 분산된 클러스터를 구성 할 수 있도록 설계되어 있다.최대 48노드까지 Master로 사용할 수 있으며, 분산 데이터베이스에 대해 오버 헤드가 적게 걸리고 유지 보수 비용이 낮은 기술이다.

![BDR-Basic-Schema3_display.jpg](/images/rdbms/postgresql/BDR-Basic-Schema3_display.jpg)

[ 출처: [2ndquadrant.com](https://2ndquadrant.com/en/resources/bdr/) ]

## BDR 과 다른 Open Source Replication Solutions 비교

아래 표를 보면 많은 기능을 지원하고 있다. 특히 DDL Replication 을 지원하고 있어서 모든 노드마다 작업을 해주지 않아도 된다는 장점이 있다.다만, DDL 작업은 서비스에 영향을 미칠 수 있으니 항상 작업은 조심히 !

![bdrfeaturematrix_display.jpg](/images/rdbms/postgresql/bdrfeaturematrix_display.jpg)

[ 출처: [2ndquadrant.com](https://2ndquadrant.com/en/resources/bdr/) ]

안정성과 버그가 수정된 1.0 버전이 2016년 8월 12일에 릴리즈 되었다. 주요 특징은 다음과 같다.

- Smoother handling of schema changes (DDL) statements allowing increased operational stability and reduced maintenance.
- Various bug fixes for operational issues demonstrating high level of maturity
- Performance tuning, especially of global sequence handling
- Removal of the now deprecated UDR
- Extensive documentation improvements based upon user feedback

전체 바뀐 내용은 [BDR Release Note](http://bdr-project.org/docs/stable/release-1.0.0.html) 를 참조하세요.

---

# Google Cloud Platform 에 인스턴스 생성

- region 별로 asia 2대, us 2대, europe 2대 씩 인스턴스를 생성한다.Server Name : bdr-asia-1, bdr-asia-2, bdr-us-1, bdr-us-2, bdr-eu-1, bdr-eu-2

```bash
# 아시아 인스턴스 생성
gcloud compute --project "" instances create "bdr-asia-1" --zone "asia-east1-c" \
--machine-type "n1-standard-4" \
--network "default" \
--maintenance-policy "MIGRATE" \
--scopes default="https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly" \
--image "/debian-cloud/debian-8-jessie-v20160803" \
--boot-disk-size "10" --boot-disk-type "pd-standard" \
--boot-disk-device-name "bdr-asia-1"

gcloud compute --project "" instances create "bdr-asia-2" --zone "asia-east1-c" \
--machine-type "n1-standard-4" \
--network "default" \
--maintenance-policy "MIGRATE" \
--scopes default="https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly" \
--image "/debian-cloud/debian-8-jessie-v20160803" \
--boot-disk-size "10" --boot-disk-type "pd-standard" \
--boot-disk-device-name "bdr-asia-2"

# 미국 인스턴스 생성
gcloud compute --project "" instances create "bdr-us-1" --zone "us-central1-c" \
--machine-type "n1-standard-4" \
--network "default" \
--maintenance-policy "MIGRATE" \
--scopes default="https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly" \
--image "/debian-cloud/debian-8-jessie-v20160803" \
--boot-disk-size "10" --boot-disk-type "pd-standard" \
--boot-disk-device-name "bdr-us-1"

gcloud compute --project "" instances create "bdr-us-2" --zone "us-central1-c" \
--machine-type "n1-standard-4" \
--network "default" \
--maintenance-policy "MIGRATE" \
--scopes default="https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly" \
--image "/debian-cloud/debian-8-jessie-v20160803" \
--boot-disk-size "10" --boot-disk-type "pd-standard" \
--boot-disk-device-name "bdr-us-2"

# 유럽 인스턴스 생성
gcloud compute --project "" instances create "bdr-eu-1" --zone "europe-west1-c" \
--machine-type "n1-standard-4" \
--network "default" \
--maintenance-policy "MIGRATE" \
--scopes default="https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly" \
--image "/debian-cloud/debian-8-jessie-v20160803" \
--boot-disk-size "10" --boot-disk-type "pd-standard" \
--boot-disk-device-name "bdr-eu-1"

gcloud compute --project "" instances create "bdr-eu-2" --zone "europe-west1-c" \
--machine-type "n1-standard-4" \
--network "default" \
--maintenance-policy "MIGRATE" \
--scopes default="https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly" \
--image "/debian-cloud/debian-8-jessie-v20160803" \
--boot-disk-size "10" --boot-disk-type "pd-standard" \
--boot-disk-device-name "bdr-eu-2"
```

---

# Postgres-BDR 설치

Postgres-BDR 은 Fedora, CentOS, & RHEL 을 위해 yum 을 통한 RPMs 설치를 지원하고, Debian 과 Ubuntu 를 위해 apt 를 통한 DEBs 설치를 지원한다.공식 설치 방법은 여기를 [클릭](https://2ndquadrant.com/en/resources/bdr/bdr-installation-instructions/).

## 패키지 설치

설치는 매우 쉽다.

```bash
sudo sh -c 'echo "deb [arch=amd64] http://packages.2ndquadrant.com/bdr/apt/ jessie-2ndquadrant main" &gt;&gt; /etc/apt/sources.list.d/2ndquadrant.list'
wget --quiet -O - http://packages.2ndquadrant.com/bdr/apt/AA7A6805.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get -y install postgresql-bdr-9.4-bdr-plugin
```

Python 을 이용한 개발을 한다면 추가 패키지를 설치한다.

```bash
sudo apt-get -y install postgresql-bdr-plpython-9.4 postgresql-bdr-server-dev-9.4
```

- BDR extension 이 반드시 설치되어야 한다. (package의 경우, postgresql-bdr-contrib)
- pg_createcluster 를 이용해서 원하는 위치에 data directory를 생성할 수도 있다.

## 소스 다운로드

- 패키지 설치를 원하지 않는다면 소스로도 설치할 수 있다. 방법은 알아서 ~~~

- 소스코드 [GitHub](https://github.com/2ndQuadrant/bdr)

---
# 초기화 (선택사항)

테스트 하다가 PostgreSQL 에 문제가 있다고 생각되어 초기화를 하고자 한다면 다음 순서로 하면 된다.

```bash
sudo service postgresql stop
sudo rm -rf /var/lib/postgresql/9.4
sudo rm -rf /etc/postgresql/9.4/main
sudo rm -rf /service/db/pgsql/9.4/main
sudo chown -R postgres.postgres /var/lib/postgresql
sudo pg_createcluster -d /var/lib/postgresql/9.4/main 9.4 main
sudo service postgresql start
```

---

# 서비스 계정 추가 (선택사항)

상용 서비스용이라면 당연히 관리자 계정을 추가해줘야하는데, 여기선 테스트만 할 것이니 필요하다면 관리자 계정을 추가한다.

```bash
sudo su -l postgres sh -c "psql -dpostgres -c \"CREATE ROLE dbadmin LOGIN PASSWORD 'dbadmin1234' superuser;\""
```

- Username : dbadmin
- Password : dbadmin1234

---

To be continue...
