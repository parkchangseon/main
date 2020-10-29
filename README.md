![저기어때](https://user-images.githubusercontent.com/73212897/96727589-2d6a0880-13ee-11eb-91ab-acf6e0cc9526.jpg)


# 객실예약서비스
- 4조

# Table of contents

- [객실예약서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항
. 고객의 결제 금액에 따라 할인쿠폰, 음료수와 같은 서비스 또는 쿠폰을 제공한다.

비기능적 요구사항
1. 트랜잭션

    i. 결제가 되지 않은 건은 프로모션 적용이 되지 않아야 한다. Sync 호출 
2. 장애격리

    i. 객실관리시스템이 수행되지 않더라도 프로모션은 365일 24시간 제공되어야 한다.  Async (event-driven), Eventual Consistency
    
    ii. 프로모션시스템이 과중되면 사용자를 잠시동안 받지 않고 프로모션을 잠시후에 하도록 유도한다  Circuit breaker, fallback
3. 성능

    i. 고객이 프로모션 상태를 마이페이지 시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS    
    
    
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
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
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
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커를 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?

  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등
    - Contract Test : 자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?

# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
![image](https://user-images.githubusercontent.com/69283674/97150539-99af8800-17b1-11eb-8b8e-c6b8bcd1d537.png)

## TO-BE 조직 (Vertically-Aligned)
![image](https://user-images.githubusercontent.com/69283674/97150593-b350cf80-17b1-11eb-8c7f-68c5be80badc.png)

[조직 KPI]
예약팀 : 고객의 예약을 최대한 많이 받아야 함.
결제팀 : 결제 안정성을 최대한 확보해야 함.
객실팀 : 객실관리 편의성이 높아야 함.

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: 
![이벤트스토밍](https://user-images.githubusercontent.com/69283816/97504967-558fd380-19bb-11eb-9659-e05dc1662765.png)

기능적 요구사항

    - 고객의 결제금액에 따라 할인쿠폰, 음료수와 같은 할인 프로모션이 제공된다. (OK)
   
    - 고객은 언제든지 본인의 프로모션 상태를 확인할 수 있다. (OK)

1. 트랜잭션
 - 결제가 되지 않은 건은 프로모션이 적용되지 않아야 한다 (동기식 호출)

2. 장애격리
 - 객실관리시스템이(객실등록,객실예약 등) 수행되지 않더라도 프로모션은 365일 24시간 제공될 수 있어야 한다 Async (event-driven), Eventual Consistency
 - 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 프로모션을 잠시후에 하도록 유도한다 Circuit breaker, fallback
 
3. 성능
 - 고객이 프로모션을 언제든지 RoomInfo 시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/69283816/97513646-2a63af00-19d0-11eb-9251-8653fc54299c.png)

    - Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    
## 폴리글랏

Promotion 서비스 pom.xml에서 H2 DB -> Hsql DB로 변경

![폴리글랏](https://user-images.githubusercontent.com/69283816/97514532-4bc59a80-19d2-11eb-8306-23e9ad7415fa.png)

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd Promotion
mvn spring-boot:run

cd Reservation
mvn spring-boot:run

cd Payment
mvn spring-boot:run 

cd Room
mvn spring-boot:run  

cd Notice
mvn spring-boot:run  

cd Gateway
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```
package accommodation;
import javax.persistence.*;
import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Promotion_table")
public class Promotion {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private int couponId;
    private int paymentId;
    private int paymentPrice;
    private String paymentStatus;
    private int point;
    private String reserveStatus;
    private String service;

    @PrePersist
    public void onPrePersist() {
        if ("promotion".equals(reserveStatus) ) {
            System.out.println("=============마일리지 적립 처리중=============");
            setReserveStatus("promotion");
            PromotionSaved couponSaved = new PromotionSaved();

            couponSaved.setPaymentId(paymentId);
            couponSaved.setPaymentPrice(paymentPrice);
            couponSaved.setPaymentStatus(paymentStatus);

            if("Y".equals(paymentStatus)) {
                if (paymentPrice >= 100000) {
                    service = "DISCOUNT COUPON";
                    point = paymentPrice / 10;
                } else if (paymentPrice >= 50000 && paymentPrice < 100000) {
                    service = "BEVERAGE";
                    point = paymentPrice / 10;
                } else {
                    point = paymentPrice / 10;
                }
            } else {
                point = 0;
            }
            BeanUtils.copyProperties(this, couponSaved);

            couponSaved.publishAfterCommit();

            try {
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
                System.out.println("=============마일리지 적립 완료=============");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public int getCouponId() {
        return couponId;
    }

    public void setCouponId(int CouponId) {
        this.couponId = couponId;
    }

    public int getPaymentId() {
        return paymentId;
    }

    public void setPaymentId(int paymentId) {
        this.paymentId = paymentId;
    }

    public int getPaymentPrice() {
        return paymentPrice;
    }

    public void setPaymentPrice(int paymentPrice) {
        this.paymentPrice = paymentPrice;
    }

    public String getPaymentStatus() {
        return paymentStatus;
    }

    public void setPaymentStatus(String paymentStatus) {
        this.paymentStatus = paymentStatus;
    }

    public int getPoint() {
        return point;
    }

    public void setPoint(int point) {
        this.point = point;
    }

    public String getReserveStatus() {
        return reserveStatus;
    }

    public void setReserveStatus(String reserveStatus) {
        this.reserveStatus = reserveStatus;
    }

    public String getService() {
        return service;
    }

    public void setService(String service) {
        this.service = service;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package accommodation;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface PromotionRepository extends PagingAndSortingRepository<Promotion, Long> {
}
```
## 동기식 호출 

분석단계에서의 조건 중 하나로 예약(Reservation)-> 결제(Payment) -> 프로모션(Promotion) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

```
# (payment) PaymentManagementService.java
package accommodation.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

@org.springframework.stereotype.Service
@FeignClient(name="paymentNumber", url="${api.url.promotion}")
public interface PaymentManagementService {

    @RequestMapping(method= RequestMethod.POST, path="/promotions", consumes = "application/json")
    public void CompletePromotion(@RequestBody Promotion promotion);

}
```

- 결제 요청 시 동기적으로 실행
```
# Reservation.java (Entity)

    @PreUpdate
    public void onPreUpdate(){
        ...
            Payment payment = new Payment();
            payment.setPaymentPrice(getPaymentPrice());
            payment.setReservationNumber(getReservationNumber());
            payment.setReservationStatus(getReserveStatus()); 
            
            if (this.getPaymentPrice() >= 100000) {
                payment.setService("DISCOUNT COUPON");
                payment.setPoint(this.getPaymentPrice() / 10);
            } else if (this.getPaymentPrice() >= 50000 && this.getPaymentPrice() < 100000) {
                payment.setService("BEVERAGE");
                payment.setPoint(this.getPaymentPrice() / 10);
            } else {
                payment.setPoint(this.getPaymentPrice() / 10);
            }
            
            Application.applicationContext.getBean(PaymentManagementService.class).CompletePayment(payment);
          
    }
```
![프로모션적립동기호출](https://user-images.githubusercontent.com/69283816/97468638-dd101f00-1988-11eb-918a-3a0f0f5b5342.png)
![프로모션확인동기호출](https://user-images.githubusercontent.com/69283816/97467722-d3d28280-1987-11eb-8318-2e8fa924bc31.png)

- 동기식으로 프로모션 요청시 결제 시스템 장애의 경우 프로모션 불가 확인 :
```
# 결제 서비스 (pament) 를 잠시 내려놓음 
kubectl scale deploy room --replicas=0
```
![image](https://user-images.githubusercontent.com/69283674/97378907-db9e1280-1906-11eb-8ea9-09d6e20272d4.png)

```
#프로모션요청
http post http://promotion:8080/promotions paymentId=1 paymentPrice=100000 paymentStatus="Y" reserveStatus="promotion"
```
![image](https://user-images.githubusercontent.com/69283674/97378963-ff615880-1906-11eb-973c-dd307e672a9b.png)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


프로모션 적용 이후 이를 예약 정보 서비스를 비동기로 호출하여 개별적으로 조회 가능하도록 구현
 
- 이를 위하여 프로모션 서비스에서 ReservStatus를 promotion 으로 셋팅 후에 곧바로 프로모션 정보가 등록 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package accommodation;

@Entity
@Table(name="Payment_table")
public class Payment   {

 ...
    @PrePersist
    public void onPrePersist() {
        if ("promotion".equals(reserveStatus) ) {
            System.out.println("=============마일리지 적립 처리중=============");
            setReserveStatus("promotion");
            PromotionSaved couponSaved = new PromotionSaved();

            couponSaved.setCustomerId(customerId);
            couponSaved.setCustomerName(customerName);
            couponSaved.setPaymentId(paymentId);
            couponSaved.setPaymentPrice(paymentPrice);
            couponSaved.setPaymentStatus(paymentStatus);

            if("Y".equals(paymentStatus)) {
                if (paymentPrice >= 100000) {
                    service = "DISCOUNT COUPON";
                } else if (paymentPrice >= 50000 && paymentPrice < 100000) {
                    service = "BEVERAGE";
                } else {
                    point = paymentPrice / 10;
                }
            } else {
                point = 0;
            }
            BeanUtils.copyProperties(this, couponSaved);

            couponSaved.publishAfterCommit();

            try {
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
                System.out.println("=============마일리지 적립 완료=============");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    } 
```
- 예약 서비스에서 해당 비동기 호출을 수신할 PolicyHandler를 구현

```
package accommodation;

...

@Service
public class PolicyHandler{

    @Autowired
    ReservationRepository reservationManagementRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPromotionCompleted_ChangeResvStatus(@Payload PromotionSaved promotionSaved){
        System.out.println(promotionSaved.toJson());
        if(promotionSaved.isMe()){
            System.out.println("====================================프로모션 적립 1차====================================");
            if(reservationManagementRepository.findById(promotionSaved.getPaymentId()) != null){
                System.out.println("====================================프로모션 적립 완료====================================");
                Reservation reservationManagement = reservationManagementRepository.findById(promotionSaved.getPaymentId()).get();
                reservationManagement.setReserveStatus("paymentComp");
                reservationManagementRepository.save(reservationManagement);
            }
        }
    }
}

```

## CQRS 패턴 
사용자 View를 위한 프로모션 정보 조회 서비스를 위한 별도의 프로모션 정보 저장소를 구현
- 이를 하여 기존 CQRS 서비스인 RoomInfo 서비스를 활용
- 모든 정보는 비동기 방식으로 호출한다.

```
Promotion.java(Entity)

@Entity
@Table(name="Promotion_table")
public class Promotion {
    ...

    @PrePersist
    public void onPrePersist() {
        if ("promotion".equals(reserveStatus) ) {
            System.out.println("============= 적립 처리중=============");
            setReserveStatus("promotion");
            PromotionSaved couponSaved = new PromotionSaved();

            couponSaved.setCustomerId(customerId);
            couponSaved.setCustomerName(customerName);
            couponSaved.setPaymentId(paymentId);
            couponSaved.setPaymentPrice(paymentPrice);
            couponSaved.setPaymentStatus(paymentStatus);

            BeanUtils.copyProperties(this, couponSaved);

            couponSaved.publishAfterCommit();

            try {
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
                System.out.println("=============마일리지 적립 완료=============");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
```
- RoomInfo에 저장하는 서비스 정책 (PolicyHandler)구현
```
PolicyHandler.java

@Service
public class PolicyHandler{

    @Autowired
    RoomInfoRepository roomInfoRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverSave_PromotionSaved(@Payload PromotionSaved promotionSaved) {
        if (promotionSaved.isMe()) {
            // external message send
            System.out.println("##### listener Promotion 저장 : " + promotionSaved.toJson());
            RoomInfo roomInfo = new RoomInfo();
            roomInfo.setPaymentId(promotionSaved.getPaymentId());
            roomInfo.setPaymentPrice(promotionSaved.getPaymentPrice());
            roomInfo.setPaymentStatus(promotionSaved.getPaymentStatus());
            roomInfo.setService(promotionSaved.getService());
            roomInfo.setCouponId(promotionSaved.getCouponId());
            roomInfo.setPoint(promotionSaved.getPoint());
            roomInfo.setReserveStatus(promotionSaved.getReserveStatus());
            roomInfo.setCustomerId(promotionSaved.getCustomerId());
            roomInfo.setCustomerName(promotionSaved.getCustomerName());

            roomInfoRepository.save(roomInfo);
        }
    }
}

![CQRS](https://user-images.githubusercontent.com/69283816/97469033-498b1e00-1989-11eb-90b5-dbff227dc7cd.png)

```
# API 게이트웨이
Cloud 환경에서는 //서비스명:8080 에서 Gateway API가 작동해야함 application.yml 파일에 profile별 gateway 설정
-  Gateway 설정 파일 

![Gateway 설정 파일](https://user-images.githubusercontent.com/69283816/97454613-4c324700-197a-11eb-8048-a09a23514556.png)

# 운영

## Deploy 

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AZURE를 사용하였다.

-  아래와 같이 pod 가 정상적으로 올라간 것을 확인하였다. 

![POD 확인](https://user-images.githubusercontent.com/69283816/97471784-6bd26b00-198c-11eb-83ea-9441b6e0cd51.png)

-  아래와 같이 쿠버네티스에 모두 서비스로 등록된 것을 확인할 수 있다. 

![모든서비스확인](https://user-images.githubusercontent.com/69283816/97471853-80166800-198c-11eb-83c3-d56a7b20b9de.png)

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 예약(Reservation)-->결제(Payment)-->프로모션(Promotion) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 
결제 요청이 과도할 경우 서킷브레이커 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 서킷브레이커 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제:Payment) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (Payment) Payment.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
        try {
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
                System.out.println("=============결재 승인 완료=============");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 1명
- 10초 동안 실시

![서킷브레이커](https://user-images.githubusercontent.com/69283816/97518037-c8a84280-19d9-11eb-91d5-03af97f9034e.png)

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌.
하지만, 50%밖에 성공하지 못하였고 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

## Configmap
- configmap.yaml 파일설정

![configmapyml](https://user-images.githubusercontent.com/69283816/97516523-9b0dca00-19d6-11eb-987b-822b196113d1.png)

- deployment.yaml파일 설정

![deploymentyml](https://user-images.githubusercontent.com/69283816/97516535-a234d800-19d6-11eb-9aff-9f6230d088e8.png)

- application.yaml 파일 설정

![applicationyml](https://user-images.githubusercontent.com/69283816/97516542-a8c34f80-19d6-11eb-995c-7102f5cca6ae.png)

- paymentService 파일 설정

![paymentservice](https://user-images.githubusercontent.com/69283816/97516610-d3ada380-19d6-11eb-8bd3-07093a58b054.png)

- 80포트로 설정하여 테스트

![configmaptest](https://user-images.githubusercontent.com/69283816/97516634-e2945600-19d6-11eb-93a9-e586d1ec8edd.png)

### 오토스케일 아웃
앞서 서킷브레이커 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy payment --min=1 --max=10 --cpu-percent=15
```
![image](https://user-images.githubusercontent.com/69283674/97295328-7b6d8900-1892-11eb-9581-f40d9b09b5de.png)


- 결제서비스의 deployment.yaml의 spec에 아래와 같이 자원속성을 설정한다:

![image](https://user-images.githubusercontent.com/69283674/97291666-8f62bc00-188d-11eb-9594-c14a11328bb0.png)

- 서킷브레이커 에서 했던 방식대로 워크로드를 1분 동안 걸어준다.
```
siege -c100 -t60S -content-type "application/json" 'http://payment:8080/payments'

```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy payment -w
```

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

![image](https://user-images.githubusercontent.com/69283674/97295982-5fb6b280-1893-11eb-89ef-741b220b2201.png)


### 무정지 재배포

- 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 서킷브레이커 설정을 제거함
- seige 로 배포작업 직전에 워크로드를 모니터링 함.
![image](https://user-images.githubusercontent.com/69283674/97297532-837af800-1895-11eb-868d-f7c70ab6b3c6.png)


- 새버전으로의 배포 시작

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

- 동일한 시나리오로 재배포 한 후 Availability 확인:

![image](https://user-images.githubusercontent.com/69283674/97297629-a1e0f380-1895-11eb-89f3-acc70aa3a30c.png)

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.
