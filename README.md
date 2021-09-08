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
```
- 실제로 view 페이지를 조회해 보면 모든 예약에 대한 전반적인 상태를 알수 있다.
![image](https://user-images.githubusercontent.com/86760552/132297054-c92b5e43-a23b-4651-8491-dc2a55195106.png)


# 3. Correlation

Vanccine 관리 프로젝트에서는 PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현을 이벤트 클래스 안의 변수로 전달받아 
서비스간 연관된 처리를 정확하게 구현하고 있습니다.

- 백신의 예약과 취소
![image](https://user-images.githubusercontent.com/86760552/132299046-18b2cb6f-a82b-4e0e-a5ad-dd7e0f8de84c.png)


# 4. Req / Resp

분석단계에서의 조건 중 하나로 백신예약->백신관리 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리함. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

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
![image](https://user-images.githubusercontent.com/86760552/132301300-47e538e2-2fa5-4746-9c12-22a4a2fc5e07.png)
- 백신 예약 에러
![image](https://user-images.githubusercontent.com/86760552/132301502-bcd9d90a-94ec-455d-8880-c4194f27d68e.png)
- 백신 서비스 개시
![image](https://user-images.githubusercontent.com/86760552/132309153-f83e5c7e-456f-41af-8f88-58bc0b23744d.png)
- 백신 예약 정상 처리
![image](https://user-images.githubusercontent.com/86760552/132309581-fd7d44e6-698d-4e21-aca8-9d65ef6fc2c1.png)


# 5. Gateway

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
![image](https://user-images.githubusercontent.com/86760552/132352995-4138756b-5acb-4a25-a251-63582a8a991b.png)


# 6. Deploy / Pipeline

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였음.

- 도커 이미지
![image](https://user-images.githubusercontent.com/86760552/132330754-ec7ca94c-e77a-4186-91c8-e962db62e74c.png)

- azure 이미지
![image](https://user-images.githubusercontent.com/86760552/132330848-ed0ea38a-2141-47df-a69e-912f393659ab.png)

# 7. Circuit Breaker

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
      execution.isolation.thread.timeoutInMilliseconds: 600

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
- 처리 결과

![image](https://user-images.githubusercontent.com/86760552/132433517-c97affa6-d119-40f0-ba40-c608fddac0d2.png)

# 8. Autoscale (HPA)

# 9. Zero-downtime deploy (readiness probe)

# 10. ConfigMap/Persistence Volume

# 11. Polyglot

# 12. Self-healing (liveness probe)
