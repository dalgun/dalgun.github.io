---
layout: post
title: "Code Refactoring"
subtitle: "소소한 리팩토링 일지"
author: "Dalgun"
category: "Coding"
comments: true
tag: 
 - refactoring
 
---

|![thumb](/assets/img/post1-thumb.png)|
|:--:| 
| *Photo by dodamdodam on velog.io* [^1] |

## 냄새 나는 소스를 바꿔보자

 그래, 이 메소드의 `목적`이 뭔지는 알겠어! 근데 이게 최선이었니? 
 

```java
public class payService {
    
    //···
    
    public String oldMethod (PayInfo payInfo){
            String format = "";
            if("A".equals(payInfo.getType())){
                //결제자(10)+결제품목(20)+결제금액(10)+카드명(8)+할부기간(2)
                format =  String.format("%-10s%-20s%-10s%-8s%-2s",
                        payInfo.getUserName(),
                        payInfo.getProduct(),
                        payInfo.getAmount(),
                        payInfo.getCardNm(),
                        payInfo.getInstallmentPeriod());
    
            }else if("B".equals(payInfo.getType())){
                //MOBILE(고정):카드명:카드번호:결제금액:할부기간:결제자
                List payInfoData = new ArrayList();
                payInfoData.add(payInfo.getCardNm());
                payInfoData.add(payInfo.getCardNo());
                payInfoData.add(payInfo.getAmount());
                payInfoData.add(payInfo.getInstallmentPeriod());
                payInfoData.add(payInfo.getUserName());
                format =  "MOBILE:"+payInfoData.stream().collect(Collectors.joining(":")).toString();
            }else{
                //결제자(5)+결제금액(10)+결제품목(10)+카드사명(5)+카드번호(16)+할부기간(2)+요청시각(20)
                format = String.format("%-5s%-10s%-10s%-5s%-16s%-2s%-20s",
                        payInfo.getUserName(),
                        payInfo.getAmount(),
                        payInfo.getProduct(),
                        payInfo.getCardNm(),
                        payInfo.getCardNo(),
                        payInfo.getInstallmentPeriod(),
                        payInfo.getRequestDateTime());
            }
    
            return format;
    }
    //···
}
```

결제 정보의 타입에 따른 분기 로직이 모두 한 메소드안에 들어가 있어 서비스 로직이 길어지고, 무엇보다 **세련** 되지 못하다.

자! 이것을 팩토리 패턴으로 재구성 해보자!

## FACTORY PATTERN [^2]
>팩토리 메소드 패턴 : 객체를 생성하기 위한 인터페이스를 정의하는데, 어떤 클래스의 인스턴스를
 만들지는 서브클래스에서 결정하게 만든다. 즉 팩토리 메소드 패턴을 이용하면
클래스의 인스턴스를 만드는 일을 서브클래스에게 맡기는 것.
>
>추상 팩토리 패턴 : 인터페이스를 이용하여 서로 연관된, 또는 의존하는 객체를 구상 클래스를 지정하지 않고도 생성

![interface](/assets/img/post1-1.png)

Format Type에 따라 Format을 만들기위한 다른 객체를 생성하고 그 객체로 전문 규격을 생성하는 로직을 그려보자.


```java
public interface Formatter {

    String getFormat(PayInfo payinfo);

    Formatter DEFAULT_TYPE = new Formatter() {
        @Override
        public String getFormat(PayInfo payInfo) {
            //결제자(5)+결제금액(10)+결제품목(10)+카드사명(5)+카드번호(16)+할부기간(2)+요청시각(20)
            return String.format("%-5s%-10s%-10s%-5s%-16s%-2s%-20s",
                    payInfo.getUserName(),
                    payInfo.getAmount(),
                    payInfo.getProduct(),
                    payInfo.getCardNm(),
                    payInfo.getCardNo(),
                    payInfo.getInstallmentPeriod(),
                    payInfo.getRequestDateTime());
        }
    };
    class A_TYPE_FORMAT implements Formatter{
        @Override
        public String getFormat(PayInfo payinfo) {
            //결제자(10)+결제품목(20)+결제금액(10)+카드명(8)+할부기간(2)
            return String.format("%-10s%-20s%-10s%-8s%-2s",
                    payinfo.getUserName(),
                    payinfo.getProduct(),
                    payinfo.getAmount(),
                    payinfo.getCardNm(),
                    payinfo.getInstallmentPeriod());
        }
    }

    class B_TYPE_FORMAT implements Formatter {
        @Override
        public String getFormat(PayInfo payinfo) {
            //MOBILE(고정):카드명:카드번호:결제금액:할부기간:결제자
            List payInfoData = new ArrayList();
            payInfoData.add(payinfo.getCardNm());
            payInfoData.add(payinfo.getCardNo());
            payInfoData.add(payinfo.getAmount());
            payInfoData.add(payinfo.getInstallmentPeriod());
            payInfoData.add(payinfo.getUserName());
            return "MOBILE:"+payInfoData.stream().collect(Collectors.joining(":")).toString();
        }
    }
}
```

결제 타입에 따라 동일한 기능을 가지는 메소드 `getFormat(PayInfo payinfo)`를 강제하였다.

서비스 로직에서는 팩토리로 각 타입에 해당하는 구현체만 인스턴스 한 후 `getRemark` 메소드만 호출하면 훨씬 간결하고 명료해진 소스를 볼수 있다.
```java
public class PayService {
    //···
    public String newMethod(PayInfo payInfo){
        FormatterFactory formatterFactory = new FormatterFactory();
        Formatter formatter = formatterFactory.getFormatterType(payInfo.getType());
        return formatter.getFormat(payInfo);
    }
}
    //···

```

## 검증을 해볼까?

~~굳이 테스트 케이스를 첨부해야만 했나, 포스트를 글자 수 늘리기 위해서가 결코 아니다~~
```java
@Slf4j
public class refactoring1 {
    PayInfo payInfo;

    @BeforeEach
    public void init(){

        payInfo = new PayInfo();
        payInfo.setAmount(10000);
        payInfo.setCardNm("아메리카익스프레스");
        payInfo.setCardNo("1234123412341234");
        payInfo.setInstallmentPeriod("03");
        payInfo.setRequestDateTime(LocalDateTime.now());
        payInfo.setProduct("프로포폴");
        payInfo.setUserName("홍길동");
    }

    @Test
    public void 타입에_따른_FORMAT(){
        PayService payService = new PayService();
        payInfo.setType("A");
        String oldString  = payService.oldMethod(payInfo);
        log.info("oldString:<{}>, length:{}", oldString, oldString.length());
        String newString = payService.newMethod(payInfo);
        log.info("newString:<{}>, length:{}", newString, newString.length());
    }
}
```

결과는 아래와 같이 동일한 결과를 보여주었다.(~~이게 무슨의미가 있나~~)

![result](/assets/img/post1-2.png)

[달군이 이번 포스트, 해당 GitHub 소스 보러가기](https://github.com/dalgun/play)


[^1]: [이미지 출처]("https://velog.io/@myeongho0812/-%EC%B2%AB-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EB%A6%AC%ED%8C%A9%ED%86%A0%EB%A7%81-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EC%86%8C%EA%B0%9C-%EC%9B%B9-%EA%B0%9C%EB%B0%9C%EC%9E%90%EB%A1%9C-%EC%84%B1%EC%9E%A5%ED%95%98%EA%B8%B0-qxk0ugiy01")
[^2]: 출처 - https://jusungpark.tistory.com/14
 
