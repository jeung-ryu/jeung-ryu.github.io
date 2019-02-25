---
layout: single
title: "Spring spring.profiles.include"
date: 2019-02-25 20:00:00 +0900
categories:
  - 기술공부
tags:
  - basecamp
comments: true
---

# spring.profiles.include 사용


real서버에서 다음과 같이 profile을 적용하여 사용중이었습니다.

```bash
export JAVA_OPTS="$JAVA_OPTS -Dspring.profiles.active=real,redis"
```

샤딩 확인 중 redis profile을 mysql으로 변경할 일이 생겼고
서버 각각을 변경해야 했습니다.

코드리뷰 중 서버가 2대가 아니라 여럿일 경우 다음과 같은 구성은 좋지 않다라고 하셨고
spring.profiles.include 를 적용하라고 하셨습니다.

## spring.profiles.include
위 옵션은 포함해야하는 profile을 지정하여
profile 구조를 쉽게 설정할 수 있습니다.

예를들어
개발에서 mysql 세션
리얼에서 redis 세션 을 사용한다면


* * *
### develop
```bash
#application-develop.properties

...

spring.profiles.include=mysql

```
```bash
#develop tomcat env

export JAVA_OPTS="$JAVA_OPTS -Dspring.profiles.active=develop"
```





* * *
### real

```bash
#application-real.properties

...

spring.profiles.include=redis

```
```bash
#real tomcat env

export JAVA_OPTS="$JAVA_OPTS -Dspring.profiles.active=real"
```


위와 같이 프로파일 간에 포함관계를 만들어
서버에서 프로파일 적용은 develop이나 real 하나만 쓰는게 가능합니다.


더 자세한 내용은 아래 URL 을 확인하시길 바랍니다.
[https://meetup.toast.com/posts/149](https://meetup.toast.com/posts/149)