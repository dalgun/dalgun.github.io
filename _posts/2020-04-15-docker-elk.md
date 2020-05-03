---
layout: post
title: "Docker ELK install on mac"
subtitle: "ELK, Docker로 쉽게 설치해보자"
author: "Dalgun"
category: "Docker"
tag: [docker, elk]
comments: true
---

![thumb](https://user-images.githubusercontent.com/16992394/65840473-f70ca780-e319-11e9-9245-29ec0a8948d6.png)

# Easy ELK Install on MAC

1. 설치파일 다운로드
![install](/assets/img/2020-04-15/install.png)
[도커 공식 설치페이지](https://hub.docker.com/editions/community/docker-ce-desktop-mac/){:target="_blank"} 로 가기
2. Dran and Drop 으로 설치 완료
![dragndrop](https://docs.docker.com/docker-for-mac/images/docker-app-drag.png)
3. Docker 설치 확인

상태바 확인
![status](/assets/img/2020-04-15/status.png)

마우스 오른쪽 클릭
![status2](/assets/img/2020-04-15/status2.png)

about Docker Desktop
![info](/assets/img/2020-04-15/info.png)

# Docker hub Join & Login

## Sign Up

![join](/assets/img/2020-04-15/join.png)

[도커 가입 바로가기](https://hub.docker.com/?utm_source=docker4mac_2.2.0.0&utm_medium=account_create&utm_campaign=referral){:target="_blank"}

## Login

![login](/assets/img/2020-04-15/login.png)


> 참고 및 출처  
> [도커 공식 문서](https://docs.docker.com/docker-for-mac/install/)

# Prepare Docker ELK
## Docker login
```bash
$> docker login
```



![docker-login](/assets/img/2020-04-15/docker-login.png)

## Git Downloads

```bash
$> git clone https://github.com/dalgun/docker.git
$> cd docker
$> cd docker-elk-master
```

## Directory Structure

![docker-elk](/assets/img/2020-04-15/docker-elk-master.png)

[Github에서 소스 보기](https://github.com/dalgun/docker){:target="_blank"}

## Docker Compose 설정 파일 확인

```yaml
version: '3.2'

services:
  delete-indexes:
    image: playdingnow/delete-outdated-es-indexes:1.3
    environment:
      - eshost=elasticsearch
      - esport=9200
      - esmaxdays=15
  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./elasticsearch/config/elasticsearch.yml
        target: /usr/share/elasticsearch/config/elasticsearch.yml
        read_only: true
      - type: volume
        source: elasticsearch
        target: /usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: changeme
      # Use single node discovery in order to disable production mode and avoid bootstrap checks
      # see https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
      discovery.type: single-node
    networks:
      - elk

  logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./logstash/config/logstash.yml
        target: /usr/share/logstash/config/logstash.yml
        read_only: true
      - type: bind
        source: ./logstash/pipeline
        target: /usr/share/logstash/pipeline
        read_only: true
    ports:
      - "5044:5044"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./kibana/config/kibana.yml
        target: /usr/share/kibana/config/kibana.yml
        read_only: true
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch

  apm-server:
    build:
      context: apm-server/
      args: 
        ELK_VERSION: $ELK_VERSION
    ports:
      - 8200:8200
    volumes:
      - type: bind
        source: ./apm-server/config/apm-server.yml
        target: /usr/share/apm-server/apm-server.yml
        read_only: true
#    environment:
#      - apm-server.host="0.0.0.0:8200"
#       - setup.kibana.host="kibana:5601"
#       - setup.template.enabled=true
#       - setup.dashboard.enabled=true
#      - logging.to_files=false
    depends_on:
      - elasticsearch
      - kibana
      - logstash
    networks:
      - elk
networks:
  elk:
    driver: bridge

volumes:
  elasticsearch:

```

yml 에 각 이미지의 version을 arguments를 통해 가져오게 되어있는데, docker-elk-master 폴더 속 .env 파일에 그 정보들이 담겨 있습니다.

```bash
$> vi .env
```

```yaml
ELK_VERSION=7.5.1
```

docker-compose.yml 에 있는 폴더에서 명령어 실행

구동할때
```bash
$> docker-compose up -d
```

![docker-up](/assets/img/2020-04-15/docker-up.png)

정지할때
```bash
$> docker-compose down
```

## 구동 확인

localhost:5601 

![kibana](/assets/img/2020-04-15/kibana.png)

초기 계정 정보 설정파일 경로 ~/docker-elk-master/kibana/config/kibana.yml

```yaml
---
## Default Kibana configuration from Kibana base image.
## https://github.com/elastic/kibana/blob/master/src/dev/build/tasks/os_packages/docker_generator/templates/kibana_yml.template.js
#
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true

## X-Pack security credentials
#
elasticsearch.username: elastic
elasticsearch.password: changeme
apm_oss.indexPattern: 'apm*'

```

# 정리
ELK Docker 로 쉽게 구동해보았습니다. Docker 와 Container의 등장으로 정말 쉽게 구동이 가능해지네요.    
좀 더 자세히 공부해보고 싶으신 분들은 Docker Compose, Docker File 등의 수정을 통해 원하는 세팅으로 변경하는 작업을 진행해보세요!

`예고!` 각 Application 에서 elastic search로 로그를 전송해서 kibana로 보여주는 포스트를 작성할 예정입니다.
