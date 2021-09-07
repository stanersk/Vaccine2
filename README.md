![image](https://user-images.githubusercontent.com/86760552/130912831-8ec7077a-8f58-4f1c-b08e-56fc51640bac.png)

# 백신 예약 시스템

### 서비스 시나리오 

백신 예약

기능적 요구사항
1. 정부관리자가 백신접종가능일자별로 접종병원, 백신수량을 등록한다.
2. 백신 등록 시 사용자에게 백신 등록 알림이 간다.
3. 예약자는 접종일자, 접종병원, 코로나 백신 종류를 선택하여 예약한다. 
4. 예약과 동시에 예약가능한 백신수량이 감소한다.
5. 예약이 되면 예약 확정 메시지가 예약자에게 전달된다.
6. 예약자는 예약을 취소할 수 있다.
7. 예약이 취소되면 백신수량이 증가하고, 취소내역 메시지가 예약자에게 전달된다. 
8. 예약자는 접종 예약정보(예약번호, 예약상태, 접종예정일자, 접종병원, 백신종류)를 조회할 수 있다.

비기능적 요구사항
1. 트랜잭션
   - 백신수량 감소가 되지 않은 건은 예약이 되지 않아야 한다.(Sync 호출)
2. 장애격리
   - 예약 시스템 과중되면 잠시 후에 하도록 유도한다.(Circuit breaker)
   - 백신 관리시스템이 문제가 있더라도 예약 취소는 받을 수 있어야 한다.(Async event-driven)
3. 성능
   - 예약정보를 한번에 확인할 수 있어야 한다.(CQRS)

# 1.Sasg

![image](https://user-images.githubusercontent.com/86760552/132290658-ae4ccd78-d94d-4c3e-a408-437215376d6f.png)

# 2. CQRS 

- Table 구조
![image](https://user-images.githubusercontent.com/86760552/132290110-0e3753eb-06c0-4382-8eaa-ae94827f72c0.png)


- viewpage MSA ViewHandler 를 통해 구현
```
@Service
public class MyPageViewHandler {


    @Autowired
    private MyPageRepository myPageRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void whenVaccineReserved_then_CREATE_1 (@Payload VaccineReserved vaccineReserved) {
        try {

            if (!vaccineReserved.validate()) return;

            // view 객체 생성
            MyPage myPage = new MyPage();
            // view 객체에 이벤트의 Value 를 set 함
            // myPage.setId(vaccineReserved.getId());
            myPage.setUserId(vaccineReserved.getUserId());
            myPage.setHospital(vaccineReserved.getHospital());
            myPage.setReservedDate(vaccineReserved.getReservedDate());
            myPage.setReservationStatus(vaccineReserved.getReservationStatus());
            myPage.setUserId(vaccineReserved.getUserId());
            myPage.setHospital(vaccineReserved.getHospital());
            myPage.setReservedDate(vaccineReserved.getReservedDate());
            myPage.setReservationStatus(vaccineReserved.getReservationStatus());
            // view 레파지 토리에 save
            myPageRepository.save(myPage);

        }catch (Exception e){
            e.printStackTrace();
        }
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void whenVaccineReserved_then_UPDATE_1(@Payload VaccineReserved vaccineReserved) {
        try {
            if (!vaccineReserved.validate()) return;
                // view 객체 조회

                    List<MyPage> myPageList = myPageRepository.findByUserId(vaccineReserved.getUserId());
                    for(MyPage myPage : myPageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setReservationStatus(vaccineReserved.getReservationStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenReservationCancelled_then_UPDATE_2(@Payload ReservationCancelled reservationCancelled) {
        try {
            if (!reservationCancelled.validate()) return;
                // view 객체 조회

                    List<MyPage> myPageList = myPageRepository.findByUserId(reservationCancelled.getUserId());
                    for(MyPage myPage : myPageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setReservationStatus(reservationCancelled.getReservationStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenReservationCancelled_then_UPDATE_3(@Payload ReservationCancelled reservationCancelled) {
        try {
            if (!reservationCancelled.validate()) return;
                // view 객체 조회

                    List<MyPage> myPageList = myPageRepository.findByUserId(reservationCancelled.getUserId());
                    for(MyPage myPage : myPageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setReservationStatus(reservationCancelled.getReservationStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```
- 실제로 view 페이지를 조회해 보면 모든 예약에 대한 전반적인 상태를 알수 있다.
![image](https://user-images.githubusercontent.com/86760552/132297054-c92b5e43-a23b-4651-8491-dc2a55195106.png)


# 3. Correlation

Vanccine 관리 프로젝트에서는 PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현을 이벤트 클래스 안의 변수로 전달받아 
서비스간 연관된 처리를 정확하게 구현하고 있습니다.

- 백신 예약
![2 백신예약](https://user-images.githubusercontent.com/86760552/131067002-f33d3330-d430-4e6d-b61d-9ead306f8ba2.PNG)

- 백신 취소
![3 백신취소](https://user-images.githubusercontent.com/86760552/131067043-e574c60c-6200-4c4a-b337-d2bdbc6b0884.PNG)






## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/86760552/130935503-58a4f6d4-7367-434f-87c6-eb7740fc07b8.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 Pub/Sub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현 

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
mvn spring-boot:run  

```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 VaccineMgmt 마이크로 서비스). 

```
package vaccinereservation;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="VaccineMgmt_table")
public class VaccineMgmt {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long reservationId;
    private Date availableDate;
    private String hospital;
    private String vaccineStatus;

    @PostPersist
    public void onPostPersist(){
        VaccineRegistered vaccineRegistered = new VaccineRegistered();
        BeanUtils.copyProperties(this, vaccineRegistered);
        vaccineRegistered.publishAfterCommit();
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getReservationId() {
        return reservationId;
    }

    public void setReservationId(Long reservationId) {
        this.reservationId = reservationId;
    }
    public Date getAvailableDate() {
        return availableDate;
    }

    public void setAvailableDate(Date availableDate) {
        this.availableDate = availableDate;
    }
    public String getHospital() {
        return hospital;
    }

    public void setHospital(String hospital) {
        this.hospital = hospital;
    }
    public String getVaccineStatus() {
        return vaccineStatus;
    }

    public void setVaccineStatus(String vaccineStatus) {
        this.vaccineStatus = vaccineStatus;
    }

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

```
package vaccinereservation;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="vaccineMgmts", path="vaccineMgmts")
public interface VaccineMgmtRepository extends PagingAndSortingRepository<VaccineMgmt, Long>{

}

```
- 적용 후 REST API 의 테스트

```

백신 등록
http post localhost:8088/vaccineMgmts id=1 qty=100 availableDate=2021-08-30 hospital=seoul

백신 예약
http post localhost:8088/reservations vaccineId=1 userId=1 reservedDate=2021-08-27 reservationStatus=“reserved”

백신 취소
http patch localhost:8088/reservations/1 reservationStatus="cancelled

```

## 폴리글랏 퍼시스턴스

별다른 작업없이 기존의 Entity Pattern 과 Repository Pattern 적용과 데이터베이스 제품의 설정 (pom.xml) 만으로 hsqldb 로 부착시켰다

```
# pom.xml - in myPage 인스턴스

		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>

```
![image](https://user-images.githubusercontent.com/86760552/131065064-33a9240f-c23e-4d18-8a4b-b893c1963c6e.png)


## 동기식 호출 과 Fallback 처리 

분석단계에서의 조건 중 하나로 백신예약->백신관리 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 백신관리 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# VaccineMgmtService.java

package vaccinereservation.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.Date;

@FeignClient(name="vaccine", url="http://localhost:8088")
public interface VaccineMgmtService {
    @RequestMapping(method= RequestMethod.PUT, path="/vaccineMgmts/updateVaccine")
    public void updateVaccine(@RequestParam("vaccineId") long vaccineId);

}

```

- 예약을 받은 직후(@PostPersist) 백신 확보 및 예약 처리를 하도록 설계
```
# VaccineMgmt.java

    @PostPersist
    public void onPostPersist(){
        VaccineRegistered vaccineRegistered = new VaccineRegistered();
        vaccineRegistered.setId(this.getId());
        vaccineRegistered.setUserId(this.getUserId());
        vaccineRegistered.setHospital(this.getHospital());
        vaccineRegistered.setAvailableDate(this.getAvailableDate());        
        // vaccineRegistered.setQty(this.getQty());
        vaccineRegistered.setVaccineStatus("registered");

        BeanUtils.copyProperties(this, vaccineRegistered);
        vaccineRegistered.publishAfterCommit();

    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 백신관리 시스템이 장애가 나면 예약을 못받는다는 것을 확인:


- 백신 서비스 다운
![1  백신서비스다운](https://user-images.githubusercontent.com/86760552/131067753-bb9323ea-31ee-4ab7-9475-c78f994e450f.PNG)

- 백신 예약 - 에러
![2  백신예약 실패](https://user-images.githubusercontent.com/86760552/131067772-00eca8c2-1dbd-4cc6-8dd4-cf16cb60df48.PNG)

- 백신 서비스 개시
![3  백신재기동완료](https://user-images.githubusercontent.com/86760552/131067806-53bad80e-f4d9-427e-b829-7a2979f0a468.PNG)

- 백신 예약 - 성공
![4 백신예약완료](https://user-images.githubusercontent.com/86760552/131067855-6e7c34e0-e41e-4725-a9b8-6687ab33a8a4.PNG)


- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트 


백신 Notification 정보 관리는 비동기식으로 처리하여 백신관리 시스템의 처리에 영향을 주지 않도록 블로킹 처리가 되지 않도록 처리한다.
 
- 이를 위하여 예약 정보의 생성/변경의 기록을 남긴 후에 곧바로 생성/변경 되었다고 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package vaccinereservation;

@Entity
@Table(name="Reservation_table")
public class Reservation {

 ...
    @PostUpdate
    public void onPostUpdate() {
        final ReservationCancelled reservationCancelled = new ReservationCancelled();
        reservationCancelled.setId(this.getId());
        reservationCancelled.setUserId(this.getUserId());
        reservationCancelled.setHospital(this.getHospital());
        reservationCancelled.setReservedDate(this.getReservedDate());
        reservationCancelled.setReservationStatus("reserved");

        BeanUtils.copyProperties(this, reservationCancelled);
        reservationCancelled.publishAfterCommit();
    }
```
- 백신 관리 서비스에서는 예약 취소 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
public class PolicyHandler{
    @Autowired NotificationRepository notificationRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverVaccineRegistered_SendSms(@Payload VaccineRegistered vaccineRegistered){

        // if(!vaccineRegistered.validate()) return;

        System.out.println("\n\n##### listener SendSms : " + vaccineRegistered.toJson() + "\n\n");

        if(vaccineRegistered.validate()){

            /////////////////////////////////////////////
            // 취소 요청이 왔을 때 -> status -> cancelled 
            /////////////////////////////////////////////
            System.out.println("##### listener vaccineRegistered : " + vaccineRegistered.toJson());
            Notification noti = new Notification();
            // 취소시킬 Id 추출
            // long id = vaccineRegistered.getId(); // 취소시킬 Id

            // Optional<Notification> res = notificationRepository.findById(id);
            // Notification noti = res.get();

            noti.setUserId(vaccineRegistered.getUserId());
            noti.setMessage("관리자에 의해 백신이 등록되었습니다.");
            noti.setVaccineStatus("registered"); 

            // DB Update
            notificationRepository.save(noti);
        }
```
Sample Test 

백신 예약 시스템은 백신취소와 메시지 전송이 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, notification이 유지보수로 인해 잠시 내려간 상태라도 이벤트 전송받는데 문제가 없다.

- notification 서비시를 잠시 내려놓음 (ctrl+c)
![notification다운](https://user-images.githubusercontent.com/86760552/131071546-679b2e04-26cb-4fb5-b01f-b7a9a1c88c62.PNG)

- 예약 취소 처리 정상
![백신예약 취소](https://user-images.githubusercontent.com/86760552/131071595-3b4525cb-a645-47d3-97b2-bb75b2e30aad.PNG)

- notification 가동
![notification 서비스기동](https://user-images.githubusercontent.com/86760552/131071637-f5e6180a-9afc-401f-84d6-45a463fe6f9c.PNG)

- 예약 변경 확인
![정상적으로 취소알람발생 확인가능](https://user-images.githubusercontent.com/86760552/131071705-52474794-1ec6-48a1-8b47-2820135b6b12.PNG)


## CQRS

- Table 구조

![스크린샷 2021-08-27 오전 11 55 18](https://user-images.githubusercontent.com/86760552/131065313-35e846d8-e5c6-42fd-a3c0-c57660e0de88.png)

- viewpage MSA ViewHandler 를 통해 구현
```
@Service
public class MyPageViewHandler {


    @Autowired
    private MyPageRepository myPageRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void whenVaccineReserved_then_CREATE_1 (@Payload VaccineReserved vaccineReserved) {
        try {

            if (!vaccineReserved.validate()) return;

            // view 객체 생성
            MyPage myPage = new MyPage();
            // view 객체에 이벤트의 Value 를 set 함
            // myPage.setId(vaccineReserved.getId());
            myPage.setUserId(vaccineReserved.getUserId());
            myPage.setHospital(vaccineReserved.getHospital());
            myPage.setReservedDate(vaccineReserved.getReservedDate());
            myPage.setReservationStatus(vaccineReserved.getReservationStatus());
            myPage.setUserId(vaccineReserved.getUserId());
            myPage.setHospital(vaccineReserved.getHospital());
            myPage.setReservedDate(vaccineReserved.getReservedDate());
            myPage.setReservationStatus(vaccineReserved.getReservationStatus());
            // view 레파지 토리에 save
            myPageRepository.save(myPage);

        }catch (Exception e){
            e.printStackTrace();
        }
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void whenVaccineReserved_then_UPDATE_1(@Payload VaccineReserved vaccineReserved) {
        try {
            if (!vaccineReserved.validate()) return;
                // view 객체 조회

                    List<MyPage> myPageList = myPageRepository.findByUserId(vaccineReserved.getUserId());
                    for(MyPage myPage : myPageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setReservationStatus(vaccineReserved.getReservationStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenReservationCancelled_then_UPDATE_2(@Payload ReservationCancelled reservationCancelled) {
        try {
            if (!reservationCancelled.validate()) return;
                // view 객체 조회

                    List<MyPage> myPageList = myPageRepository.findByUserId(reservationCancelled.getUserId());
                    for(MyPage myPage : myPageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setReservationStatus(reservationCancelled.getReservationStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenReservationCancelled_then_UPDATE_3(@Payload ReservationCancelled reservationCancelled) {
        try {
            if (!reservationCancelled.validate()) return;
                // view 객체 조회

                    List<MyPage> myPageList = myPageRepository.findByUserId(reservationCancelled.getUserId());
                    for(MyPage myPage : myPageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setReservationStatus(reservationCancelled.getReservationStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
                }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```
- 실제로 view 페이지를 조회해 보면 모든 예약에 대한 전반적인 상태를 알수 있다.
![5  마이페이지 조회](https://user-images.githubusercontent.com/86760552/131079768-68df7fc5-a423-42c5-a1ac-751123c72714.PNG)

'''

## Correlation
Vanccine 관리 프로젝트에서는 PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현을 이벤트 클래스 안의 변수로 전달받아 
서비스간 연관된 처리를 정확하게 구현하고 있습니다.

- 백신 예약
![2 백신예약](https://user-images.githubusercontent.com/86760552/131067002-f33d3330-d430-4e6d-b61d-9ead306f8ba2.PNG)

- 백신 취소
![3 백신취소](https://user-images.githubusercontent.com/86760552/131067043-e574c60c-6200-4c4a-b337-d2bdbc6b0884.PNG)

## API 게이트웨이
```
  1. gateway 스프링부트 App을 추가 후 application.yaml내에 각 마이크로 서비스의 routes 를 추가하고 gateway 서버의 포트를 8080 으로 설정함
   
      - application.yaml 예시
        spring:
          profiles: docker
          cloud:
            gateway:
              routes:
                - id: payment
                  uri: http://payment:8080
                  predicates:
                    - Path=/payments/** 
                - id: room
                  uri: http://room:8080
                  predicates:
                    - Path=/rooms/**, /reviews/**, /check/**
                - id: reservation
                  uri: http://reservation:8080
                  predicates:
                    - Path=/reservations/**
                - id: message
                  uri: http://message:8080
                  predicates:
                    - Path=/messages/** 
                - id: viewpage
                  uri: http://viewpage:8080
                  predicates:
                    - Path= /roomviews/**
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

  2. Kubernetes용 Deployment.yaml 을 작성하고 Kubernetes에 Deploy를 생성함
      - Deployment.yaml 예시
      
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: gateway
	  labels:
	    app: gateway
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      app: gateway
	  template:
	    metadata:
	      labels:
		app: gateway
	    spec:
	      containers:
		- name: gateway
		  image: user09acr.azurecr.io/gateway:latest
		  ports:
		    - containerPort: 8080

  3. Kubernetes용 Service.yaml을 작성하고 Kubernetes에 Service/LoadBalancer을 생성하여 Gateway 엔드포인트를 확인함. 
      - Service.yaml 예시
      
	apiVersion: v1
	kind: Service
	metadata:
	  name: gateway
	  labels:
	    app: gateway
	spec:
	  ports:
	    - port: 8080
	      targetPort: 8080
	  selector:
	    app: gateway
	  type:
	    LoadBalancer         
     
        Service 생성
        kubectl apply -f service.yaml      
```
Gateway Loadbal 확인
![gateway_LB](https://user-images.githubusercontent.com/86760552/131075921-affd92fb-b9e8-43ed-9530-e62c9eaba94e.jpg)



# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였음.


- 도커 이미지


![도커 이미지](https://user-images.githubusercontent.com/86760552/131076024-b138926d-43b3-4ffe-9cf3-61b935d3bc6e.png)


- Azure Portal


![azure_potal](https://user-images.githubusercontent.com/86760552/131076080-9043917d-d1cc-4b8e-bdd0-a69157bf2e68.PNG)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 백신예약 요청--> 예약관리 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(백신관리)의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# Reservation.java 

    @PrePersist
    public void onPrePersist(){ 
        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
$ siege -c100 -t60S -v --content-type "application/json" 'http://localhost:8081/reservations POST {"vaccineId": 1}'

```
![부하테스트2](https://user-images.githubusercontent.com/86760552/131077990-b536e8dd-2254-43bf-8a13-0f7f2e71f21f.png)


- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 91.55% 가 성공한 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy reservations --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 1분 동안 걸어준다.
```
siege -c100 -t60S -v --content-type "application/json" 'http://localhost:8081/reservations POST {"vaccineId": 1}'

```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy reservation -w

```
- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![autoscale 결과](https://user-images.githubusercontent.com/86760552/131078744-4ba84440-e04b-4727-87b7-d3fac60597ce.png)


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t60S -v --content-type "application/json" 'http://localhost:8081/reservations POST {"vaccineId": 1}'


HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/reservations
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/reservations
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/reservations
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/reservations
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        1278 hits
Availability:		       81.45 %
Elapsed time:		       60 secs
Data transferred:	        0.32 MB
Response time:		        5.60 secs
Transaction rate:	       16.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
배포기간중 Availability 가 평소 100%에서 80% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        1278 hits
Availability:		       100 %
Elapsed time:		        60 secs
Data transferred:	        0.34 MB
Response time:		        5.40 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

## Config Map

1: cofingmap.yml 파일 생성
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: reservation
data:
  API_URL_VACCINE: "http://vaccine:8080"

```
2: deployment.yml에 적용하기
```
spec:
containers:
- name: reservation
  image: user09acr.azurecr.io/reservation:latest
  ports:
    - containerPort: 8080
  envFrom: 
    - configMapRef:
	name: reservation
```
- 실제 모습

![configmap_kubectl](https://user-images.githubusercontent.com/86760552/131078470-747d7f86-f066-416b-ad54-59c808fb6181.jpg)

## Persistent Volume

1. persistent volume claim 생성 
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vaccine-pvc
  labels:
    app: vaccine-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 2Ki
  storageClassName: azurefile

```
2. deployment 에적용

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vaccine
  labels:
    app: vaccine
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vaccine
  template:
    metadata:
      labels:
        app: vaccine
    spec:
      containers:
        - name: vaccine
          image: user09acr.azurecr.io/vaccine:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
          volumeMounts:
            - name: volume
              mountPath: "/apps/data"
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: vaccine-pvc
```
3. A pod에서 파일을 올리고 B pod 에서 확인

```
kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
gateway-7cbb87bc9f-6zj5m        1/1     Running   0          3h13m
mypage-77dcbcb79b-q8n6b         1/1     Running   0          3h8m
notification-5c5b768898-8bg65   1/1     Running   0          3h7m
reservation-6b86b764bf-rmcwg    1/1     Running   0          18m
siege                           1/1     Running   0          124m
vaccine-5b54ccfc8-nf5lh         1/1     Running   0          56s
vaccine-fc887689b-5mjjd         1/1     Running   0          56s

kubectl exec -it vaccine-5b54ccfc8-nf5lh /bin/sh
/ # cd /apps/data
/apps/data # touch intensive_course_work

```
![pvc 파일 업로드 결과](https://user-images.githubusercontent.com/86760552/131083001-8fb955f3-f0ea-4272-a64f-0b4e3c678710.png)

