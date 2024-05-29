---
layout: post
title: "Spring Batch3"
description: "Spring Batch  메타데이터 테이블에 대해 알아봅시다."
date: 2021-10-25
tags: [SpringBatch]
writer: syh8088
category: SpringBatch
comments: true
share: true
---

# Spring Batch - 메타데이터 스키마

이번에는 Spring Batch 에서 사용되는 메타데이터 보관하는 스키마에 대해 알아보겠습니다.

우리는 mysql 기준으로 Spring Batch 에서 사용되는 메타데이터를 보관 해야 하니

``schema-mysql.sql``

해당 파일을 검색하면 여러 스키마를 볼수 있습니다.

이렇게 메타데이터가 존재하는 이유는 이전 과거 또는 현재 배치 실행에 있어서 상태정보를 저장시켜서 

체계적인 관리 및 처리를 하기 위함입니다.


각각의 테이블 정보는 

https://docs.spring.io/spring-batch/docs/3.0.x/reference/html/metaDataSchema.html

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/spring_batch_meta_data_erd.png)

여기서 확인이 가능합니다.

그럼 Job 을 실행시켜 저장되는 각각의 메타데이터를 알아보도록 하겠습니다.

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/BasicJobConfigurationStart.PNG)

총 9개 테이블이 존재하지만 여기서는 대표적인 테이블만 보도록 하겠습니다.


### BATCH_JOB_INSTANCE

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/BATCH_JOB_INSTANCE.PNG)

Job 을 실행하면 Job_instance 에 관련 메타데이터를 저장하는 테이블입니다.
* JOB_NAME: 실행된 Job 이름 입니다. 우리는 'basicJob' 이라고 실행했으니 'basicJob' 이라고 저장 되었습니다.
* JOB_KEY: 실행된 Job 의 이름과 파라미터값 합쳐서 해시 알고리즘 통해 저장된 값 입니다. 실행 할때 파라미터값을 지정하지 않으면 Job 이름만 통해 해시 알고리즘으로 저장 됩니다.

Job 실행전 먼저 BATCH_JOB_INSTANCE 테이블 이미 실행된 JOB_NAME, JOB_KEY 값인지 확인 합니다.

즉 JOB_NAME, JOB_KEY 은 묶어서 유니크로 지정되어 있기 때문에 같은 Job 이름 Job 파라미터를 지정 할 수 없습니다.


### BATCH_JOB_EXECUTION

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/BATCH_JOB_EXECUTION.PNG)

Job 실행에 있어서 배치 시간, 상태코드 등 상태정보를 저장 하는 테이블 입니다.

핵심 컬럼만 보겠습니다.
* JOB_INSTANCE_ID: 매핑된 BATCH_JOB_INSTANCE 테이블의 PK 값입니다.
* CREATE_TIME: 배치 실행시 JobExecution 객체 생성 시간 입니다.
* START_TIME: 배치 실행 시간 입니다.
* END_TIME: 배치 실행 완료 시간 입니다.
* STATUS: 상태 코드 입니다. (상태값 소개 part 에서 자세히 말씀 드리겠습니다.)
* EXIT_CODE: 종료 코드 입니다. (상태값 소개 part 에서 자세히 말씀 드리겠습니다.)
* EXIT_MESSAGE: 만약 에러 발생시 에러 메세지 입니다.


### BATCH_JOB_EXECUTION_PARAMS

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/BATCH_JOB_EXECUTION_PARAMS.PNG)

실행시 특별히 파라미터를 지정하지 않아 빈값으로 나왔지만 파라미터를 지정 하게 된다면 저장 되는 값 입니다.

* JOB_EXECUTION_ID: BATCH_JOB_EXECUTION PK 값 입니다.
* TYPE_CD: Job Parameter 에서는 'String', 'Long', 'DOUBLE', 'DATE' 타입으로 지정 할 수 있는데 이 중 어떤 데이터 타입인지 확인하는 컬럼 입니다.
* KEY_NAME: Job Parameter 지정시 key 이름 입니다.
* STRING_VAL: Job Parameter 데이터 타입이 String 일때 value 값 입니다.
* DATE_VAL : Job Parameter 데이터 타입이 DATE 일때 value 값 입니다.
* LONG_VAL: Job Parameter 데이터 타입이 Long 일때 value 값 입니다.
* DOUBLE_VAL : Job Parameter 데이터 타입이 DOUBLE 일때 value 값 입니다.
* IDENTIFYING: 해당 job Parameter 값의 식별 여부 입니다. 
	
### BATCH_STEP_EXECUTION

![Formula]({{ site.url }}{{ site.baseurl }}/images/SpringBatch/BATCH_STEP_EXECUTION.PNG)

Step 실행에 있어서 배치 시간, 상태코드 등 상태정보를 저장 하는 테이블 입니다.

* STEP_EXECUTION_ID: BATCH_STEP_EXECUTION pk 값 입니다.
* VERSION: 
* STEP_NAME: 실행된 Step 이름 입니다.
* JOB_EXECUTION_ID: BATCH_JOB_EXECUTION 테이블 Pk 값 입니다.
* START_TIME: 배치 실행 시간 입니다.
* END_TIME: 배치 실행 완료 시간 입니다.
* STATUS: 상태 코드 입니다. (상태값 소개 part 에서 자세히 말씀 드리겠습니다.)

여기서 부터는 후에 청크 기반의 itemReader, ItemWriter, ItemProcessor 설명때 자세히 설명 드리겠습니다.

* COMMIT_COUNT: 실행 중 트렌젝션당 커밋 카운트 입니다.
* READ_COUNT: 실행 시점에 읽은 item 카운트 입니다.
* FILTER_COUNT: 실행 도중 필터링 item 카운트 입니다.
* WRITE_COUNT: 실행 도중 쓰기 item 카운트 입니다.
* READ_SKIP_COUNT: 실행 시점에 읽지 않는 item 카운트 입니다.
* WRITE_SKIP_COUNT: 실행 도중 쓰기를 하지 않는 item 카운트 입니다.
* PROCESS_SKIP_COUNT: 실행 도중 process 하지 않는 item 카운트 입니다.
* ROLLBACK_COUNT: 실행 도중 롤백 item 카운트 입니다.
* EXIT_CODE: 종료 코드 입니다. (상태값 소개 part 에서 자세히 말씀 드리겠습니다.)
* EXIT_MESSAGE: 만약 에러 발생시 에러 메세지 입니다.
* LAST_UPDATED: 마지막에 실행된 날짜 정보 입니다.
