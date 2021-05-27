---
layout: post
title:  "[GCP] 아무도 알려주지 않는 GCP 실전 비법"
subtitle:   "IAP(Identity-Aware Proxy)를 이용한 SSH 접속"
tags: gcp google cloud platform ssh
date: 2021-05-23 12:40:13 +0900
background: '/img/csp/gcp/google-cloud-platform.png'
comments: true
---
# 개요
>External IP가 없는 인스턴스가 있다면, 이 서버는 인터넷과 격리되어 관리자가 직접 연결하여 CLI 명령어를 실행할 수가 없다는 것은 모든 이들이 잘 알고 있다. 그런데, GCP에서는 External IP가 없어도 인스턴스에 연결할 수 있는 방법을 제공하고 있다. IAP(Identity-Aware Proxy) 서비스를 이용하면 External IP 없이도 인스턴스에 접속이 가능하다.

- 목차
    - [Identity-Aware Proxy 란 무엇입니까?](#identity-aware-proxy-란-무엇입니까)
    - [구성 방법](#구성-방법)
    - [SSH 버튼 조건](#ssh-버튼-조건)
    - [결론](#결론)

---

# Identity-Aware Proxy 란 무엇입니까?
IAP (Identity-Aware Proxy)는 애플리케이션으로 전송 된 웹 요청을 가로 채고 Google ID 서비스를 사용하여 요청하는 사용자를 인증하며 승인 된 사용자가 요청한 경우에만 요청을 허용하는 Google Cloud Platform 서비스이다. 또한 인증 된 사용자에 대한 정보를 포함하도록 요청 헤더를 수정할 수 있다.
IAP의 활용 방법 2가지 중에서 여기서는 "HTTPS RESOURCES"에 대한 제어가 아닌 "SSH AND TCP RESOURCES"에 대한 제어 방법을 알아본다.


# 구성 방법
1. **IAP(Identity-Aware Proxy) 서비스를 사용을 위해 API를 활성화한다.**
    _GCP 메뉴 > APIs & Services > Library > IAP를 검색_ 하여 Enable 해주거나 _GCP 메뉴 > Security > Identity-Aware Proxy_ 에 처음 접속하면 다음과 같이 활성화 하라고 나온다.

    ![2021-05-25-csp-gcp-iap-1.jpg](/img/csp/gcp/2021-05-25-csp-gcp-iap-1.jpg)


2. **SSH AND TCP RESOURCES 메뉴에서 인스턴스를 확인 할 수 있다.**

    ![2021-05-25-csp-gcp-iap-2.jpg](/img/csp/gcp/2021-05-25-csp-gcp-iap-2.jpg)


3. **"Warning" 이라는 문구가 너무 신경쓰인다. 클릭해서 내용을 확인하자.**

    아래 내용대로 방화벽 규칙을 추가 생성하고, 인스턴스에 적용해주자.


    ![2021-05-25-csp-gcp-iap-3.jpg](/img/csp/gcp/2021-05-25-csp-gcp-iap-3.jpg)
    - 소스필터 : 35.235.240.0/20
    - 프로토콜 : TCP


4. **IAM 에서 사용자/그룹들에 대해 최소 권한의 Role을 구성 한다.**

    소유자 권한을 부여하지 말고, 각 사용자 또는 그룹별로 최소한의 권한을 가질 수 있도록 Role 을 적절히 구성한다.


5. **IAM 또는 IAP 메뉴에서 Web Console에서의 SSH 접속을 허용할 구성원 또는 그룹에 Role을 추가한다.**

    1) IAP 메뉴에서 방화벽 설정이 잘 되었음(OK)을 확인한 후, 인스턴스를 선택하고, "ADD MEMBER"를 클릭하여 "IAP-Secured Tunnel User" Role을 추가한다.

    ![2021-05-25-csp-gcp-iap-4.jpg](/img/csp/gcp/2021-05-25-csp-gcp-iap-4.jpg){: width="800"}

    2) IAM에서 구성원 또는 그룹에 "IAP-Secured Tunnel User" Role을 추가한다.

    ![2021-05-25-csp-gcp-iap-5.jpg](/img/csp/gcp/2021-05-25-csp-gcp-iap-5.jpg)

6. **SSH 버튼 활성화 확인**

    ![2021-05-25-csp-gcp-iap-6.jpg](/img/csp/gcp/2021-05-25-csp-gcp-iap-6.jpg){: width="800"}


# SSH 버튼 조건
_GCP Web Console > Compute Engine > VM Instances_ 에서 SSH 버튼 활성화/비활성화 조건

| No. | External IP | IAP API | IAP 역할 | SSH 버튼 | SSH 접속여부 | 이유                                                                           |
|:---:|:-----------:|:-------:|:--------:|:--------:|:------------:|:-------------------------------------------------------------------------------|
|  1  |      O      | Enable  |    X     |   활성   |   **가능**   | External IP가 존재해서                                                         |
|  2  |      O      |    X    |    X     |   활성   |   **가능**   | External IP가 존재해서                                                         |
|  3  |      X      | Enable  |    X     | `비활성` |   `불가능`   | IAP 보안 터널을 사용할 수 없어서                                               |
|  4  |      X      |    X    |    X     | `비활성` |   `불가능`   | IAP 보안 터널을 사용할 수 없어서                                               |
|  5  |      O      | Enable  |    O     |   활성   |   **가능**   | External IP가 존재해서                                                         |
|  6  |      O      |    X    |    O     |   활성   |   **가능**   | External IP가 존재해서                                                         |
|  7  |      X      | Enable  |    O     |   활성   |   **가능**   | IAP 보안 터널을 사용할 수 있어서                                               |
|  8  |      X      |    X    |    O     |   활성   |   `불가능`   | IAP 보안 터널 사용자 권한은 있어서 버튼은 활성화 되나 API 비활성화로 접속 안됨 |



# 결론

1. Google Cloud Web Console에서 SSH 버튼을 비활성화 하기 위해서는, 인스턴스에서 External IP를 제거하면 된다. (3,4번 테스트 결과)
2. 브라우저에서 External IP가 없는 인스턴스로 SSH 연결을 하기 위해서는 IAP(Identity-Aware Proxy) 서비스를 사용하면 된다.
3. Windows(cmd) 또는 Mac(terminal)에서 접속시에는 "gcloud compute ssh " 명령어를 사용하여 접속하면 된다.
