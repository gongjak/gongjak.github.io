---
title:  "[GCP] SSH를 이용해서 VM에 접속하기"
excerpt: "SSH를 이용해서 VM에 접속해보자"
toc: true
toc_sticky: true
header:
  teaser: /images/csp/gcp/gcp-book1.png

categories:
  - GCP
tags:
  - GCP(Google Cloud Platform)
  - SSH
last_modified_at: 2016-08-01T10:13:13+0900
comments: true
---

> 이 내용은 "빠르게 훓어보는 구글 클라우드 플랫폼(한빛미디어. 2016-08-23)"을 집필하면서 작성했던 원고이다.
> 출판된 책은 PDF 파일로 무료로 [다운로드](https://www.hanbit.co.kr/store/books/look.php?p_code=E5359426070) 받을 수 있다.

작성자: [윤성재](https://gongjak.github.io/), [조대협](http://bcho.tistory.com/)

# 개요

클라우드에서 VM을 생성하면, 이 VM에 접속할 필요가 있는데, 일반적으로 SSH를 이용한다.SSH로 접속하는 방법은 크게 다음과 같이 4가지 방법이 있다.

1. 브라우저 창에서 열기
2. 맞춤 포트의 브라우저 창에서 열기
3. gcloud 명령어로 접속하기
4. 다른 SSH 클라이언트를 사용하여 접속하기

이 중 가장 쉬운 방법은 1번일 것이다. 웹 브라우저에서 그냥 클릭만 하면 바로 접속 된다. 인스턴스 한 대에만 접속한다면 분명 가장 쉽고 편하게 접속할 수 있는 방법이다.
그 다음으로는 gcloud 명령어로 접속하는 방법일 것이다. 이는 맥킨토시나 리눅스를 PC로 사용하는 사람들에게는 무척이나 쉽고 편하게 강력한 기능들을 제공해줄 것이다.하지만 많은 이들이 윈도우를 사용하고 있고, 윈도우에서 서버로 접속하기 위해 SecureCRT 나 PuTTY 같은 ssh client 프로그램을 사용하고 있다.
이 장에서는 윈도우와 맥에서 SSH를 이용하여 생성된 VM에 접속하는 방법을 알아보도록 한다.


# MAC OS에서 SSH로 접속하기

## SSH 키페어 생성

ssh-keygen 명령어를 이용하여 다음과 같이 RSA 키페어를 생성한다.

```bash
$ ssh-keygen -t rsa -C "구글계정명"
```

![2016-08-01-csp-gcp-ssh-1.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-1.jpg)


## 퍼블릭 키 등록하기

키페어가 생성이 되었으면 *.pub으로 끝나는 이름의 퍼블릭키의 내용(텍스트)를 복사한후, 콘솔창에서 _"Compute Engine > Metadata > SSH Keys"_ 부분에 복사한 텍스트를 붙여 넣어서 키를 등록한다.

![2016-08-01-csp-gcp-ssh-2.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-2.jpg)

## SSH 터미널로 접속하기

키 등록이 끝났으면, 이 키를 이용하여 SSH로 접속한다.

```bash
$ ssh -i [Private 키 파일] 계정@호스트명
```
![2016-08-01-csp-gcp-ssh-3.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-3.jpg)

---

# Windows에서 Putty로 SSH 접속하는 방법

윈도우에서 서버로 접속하기 위해 SecureCRT 나 PuTTY 같은 ssh client 프로그램을 사용하고 있다.이에 무료로 사용이 가능한 PuTTY 를 이용한 접속 방법을 알아보자.

### PuTTY 를 다운로드 받아 설치하기

Putty는 [http://www.putty.org/](http://www.putty.org/) 를 통해 다운로드 받을 수 있다.

### SSH 키페어 생성

만약 GCP 프로젝트에서 승인되어진 public key가 없다면, 새로운 key-pair를 생성하고 프로젝트에 적용할 수 있다.

SSH나 SCP를 이용하여 인스턴스에 접속하기 전에 프로젝트에서 사용할 SSH key-pair 를 생성해야만 한다. 만약 gcloud tools 를 이용하여 인스턴스에 접속하고 있다면, 이 key-pair는 이미 생성되어 있고 다음의 위치에서 확인할 수 있다.

Linux OSX

- Public key : $HOME/.ssh/google_compute_engine.pub
- Private key : $HOME/.ssh/google_compute_engine

Windows

- Public key : C:\Users\[USER_NAME]\.ssh\.google_compute_engine.pub
- Private key : C:\Users\[USER_NAME]\.ssh\google_compute_engine

만약에 키 페어가 없다면, 앞의 “MAC OS에서 SSH로 접속하기" 부분에서 SSH 키페어 생성 부분을 참고하여 키페어를 생성하거나 아래 가이드에 따라서 Putty에서 키페를 생성하도록 하자.

gcloud tools를 설치했다면 앞에서 key-pair를 확인했을 것이고, 여기는 넘어가도 된다. gcloud tools는 GCP를 계속 사용하고자 한다면 반드시 필요한 툴이므로 꼭 설치하길 바란다. [https://cloud.google.com/sdk/downloads](https://cloud.google.com/sdk/downloads) 에서 다운로드 받아 설치할 수 있다. 아직 gcloud tools를 설치하지 않았다면 새로운 key-pair를 생성해서 접속할 수 있다.

puttygen.exe 를 다운로드 받아 실행하고 Generate 버튼을 클릭한다.

![2016-08-01-csp-gcp-ssh-4.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-4.jpg)

![2016-08-01-csp-gcp-ssh-5.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-5.jpg)

private key 와 public key 를 파일로 저장하고 아직은 창을 닫지 않는다.

### 퍼블릭 키 등록하기

다음으로 *'GCP 콘솔 > Compute Engine > 메타데이터 > SSH 키'* 페이지로 간다.웹 콘솔에서 '수정' 버튼을 클릭하고, '전체 키 데이터 입력' 이라고 적힌 곳에 앞의 Putty 화면에서 ‘Public key for pasting into Open SSH authorized_keys file:’ 이라고 되어 있는 부분의 텍스트를 아래 그림과 같이 복사하여 붙혀넣고 저장한다.

![2016-08-01-csp-gcp-ssh-6.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-6.jpg)

### Putty로 SSH 접속확인

먼저 접속할 VM의 IP를 확인하자.

구글 클라우드 콘솔에서 Compute Engine 메뉴의 VM 인스턴스 로 가면 외부IP를 쉽게 확인할 수 있다.

![2016-08-01-csp-gcp-ssh-7.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-7.jpg)

다음 이 IP로 접속을 해보자.PuTTY 를 실행한 후 Session 에서 Host Name 에는 puttyuser@[인스턴스 외부IP주소] 를 입력한다.

![2016-08-01-csp-gcp-ssh-8.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-8.jpg)

다음 Connection > SSH > Auth 의 Private key file for authentication 에서 저장해둔 private key 파일을 선택한다.

![2016-08-01-csp-gcp-ssh-9.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-9.jpg)

이제 Open 을 눌러 접속해보자. 처음 접속하기에 보안 경고 화면이 나오면 '예(Y)’를 눌러주자.

![2016-08-01-csp-gcp-ssh-10.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-10.jpg)

Passphrase for key “puttyuser" 가 나오면 key-pair를 생성할 때 입력했던 암호를 넣어준다.

![2016-08-01-csp-gcp-ssh-11.jpg](/images/csp/gcp/2016-08-01-csp-gcp-ssh-11.jpg)

이와 같이 authorized_key 를 *"메타데이터 > SSH"* 키에 등록하면 모든 인스턴스에 접속할 수 있게 되지만, 인스턴스를 생성할 때 SSH 키 항목에 등록하면 해당 인스턴스만 접속이 가능하게 된다.
