# 예제 - 음식배달

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


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


# 체크포인트

- 분석 설계

  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?


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

## Request/Response
- orderId가 있으면 요리를 시작한다
![image](https://user-images.githubusercontent.com/38934586/206200554-f74ac119-46a4-41f2-99f0-74897bda11a5.png)

## Gateway

- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

![image](https://user-images.githubusercontent.com/38934586/205929638-ac2b0e80-76bc-4b3d-8b42-41df2082c9a9.png)


