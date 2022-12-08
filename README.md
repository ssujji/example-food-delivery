# 배민 분석/설계/구현 실습

# Table of contents

  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
    - Event Storming 결과
  - [구현](#구현)
    - Saga(Pub/Sub)
    - CQRS
    - Request/Response
    - Gateway

# 서비스 시나리오

기능적 요구사항
- 고객이 메뉴를 선택하여 주문한다
- 고객이 배달 타입을 선택한다(추가) 예) 도보, 자전거, 바이크
- 고객이 결제한다
- 주문이 되면 주문 내역이 입점상점주인에게 전달된다
- 상점주인이 확인하여 요리한다
- 요리가 완성되면 라이더는 배달타입에 맞게 배달한다
- 고객이 주문을 취소할 수 있다
- 주문이 취소되면 배달이 취소된다
- 고객이 주문상태를 중간중간 조회한다
- 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다
- 배달이 완료되면 고객은 리뷰를 작성한다(추가)

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다  Sync 호출 
2. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
3. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과

### 완성된 모형

![image](https://user-images.githubusercontent.com/38934586/205815234-86986993-7f35-456b-be81-8d1cd3b3d242.png)
    

### 완성본에 대한 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/38934586/205931351-d504ecbe-c748-4a70-9a0f-db007446d8a1.png)

    - 고객이 메뉴를 선택하여 주문한다 (ok)
    - 고객이 배달 타입을 선택한다(ok) 예) 도보, 자전거, 바이크
    - 고객이 결제한다 (ok)
    - 주문이 되면 주문 내역이 상점주인에게 전달된다 (ok)
    - 상점주인이 확인하여 요리한다 (ok)
    - 요리가 완성되면 라이더는 배달타입에 맞게 배달한다 (ok)

![image](https://user-images.githubusercontent.com/38934586/205923872-890e29d1-2925-4bed-b16d-180d1f775124.png)

    - 고객이 주문을 취소할 수 있다 (ok)
    - 주문이 취소되면 배달이 취소된다 (ok)
   
![image](https://user-images.githubusercontent.com/38934586/205925481-535dcd0c-e568-48a6-83d5-3f8ae24b0c13.png)
    
    - 고객이 주문상태를 중간중간 조회한다 (ok) 
    - 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다 (ok)
    - 배달이 완료되면 고객은 리뷰를 작성한다(ok)


# 구현

## Saga(Pub/Sub)

- 마이크로 서비스간의 통신에서 이벤트 메세지를 Pub/Sub하는 방법을 실습한다
- app서비스에서 orderPlaced 이벤트를 발생하였을 때 store 서비스에서 orderPlaced 이벤트를 수신하여 주문을 접수한다

### app 서비스의 Publish

- app 마이크로 서비스를 실행한다
- app 폴더 -> Open In Terminal
```
mvn spring-boot:run
```
  - 기동된 app 서비스를 호출하여 주문을 요청한다

![image](https://user-images.githubusercontent.com/38934586/205934188-14abb090-ca20-46a0-969b-d37e990f7c74.png)

  - kafka 유틸리티가 포함된 위치에 접속하기 위해 docker를 통하여 shell에 진입한다

```
cd kafka
docker-compose exec -it kafka /bin/bash

cd /bin
./kafka-console-consumer --bootstrap-server localhost:9092 --topic fooddelivery
```
### store 서비스의 Publish
- store 마이크로 서비스를 실행한다
- store 폴더 -> Open In Terminal
```
mvn spring-boot:run
```
  - 기동된 store 서비스를 호출하여 주문을 확인한다
![image](https://user-images.githubusercontent.com/38934586/205936503-0cf083a6-2012-44be-8f57-df492c43750a.png)

  - accept 상태를 true로 변경하여 상태가 update되는 것을 확인한다
![image](https://user-images.githubusercontent.com/38934586/205937886-7a0f23f2-7322-431a-9f1e-dbd3ddceb624.png)

## CQRS
- 고객이 주문상태를 중간중간 조회할 수 있도록 CQRS로 구현하였다
  - viewpage MSA ViewHandler 를 통해 구현 
![image](https://user-images.githubusercontent.com/38934586/206192402-a6bd7fdb-da22-4afa-90a7-b06a2df82d7a.png)

  - MyPage CRUD 상세 설계
![image](https://user-images.githubusercontent.com/38934586/206194316-52273453-30d2-424c-a5f7-ef13b82b6ffc.png)

## Compensation/Correlation
- 주문이 취소될 때 Compensation이 발생한다.
- 주문취소 이벤트 OrderCanceled를 publish하면서 store 서비스에서 상태를 변경한다

## Request/Response
- orderId가 있으면 요리를 시작한다
![image](https://user-images.githubusercontent.com/38934586/206200554-f74ac119-46a4-41f2-99f0-74897bda11a5.png)

## Gateway

- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

![image](https://user-images.githubusercontent.com/38934586/205929638-ac2b0e80-76bc-4b3d-8b42-41df2082c9a9.png)


