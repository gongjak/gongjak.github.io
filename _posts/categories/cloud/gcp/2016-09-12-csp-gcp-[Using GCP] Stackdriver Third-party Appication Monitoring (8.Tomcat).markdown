---
title:  "[Using GCP] Stackdriver Third-party Appication Monitoring (8.Tomcat)"
excerpt: "Stackdriver Third-party Appication Monitoring (8.Tomcat)"
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

Stackdriver의 Tomcat Plugin에 대한 포스팅이다.


# Tomcat Monitoring

## Tomcat 설치와 시작

< Debian 8 >

```bash
sudo apt-get install tomcat7
```

< CentOS 7 >

```bash
sudo yum install tomcat
sudo systemctl start tomcat
```

## 사전 준비

JMX 모니터링을 사용하기 위해 catalina-jmx-remote.jar 파일을 설치해줘야 하는데, Debian 8 은 tomcat7을 설치하면 함께 설치가 되나 CentOS 7 은 직접 설치해줘야 한다.

< Debian 8 >

다음 내용으로 /usr/share/tomcat7/bin/setenv.sh 파일을 만든다.

```bash
sudo vi /usr/share/tomcat7/bin/setenv.sh
#!/bin/sh
JMX_OPTS=" -Dcom.sun.management.jmxremote \
-Dcom.sun.management.jmxremote.ssl=false \
-Dcom.sun.management.jmxremote.authenticate=false \
-Djava.rmi.server.hostname=localhost \
-Dcom.sun.management.jmxremote.ssl=false "
CATALINA_OPTS=" ${JMX_OPTS} ${CATALINA_OPTS}"
```

/etc/tomcat7/server.xml 파일을 열고 Server 항목에 다음 Listener를 추가한다.

```bash
<Listener className="org.apache.catalina.mbeans.JmxRemoteLifecycleListener"
rmiRegistryPortPlatform="9012" rmiServerPortPlatform="9013"/>
```

Tomcat을 재기동하고, 9012 포트가 열렸는지 확인한다.

```bash
sudo service tomcat7 restart
netstat -na | grep 9012
```

< CentOS 7 >

JMX 모니터링을 위한 jar 파일이 기본 패키지에 포함되어 있지 않으니 톰캣 다운로드 사이트 (http://tomcat.apache.org/download-70.cgi) 에서 최신버전을 다운로드 받아야 한다.

JMX_Remote.jar Download :

```bash
cd /usr/share/tomcat/lib && sudo wget http://apache.mirror.cdnetworks.com/tomcat/tomcat-7/v7.0.70/bin/extras/catalina-jmx-remote.jar
```

tomcat 실행 파일을 열어 if 문 앞(15번행)에 다음 내용을 추가한다.

/usr/libexec/tomcat/server

```bash
JMX_OPTS=" -Dcom.sun.management.jmxremote \
-Dcom.sun.management.jmxremote.ssl=false \
-Dcom.sun.management.jmxremote.authenticate=false \
-Djava.rmi.server.hostname=localhost \
-Dcom.sun.management.jmxremote.ssl=false "
OPTIONS=" ${JMX_OPTS} ${OPTIONS}"
```

/etc/tomcat/server.xml 파일을 열고 Server 항목에 다음 Listener를 추가한다.

```bash
<Listener className="org.apache.catalina.mbeans.JmxRemoteLifecycleListener"
rmiRegistryPortPlatform="9012" rmiServerPortPlatform="9013"/>
```

Tomcat을 재기동하고, 9012 포트가 열렸는지 확인한다.

```bash
sudo systemctl restart tomcat
netstat -na | grep 9012
```

## Plugin 설치

```bash
cd /opt/stackdriver/collectd/etc/collectd.d/ && sudo curl -O https://raw.githubusercontent.com/Stackdriver/stackdriver-agent-service-configs/master/etc/collectd.d/tomcat-7.conf
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

Tomcat 은 JVM 기반으로 동작하기에 친절한 Stackdriver는 JVM 모니터링까지 함께 할 수 있게 자료를 수집해서 보여준다.

![2016-09-12-csp-gcp-stackdriver-8-1.jpg](/images/csp/gcp/2016-09-12-csp-gcp-stackdriver-8-1.jpg)


## 현재 모니터링 가능한 항목

Tomcat

- Threads: Total and busy threads in the Tomcat process.
- Requests: The number of completed and error requests that took place.
- Sessions: How many sessions are active in Tomcat.

JVM

- Active JVM Threads
- JVM Heap memory usage
- JVM Non-Heap memory usage
- JVM Open File Descriptors
- JVM Garbage Collection Count

---
# Plugin 설정을 마치며…

Stackdriver 라는 도구가 GCP 에 추가되어 들어오고나서 각 인스턴스들에 대해 좀 더 세밀한 모니터링이 가능하게 되었다.

서비스 구성에서 필수적으로 사용해야할 각 종 Application 들의 모니터링까지 쉽게 설정할 수 있게 되어 시스템 운영자들의 필수 과제인 "모니터링과 알람" 에 대한 1차 문제를 해결할 수 있게 되었는데, 이 중 모니터링에 대한 부분만을 해결 하였다.그런데 아직도 남은 숙제가 있어 보인다.

바로 각 서비스에서 개발자들이 남기는 User-Defined Log 들에 대한 모니터링이다.

다음에는 이 로그들에 대한 모니터링 방법과 이렇게 모인 Metric을 이용해서 어떻게 알람을 설정할 수 있는지 알아보도록 하겠다.
