---
title:  "[GCP] 아무도 알려주지 않는 GCP 실전 비법"
subtitle:   "운영 서버에서 디스크 구성 방안"
categories: csp
tags: gcp google cloud platform persistent disk 영구디스크
date: 2021-05-18 16:40:13 +0900
background: '/img/csp/gcp/google-cloud-platform.png'
comments: true
---
## 개요
>시스템 운영팀마다 디스크 구성 방안은 많이 다르겠지만, 전통적인 서버의 디스크 구성 방안을 클라우드에서도 그대로 적용하는 곳이 많이 있는거 같다. 전통적인 디스크 구성은 I/O의 분산과 안정성을 고려하여 여러 디스크를 분리하여 구성하였지만, 클라우드에서도 동일하게 구성하는 것이 좋을까?
>
>기존에 이렇게 했으니까, 잘 모르겠으니까 대충 구성하자는 식으로 서버를 구성하지 말자. 서버가 운영을 시작하면 이러한 구성은 변경하기가 매우 어렵다.
>
>클라우드에서 실제 운영 서버의 디스크 구성 방안에 대한 개인적인 의견을 정리해 본다.


- 목차
    - [운영을 위한 디스크 구성 조건](#운영을-위한-디스크-구성-조건)
    - [Persistent Disk란](#persistent-disk란)
    - [Persistent DISK의 특징](#persistent-disk의-특징)
    - [클라우드에서의 디스크 구성에 대한 의견](#클라우드에서의-디스크-구성에-대한-의견)
        - [최소 디스크 구성](#최소-디스크-구성)
        - [일반적인 Application 서버의 디스크 구성](#일반적인-application-서버의-디스크-구성)
        - [Database용 서버의 디스크 구성](#database용-서버의-디스크-구성)


---

오랜 시간 서비스가 운영되다 보면 늘어나는 파일들로 인해 디스크 용량 부족으로 증설을 해야하는 경우가 발생할 수 있다. On-Premise 환경에서는 이러한 작업이 매우 복잡하다. (서비스 담당자의 증설 요청 > 인프라 팀에서 물리적인 디스크 증설 (필요시 서비스 중지 및 서버 재기동) > OS 팀에서  논리적인 증설 작업 등)

하지만, Google Cloud Platform 에서는 Console 화면에서 클릭만으로 디스크 증설이 된다.

분명히 On-Premise 환경에서 보다 많이 편해지기는 했다. 이렇게 편해진 디스크 증설 방법을 잘 이용하기 위해서는 초기 VM instance 구성 시에 잘 만들어 놓아야 할 것이다.

## 운영을 위한 디스크 구성 조건

1. System(OS)의 업데이트, 패치에 영향이 없어야 한다.
2. Application 으로 인해 System Disk 용량이 Full 되면 안된다.
3. 각 영역의 공간이 부족할 경우에 증설이 용이해야 한다.
4. 감시 솔루션에서의 감시 설정이 용이해야 한다.
5. Raid, LVM, fdisk partion 관리 등 복잡한 리눅스 시스템 관리 및 작업은 피하고 싶다.
6. 관리에 용이하게 디스크를 분리하면서도 성능이 떨어지면 안된다.

## Persistent Disk란

Google Cloud Platform에 정의된 내용은 이렇습니다.

"영구 디스크(Persistent Disk)는 데스크톱 또는 서버의 물리적 디스크와 같이 인스턴스에서 액세스할 수 있는 내구성이 있는 네트워크 스토리지 기기입니다. 각 영구 디스크의 데이터는 여러 물리적 디스크에 분산됩니다. Compute Engine은 중복을 보장하고 성능을 최적화하기 위해 물리적 디스크 및 데이터 분산을 관리합니다."

참고) [영구 디스크](https://cloud.google.com/compute/docs/disks#pdspecs)


## Persistent DISK의 특징

Google Cloud Platform에서 GCE(Google Compute Engine, VM  instance)에 직접 연결하여 사용할 수 있는 Storage이다. 이는 다음과 같은 특징을 가지고 있다.

1. 스냅샷이 지역별로 복제되고 모든 리전에서 복원이 가능하다.
2. 자동으로 암호화되어 전송 중 데이터 또는 저장 데이터를 보호한다.
3. Storage는 가상 머신 인스턴스로부터 독립된 위치에 존재한다.
4. 크기 조절이 가능한 볼륨. 최대 64TB까지 가능하므로 대용량의 논리 볼륨을 생성하기 위해 다수의 디스크를 구성할 필요가 없다.
5. 디스크 성능은 크기에 따라 자동으로 늘어나므로 기존 디스크의 크기를 조절하여 성능 및 저장 공간 요구 사항을 충족할 수 있다. IOPS는 Storage 옵션에 따라 GB당 0.75 ~ 30까지이다.
6. VM instance의 vCPU가 늘어날수록 최대 디스크 성능도 늘어난다. 최대 IOPS는 7,500 ~ 2,400,000 이다.

참고1) [Persistent Disk 모든 특징](https://cloud.google.com/persistent-disk#all-features)

참고2) [블록 스토리지 성능](https://cloud.google.com/compute/docs/disks/performance)


## 클라우드에서의 디스크 구성에 대한 의견

1. 성능 향상

    Persistent Disk는 **네트워크 스토리지 기기**이기에 디스크를 여러 개 분리하여 연결하여도 성능 향상에 도움이 되지 않는다. 적정 용량이 확보되지 않은 디스크를 여러개로 분리하여 구성하면 오히려 성능(IOPS, 최대 지속 처리량)이 떨어진다.

    즉, 10GB(pd-balanced IOPS 60)짜리 디스크 4개를 사용하는 것보다 40GB(pd-balanced IOPS 240)짜리 디스크 1개를 마운트하여 4개의 디렉토리로 구분하여 사용하는 것이 성능면에서 도움이 될 것이다.

2. 운영 관리의 편의성

    반대로 충분한 성능을 확보할 수 있는 디스크 크기라면 분리하는 것이 관리에 도움이 될 수 있을 것이다.

3. 추가 또는 증설 편의성

    Persistent Disk 는 언제든지 추가하거나 증설이 가능하므로 필요할 때 추가/증설 하면 된다.

4. 자동화 편의성

    자동화 구성을 한다면 디스크가 추가됨에 따라 신경써야 할 코드가 더 늘어날 것이기에 주의가 필요하다.

5. 백업 및 복구

    snapshot 을 사용할 경우 디스크가 분리되어 있는 편이 도움이 될 수도 있겠지만, 이는 운영 정책에 따라 영향이 달라질 수도 있겠다.

6. 사용 비용

    기업에서 본다면 Persistent Disk가 저렴할 수도 있는데, 한 번 늘어난 Disk는 줄이기가 쉽지 않아 고정 비용이 증가하는 주된 요인이 된다. 서울 리전의 Zonal pd-balanced 기준 100 GiB는 US$ 13.00 (약 15,000원 정도)이니, 1TiB는 약 150,000원 정도된다.

이렇기에 서비스 운영 초기에는 적은 용량으로 생성하여 언제든 추가/증설이 가능한 구조를 갖도록 구성하는 것이 좋겠다.

### 최소 디스크 구성

기본적인 권고안으로 **System과 Application 영역을 분리**하여 구성한다.

이 때, Application 영역으로 추가한 disk인  /dev/sdb는 별도의 Partition 작업은 하지 않고 전체 디스크를 하나로 Format 하고 /svc에 마운트 한다. 각 저장 영역은 /svc 하위에 디렉토리로 나누어 생성하고 관리한다.

```bash
# Device      Mount point          Size      memo
/dev/sda2    /                     30 GB    (system)
/dev/sdb     /svc                  50 GB
             /svc/engn001                   (솔루션 설치 영역)
             /svc/sorc001                   (소스 파일 저장 영역)
             /svc/data001                   (data 저장 영역)
             /svc/logs001                   (logs 저장 영역)
```

- Application을 구성할 Apache HTTP 서버, Tomcat 등은 sorc001 에 바이너리 파일로 다운로드 받아 관리하고, 개발한 소스 파일도 sorc001 에 폴더를 생성하여 관리한다.
- 솔루션은 engn001 에 압축을 풀고 설정한다. 이 때, 솔루션 configuration에서 data 저장 영역과 log 저장 영역을 디렉토리 구성에 맞게 설정한다.
- log의 경우, logrotate 같은 관리 툴로 시간 지난 파일은 GCS(Google Cloud Storage)로 백업을 한다면 적은 디스크 용량으로도 충분히 서비스가 가능하다.
- 서비스를 운영하면서 상황에 맞춰 디스크를 추가하거나 증설하도록 한다.

### 일반적인 Application 서버의 디스크 구성

Application 사용량에 따라 용량이 증가하여 지속적으로 관심을 가지고 관리해야 할 특정 영역(여기서는 log 영역)을 분리한다.

```bash
# Device      Mount point          Size      memo
/dev/sda2    /                     30 GB    (system)
/dev/sdb     /svc                  50 GB
             /svc/engn001                   (솔루션 설치 영역)
             /svc/sorc001                   (소스 파일 저장 영역)
             /svc/data001                   (data 저장 영역)
/dev/sdc     /svc/logs001          50 GB    (logs 저장 영역)
```

- 서비스를 운영하면서 상황에 맞춰 디스크를 추가하거나 증설하도록 한다.

### Database용 서버의 디스크 구성

Database는 지속적으로 데이터 저장 용량이 증가하기 때문에 VM instance에 설치형으로 구성한다면, 가장 사용량이 높은 data 저장 영역을 추가로 분리한다.

```bash
# Device      Mount point          Size      memo
/dev/sda2    /                     30 GB    (system)
/dev/sdb     /svc                  50 GB
             /svc/engn001                   (솔루션 설치 영역)
             /svc/sorc001                   (소스 파일 저장 영역)
/dev/sdc     /svc/data001         100 GB    (data 저장 영역)
/dev/sdd     /svc/logs001          50 GB    (logs 저장 영역)
```

- 서비스를 운영하면서 상황에 맞춰 디스크를 추가하거나 증설하도록 한다.
