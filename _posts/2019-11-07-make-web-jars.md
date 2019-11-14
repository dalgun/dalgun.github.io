---
layout: post
title: "Web Jar 만들어 보기"
subtitle: "사설 저장소용 web jar 쉽게 만들기"
author: "Dalgun"
category: "Spring boot"
tag: [web jar, third party library]
comments: true
---

# 유료 UI FRAMEWORK 을 Private Repository 에서 관리하기

팀내에서 쓰는 유료 Third-party Library 들을 보통 Nexus[^1] 같은 사설 리파지토리에서 관리, 배포, 사용을 권장한다.

> Nexus를 써야 하는 이유! [^2]
- 회사/단체의 화이트 리스트로 인해 외부 리포지토리에 접속하기 어려운 경우 프록시 역활.
- 특히나 비상시 외부 인터넷이 느리거나 리포지토리가 다운되는등 여러 상황에서도 빠르게 받을 수 있다.
- 현재 메이븐에 올라와 있지 않은 자료들은 효율적으로 관리하기 위하여.
- 한번 다운로드 받은 디펜던시는 로컬에 저장되지만 컴퓨터를 포멧하거나 동료가 시작할때 설정을 해야한다.
- 서버에도 동일한 설정들을 해줘야함으로 서버 구조가 복잡할 수록 잔업도 늘어난다.
- 예외 파일로 인한 설정이 줄어들어 전체적인 일관성이 증가한다.

그래서 정적 라이브러리 파일들도 Maven 으로 관리하며, 사설 리파지토리를 통해 배포하기 위해 Web Jar[^3]를 생성해 보기로 하자.
 
## Step 1. Jar 생성을 위한 프로젝트 생성 및 리소스 위치 시키
일단 새로운 빈 프로젝트를 만들고 각각의 요소에 해당하는 파일들을 위치 합니다.(~~저작권 문제로~~ 무료 배포 bootstrap 을 예시로 사용하였습니다)
![image1](/assets/img/post5-1.png)

## Step 2. POM 설정하기
생성한 Project 의 pom.xml 에 라이브러리 정보를 기입하여 준다.<br>
이 정보들은 사용하려는 프로젝트에서 의존성을 걸어주는 정보로 활용된다.
```text
    <packaging>jar</packaging>
    <groupId>그룹아이디</groupId>
    <artifactId>아티팩트 아이디</artifactId>
    <version>버전</version>
    <name>Webjar 이름</name>
    <description>WebJar에 대한 설명</description>
```

그리고 제일 중요한 build 정보
```text
<build>
        <plugins>
            <plugin>
                <artifactId>maven-antrun-plugin</artifactId>
                <version>1.7</version>
                <executions>
                    <execution>
                        <phase>process-resources</phase>
                        <goals>
                            <goal>run</goal>
                        </goals>
                        <configuration>
                            <target>
                                //밑에 설정한 프로젝트의 파일들을 META-INF 안에 위치시켜 web jar 파일로 생
                                <copy todir="./target/classes/META-INF/resources/webjars/play-web-jars/0.0.1-SNAPSHOT">
                                    //루트로 부터 바로 하위 resources 폴더에 js, css, fonts 폴더를 가져오겠다.
                                    <fileset dir="./resources" includes="js/,css/,fonts/"></fileset>
                                </copy>
                            </target>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

```
*target/classes/META-INF* 이 경로로 반드시 설정을 해줘야 html 에서 link 사용시 문제가 없다!

만약, 이 Web Jar 가 종속으로 다른 라이브러리를 포함해야 한다면 의존성 설정도 가능하다.
bootstrap 의 경우 jquery 가 반드시 포함되어야 하므로 jquery 의존성 설정 정보는 다음과 같다.

```text
    <dependencies>
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>jquery</artifactId>
            <version>1.11.1</version>
        </dependency>
    </dependencies>
```

이렇게 하면 빌드 환경 구성은 완료이다. <br>
환경 구성 중 가장 중요한 것은  **BUILD 정보에서 포함해야할 리소스들 과 위치할 경로** 이다.

이렇게 설정하고 빌드하면, 다음과 같이 target 폴더가 위치하게 되고 jar 파일 생성이 된다.
![directory](/assets/img/post5-2.png)

## STEP 3. 적용
생성된 Jar 와 Pom 파일을 Nexus 에 업로드 하고, 적용하고자 하는 프로젝트에 의존성을 설정한다.
<br>그리고 HTML 에 다음과 같이 IMPORT 를 한다.

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8"/>
    <title th:text="'Hello, ' + ${name} + '!'"></title>
    <link href="/webjars/play-web-jars/0.0.1-SNAPSHOT/css/bootstrap.css" rel="stylesheet"/>
</head>
<body>
<h2 class="hello-title" th:text="'Hello, ' + ${name} + '!'"></h2>
<script src="/webjars/jquery/1.11.1/jquery.js"></script>
<script src="/webjars/play-web-jars/0.0.1-SNAPSHOT/js/bootstrap.js"></script>
</body>
</html>
```

우리가 POM 에 빌드 경로 설정시 META-INF 밑 경로가 Web Jar 경로가 된다.
종속 라이브러리로 지정했던 jquery 의 경우도 /webjars 밑에 아티팩트 아이디와 버전정보를 기입해주면 쉽게 import 하여 사용할 수 있다.

![RESULT](/assets/img/post5-3.png)

## 결론

- 정적파일 위치 설정
- POM 정보 및 빌드 설정 
- 적용

WEB JAR 어렵지 않다. 세가지만 기억하자!

[All the sources on the blog can be found here.](https://github.com/dalgun/play)




 


 
[^1]: Nexus는 Sonatype 에서 만든 저장소 관리자 프로젝트로, 다양한 Format 의 사설 저장소를 만들 수 있으며 메인 저장소를 Cache할 수 있는 기능 또한 제공하여 저장소를 관리할 수 있도록 도와주는 관리자 도구입니다. Maven 에서 사용할 수 있는 가장 널리 사용되는 무료 저장소로 잘 알려져있습니다.[출처](https://blog.kingbbode.com/posts/nexus-3xx-maven-npm)
[^2]: [인용, 출처](https://gs.saro.me/dev?tn=466)
[^3]: [WEB JAR](https://www.webjars.org/)