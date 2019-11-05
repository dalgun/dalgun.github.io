---
layout: post
title: "JsonSubType 으로 Polymorphism Rest 만들기"
subtitle: "잔머리로 만들어 보는 REST"
author: "Dalgun"
category: "Coding"
comments: true
tag: 
 - spring boot
 - rest api
 - json sub type
 - polymorphism
---

![다형성](https://miro.medium.com/max/1276/1*cFSJ95jyw-ACiWaIRxAbbg.jpeg)

# 다형성은 비슷한 서비스 로직을 공통화 할 수 있다!

로직안에서 상속받는 객체들을 다형성으로 처리한 경험은 많이들(~~난 초보 개발자라 아직 아니지만~~) 있으실 거다. 뭐 예를 들면,

```java

class FireEngine extends Car{
    
}

class Ambulance extends Car {
    
}

FireEngine fireEngine = new FireEngine();
Ambulance ambulance = new Ambulance();

// 이렇게 매개변수를 다형성으로 처리하게 되면, 메소드 하나로 처리할 수 있게 되지만,
void buy(Car car){
    
}
buy(fireEngine);
buy(ambulance);

//만약 다형성을 사용하지 않는 다면, 각 객체마다 해당 메소드를 선언해주어야 한다.
void buyFireEngine(FireEngine fireEngine){
    
}
void buyAmbulance(Ambulance ambulance){
    
}
buyFireEngine(fireEngine);
buyAmbulance(ambulance);

```
> 다형성에 관한 자세한 내용은 [W3School](https://www.w3schools.com/java/java_polymorphism.asp)을 참고하시는게 좋을 것 같다.

# 난 REST API 에도 적용하고 싶은걸!

`목표`: REST API 에도 다형성을 적용 해보자(전 MAP 사용을 지양합니다, ~~MAP을 사용하면 이건 안해도 되겠지만..~~)<br>
`상황`: 예약을 받는 REST API, 요청 JSON 에 따라 POJO[^1] 에 매핑하고 예약 내용을 리턴해주기!

목표에 밝혔듯이 전 데이터의 정합성과 안정성을 위해 MAP 보단 POJO 로 코딩하는 것을 선호 합니다.

일단 REST 는,
```java
@Slf4j
@RestController
@RequestMapping("/reservation")
public class ReservationController {
    @PostMapping("/v1")
    public ResponseEntity post(@Validated @RequestBody ReservationBase reservationBase){
        log.debug("TYPE", reservationBase.getReservationType());
        return ResponseEntity.ok().body(reservationBase);
    }
}
```
Reservation Base 객체를 기본으로 보내는 JSON 에 따라 매핑을 달리 해주기 위해서는
jackson library 에 Json Sub Type 을 사용하기로 결정했다.

```java
@Data
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME
        // 이 설정이 들어가면, super 와 그것을 상속받는 객체에 중복값이 안나온다
        , include = JsonTypeInfo.As.EXISTING_PROPERTY 
        //json 안의 어떤 키값으로 타입을 결정할지 키값
        , property = "reservationType"
        // 키값이 없을 경우 디폴트로 설정할 class 
        , defaultImpl = SimpleReservationBase.class) 
@JsonSubTypes({
        @JsonSubTypes.Type(name = "SIMPLE", value = SimpleReservationBase.class),
        @JsonSubTypes.Type(name = "CONSULTING", value = ConsultingReservationBase.class),
        @JsonSubTypes.Type(name = "DIAGNOSIS", value = DiagnosisReservationBase.class)
})
public class ReservationBase {

    public ReservationBase(ReservationType reservationType){this.reservationType = reservationType;}

    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE)
    private String reservationDt;

    @DateTimeFormat(iso = DateTimeFormat.ISO.TIME)
    private String reservationTm;

    private ReservationType reservationType;

}
```

이와 같은 POJO 를 설정해주면, reservationType 은 value 에 따라 POJO 가 결정이 된다.
예를 들면,

![DIAGNOSIS](/assets/img/post4-1.png)

DIAGNOSIS TYPE 의 POJO 가 그대로 응답을 받아왔다. 그렇다면, TYPE 을 CONSULTING 으로 바꾸면 어떻게 될까?

![CONSULTING](/assets/img/post4-2.png)

CONSULTING POJO 에는 symptom 키가 없고 대신 consulting type 과 memo 가 있는데 null 이기 때문에 POJO 를 Serialization 을 하면 그림과 같이 나오게 된다.

좀 더 다양한 테스트를 해보기 위해 unit test 를 구동해보자
```java
@Slf4j
@RunWith(SpringRunner.class)
//@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@WebMvcTest(ReservationController.class)
public class MockMvcTest {

    @Autowired
    private MockMvc mvc;

    Map<String, String> reqMap;


    @Before
    public void init(){
        reqMap = new HashMap<>();
        reqMap.put("reservationDt", LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd")));
        reqMap.put("reservationTm", LocalDateTime.now().format(DateTimeFormatter.ofPattern("HHMMss")));
    }

    @Test
    public void SIMPLE_RESERVATION_TEST() throws Exception{

        reqMap.put("reservationType", "SIMPLE");

        mvc.perform(MockMvcRequestBuilders
                .post("/reservation/v1")
                .content(asJsonString(reqMap))
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .accept(MediaType.APPLICATION_JSON_UTF8))
                .andDo(MockMvcResultHandlers.print())
                .andExpect(status().is2xxSuccessful());
    }

    @Test
    public void CONSULTING_RESERVATION_TEST() throws Exception{

        reqMap.put("reservationType", "CONSULTING");
        reqMap.put("consultingType", "FAMILY");
        reqMap.put("memo", "Long Long ago..");

        mvc.perform(MockMvcRequestBuilders
                .post("/reservation/v1")
                .content(asJsonString(reqMap))
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .accept(MediaType.APPLICATION_JSON_UTF8))
                .andDo(MockMvcResultHandlers.print())
                .andExpect(status().is2xxSuccessful());
    }

    @Test
    public void DIAGNOSIS_RESERVATION_TEST() throws Exception{

        reqMap.put("reservationType", "DIAGNOSIS");
        reqMap.put("symptom", "Ouch..That's hurts");

        mvc.perform(MockMvcRequestBuilders
                .post("/reservation/v1")
                .content(asJsonString(reqMap))
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .accept(MediaType.APPLICATION_JSON_UTF8))
                .andDo(MockMvcResultHandlers.print())
                .andExpect(status().is2xxSuccessful());
    }



    public static String asJsonString(final Object obj) {
        try {
            return new ObjectMapper().writeValueAsString(obj);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

테스트 케이스 중 하나의 결과를 보여주자면,
```
MockHttpServletResponse:
           Status = 200
    Error message = null
          Headers = {Content-Type=[application/json;charset=UTF-8]}
     Content type = application/json;charset=UTF-8
             Body = {"reservationDt":"20191106","reservationTm":"011145","reservationType":"CONSULTING","consultingType":"FAMILY","memo":"Long Long ago.."}
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
2019-11-06 01:17:46.140  INFO 10751 --- [       Thread-2] o.s.w.c.s.GenericWebApplicationContext   : Closing org.springframework.web.context.support.GenericWebApplicationContext@482bce4f: startup date [Wed Nov 06 01:17:44 KST 2019]; root of context hierarchy

```

# 결론
REST API 도 각 POJO 에 따라서 여러개의 API 를 만들 것 없이 JSON SUB TYPE 을 이용한다면, 쉽게 공통화 할 수 있다! 
~~소스와 그림만 많고 글이 적어 보이는 것은 기분탓입니다.~~




[^1]: PLAIN OLD JAVA OBJECT 로, "getter / setter를 가진 단순한 자바 오브젝트"로 정의


