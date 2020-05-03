---
layout: post
title: "Cloud Infrastructure Draw"
subtitle: "클라우드 인프라 구조도 그리"
author: "Dalgun"
category: "Cloud"
tag: [aws, cloud, draw]
comments: true
---

![thumb](/assets/img/2020-05-03/architecture.png)

# 클라우드 구조도를 쉽게 그려보기

상기 이미지와 같이 클라우드 인프라를 구성도를 한눈에 보기 쉽게 이미지화 시키려고 할때 좋은 사이트를 소개합니다!

[클라우드 크래프트 바로가기](https://cloudcraft.co){:target="_blank"}


# 간단 사용법

![usage](/assets/img/2020-05-03/1.png)

구성하는 요소들은 모두 AWS 기준으로, AWS를 사용하고 있다면 더 없이 좋은 툴이다.(전 네이버 클라우드를 사용하고 있지만..그래도 사용함ㅋㅋ)
사용법은 왼쪽에 구성요소들을 선택하고 드래그 앤 드롭으로 원하는 위치에 놓으면 끝이다.

# AWS 사용자들은 유요할 수 있는 몇가지 기능

## 한달 예상 요금

![usage](/assets/img/2020-05-03/2.png)
![usage](/assets/img/2020-05-03/3.png)

## 구성도 그대로 코드로

Terraform 을 사용하고 계신다면! 최고의 옵션! `Terraform 에 대해선 조금 더 공부해세 포스팅 할게요!`

Export 누르고(물론 PNG 파일로도 변환 가능해요!)

![usage](/assets/img/2020-05-03/4.png)

동의해주시면!
![usage](/assets/img/2020-05-03/5.png)


### 짜잔

완벽하게 바로 가져다 쓸 수 있는 정도는 아니지만, 어느정도 기반을 만들어주기엔 충분한 것 같습니다. 

*AWS 로그인하면 현재 구성된 정보 그대로 가져올 수 있는 기능이 있는지 체크 해봐야겠어요!*

```yaml
terraform {
  source = "git::git@github.com:terraform-aws-modules/terraform-aws-elb.git?ref=v2.3.0"
}

include {
  path = find_in_parent_folders()
}

###########################################################
# View all available inputs for this module:
# https://registry.terraform.io/modules/terraform-aws-modules/elb/aws/2.3.0?tab=inputs
###########################################################
inputs = {
  # A health check block
  # type: map(string)
  health_check = {"healthy_threshold": 2, "interval": 30, "target": "HTTP:80/", "timeout": 5, "unhealthy_threshold": 2}

  # A list of listener blocks
  # type: list(map(string))
  listener = [{"instance_port": "80", "instance_protocol": "http", "lb_port": "80", "lb_protocol": "http"}]

  # The name of the ELB
  # type: string
  name = "super-poodle"

  # A list of security group IDs to assign to the ELB
  # type: list(string)
  security_groups = []

  # A list of subnet IDs to attach to the ELB
  # type: list(string)
  subnets = []

  
}
```
 
#설마 무료입니까?

이렇게 좋은 온라인 툴이 제한된 범위안에서는 무료입니다!
가격 정책은 다음과 같으니 참고해주세요!

![usage](/assets/img/2020-05-03/6.png)

저 같은 라이트 유저에겐 프리티어도 감지덕지 입니다 !

또 좋은 정보 있으면 가지고 올게요! 즐거운 연휴 되시길 :)
를