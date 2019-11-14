---
layout: post
title: "Linux Log Rotate 설정"
subtitle: "몇일만 쌓여도 로그보기 힘들어요!"
author: "Dalgun"
category: "Linux"
tag: [Linux, Log Rotate]
comments: true
---

![thumb](https://www.ostechnix.com/wp-content/uploads/2017/04/Manage-Log-Files-Using-Logrotate-In-Linux-720x340.png)

# Definition of Log Rotate

>리눅스내에서 시스템에 있는 모든 로그파일들을 관리할 수 있으며, 이들 로그파일들을 날짜를 지정하여 주기적으로 압축, 백업, 삭제, 메일로 보내는 등의 작업을 할 수 있도록 하여주는 기능[^1]

# How do I install Log Rotate?
일반적으로 로그로테이트는 Ubuntu 나 CentOS, 리눅스 계열의 OS 설치시 자동으로 설치 되며 default 디렉토리는 /etc/logrotate.d 입니다.

# Log Rotate Structure

- Daemon : /usr/sbin/logrotate
- Daemon config : /etc/logrotate.conf
- Log rotate Process config : /etc/logrotate.d
- Log rotate Working Log : /etc/cron.daily/logrotate

# /etc/logrotate.d 설정하기

`/var/log/test/test.log 에 rotate 를 설정하는 상황`

편집기를 열어 test 란 설정 파일을 생성,

![bash](/assets/img/2019-11-14/1.png)

설정파일에 rotate 할 파일들의 경로와 패턴을 작성

![step1](/assets/img/2019-11-14/2.png)

실제 운영 중인 서버에 적용한 설정들을 보며, 세부설정 확인해보기

![step2](/assets/img/2019-11-14/3.png)

`Weekly` : 로그 회전 주기, 다른 설정으로 daily, monthly, yearly <br>
`rotate 30` : 보관할 Rotate 갯수, 현재 설정으로 30주 에 해당하는 분량을 보관(주기가 daily 였다면 한달분량 이란 의미다)<br>
`dateext` : 로테이트시 날짜를 부여 (ex. test-api-server-8082.log-20191114)<br>
`missingok`: 내용이 비어 있어도 오류 발생시키지 않음<br>
`copytruncate` : 로테이트 후 어플리케이션이 사용할 로그파일 재생성<br>
`notifempty` : Log 내용이 없더라도 로테이트 

나는 이정도 설정만으로도 잘 활용하고 있지만, 좀 더 세부적인 설정을 원하신다면!<br>
[요기](http://blog.naver.com/PostView.nhn?blogId=didim365_&logNo=220405747299) ~~이분이 정리를 더 잘해주셨다!!~~


# 실제 Log Rotate 의 결과는 어떻게 나올까?

Log Rotate 의 경우 Cron Daily 가 보통 새벽 2시 ~ 3시쯤 사이에 한번 돌기 때문에, 정상적으로 세팅이 되었는지 확인하기 어려울 것이다. 하여 제공하는 것이 `dry-run` !!<br>
다음과 같은 명령어를 Shell 에 입력하면, 해당 설정으로 어떻게 작업이 되는지 확인 할 수 있다!
```bash
$> logrotate -d -f /etc/logrotate.d/test
```

# 수행된 Rotate 의 로그 를 확인해보자
`CentOS 7 ` 기준으로 다음과 같이 입력하면, 최근 동작한 로테이트의 정보를 확인 할 수 있다.

```bash
$> cat /var/lib/logrotate.status
```


# 결론





[^1]: 출처:http://suricata.tistory.com/entry/Linux-리눅스-로그관리-logrotate[변화를 두려워 말고 즐겨라 - 시스템 엔지니어 성장기]
                                      