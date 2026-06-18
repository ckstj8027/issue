
---
 1 동시성 고려 안했을경우 

 스레드 10개
 
<img width="185" height="102" alt="image" src="https://github.com/user-attachments/assets/295eec16-78b4-46f5-ac92-57b9cb19dcff" />

 스레드 100개
 
<img width="183" height="87" alt="image" src="https://github.com/user-attachments/assets/702eacc3-e441-41d0-9c5c-59b77a21cca2" />


 방이 2개 이상일 경우 
 
 <img width="208" height="95" alt="image" src="https://github.com/user-attachments/assets/cc1b2436-fc11-4558-8b21-b0f00a2c6e55" />

---
2 synchronized를 활용해서 

멀티스레드 환경에서 공유 자원을 보호할 경우

 스레드 10개
 
<img width="184" height="98" alt="image" src="https://github.com/user-attachments/assets/abbef1a3-100e-4114-a784-0b101bcb2951" />

스레드 100개 일경우 

<img width="188" height="94" alt="image" src="https://github.com/user-attachments/assets/b92a85c5-ad29-48f1-934d-894975764907" />

방이 2개 이상일 경우

<img width="206" height="90" alt="image" src="https://github.com/user-attachments/assets/061b41c2-f05e-4669-8d21-372d897093ae" />



---


3 curcurrentmap

스레드 100개 일경우 

<img width="189" height="89" alt="image" src="https://github.com/user-attachments/assets/ec23ae6d-21ac-43e7-b777-e2819da19cc3" />

방이 2개 이상일 경우

<img width="204" height="90" alt="image" src="https://github.com/user-attachments/assets/3f11ad60-1348-429e-a88e-3f7816ac4256" />



---
4 낙관락

단일 서버 일때 테스트 결과 

<img width="187" height="90" alt="image" src="https://github.com/user-attachments/assets/6ba39c9a-1960-485e-b32d-cfcee3988a9b" />


<img width="199" height="87" alt="image" src="https://github.com/user-attachments/assets/94b8c595-b259-45cc-868d-3ef0a61c5371" />

낙관적 락은 데이터 원자성을 보장하는 데이터베이스의 버전 관리 메커니즘에 의존하므로, 

분산 서버 에서도 동시성 제어가 가능할 것이라 판단하여 아키텍처적 검증을 시작했습니다.


---


## 문제 제기 및 배경

멀티 인스턴스 환경에서도 예약 시스템의 정합성을 보장하기 위해 **낙관적 락을** 도입했습니다.

동시에 트래픽이 몰릴 때 낙관적 락이 정상적으로 트랜잭션을 튕겨내는지 검증하고자 합니다.

## 1차 시도: 단일 통합 테스트 코드 내 객체 직접 생성

스프링에서 빈은 싱글톤  객체임으로 **메모리상에 존재하는 단 하나의 ReservationService 인스턴스**를 공유해서 사용할 수 없었습니다.

따라서 val로 직접 선언해서 테스트를해봤습니다.

동일한 DB 레포지토리를 공유하되 메모리만 격리된 두 개의 서버를 시뮬레이션하기 위해, 통합 테스트 코드 내부에서 서비스 인스턴스를 직접 생성하여 테스트코드를 작성했습니다.




<img width="528" height="92" alt="image" src="https://github.com/user-attachments/assets/0688c823-0beb-4798-9c5f-8d9230442100" />



의도대로 잘 작동했으나 실제 db에는  20개로의도와 다르게 모든 경합 시도가 저장됩니다 .

실제로도 로그를보면 버전이 다르면  OptimisticLockingFailureException 예외를 던지며,

콘솔에도 예상한 숫자가 찍힙니다.

 하지만 빈이아닌 객체를 직접 생성 했기에 Transactional이 안걸려있어 예외터져도 롤백하지 않습니다.




## 2차시도: 다중 실행환경 구축

db까지 확실하기위해서 테스트가아닌 실행환경에서 구축해봅시다.

./gradlew :application-api:bootRun --args='--server.port=8081'

./gradlew :application-api:bootRun --args='--server.port=8082'


이렇게 포트를 다르게 실행해도 testcontainer가 각각의 임시 db 를 만들기 때문에 

다중 인스턴스 환경이라고 볼 수 없었습니다. 

찾아본 결과 컨테이너가 이미 있으면 공유가되는 옵션이있어서 활성화시켰습니다

<img width="284" height="94" alt="image" src="https://github.com/user-attachments/assets/e9d90a89-1270-4224-8558-b442b2a22187" />


이제 다중 인스턴스 환경 세팅은 완료 되었습니다. 


---

## TEST과정

.http 파일을 만들어서 테스트해봤습니다.


<img width="328" height="444" alt="image" src="https://github.com/user-attachments/assets/21667ee6-2de1-49fa-9828-539765608f2f" />


응답으로 에러코드가 왔고 

낙관락이 동시저장을 잘 막고있구나 라고 생각하고 서버 로그를 살펴봤는데 

하지만 비지니스로직 검증 에서 예외를 뱉고 있었습니다.


<img width="390" height="35" alt="image" src="https://github.com/user-attachments/assets/591e440b-76aa-4136-9e6e-b4df76c402e7" />



코드를 살펴봅니다.


<img width="541" height="489" alt="image" src="https://github.com/user-attachments/assets/5759d371-926e-419d-abd1-0c33be97716c" />




따라서 저 1번과 2번 사이 Thread.sleep(5000) 를 두어서 네트워크 지연 상황을 강제하였습다.

즉 두 트랜잭션이 같은 시점의 데이터를 보고 경쟁하도록 만들었습니다.

다시 테스트 해봅니다. 


<img width="553" height="178" alt="image" src="https://github.com/user-attachments/assets/3d7d1442-d6a0-4bfa-94d1-b6c2f3f2689d" />




<img width="563" height="35" alt="image" src="https://github.com/user-attachments/assets/685356f7-1595-4407-881e-51d450774ea5" />


낙관적 락에 의해 예외가 발생하는 것을 관측할 수 있었습니다.



---

결과의 신뢰성을 확보하기 위해 반복 테스트를을 수행했습니다.

방이 6개가 있다고 가정하에   

유저 1과 유저 101이 경합을 시켰습니다.

대부분의 경합은 유저 1이 승리한 모습입니다. 

(사실 마지막 시도는 스크립트의 순서가 유저1이 좀더 앞섬 6번째는 순서바꿔서 날림) 

여기서 reservation의 id가 모두 홀수 인 모습인데 

짝수는 낙관적 락에 의해 패배하여 롤백된 유저의 예약 시도입니다.



<img width="558" height="308" alt="image" src="https://github.com/user-attachments/assets/3ac582db-f2b8-422c-bac5-12f4105fdb80" />



또한 

좌석예약은 한 번의 경합으로 승패가 결정되기 때문에 retry 로직은 발생 시키니 않았습니다.


















