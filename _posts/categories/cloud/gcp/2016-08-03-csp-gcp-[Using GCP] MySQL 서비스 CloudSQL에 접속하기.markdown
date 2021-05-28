---
title:  "[GCP] MySQL 서비스 CloudSQL에 접속하기"
excerpt: "MySQL 서비스 CloudSQL에 접속하기"
toc: true
toc_sticky: true
header:
  teaser: /assets/images/csp/gcp/gcp-book1.png

categories:
  - GCP
tags:
  - GCP(Google Cloud Platform)
  - SSH
last_modified_at: 2016-08-03T10:38:13+0900
comments: true
---

> 이 내용은 "빠르게 훓어보는 구글 클라우드 플랫폼(한빛미디어. 2016-08-23)"을 집필하면서 작성했던 원고이다.
> 출판된 책은 PDF 파일로 무료로 [다운로드](https://www.hanbit.co.kr/store/books/look.php?p_code=E5359426070) 받을 수 있다.

## 개요

구글 클라우드에서는 MySQL의 매니지드 서비스 형태로 CloudSQL 서비스를 제공한다. 이 글에서는 CloudSQL을 서버에서 접근하는 방법과, 일반적인 MySQL 클라이언트로 접근하는 방법에 대해서 설명하고자 한다.

- 목차
    - [1세대 Cloud SQL 연결](#1세대-cloud-sql-연결)
    - [2세대 Cloud SQL 연결](#2세대-cloud-sql-연결)
        - [ Cloud SQL 생성](#cloud-sql-생성)
        - [ MySQL 클라이언트에서 Proxy를 통해서 CloudSQL 2세대 접속하기](#mysql-클라이언트에서-proxy를-통해서-cloudsql-2세대-접속하기)
        - [ Proxy 준비](#proxy-준비)
        - [ CloudSQL API 활성화](#cloudsql-api-활성화)
        - [ Proxy가 사용할 서비스 개정 및 비공개키 생성](#proxy가-사용할-서비스-개정-및-비공개키-생성)
        - [ root 유저 비밀번호 변경](#root-유저-비밀번호-변경)
        - [ Linux VM에 접속하기](#linux-vm에-접속하기)
        - [ Proxy 설치](#proxy-설치)
        - [ Proxy 시작](#proxy-시작)
        - [ Proxy 를 이용한 MySQL Client 에서 접속하기](#proxy-를-이용한-mysql-client-에서-접속하기)
        - [ Cloud SQL 접속을 위한 네트워크 추가하기](#cloud-sql-접속을-위한-네트워크-추가하기)
    - [ MAC에서 접속하기](#mac에서-접속하기)
        - [ MySQL 설치하기](#mysql-설치하기)
    - [ Windows에서 접속하기](#windows에서-접속하기)
        - [ MySQL Workbench 설치](#mysql-workbench-설치)
    - [ 애플리케이션에서 접속하기](#애플리케이션에서-접속하기)
        - [ Apache에 php 설치하기](#apache에-php-설치하기)
        - [ mysql connection example](#mysql-connection-example)

---

작성자: [윤성재](https://gongjak.github.io/), [조대협](http://bcho.tistory.com/)

CloudSQL은 매니지드 MySQL서비스이다. 아마존에 RDS서비스와 같다고 보면 되는데 현재는 1세대를 서비스하고 있고, 곧 2세대가 서비스 예정이다.1세대는 500GB까지의 용량까지 지원하고 있지만 2세대는 10테라까지 지원을 한다.현재 지원되는 MySQL버전은 5.5와 5.6 지원하고, 내부 엔진으로는 InnoDB만을 제공한다.

2 세대에 기대되는 기능으로는 On Prem(호스팅 센터에 있는) MySQL과 복제 구성이 가능하다. On Prem에 있는 서버를 마스터로 할 수 도 있고, 반대로 마스터를 CloudSQL로 사용하고 읽기 노드를 On Prem에 구성하는 등 다양한 하이브리드 구성이 가능하다. (기대되는 부분)

**자동 백업, 확장을 위한 읽기 전용 노드 (Read replica)등 필수적인 기능을 제공하고 있다.**

---

# 1세대 Cloud SQL 연결

CloudSQL은 RDS와는 다르게 private ip (10.x.xx)를 아직 지원하지 않고 public ip만을 지원한다. 그래서 서버에 접근하려면 이 public ip를 통해서 접근하면 된다. 보안을 위한 접근 통제 방법으로 특정 IP 주소에서 들어오는 트래픽만을 받아드리도록 설정이 가능하다.

또는 PaaS서비스인 구글 앱앤진을 사용하는 경우에는 구글 앱앤진의 인스턴스나 그룹 단위로 접근 통제가 가능하다.다음 그림은 콘솔상에서 접근이 가능한 IP 주소를 지정하는 화면이다.

![2016-08-03-csp-gcp-cloudsql-mysql-1.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-1.jpg)

MySQL 클라이언트를 이용하여 접속을 할때 mysqlclient를 이용하여 직접 ip등을 입력하고 접속을해도 되지만 이 경우에는 CloudSQL에서 Ip를 허용해줘야 하기 때문에 개발이나 기타 운영 환경등에서 IP 주소가 바뀌면 그때 마다 설정을 해줘야 하기 때문에 불편할 수 있다.이를 조금 더 편하게 하려면 mysqlclient를 사용하지 않고 구글에서 제공하는 "gcloud" 라는 도구를 이용하면, 별도로 접속 IP를 열지 않더라도 접속이 가능하다.접속 방법은 먼저 gcloud 명령어를 인스톨 한 후에

```bash
$ gcloud config set project [클라우드 프로젝트 이름]
$ gcloud beta sql connect [CloudSQL 인스턴스이름] —user=root
(여기서 --user=root 에서 root 사용자는 MySQL 인스턴스내의 사용자이다)
```

으로 접속하면 MySQL 클라이언트와 동일한 툴로 접속이 된다.

![2016-08-03-csp-gcp-cloudsql-mysql-2.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-2.jpg)

gcloud 툴킷을 이용한 자세한 접속 방법은 [https://cloud.google.com/sql/docs/mysql-client#connect-ssl](https://cloud.google.com/sql/docs/mysql-client#connect-ssl) 를 참고하기 바란다.

> 참고) [Toad, MySQL Workbench 등에서 안전하게 연결하는 방법](https://cloud.google.com/sql/docs/mysql/admin-tools)

Google Cloud Platform (이하 GCP)에는 Fully-Managed 해주는 RDBMS 서비스로 CloudSQL이라는 상품이 있다. 얼마전 2세대 제품을 베타로 시작하면서 현재는 두 가지 제품이 함께 운영되고 있다.

---

# 2세대 Cloud SQL 연결

2세대 ClouodSQL은 1세대의 단점인 Public IP를 통해서만 접속이 가능한 방식을 CloudSQL에 접속할 클라이언트에 Proxy를 설치 하는 방식을 이용하여, Public IP를 이용한 방식에 비해서 안전한 보안 방식을 조금 더 쉽게 제공할 수 있게 되었다.

## Cloud SQL 생성

*클라우드 콘솔 > CloudSQL* 에 접속하면 다음과 같은 화면을 볼 수 있다.

![2016-08-03-csp-gcp-cloudsql-mysql-3.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-3.jpg)

![2016-08-03-csp-gcp-cloudsql-mysql-4.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-4.jpg)

![2016-08-03-csp-gcp-cloudsql-mysql-5.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-5.jpg)

2세대 서비스는 현재 베타 서비스 중으로 일부 베타 제한 사항이 있으며, 다음 [링크](https://cloud.google.com/sql/docs/1st-2nd-gen-differences?hl=ko&_ga=1.157093051.1844765000.1467088662#differences) 에서 확인할 수 있다.

2세대를 선택하면 인스턴스를 만들 수 있는 화면으로 바뀐다. 여기에 다음과 같은 값을 입력해준다.

- 인스턴스 ID : testdb-001
- 지역 : asia-east1
- 영역 : 자동선택
- 머신유형 : db-n1-standard-1
- 저장용량 크기 : 50GB
- 고가용성 : 장애 조치용 복제본 만들기 체크
- 자동 백업 사용 설정 : AM 2:00 - AM 6:00
- 승인된 네트워크 : 집이나 사무실 등 직접 접속을 할 네트워크를 입력해주세요.
- 고급 옵션 > 데이터베이스 버전 : MySQL 5.6
- 고급 옵션 > MySQL 플래그 : 항목 추가를 눌러보면 주요 옵션에 대해 추가할 수 있으니 원하는 옵션을 선택하면 된다.

여기까지 입력하면 다음과 같은 모습이 될 것이다.마지막의 '생성'을 클릭하면 인스턴스가 만들어진다.

![2016-08-03-csp-gcp-cloudsql-mysql-6.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-6.jpg)

![2016-08-03-csp-gcp-cloudsql-mysql-7.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-7.jpg)

![2016-08-03-csp-gcp-cloudsql-mysql-8.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-8.jpg)

![2016-08-03-csp-gcp-cloudsql-mysql-9.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-9.jpg)

![2016-08-03-csp-gcp-cloudsql-mysql-10.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-10.jpg)

---

## MySQL 클라이언트에서 Proxy를 통해서 CloudSQL 2세대 접속하기

### Proxy 준비

클라이언트에서 Proxy를 통해서 CloudSQL 2세대를 접속할때는 클라이언트에 Proxy를 설치해야 한다.인스턴스가 생성되었으면 이제, Proxy를 이용하여 CloudSQL에 접속해보자.

Linux, Mac, Windows를 지원하는데 각각의 설정 방법은 다음과 같다.각 OS 별 설정전에 먼저 공통적으로 접속키 설정 등 사전 작업이 필요한데 다음과 같다.

IP를 통한 접속 방법은 앞의 1세대 CloudSQL 을 이용한 접속 방식과 동일하니 참고하기 바란다.

### CloudSQL API 활성화

우선 클라우드 콘솔에서 Google CloudSQL API 를 사용할 수 있도록 해줘야 한다.*클라우드 콘솔 > API 관리자 > Google Cloud API > CloudSQL API* 를 선택 한다.개요 화면에서 '사용 설정' 을 클릭해주면 잠시 후에 활성화된다. 여기에 '사용 중지' 라고 나온다면 이미 사용 설정이 되어 있는 상태이다.

![2016-08-03-csp-gcp-cloudsql-mysql-11.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-11.jpg)

### Proxy가 사용할 서비스 개정 및 비공개키 생성

When you connect using the proxy, you provide the proxy with a path to a local key file. The proxy uses that key to authenticate with the Google Cloud Platform.

For more information about service accounts, see the [Google Cloud Platform Auth Guide](https://cloud.google.com/docs/authentication#user_accounts_and_service_accounts).

콘솔에서 IAM 및 관리자 로 가보자.

![2016-08-03-csp-gcp-cloudsql-mysql-12.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-12.jpg)

'서비스 계정 만들기'를 클릭하고 다음과 같이 입력한 뒤, '새 비공개 키 제공'을 선택하고 '만들기'를 클릭한다.

![2016-08-03-csp-gcp-cloudsql-mysql-13.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-13.jpg)

아래와 같은 메시지가 보인다면 서비스 계정 생성이 완료된 것이다.Downloads 폴더에 가보면 비공개 키 파일이 json 파일 형태로 저장되어 있는 것을 찾을 수 있다.

![2016-08-03-csp-gcp-cloudsql-mysql-14.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-14.jpg)

이 키 파일은 DB 접속에 사용되는 매우 중요한 파일이므로 적절한 곳에 잘 보관하도록 한다.

### root 유저 비밀번호 변경

MySQL을 생성한 후에, root의 비밀 번호를 변경한다. 이 예제에서는 클라이언트가 root 사용자로 붙도록 하였는데, 만약에 필요하다면 새로운 사용자를 생성해서 그 사용자를 이용하는 것을 권장한다.Google 콘솔의 CloudSQL 에서 루트 비밀번호를 바꿔보자.

*클라우드 콘솔 > CloudSQL > testdb-001 인스턴스 > 엑세스 제어 > 루트 비밀번호 변경* 클릭

다음과 같은 창이 열리면 사용하고자 하는 암호를 입력해주면 된다.

![2016-08-03-csp-gcp-cloudsql-mysql-15.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-15.jpg)

자아 이제 클라이언트에서 CloudSQL 에 Proxy를 통해서 접속하기 위한 준비가 끝났다.

### Linux VM에 접속하기

Linux에서 접속을 하는 방법을 살펴보기 위해서 편의상 구글 클라우드 내에 VM을 생성해놓고,이 VM으로 부터 Proxy를 이용하여 접속을 해보도록 하겠다.이 예제는 Debian Linux를 기준으로 작성되었다.

*클라우드 콘솔 > Compute Engine* 을 선택하여 다음과 같이 인스턴스를 생성해 준다.

- 이름 : testapp-001
- 영역 : asia-east1-a
- 머신 유형 : vCPU 1개
- 부팅 디스크 : Debian GNU/Linux 8 (jessie)
- ID 및 API 액세스
- 방화벽 : HTTP 트래픽 허용

![2016-08-03-csp-gcp-cloudsql-mysql-16.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-16.jpg)

인스턴스 생성이 완료되면 다음과 같이 나타날 것이다.오른쪽 끝의 SSH를 눌러 서버에 접속한다.

![2016-08-03-csp-gcp-cloudsql-mysql-17.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-17.jpg)

### Proxy 설치

로그인에 성공했으면 다음 명령어를 입력하여 proxy를 다운로드 받는다.

```bash
$ wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64
$ mv cloud_sql_proxy.linux.amd64 cloud_sql_proxy
$ chmod +x cloud_sql_proxy
```

다음과 비슷한 실행 결과 화면을 볼 수 있다.

![2016-08-03-csp-gcp-cloudsql-mysql-18.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-18.jpg)

### Proxy 시작

인스턴스의 콘솔로 가서 proxy를 시작해보자.

```bash
$ sudo mkdir /cloudsql; sudo chmod 777 /cloudsql
$ sudo ./cloud_sql_proxy -dir=/cloudsql -instances=[INSTANCE_CONNECTION_NAME] -credential_file=[PATH_TO_KEY_FILE] &;
```

[INSTANCE_CONNECTION_NAME] 은 CloudSQL 의 개요 화면에서 확인할 수 있다. 한글로는 "인스턴스 연결 이름" 이다.

![2016-08-03-csp-gcp-cloudsql-mysql-19.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-19.jpg)

[PATH_TO_KEY_FILE] 은 앞서 사전 작업에서 생성한 비공개키 JSON파일을 서버에 복사한 후 이 파일의 위치를 적어주면 된다.

```bash
$ sudo ./cloud_sql_proxy -dir=/cloudsql -instances=(project name):asia-east1:testdb-001 -credential_file=/home/sjyun/.keys/78a8c4166b03.json &amp;
```

실행 후 "Ready for new connections” 이란 메시지가 보이면 된다.

![2016-08-03-csp-gcp-cloudsql-mysql-20.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-20.jpg)

### Proxy 를 이용한 MySQL Client 에서 접속하기

먼저 다음 명령어를 이용해서 MySQL Client를 설치하자.

```bash
$ sudo apt-get update
$ sudo apt-get install mysql-client-5.6
```

![2016-08-03-csp-gcp-cloudsql-mysql-21.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-21.jpg)

MySQL 클라이언트 설치가 끝났으면 이제 직접 연결을 해보자. 다음 명령어를 이용하면 접속할 수 있다.

```
$ mysql -u root -p -S /cloudsql/[INSTANCE_CONNECTION_NAME]
```

Enter password: 가 나오면 루트 암호를 입력하면 접속 된다.

![2016-08-03-csp-gcp-cloudsql-mysql-22.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-22.jpg)

---

## Cloud SQL 접속을 위한 네트워크 추가하기

GCP에는 기본으로 방화벽이 설정되어 있어 외부로부터의 모든 접근이 차단되어 있기 때문에 현재 접속하고자 하는 IP를 Cloud SQL에 등록해줘야만 접속을 할 수 있다.

이미 생성된 Cloud SQL 인스턴스에서는 "액세스 제어 > 승인" 메뉴에서 네트워크를 추가해줄 수 있다.

![2016-08-03-csp-gcp-cloudsql-mysql-23.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-23.jpg)

"+네트워크 추가"를 클릭해서 현재 내가 사용중인 외부 IP를 등록해주면 된다.IP를 모르겠다면 아래 사이트에서 쉽게 확인할 수 있다.

IP확인 [whatismyipaddress.com](http://whatismyipaddress.com/)

**IP는 CIDR 형식에 맞게 입력해주면 되는데, 잘 모르겠다면 확인된 IP 뒤에 "/32” 를 붙혀서 등록해준다. 이는 본인 IP 하나만 접속을 허용해주겠다는 의미이다.**

---

## MAC에서 접속하기

Apple MAC을 사용하는 유저들은 Cloud SQL 에 접속하기 위해 MySQL Workbench 와 터미널을 이용한 두 가지 방법을 모두 이용할 수 있다.Workbench 를 이용하는 방법은 Windows에서 접속하기에서 설명하도록 하고, 여기에서는 많은 유저들이 선호하는 터미널에서 접속하는 방법을 알아보도록 하겠다. 만약 터미널에서의 접속은 이용하지 않고 Workbench 만을 사용한다면 MySQL을 설치하지 않고도 바로 Workbench에서 접속이 가능하기에 “Windows에서 접속하기" 방법을 보기 바란다.

### MySQL 설치하기

이미 자신의 맥에 MySQL이 설치되어 있다면 바로 접속하면 되는데, 없다면 설치해주도록 하자.mysql.com 사이트에서 다운로드 받아 설치하면 된다.

현재 사용중인 Mac OS X 의 버전이 Maverick(10.9) 이상 이라면 "Mac OS X 10.11 (x86, 64-bit), Compressed TAR Archive" 버전을 다운로드 받으면 된다.만약 Mountain Lion (10.8) 이라면 다운로드 창의 "Looking for previous GA versions?” 를 클릭하여 “Mac OS X 10.8 (x86, 64-bit), DMG Archive” 버전을 다운로드 받아서 설치한다.

[Downlaod Link](http://dev.mysql.com/downloads/mysql/)

![2016-08-03-csp-gcp-cloudsql-mysql-24.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-24.jpg)

자신에게 맞는 파일의 Download 를 클릭하면 다음과 같이 로그인이나 가입을 하라고 나온다.그냥 무시하고 아래의 "No thanks, just start my download.”를 클릭하면 바로 다운로드를 시작한다.

![2016-08-03-csp-gcp-cloudsql-mysql-25.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-25.jpg)

설치 방법은 다른 맥용 App을 설치하는 것과 다르지 않다. Finder를 열어서 더블 클릭하고, pkg 파일을 더블 클릭하면 설치 화면이 나온다.

![2016-08-03-csp-gcp-cloudsql-mysql-26.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-26.jpg)

![2016-08-03-csp-gcp-cloudsql-mysql-27.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-27.jpg)

'계속'을 클릭한다.

![2016-08-03-csp-gcp-cloudsql-mysql-28.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-28.jpg)

소프트웨어 사용권 계약에 대한 내용이다.'계속'을 클릭한다.

![2016-08-03-csp-gcp-cloudsql-mysql-29.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-29.jpg)

동의하지 않으면 설치를 할 수 없으므로 ‘동의'를 클릭한다.

![2016-08-03-csp-gcp-cloudsql-mysql-30.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-30.jpg)

설치 디스크 위치를 기본적으로 'Macintosh HD’에 설치하는데, 다른 곳에 설치하고 싶다면 '설치 위치 변경'을 클릭하여 바꿔준다. 준비가 되었으면 '설치'를 클릭한다.

![2016-08-03-csp-gcp-cloudsql-mysql-31.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-31.jpg)

암호를 넣고 '소프트웨어 설치'를 클릭하면 설치를 시작한다.

![2016-08-03-csp-gcp-cloudsql-mysql-32.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-32.jpg)

잠시 기다리면 다음과 같은 창이 열리면서 root 유저의 암호가 나온다. MySQL DB를 서버로 사용할 거라면 반드시 기억해야만 한다. 여기서는 Client 만 사용할 것이기에 무시해도 된다.

![2016-08-03-csp-gcp-cloudsql-mysql-33.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-33.jpg)

OK를 클릭하고 닫으면 된다.

이제 터미널을 열어서 mysql 을 입력해보자. 어? 에러가 발생한다.

![2016-08-03-csp-gcp-cloudsql-mysql-34.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-34.jpg)

분명 설치했는데, 왜 명령어를 못찾을까?맥용 MySQL이 설치된 기본 위치는 /usr/loca/mysql 인데, 인스톨러가 PATH를 추가해주지 않아서 발생하는 에러이다.에디터로 자신의 $HOME 폴더에 있는 .bash_profile 파일을 열어 다음과 같이 PATH 를 추가해주자.

```bash
export PATH=”$PATH:/usr/local/mysql/bin”
```

이제 터미널을 닫았다가 다시 열거나 다음 명령을 실행해주면 변경된 내용이 적용된다.

```bash
$ . .bash_profile
```

이제 mysql을 입력해보자.

![2016-08-03-csp-gcp-cloudsql-mysql-35.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-35.jpg)

이런 메시지가 나온다면 잘 적용된 것이다.

이제 Cloud SQL에 연결해보자. 접속을 위한 외부 IP는 Cloud SQL 의 처음 화면에서 확인할 수 있다. 다음과 같이 입력하고 root 암호를 넣어주면 접속이 된다.

```
$ mysql -h [Cloud SQL 외부 IP] -u root -p
```

![2016-08-03-csp-gcp-cloudsql-mysql-36.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-36.jpg)

---

## Windows에서 접속하기

이번엔 MS Windows 에서 MySQL Workbench 를 이용하여 접속하는 방법을 알아보자.

MySQL Workbench 를 이용하는 방법은 PC에 MySQL 서버 또는 클라이언트를 설치하지 않고도 서버로의 접속이 가능하므로 Workbench만 다운로드 받아 설치하고 사용하면 된다.

MySQL Workbench [Download Link](http://dev.mysql.com/downloads/workbench/)

Workbench는 윈도우 뿐아니라 Ubuntu, Red Hat, Fedora, Mac OS X 등 다양한 플랫폼을 지원하는데 여기서는 윈도우에 설치할 것이기에 'Microsoft Windows’를 선택하고 다운로드 받는다.

MSI installer 버전을 선택하면 설치를 하고 사용할 수 있고, ZIP버전은 압축을 풀고 바로 사용할 수 있다.

![2016-08-03-csp-gcp-cloudsql-mysql-37.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-37.jpg)

Download 를 클릭하면 역시 로그인하라는 메시지가 나오는데, “No thanks, just start my download.” 를 선택하면 바로 다운로드가 시작된다.

![2016-08-03-csp-gcp-cloudsql-mysql-38.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-38.jpg)

### MySQL Workbench 설치

설치를 시작하면 다음과 같은 화면이 나타날 수 있다.

![2016-08-03-csp-gcp-cloudsql-mysql-39.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-39.jpg)

이는 Workbench 가 필요로 하는 두 가지 프로그램이 설치되어 있지 않을 때 나타나며, OK 를 클릭하면 설치가 진행되지 않고 "Download Prerequisites” 링크와 함께 바로 “Finish” 가 나타난다.

![2016-08-03-csp-gcp-cloudsql-mysql-40.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-40.jpg)

"Download Prerequisites” 을 클릭하면 두 가지 필요한 프로그램을 다운로드 받을 수 있는 페이지가 열린다.

![2016-08-03-csp-gcp-cloudsql-mysql-41.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-41.jpg)

윈도우 10 사용자는 .NET Framework 4 이 이미 설치되어 있으므로 아래의 Visual C++ 2013 만 설치하면 된다.

[Microsoft .NET Framework 4 Client Profile](http://www.microsoft.com/en-us/download/details.aspx?id=17113)

[Microsoft Visual C++ 2013 Redistributable Package (x86 or x64)](http://www.microsoft.com/en-us/download/details.aspx?id=40784)

두 가지 프로그램의 설치가 완료되었으면 다시 Workbench 설치를 진행한다.

![2016-08-03-csp-gcp-cloudsql-mysql-42.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-42.jpg)

이제는 여느 다른 프로그램들과 설치 방법이 다르지 않다. 내용을 보면서 “Next”를 눌러주면 설치가 진행된다. 설치가 끝나고 마지막 화면에서 “Finish”를 클릭하면 Workbench 가 자동으로 실행된다.

MySQL Workbench가 실행된 화면이다.

![2016-08-03-csp-gcp-cloudsql-mysql-43.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-43.jpg)

새로운 연결 설정을 하기 위해 "MySQL Connections" 옆의 "+" 기호를 클릭하고, 새로 열린 창에 접속 정보를 입력하자.

Connection Name : testdb-001Hostname : [Cloud SQL 의 외부 IP]

![2016-08-03-csp-gcp-cloudsql-mysql-44.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-44.jpg)

창 아래쪽의 “Test Connection”을 클릭하면 Password 를 입력하는 창이 새로 열린다.암호를 입력하고 정상적으로 연결이 잘 되면 다음과 같은 연결 성공 메시지 창이 나타닌다.

![2016-08-03-csp-gcp-cloudsql-mysql-45.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-45.jpg)

Test Connection 이 성공했다면 “OK” 를 누르자.초기 화면에 "testdb-001"에 대한 내용이 추가 되었을 것이고, 이를 더블 클릭하면 DB에 접속을 하게된다.

가운데 'Query 1’ 이란 탭에 SQL문을 입력하고 실행하면 바로 아래 창에서 수행 결과를 볼 수 있다.

![2016-08-03-csp-gcp-cloudsql-mysql-46.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-46.jpg)

이제 MySQL Workbench 의 모든 기능을 사용할 수 있다.

---

## 애플리케이션에서 접속하기

MAC,Windows,Linux 에서 직접 MySQL 클라이언트를 이용하여 CloudSQL 에 접속하는 방법을 알아보았다. 이번에는 애플리케이션에서 Proxy를 통하여 CloudSQL에 접속하는 방법을 알아보자.여기서는 Apache + PHP를 이용하여 MySQL에 연결하는 예제를 통해서 설명한다.

### Apache에 php 설치하기

먼저 Apache 2 서버와 PHP5를 설치해보자.

```bash
$ sudo apt-get install apache2 php5
```

![2016-08-03-csp-gcp-cloudsql-mysql-47.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-47.jpg)

구글 콘솔에서 *‘Compute Engine > VM 인스턴스'* 로 가서 외부 IP를 클릭하면 새창이 열리면서 Apache2 Debian Default Page 화면을 볼 수 있을 것이다.

![2016-08-03-csp-gcp-cloudsql-mysql-48.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-48.jpg)

![2016-08-03-csp-gcp-cloudsql-mysql-49.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-49.jpg)


만약 이 화면이 보이지 않는다면 apache2 가 실행되고 있는지 확인하고 시작해 준다.

```bash
$ sudo service apache2 status
$ sudo service apache2 start
```

### mysql connection example

php5 용 mysql 모듈을 설치한다.

```bash
$ sudo apt-get install php5-mysql
$ sudo service apache2 stop
$ sudo service apache2 start
```

아래 예제에서 (project_name) 부분을 현재 프로젝트 이름으로 바꿔주고,‘/var/www/html’ 폴더에 mysql_connect.php라는 이름으로 저장한다.

```bash
$ sudo vi /var/www/html/mysql_connect.php
```

```php
<?php
// we connect to localhost and socket
$link = mysql_connect(':/cloudsql/<프로젝트_이름>:asia-east1:testdb-001', 'root',
'testdb');
if (!$link) {
 die('Could not connect: ' . mysql_error());
}
echo 'Socket Connected successfully';
mysql_close($link);
?>
```

이제 브라우저에서 접속해보자. 아래와 같은 결과를 볼 수 있을 것이다.

![2016-08-03-csp-gcp-cloudsql-mysql-50.jpg](/img/csp/gcp/2016-08-03-csp-gcp-cloudsql-mysql-50.jpg)

지금까지 최근 업데이트 된 Google CloudSQL 2nd Generation 의 생성부터 접속하는 방법까지 알아보았다.

처음 CloudSQL 을 사용하고자 하는 이들에게 도움이 되길 바란다.
