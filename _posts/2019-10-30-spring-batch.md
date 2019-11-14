---
layout: post
title: "Spring batch+Scheduler 구현 해보기"
subtitle: "Tasklet 으로 Spring Batch, 날로 먹기"
author: "Dalgun"
category: "Spring boot"
tags: [Spring batch]
comments: true
 
---

![명수옹](http://www.sportsq.co.kr/news/photo/201510/88932_109995_455.png)

### 주기적으로 처리해줘야 할 일이 있다면?

선배님 : 결제 승인 이후에 승인내역을 특정 VAN 사로 전송해줘야 하는 이슈 가 있다. 

달군 : 헐, 실패하면요?

선배님 : 재시도 해야지, `(너가 할꺼지, 달군아?)`

달군 : (반짝반짝) 그럼 일단 실패한건 재시도 테이블에 넣고 주기적으로 bulk 로 읽어와서, 재전송하고, 결과를 정리할께요! 

(~~블로그의 첫 도입부는 항상 어렵다 ㅠㅠ~~)

그래서 선택한것이,

>Spring Batch + Scheduler


[Spring Batch 공식문서](https://spring.io/projects/spring-batch) 를 읽어보면, ~~이해는 잘 되지만~~ 어떻게 적용해야할지 난감했다.

### 기본적인 구조

![batch_structure](https://uploads.toptal.io/blog/image/127593/toptal-blog-image-1543272709210-20c67748adda63724f6af2cbf2d73f96.png)

배치 하나가 하나의 Job 이며, 여러개의 Step 이 모여 Job 을 이루게 된다. 다만, Step 은 Reader , Processor, Writer 가 Tasklet 로 대체 될 수 있는데 이것은 하나의 큰 덩어리로 나의 경우 굳이 3분류로 나누지 않고 각 스텝별로 Tasklet 단위로 작성을 하는게 기능 별로 정의하는 느낌이라 편했다.<br>`(맞게 쓰는건지는 모르겠어서, 날로 먹..)`

> Reader + Processor + Writer = Tasklet


자 그러면, 내 상황에 맞게 코딩을 해보자!

- 이중화 구조를 감안하여 Lock Table 조회하여 다른 Instance가 사용중인지 확인하기

```java
@Bean
    public Step step1(){
        return stepBuilderFactory.get("step1")
                .tasklet((stepContribution, chunkContext) -> {

                    log.debug("======= 전송 배치 시작 =======");
                    LockTable lockTable = Optional
                            .ofNullable(lockTableRepository.findOne("play-batch"))
                            .orElse(LockTable
                                    .builder()
                                    .instanceId(instanceId)
                                    .useYn(false)
                                    .checkDataTime(LocalDateTime.now())
                                    .build());
                    log.debug("======= Lock Table 사용 여부:{},", lockTable.isUseYn());
                    if(lockTable.isUseYn()){
                        log.debug("======= Table 사용 중으로 종료 ");
                        stepContribution.setExitStatus(ExitStatus.FAILED);
                    }else{
                        lockTable.setUseYn(true);
                        lockTableRepository.save(lockTable);
                    }
                    return RepeatStatus.FINISHED;
                }).build();
    }
```

- 처리할 아이템이 있는지 확인하기

```java
@Bean
    public Step step2(){
        return stepBuilderFactory.get("step2")
                .tasklet((stepContribution, chunkContext) -> {
                    int itemCnt = payInfoRepository.countAllBySuccessYn(false);
                    log.debug("======= 처리할 Item 개수 : {}", itemCnt);
                    if(0 == itemCnt){
                        log.debug("======= 처리할 Item 이 없어 종료 ");
                        LockTable lockTable = lockTableRepository.findOne(instanceId);
                        lockTable.setUseYn(false);
                        lockTable.setCheckDataTime(LocalDateTime.now());
                        stepContribution.setExitStatus(ExitStatus.FAILED);
                    }
                    return RepeatStatus.FINISHED;
                }).build();
    }
```
- 성공한 아이템의 후처리
```java
@Bean
    public Step step3(){
        return stepBuilderFactory.get("step3")
                .tasklet((stepContribution, chunkContext) -> {

                    List<PayInfo> payInfoList = payInfoRepository.findAllBySuccessYn(false);

                    for(PayInfo payInfo: payInfoList){
                        // 무언가를 처리하고 결과값이 TRUE 일때 결과 성공 처리
                        if(true){
                            payInfo.setRequestDateTime(LocalDateTime.now());
                            payInfo.setSuccessYn(true);
                            payInfoRepository.save(payInfo);
                        }
                    }
                    return RepeatStatus.FINISHED;
                }).build();
    }
```
- Lock Table 초기화 후 종료처리

```java
@Bean
    public Step step4(){
        return stepBuilderFactory.get("step4")
                .tasklet((stepContribution, chunkContext) -> {
                    LockTable lockTable = lockTableRepository.findOne(instanceId);
                    lockTable.setUseYn(false);
                    lockTable.setCheckDataTime(LocalDateTime.now());
                    lockTableRepository.save(lockTable);
                    log.debug("======= 전송 배치 종료 =======");
                    return RepeatStatus.FINISHED;
                }).build();
    }
```

이것들을 조합하여 하나의 Job 으로 엮어보면 다음과 같다.
 
```java
@Bean
    public Job job(){
        return jobBuilderFactory.get("job")
                .start(step1()) //STEP 1 실행
                .on("FAILED")// STEP 1 결과가 FAILED 일 경우
                .end()// 종료
                .from(step1()) // STEP1 의 결과로부터
                .on("*") // FAILED 를 제외한 모든 경우
                .to(step2()) // STEP 2로 이동 후 실행
                .on("FAILED")// STEP 2의 결과가 FAILED 일 경우
                .end()// 종료
                .from(step2()) // STEP2 의 결과로부터
                .on("*") // FAILED 를 제외한 모든 경우
                .to(step3()) // STEP 3로 이동 후 실행
                .next(step4()) // STEP 3의 결과에 상관없이 STEP 4 실행
                .on("*") // STEP4 의 모든 결과에 상관없이
                .end() // FLOW 종료
                .end() // JOB 종료
                .build();
    }
```

이렇게 하나의 Job이 생성이 되었고 이것을 이제 Scheduler 에 등록하여 주기적으로 실행되게 적용하였다.
```java
@Slf4j
@Component
public class JobScheduler {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private JobConfiguration jobConfiguration;

    @Scheduled(initialDelay = 10000, fixedDelay = 30000)
    public void runJob() {

        Map<String, JobParameter> confMap = new HashMap<>();
        confMap.put("time", new JobParameter(System.currentTimeMillis()));
        JobParameters jobParameters = new JobParameters(confMap);

        try {

            jobLauncher.run(jobConfiguration.job(), jobParameters);

        } catch (JobExecutionAlreadyRunningException | JobInstanceAlreadyCompleteException
                | JobParametersInvalidException | org.springframework.batch.core.repository.JobRestartException e) {

            log.error(e.getMessage());
        }
    }

}
``` 

[블로그에 있는 모든 소스는 여기서 확인할 수 있어요](https://github.com/dalgun/play)

## 결론
Spring Boot의 Batch 는 개발자가 비지니스 로직에만 집중할 수 있도록 자동화가 잘 되어있다(날로 먹었).<br>~~item reader, processor, writer 로 짜보려다 일주일 날린거는 안비밀~~



