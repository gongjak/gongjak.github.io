---
layout: post
title:  "[Sublime Text] Sublime Text 3 에서 폰트 바꾸기"
subtitle:   "Sublime Text 3 에서 폰트 바꾸기"
categories: devops
tags: tools
comments: true
---

## 개요
> Sublime Text 3 에서 폰트를 바꾸는 방법에 대한 포스트입니다.
> 
> 코딩 글꼴 다운로드 링크 포함


Sublime Text 3 에서 폰트를 바꾸는 방법은 쉽다.

우선 사용을 원하는 다운로드 받아 폰트를 설치한다.

링크는 맥용을 기준으로 했기에 윈도우용은 각 홈페이지에서 받으면 된다.

개인적으로 아래 2가지 폰트가 가장 맘에 든다.

Sublime Text 3을 실행하고, Preferences > Settings-User에 아래의 내용을 Paste해주면 된다.

---

네이버 나눔고딕 코딩글꼴 [Homepage](http://dev.naver.com/projects/nanumfont)

나눔고딕 코딩글꼴 [Download](http://dev.naver.com/projects/nanumfont/download/441?filename=NanumGothicCoding-2.0.zip)

```
&lt;br /&gt;{
"ignored_packages":
[
"Markdown",
"Vintage"
],
"font_face": "NanumGothicCoding",
"font_size": 12,
"line_padding_top" : 2,
"line_padding_bottom" : 2
}
```
---

D2 Coding 글꼴 [Homepage](http://dev.naver.com/projects/d2coding)

D2 Coding 글꼴 [Download](http://dev.naver.com/projects/d2coding/download/11300?filename=D2Coding-Ver1.0-TTC-20150911.zip)

```
&lt;br /&gt;{
"ignored_packages":
[
"Markdown",
"Vintage"
],
"font_face": "D2Coding",
"font_size": 12,
}
```
