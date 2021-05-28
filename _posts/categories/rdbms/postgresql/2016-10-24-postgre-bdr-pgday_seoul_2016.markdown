---
title:  "Postgre-BDR with Google Cloud Platform (PGDay Seoul 2016)"
excerpt:   "PGDay Seoul 2016에서 발표 자료 입니다."
toc: true
toc_sticky: true
header:
  teaser: /images/rdbms/postgresql/pgday.jpg

categories:
  - PostgreSQL
tags:
  - PostgreSQL
  - BDR
  - Multi-Master
last_modified_at: 2016-10-24T14:42:13+0900
comments: true
---

pgday.seoul 2016 에서 발표한 자료입니다.

아래의 running Postgres-BDR with Google Cloud Platform 1 ~ 4 을 요약한 내용과 최종 테스트 결과가 정리되어 있습니다.

<iframe src="//www.slideshare.net/slideshow/embed_code/key/7Myy1mVLvaD42M" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/sungjaeyun/postgresbdr-with-google-cloud-platform" title="Postgres-BDR with Google Cloud Platform" target="_blank">Postgres-BDR with Google Cloud Platform</a> </strong> from <strong><a href="https://www.slideshare.net/sungjaeyun" target="_blank">SungJae Yun</a></strong> </div>


여러 복제 솔루션 중에서 Bucardo 가 Multi-master 를 지원하는데 Multi-Region 을 구성하여 테스트 해 본 결과, Manage node의 부하가 너무 심하게 걸리고 데이타 복제에 Delay 가 너무 심하게 발생하여 사용할 수 없을 정도였습니다.

Global 서비스 환경에서 Multi-Region 의 동기화를 위한 RDBMS 동기화 솔루션으로는 Postgres-BDR 이 안정적으로 동작하였으며, 네트워크 상태와 시스템 부하에 따라 Delay 가 발생하기도 하였지만, 어느 정도 시간이 지나면 데이타의 손실 없이 전송/수신함을 확인할 수 있었습니다.

RDBMS 중 Multi-Region 동기화를 지원하는 몇 안되는 솔루션으로, 글로벌 서비스를 할 계획이고, PostgreSQL을 사용하고자 한다면 사용을 고려해 보세요.

이상 공작명왕 입니다.
