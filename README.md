# 개인과제 - 12번가

![12번가](https://user-images.githubusercontent.com/88864433/133467597-709524b1-4613-4dab-bc57-948f433ad565.png)
---------------------------------

# Table of contents

- [개인과제 - 12번가 배송서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#DDD의-적용)
    - [동기식 호출과 Fallback 처리](#동기식-호출과-Fallback-처리) 
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency) 
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [API 게이트웨이](#API-게이트웨이)
  - [운영](#운영)
    - [Deploy/Pipeline](#deploypipeline)
    - [동기식 호출 / Circuit Breaker / 장애격리](#동기식-호출-circuit-breaker-장애격리)
    - [Autoscale (HPA)](#autoscale-(hpa))
    - [Zero-downtime deploy (Readiness Probe)](#zerodowntime-deploy-(readiness-Probe))
    - [Self-healing (Liveness Probe)](#self-healing-(liveness-probe))
    - [운영유연성](#운영유연성)



# 서비스 시나리오
	
[서비스 아키텍쳐]
주문팀, 상품배송팀, 마케팅팀, 메시지팀(추가구현대상)

[서비스 시나리오]

가. 기능적 요구사항

1. [주문팀]고객이 상품을 선택하여 주문 및 결제 한다.

2. [주문팀]주문이 되면 주문 내역이 상품배송팀에게 전달된다. orderplaced (Pub/sub)

3-1. [상품배송팀] 주문을 확인하고 쿠폰 발행을 요청한다. ( Req/Res )
3-2. [상품배송팀] 주문을 확인하고 메시지 발송이 전달된다 ( Pub/Sub )

4-1. [마케팅팀] 쿠폰을 발행하고 상품배송팀에 알린다. ( Req/Res )
4-2. [메시지팀] 상품배송을 확인하고 메시지를 발송한다.

5. [상품배송팀] 쿠폰이 발행이 완료되면(반드시), 배송을 출발한다.

6. [주문팀]고객이 주문을 취소한다.

7. [주문팀]주문 취소를 상품배송팀에 알린다.

8-1. [상품배송팀] 주문 취소를 확인하고 쿠폰 취소를 요청한다. ( Req/Res )
8-2. [상품배송팀] 주문 취소를 확인하고 취소메시지 발송을 요청한다. ( Req/Res )

9-1. [마케팅팀] 발행된 쿠폰을 취소하고 상품배송팀에 알린다. ( Req/Res )
9-2. [메시지팀] 취소메시지를 발송하고 상품배송팀에 알린다. ( Req/Res )

11. [상품배송팀] 쿠폰이 발행이 취소되고 취소메시지가 발송되면 (반드시), 배송을 취소한다.


나. 비기능적 요구사항

1. [설계/구현]Req/Resp : 쿠폰이 발행된 건에 한하여 배송을 시작한다. 

2. [설계/구현]CQRS : 고객이 주문상태를 확인 가능해야한다.

3. [설계/구현]Correlation : 주문을 취소하면 -> 쿠폰을 취소하고 취소메시지를 발송하고 -> 배달을 취소 후 주문 상태 변경한다.

4. [설계/구현]saga : 서비스(상품팀, 상품배송팀, 마케팅팀, 메시지팀)는 단일 서비스 내의 데이터를 처리하고, 각자의 이벤트를 발행하면 연관된 서비스에서 이벤트에 반응하여 각자의 데이터를 변경시킨다.

5. [설계/구현/운영]circuit breaker : 배송 요청 건수가 임계치 이상 발생할 경우 Circuit Breaker 가 발동된다. 

다. 기타 

1. [설계/구현/운영]polyglot : 상품팀과 주문팀은 서로 다른 DB를 사용하여 polyglot을 충족시킨다.


# 체크포인트

+ 분석 설계

	- 이벤트스토밍:
		- 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
		- 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
		- 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
		- 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?
	- 서브 도메인, 바운디드 컨텍스트 분리
		- 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
			- 적어도 3개 이상 서비스 분리
		- 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
		- 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
	- 컨텍스트 매핑 / 이벤트 드리븐 아키텍처
		- 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
		- Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
		- 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
		- 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
		- 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?
	- 헥사고날 아키텍처
		- 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?

- 구현

	- [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
		-Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
		- [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
		- 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
	- Request-Response 방식의 서비스 중심 아키텍처 구현
		- 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
		- 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
	- 이벤트 드리븐 아키텍처의 구현
		- 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
		- Correlation-key: 각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
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
		- 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
		- 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
		- 모니터링, 앨럿팅:

	- 무정지 운영 CI/CD (10)
		- Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명
		- Contract Test : 자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?
---

# 분석/설계

## Event Stoming 결과

- MSAEz로 모델링한 이벤트스토밍 결과
https://www.msaez.io/#/storming/0MMWUdqHaeZClTr5NOawTiqn8rp1/4c0d2945840662a28cfa3dc786bdbe3a


### 이벤트 도출
![1](https://user-images.githubusercontent.com/60597727/135461261-a2af25b7-9657-4c24-b7b0-0bb9c20b0cc9.png)

```
1차적으로 필요하다고 생각되는 이벤트를 도출하였다 
``` 

### 부적격 이벤트 탈락

![2](https://user-images.githubusercontent.com/60597727/135461308-4c54460a-19dd-4958-8294-f89582dcdff9.png)

```
- 과정 중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
- ‘재고가 충족됨’, ‘재고가 부족함’ 은 배송 이벤트를 수행하기 위한 내용에 가까우므로 이벤트에서 제외
- 주문과 결제는 동시에 이루어진다고 봐서 주문과 결제를 묶음 
```

![3](https://user-images.githubusercontent.com/60597727/135461354-c03ec745-1515-47d6-b757-6a1b774eebe0.png)


### 액터, 커맨드를 부착하여 읽기 좋게 

![4](https://user-images.githubusercontent.com/60597727/135462456-b81bd1d3-9faf-4ea3-89aa-666cf7807e1b.png)


 
### 어그리게잇으로 묶기

![5](https://user-images.githubusercontent.com/60597727/135462461-2e48f744-801b-48d9-abcc-8bbe4b11064d.png)
 
``` 
- 고객의 주문후 배송팀의 배송관리, 마케팅의 쿠폰관리, 메시지팀의 메시지관리는 command와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어줌
```

### 바운디드 컨텍스트로 묶기

![6](https://user-images.githubusercontent.com/60597727/135468185-5561de4d-45bb-417b-a438-046055095833.png)
 
```
- 도메인 서열 분리 
    - Core Domain:  order, delivery : 본 서비스의 핵심 업무이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 order의 경우 1주일 1회 미만, delivery의 경우 1개월 1회 미만
    - Supporting Domain:  marketing, message : 고객관리를 위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함
```
### 폴리시 부착

![7](https://user-images.githubusercontent.com/60597727/135468071-320a2b6b-6984-4313-95e0-41206762462d.png)
 

### 폴리시의 이동과 컨텍스트 맵핑 (점선은 Pub/Sub, 실선은 Req/Resp) 

![8](https://user-images.githubusercontent.com/60597727/135468568-acb1d8c1-5202-4ec2-ab5e-1670d016737f.png)
 

### 완성된 모형

![image](https://user-images.githubusercontent.com/60597727/135468919-dce9a46a-d3c2-435d-b654-6a617d365164.png)
 
### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![10](https://user-images.githubusercontent.com/60597727/135471162-124e4e1c-b17d-4cd4-805c-626a875fadaa.png)

```
- 고객이 물건을 주문하고 결제한다 (ok)
- 결제가 완료되면 주문 내역이 배송팀에 전달된다 (ok)
- 마케팅팀에서 쿠폰을 발행한다 (ok)
- 메시지팀에서 주문된 문자를 발송한다 (ok)
- 쿠폰이 발행된 것을 확인하고 배송을 시작한다 (ok)
```
![11](https://user-images.githubusercontent.com/60597727/135471169-bb48efba-4689-438f-9a3d-536514d1ad60.png)

``` 
- 고객이 주문을 취소한다 (ok)
- 주문을 취소하면 결제도 함께 취소된다 (ok)
- 주문이 취소되면 배송팀에 전달된다 (ok)
- 마케팅팀에서 쿠폰발행을 취소한다 (ok)
- 메시지팀에서 취소된 문자를 발송한다 (ok)
- 쿠폰발행이 취소되고 취소문자가 발송되면 배송팀에서 배송을 취소한다 (ok)
```

### 비기능 요구사항에 대한 검증

![12](https://user-images.githubusercontent.com/60597727/135473895-531c3fda-90b0-428b-8a95-b85d4c510a55.png)

```
1. [설계/구현]Req/Resp : 쿠폰이 발행된 건에 한하여 배송을 시작한다. 취소문자가 발송되어야 배송팀에서 배송을 취소할수 있다
2. [설계/구현]CQRS : 고객이 주문상태를 확인 가능해야한다.
3. [설계/구현]Correlation : 주문을 취소하면 -> 쿠폰을 취소하고 -> 취소 문자가 발송되고 -> 배달을 취소 후 주문 상태 변경
4. [설계/구현]saga : 서비스(상품팀, 상품배송팀, 마케팅팀, 메시지팀)는 단일 서비스 내의 데이터를 처리하고, 각자의 이벤트를 발행하면 연관된 서비스에서 이벤트에 반응하여 각자의 데이터를 변경시킨다.
5. [설계/구현/운영]circuit breaker : 배송 취소가 임계치 이상 발생할 경우 Circuit Breaker 가 발동된다. 
``` 

### 헥사고날 아키텍처 다이어그램 도출 (그림 수정필요없는지 확인 필요)

![13](https://user-images.githubusercontent.com/60597727/135475246-5d3cbc7e-83a8-49c1-8ed0-93536642ebee.png)
 

```
- Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
- 호출관계에서 PubSub 과 Req/Resp 를 구분함
- 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
```

# 구현

- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 바운더리 컨텍스트 별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd order
mvn spring-boot:run

cd productdelivery 
mvn spring-boot:run

cd marketing
mvn spring-boot:run 

cd message
mvn spring-boot:run 
```

# DDD의 적용
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가? 

각 서비스 내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다. (주문(order), 배송(productdelivery), 마케팅(marketing), 메시지(message)) 

주문 Entity (Order.java) 
```
@Entity
@Table(name="Order_table")
public class Order {

    
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String username;
    private String address;
    private String phoneNo;
    private String productId;
    private int qty; //type change
    private String payStatus;
    private String userId;
    private String orderStatus;
    private Date orderDate;
    private String productName;
    private Long productPrice;
    private String couponId;
    private String couponKind;
    private String couponUseYn;

    @PostPersist
    public void onPostPersist(){
    	
         Logger logger = LoggerFactory.getLogger(this.getClass());

    	
        OrderPlaced orderPlaced = new OrderPlaced();
        BeanUtils.copyProperties(this, orderPlaced);
        orderPlaced.publishAfterCommit();
        System.out.println("\n\n##### OrderService : onPostPersist()" + "\n\n");
        System.out.println("\n\n##### orderplace : "+orderPlaced.toJson() + "\n\n");
        System.out.println("\n\n##### productid : "+this.productId + "\n\n");
        logger.debug("OrderService");
    }

    @PostUpdate
    public void onPostUpdate() {
    	
    	OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        orderCanceled.publishAfterCommit();
    }
    
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }
    
....생략 

```

Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 하였고 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

OrderRepository.java

```
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{
	
}
```

배송팀의 StockDelivery.java

```
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
```
```
@Entity
@Table(name="StockDelivery_table")
public class StockDelivery {

     //Distance 삭제 및 Id auto로 변경
    
    private Long orderId;
    private String orderStatus;
    private String userName;
    private String address;
    private String productId;
    private Integer qty;
    private String storeName;
    private Date orderDate;
    private Date confirmDate;
    private String productName;
    private String phoneNo;
    private Long productPrice;
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String customerId;
    private String deliveryStatus;
    private Date deliveryDate;
    private String userId;
    
    private static final String DELIVERY_STARTED = "delivery Started";
    private static final String DELIVERY_CANCELED = "delivery Canceled";
... 생략 
```

마케팅의 promote.java 

``` 
@Entity
@Table(name="Promote_table")
public class Promote {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String phoneNo;
    private String username;
    private Long orderId;
    private String orderStatus;
    private String productId;
    private String payStatus;
    private String couponId;
    private String couponKind;
    private String couponUseYn;
    private String userId;

    @PostPersist
    public void onPostPersist(){
        CouponPublished couponPublished = new CouponPublished();
        BeanUtils.copyProperties(this, couponPublished);
        couponPublished.publishAfterCommit();

    }

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	public String getPhoneNo() {
		return phoneNo;
	}
.... 생략 

```

PromoteRepository.java

```
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
public interface PromoteRepository extends PagingAndSortingRepository<Promote, Long>{

	List<Promote> findByOrderId(Long orderId);

}
```

- 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
가능한 현업에서 사용하는 언어를 모델링 및 구현시 그대로 사용하려고 노력하였다. 

- 적용 후 Rest API의 테스트
주문 결제 후 productdelivery 주문 접수하기 POST

```
[시나리오 1]
http POST http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orders address=“Seoul” productId=“1001" payStatus=“Y” phoneNo=“01011110000" productName=“Mac” productPrice=3000000 qty=1 userId=“goodman” username=“John”
http POST http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orders address=“England” productId=“2001” payStatus=“Y” phoneNo=“0102220000” productName=“gram” productPrice=9000000 qty=1 userId=“gentleman” username=“John”
http POST http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orders address=“USA” productId=“3001" payStatus=“Y” phoneNo=“01030000" productName=“Mac” productPrice=3000000 qty=1 userId=“goodman” username=“John”
http POST http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orders address=“USA” productId=“3001” payStatus=“Y” phoneNo=“01030000” productName=“Mac” productPrice=3000000 qty=1 userId=“last test” username=“last test”
[시나리오 2]
http PATCH http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orders/1 orderStatus=“Order Canceled”
http PATCH http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orders/3 orderStatus=“Order Canceled”
http PATCH http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orders/5 orderStatus=“Order Canceled”
[체크]
http GET http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orders
http GET http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/orderStatus
http GET http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/stockDeliveries
http GET http://aedb7e1cae2d84953b471cb6b57ed58f-1249713815.ap-southeast-1.elb.amazonaws.com:8080/promotes
```


# 동기식 호출과 Fallback 처리

(Request-Response 방식의 서비스 중심 아키텍처 구현)

- 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)

요구사항대로 배송팀에서는 쿠폰이 발행된 것을 확인한 후에 배송을 시작한다.

StockDelivery.java Entity Class에 @PostPersist로 쿠폰 발행 후에 배송을 시작하도록 처리하였다.

```
    @PostPersist
    public void onPostPersist() throws Exception{

    	Promote promote = new Promote();
        promote.setPhoneNo(this.phoneNo); 
        promote.setUserId(this.userId); 
        promote.setUsername(this.userName); 
        promote.setOrderId(this.orderId); 
        promote.setOrderStatus(this.orderStatus); 
        promote.setProductId(this.productId); 
        System.out.println("\n\npostpersist() : "+this.deliveryStatus +"\n\n");
        // deliveryStatus 따라 로직 분기
        if(DELIVERY_STARTED == this.deliveryStatus){
        	
	        boolean result = (boolean) ProductdeliveryApplication.applicationContext.getBean(food.delivery.work.external.PromoteService.class).publishCoupon(promote);
	
	        if(result){
	        	System.out.println("----------------");
	            System.out.println("Coupon Published");
	            System.out.println("----------------");
		       	DeliveryStarted deliveryStarted = new DeliveryStarted();
		        BeanUtils.copyProperties(this, deliveryStarted);
		        deliveryStarted.publishAfterCommit();
	        }else {
	        	throw new RollbackException("Failed during coupon publish");
	        }
        
        }
  
    }
    
```

##### 동기식 호출은 PromoteService 클래스를 두어 FeignClient 를 이용하여 호출하도록 하였다.

- PromoteService.java

```
  
package food.delivery.work.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

import food.delivery.work.Promote;

import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(name="marketing", url = "${api.promote.url}", fallback = PromoteServiceFallback.class)
public interface PromoteService {
  
    @RequestMapping(method=RequestMethod.POST, path="/createPromoteInfo")
    public boolean publishCoupon(@RequestBody Promote promote);
    
    @RequestMapping(method=RequestMethod.POST, path="/cancelCoupon")
    public boolean cancelCoupon(@RequestBody Promote promote);
}
```

- PromoteServiceFallback.java

```
  
package food.delivery.work.external;

import org.springframework.stereotype.Component;

import food.delivery.work.Promote;

@Component
public class PromoteServiceFallback implements PromoteService {
    @Override
    public boolean publishCoupon(Promote promote) {
        //do nothing if you want to forgive it

        System.out.println("Circuit breaker has been opened. Fallback returned instead.");
        return false;
    }
    
    @Override
    public boolean cancelCoupon(Promote promote) {
        //do nothing if you want to forgive it

        System.out.println("Circuit breaker has been opened. Fallback returned instead.");
        return false;
    }
}
```


# 비동기식 호출과 Eventual Consistency (작성완료)

(이벤트 드리븐 아키텍처)

- 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?

주문/주문취소 후에 이를 배송팀에 알려주는 트랜잭션은 Pub/Sub 관계로 구현하였다.
아래는 주문/주문취소 이벤트를 통해 kafka를 통해 배송팀 서비스에 연계받는 코드 내용이다. 

```

    @PostPersist
    public void onPostPersist(){
    	
         Logger logger = LoggerFactory.getLogger(this.getClass());

    	
        OrderPlaced orderPlaced = new OrderPlaced();
        BeanUtils.copyProperties(this, orderPlaced);
        orderPlaced.publishAfterCommit();
        System.out.println("\n\n##### OrderService : onPostPersist()" + "\n\n");
        System.out.println("\n\n##### orderplace : "+orderPlaced.toJson() + "\n\n");
        System.out.println("\n\n##### productid : "+this.productId + "\n\n");
        logger.debug("OrderService");
    }

    @PostUpdate
    public void onPostUpdate() {
    	
    	OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        orderCanceled.publishAfterCommit();
    }
```
- 배송팀에서는 주문/주문취소 접수 이벤트에 대해 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler를 구현한다. 

```
Service
public class PolicyHandler{
    @Autowired StockDeliveryRepository stockDeliveryRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderPlaced_AcceptOrder(@Payload OrderPlaced orderPlaced){

        if(!orderPlaced.validate()) return;

        // delivery 객체 생성 //
         StockDelivery delivery = new StockDelivery();

         delivery.setOrderId(orderPlaced.getId());
         delivery.setUserId(orderPlaced.getUserId());
         delivery.setOrderDate(orderPlaced.getOrderDate());
         delivery.setPhoneNo(orderPlaced.getPhoneNo());
         delivery.setProductId(orderPlaced.getProductId());
         delivery.setQty(orderPlaced.getQty()); 
         delivery.setDeliveryStatus("delivery Started");

         System.out.println("==================================");
         System.out.println(orderPlaced.getId());
         System.out.println(orderPlaced.toJson());
         System.out.println("==================================");
         System.out.println(delivery.getOrderId());

         stockDeliveryRepository.save(delivery);

    }
    
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderCanceled_CancleOrder(@Payload OrderCanceled orderCanceled) {
    	
    	if(!orderCanceled.validate()) return;
... 중략
        for (StockDelivery delivery:deliveryList)
        {
        	System.out.println("\n\n"+orderCanceled.getId());
            delivery.setDeliveryStatus("delivery Canceled");
            stockDeliveryRepository.save(delivery);
        }
     
    }

}
```


# SAGA 패턴
- 취소에 따른 보상 트랜잭션을 설계하였는가?(Saga Pattern)

상품배송팀의 기능을 수행할 수 없더라도 주문은 항상 받을 수 있게끔 설계하였다. 
다만 데이터의 원자성을 보장해주지 않기 때문에 추후 order service 에서 재고 정보를 확인한 이후에 주문수락을 진행하거나, 상품배송 서비스에서 데이터 변경전 재고 여부를 확인하여 롤백 이벤트를 보내는 로직이 필요할 것으로 판단된다. 


order 서비스가  고객으로 주문 및 결제(order and pay) 요청을 받고
[order 서비스]
Order aggegate의 값들을 추가한 이후 주문완료됨(OrderPlaced) 이벤트를 발행한다. - 첫번째 

![saga1](https://user-images.githubusercontent.com/88864433/133546289-8b2cf493-7296-4464-944a-1c112f77b500.PNG)

서비스의 트랜젝션 완료

[product delivery 서비스]

![saga2](https://user-images.githubusercontent.com/88864433/133546388-3d5da7c0-8609-4a5b-8143-270b761a7a54.PNG)

주문완료됨(OrderPlaced) 이벤트가 발행되면 상품배송 서비스에서 해당 이벤트를 확인한다.
재고배송(stockdelivery) 정보를 추가 한다. - 두번째 서비스의 트렌젝션 완료

![saga3](https://user-images.githubusercontent.com/88864433/133546519-f224c831-4a34-4360-bd79-23a5f077949e.PNG)



# CQRS
- CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

주문/배송상태가 바뀔 때마다 고객이 현재 상태를 확인할 수 있어야 한다는 요구사항에 따라 주문 서비스 내에 OrderStatus View를 모델링하였다

OrderStatus.java 
```
@Entity
@Table(name="OrderStatus_table")
public class OrderStatus {

        @Id
        @GeneratedValue(strategy=GenerationType.AUTO)
        private Long id;
        private String username;
        private String userId;
        private Long orderId;
        private String orderStatus;
        private String productId;
        private String productName;
        private Long productPrice;
        private int qty; 
        private String couponId;
        private String couponKind;
        private String couponUseYn;
.... 생략 
```

OrderStatusViewHandler 를 통해 구현

Pub/Sub 기반으로 별도 ProductPage_table 테이블에 저장되도록 구현하였다.

```
@Service
public class OrderStatusViewHandler {


    @Autowired
    private OrderStatusRepository orderStatusRepository;
    
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderPlaced_then_CREATE_1 (@Payload OrderPlaced orderPlaced) {
        try {

            if (!orderPlaced.validate()) return;

            // view 객체 생성
            OrderStatus orderStatus = new OrderStatus();
            orderStatus.setUsername(orderPlaced.getUsername());
            orderStatus.setUserId(orderPlaced.getUserId());
            orderStatus.setOrderId(orderPlaced.getId());
            orderStatus.setOrderStatus("OrderPlaced");
            orderStatus.setProductId(orderPlaced.getProductId());
            orderStatus.setProductName(orderPlaced.getProductName());
            orderStatus.setProductPrice(orderPlaced.getProductPrice());
            orderStatus.setQty(orderPlaced.getQty());
           
            orderStatusRepository.save(orderStatus);
            
            System.out.println("\n\n##### OrderStatus : whenOrderPlaced_then_CREATE_1" + "\n\n");

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

주문에 대한 결제완료(PayStatus) 시 orderId를 키값으로 OrderStatus 데이터도 생성되며 (주문과 결제를 동시에 처리했을 때 배송을 시작하므로)

"결제완료(주문완료), 주문접수, 배송시작, 결제취소(주문취소)"의 이벤트에 따라 주문상태가 업데이트되도록 모델링하였다.




- CQRS 테스트 

![CQRS](https://user-images.githubusercontent.com/88864433/133558737-0d82429e-add2-403b-9750-c1a723beeb86.PNG)




# 폴리글랏 퍼시스턴스
- 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?

메세지 발송/취소 이력의 경우 고객의 VOC가 발생할 소지가 크기 때문에 mysql DB를 사용해 영구 보관하도록 구현하였다.

- message서비스 pom.xml에 mysql 연결 설정 추가
```
...생략
                <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <scope>provided</scope>
                </dependency>

                <dependency>
                    <groupId>org.javassist</groupId>
                <artifactId>javassist</artifactId>
                <version>3.25.0-GA</version>
                </dependency>
---생략
```

- message서비스 application.yml에 mysql DB접속 정보 추가
```
...생략
spring:
  profiles: docker

  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://cloud12st.ck7n6wloicx4.ap-northeast-2.rds.amazonaws.com:3306/cloud12st
    username: root
    password: cloud#1234

  jpa:
    open-in-view: false
    show-sql: true
    hibernate:
      format_sql: true
      ddl-auto: create
...생략
```
- DB에 저장된 결과
![image](https://user-images.githubusercontent.com/60597727/135447926-eb1e3146-93d3-4644-899f-59b5293dafb5.png)


# API 게이트웨이
- API GW를 통하여 마이크로 서비스들의 진입점을 통일할 수 있는가?

- gateway서비스 application.yml에서 각 마이크로서비스의 URI PATH에 따라 접근이 가능하도록 함
```
...생략
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: productdelivery
          uri: http://productdelivery:8080
          predicates:
            - Path=/stockDeliveries/**
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/**
        - id: orderstatus
          uri: http://orderstatus:8080
          predicates:
            - Path=/orderStatus/**
        - id: marketing
          uri: http://marketing:8080
          predicates:
            - Path=/promotes/**
        - id: message
          uri: http://message:8080
          predicates:
            - Path=/messages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080
```
- message서비스 8080포트 호출 결과
![image](https://user-images.githubusercontent.com/60597727/135450138-b47f2eff-c911-446e-b5aa-d885f7157c6c.png)

# 운영
--
# Deploy/Pipeline

- deployment.yml파일을 통해 deploy와 service생성을 진행함
```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: productdelivery
  labels:
    app: productdelivery
spec:
  replicas: 1
  selector:
    matchLabels:
      app: productdelivery
  template:
    metadata:
      labels:
        app: productdelivery
    spec:
      containers:
        - name: productdelivery
          image: 879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/productdelivery:latest
          ports:
            - containerPort: 8080
...생략

apiVersion: apps/v1
kind: Deployment
metadata:
  name: order
  labels:
    app: order
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
        - name: order
          image: 879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/order:latest
          ports:
            - containerPort: 8080
...생략

apiVersion: apps/v1
kind: Deployment
metadata:
  name: marketing
  labels:
    app: marketing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: marketing
  template:
    metadata:
      labels:
        app: marketing
    spec:
      containers:
        - name: marketing
          image: 879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/marketing:latest
          ports:
            - containerPort: 8080
...생략

apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderstatus
  labels:
    app: orderstatus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orderstatus
  template:
    metadata:
      labels:
        app: orderstatus
    spec:
      containers:
        - name: orderstatus
          image: 879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/orderstatus:latest
          ports:
            - containerPort: 8080
...생략

apiVersion: apps/v1
kind: Deployment
metadata:
  name: message
  labels:
    app: message
spec:
  replicas: 1
  selector:
    matchLabels:
      app: message
  template:
    metadata:
      labels:
        app: message
    spec:
      containers:
        - name: message
          image: 879772956301.dkr.ecr.ap-southeast-1.amazonaws.com/message:latest
          ports:
            - containerPort: 8080
...생략

apiVersion: v1
kind: Service
metadata:
  name: productdelivery
  labels:
    app: productdelivery
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: productdelivery


---


apiVersion: v1
kind: Service
metadata:
  name: order
  labels:
    app: order
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: order


---


apiVersion: v1
kind: Service
metadata:
  name: marketing
  labels:
    app: marketing
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: marketing


---


apiVersion: v1
kind: Service
metadata:
  name: orderstatus
  labels:
    app: orderstatus
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: orderstatus


---


apiVersion: v1
kind: Service
metadata:
  name: message
  labels:
    app: message
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: message
```
- 배포 적용 결과
![image](https://user-images.githubusercontent.com/60597727/135547865-8b4828c0-aa4a-4190-b13e-05fb23de8452.png)


# 동기식 호출 / Circuit Breaker / 장애격리
오더 요청이 과도할 경우 서킷 브레이크를 통해 장애 격리를 하려고 한다.

- 부하테스터 siege툴을 통한 Circuit Breaker 동작 확인 : 
- 동시사용자 50명
- 30초간 실시
- marketing 서비스의 req/res 호출 후 저장전 sleep 을 진행한다.

```
siege -c50 -t30S -r10 -v --content-type "application/json" 'http://localhost:8081/stockDeliveries POST {"orderId": 1, "orderStatus": "test", "userName": "test", "qty": 10, "deliveryStatus": "delivery Started"}'
```


![ciruit1](https://user-images.githubusercontent.com/88864433/133549822-19fa0ac7-6876-4b76-b2fb-9d64e0feace3.PNG)

![circuit2](https://user-images.githubusercontent.com/88864433/133549882-3b653f1e-6c84-4abb-b073-b5cca21ddda2.PNG)

![circuit3](https://user-images.githubusercontent.com/88864433/133549892-99e332ac-18fe-4b4e-9737-b4341b66985f.PNG)

![circuit4](https://user-images.githubusercontent.com/88864433/133550076-1789913a-d545-4c18-9fc3-3afe0e03c8e2.PNG)

![circuit5](https://user-images.githubusercontent.com/88864433/133550122-22b8de48-faeb-4079-8bcf-9d6b48f5a457.PNG)



# Autoscale(HPA)
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.


![hpa1](https://user-images.githubusercontent.com/88864433/133547537-2a3d5954-305b-443e-9f06-ecd0913fdc1a.PNG)

평소에 order pod이 정상적으로 존재하던 중에

![hpa2](https://user-images.githubusercontent.com/88864433/133547635-04bbab9e-8373-4e40-94b2-6b23cadab2bb.PNG)

Autoscale 설정 명령어 실행

![hpa3](https://user-images.githubusercontent.com/88864433/133547683-607efd3d-b1a4-47fc-b3a2-3c19700de609.PNG)

Autoscale 설정됨을 확인

![hpa4](https://user-images.githubusercontent.com/88864433/133547727-9e4fb0bd-cbc9-45d5-ab08-606088272f7c.PNG)

siege 명령어를 수행 

![hpa5](https://user-images.githubusercontent.com/88864433/133547764-705a846d-c211-44b5-ae1f-bbb683fce886.PNG)

CPU 사용량이 5% 이상인 경우 POD는 최대 10개까지 늘어나는 것을 확인

![hpa6](https://user-images.githubusercontent.com/88864433/133547800-ea2c92cc-7733-4605-b58f-bc408a5c635b.PNG)

siege 가용성은 100%을 유지하고 있다.


# Zero-downtime deploy (Readiness Probe) 
(무정지 배포) 

서비스의 무정지 배포를 위하여 오더(Order) 서비스의 배포 yaml 파일에 readinessProbe 옵션을 추가하였다.

![HPA8](https://user-images.githubusercontent.com/88864433/133559651-9169b961-c0f8-47db-b8df-8b3c274bbd91.PNG)

![readness1](https://user-images.githubusercontent.com/88864433/133539552-06cc7425-1cb5-4319-b92b-c7c20d807c69.PNG)

파일의 버전이 v1을 적용하고 siege를 실행한 상태에서 v2로 배포를 진행하였다. 

![readness2](https://user-images.githubusercontent.com/88864433/133539593-37ea6cf1-ce76-4d5e-bf21-b6f3ec85079c.PNG)

서비스의 끊김없이 무정지 배포가 실행됨을 확인하였다. 


# Self-healing (Liveness Probe)

- port 및 정보를 잘못된 값으로 변경하여 yml 적용

![liveness1](https://user-images.githubusercontent.com/88864433/133550800-5c481182-5e46-4572-b5c8-738fe5356653.PNG)

- 해당 yml을 배포

![liveness2](https://user-images.githubusercontent.com/88864433/133550866-21e9ca23-9d2c-41a0-bc60-0f6a7596279f.PNG)

- 잘못된 경로와 포트여서 kubelet이 자동으로 컨테이너를 재시작하였다. 

![LIVENESS4](https://user-images.githubusercontent.com/88864433/133563189-377ef1fe-7e86-4ea6-b387-87739edcdf61.PNG)

- POD가 재시작되었다. 

![liveness3](https://user-images.githubusercontent.com/88864433/133550970-0f13cf46-7b96-4034-aeaa-c24750597973.PNG)



# 운영유연성
- 데이터 저장소를 분리하기 위한 Persistence Volume과 Persistence Volume Claim을 적절히 사용하였는가?

- kubectl apply -f efs-provisioner-deploy.yml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: efs-provisioner

---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: efs-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: efs-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
      serviceAccount: efs-provisioner
      containers:
        - name: efs-provisioner
          image: quay.io/external_storage/efs-provisioner:latest
          env:
            - name: FILE_SYSTEM_ID
              value: fs-5829ad18
            - name: AWS_REGION
              value: ap-southeast-1
            - name: PROVISIONER_NAME
              value: enyasio99.com/aws-efs
          volumeMounts:
            - name: pv-volume
              mountPath: /persistentvolumes
      volumes:
        - name: pv-volume
          nfs:
            server: fs-5829ad18.efs.ap-southeast-1.amazonaws.com
            path: /
```
- kubectl apply -f rbac.yml
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: efs-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
...생략    
```

- kubectl apply -f storageClass.yml
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: aws-efs
provisioner: enyasio99.com/aws-efs
```
- kubectl apply -f volume-pvc.yml
```
    kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: efs
  annotations:
    volume.beta.kubernetes.io/storage-class: "aws-efs"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```
- 바운드된 볼륨 상태 확인
![image](https://user-images.githubusercontent.com/60597727/135402872-28ee91fc-68f9-41bf-9d54-5c18321a1b43.png)

- message서비스의 application.yml에 로깅 설정 추가
```
...생략
logging:
  path: /logs
  file:
    max-history: 30
  level:
    org.springframework.cloud: debug
```

- 서비스 호출 후 pod내 마운트된 볼륨에서 로그 파일 생성 확인

![image](https://user-images.githubusercontent.com/60597727/135403871-375b0d48-407a-4dda-876f-e092f61d3e29.png)

