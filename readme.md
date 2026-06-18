초기 예약 시스템의 동시성 문제를 해결하기 위해 가장 먼저 시도한 방식은 자바 언어의 synchronized 키워드였습니다. meetingroom/application-api 모듈 내 예약 검증 로직에 이를 적용해 단일 인스턴스 환경에서 100개의 스레드로 테스트한 결과 데이터 정합성은 확보되었으나, 어떤 회의실을 예약하든 모든 사용자가 순서를 기다려야 하는 구조가 되어 예약 처리 속도가 현저히 느려지는 문제가 있었습니다. 이를 개선하고자 자바의 ConcurrentHashMap을 도입하여 회의실 ID별로 동기화 객체를 분리했습니다. 이 방식을 통해 A 회의실 예약자가 B 회의실 예약자의 대기를 기다릴 필요가 없게 되어 여러 회의실 환경에서 독립적인 동시 처리가 가능해졌고, 단일 JVM 내에서는 효율적인 해결책임을 확인했습니다. 그러나 포트 8081과 8082로 이중화된 분산 환경을 시뮬레이션하여 테스트한 결과, 인스턴스마다 메모리 영역이 독립적으로 존재하여 서로 다른 서버에서 동시에 같은 회의실을 예약할 경우, 각 서버가 예약 가능한 빈 좌석으로 판단해 중복 예약을 승인하는 현상을 확인했습니다.

이후 분산 환경에서의 정합성을 확보하기 위해 데이터베이스 레벨의 락 방식을 검토했습니다.

비관적 락(Pessimistic Lock)은 ReservationServiceImpl.kt의 36라인에서 roomRepository.findByIdWithLock(roomId)를 호출하여, 예약하려는 특정 회의실 데이터에 미리 잠금을 거는 방식입니다. 테스트 결과, 동일한 회의실에 예약이 몰릴 경우 뒤따르는 요청들은 앞선 트랜잭션의 작업이 끝날 때까지 대기해야 했고, 이로 인해 전체 소요 시간이 442ms~462ms까지 늘어나는 병목 현상을 측정했습니다. 이는 데이터베이스 수준에서 특정 자원을 점유하려는 트랜잭션들이 순차적으로 처리되면서 발생하는 대기 시간이 전체 시스템의 응답 속도 저하로 이어진 결과임을 직접 확인했습니다.

낙관적 락(Optimistic Lock)은 잠금을 걸지 않으므로, 충돌이 적은 본 서비스의 환경에서는 락 대기 시간이 발생하지 않아 높은 처리량을 확보할 수 있습니다. 이는 물리적으로 데이터를 잠그고 해제하는 과정에서 발생하는 데이터베이스의 대기 시간과 스레드 문맥 교환 비용이 발생하지 않기 때문입니다. 네트워크 지연을 모의하기 위해 검증과 저장 로직 사이에 Thread.sleep(5000)을 삽입해 테스트한 결과, 선행 트랜잭션이 성공하는 순간 후행 트랜잭션들은 즉시 ReservationConflictException을 발생시켰습니다. 대기 과정이 없어 전체 소요 시간은 198ms로 측정되었습니다. 저는 여기서 발생하는 예외를 실패로 간주하고 try-catch로 처리하여, 대기 없이 즉각 사용자에게 결과를 알려주어 다른 시간대나 다른 회의실을 선택하도록 유도하는 방식을 택했습니다.

저는 이 과정에서 두 가지 대안을 기각했습니다. 첫째, 메모리 기반 동기화(synchronized, ConcurrentHashMap)는 이중화 서버 환경에서 인스턴스 간 상태를 공유할 수 없어 분산 환경에서의 데이터 무결성을 보장할 수 없기에 기각했습니다. 둘째, 비관적 락은 동일 자원에 대한 트래픽 집중 시 대기로 인해 전체 처리량(Throughput)을 저하시키고 본 도메인의 확장성 요구사항과 맞지 않아 기각했습니다.

결론적으로 저는 낙관적 락을 최종 선택했습니다. 제가 낙관적 락을 선택한 결정적인 근거 중 하나는 회의실 예약 도메인의 자원 특성입니다. 본 시스템은 고정된 회의실 한 곳만을 사용하는 것이 아니라, POST 요청을 통해 회의실을 동적으로 추가할 수 있도록 설계되었습니다. 따라서 예약 요청이 여러 회의실로 분산되기 때문에 특정 자원에 대한 경합 빈도가 상대적으로 낮습니다. 이러한 환경에서는 비관적 락을 통해 유저를 대기(Block)시키며 커넥션을 점유하게 만드는 것보다, 낙관적 락을 통해 빠른 응답을 제공하는 것이 전체 Throughput 및 사용자 경험 관점에서 훨씬 적합하다고 생각합니다.




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



---
비관락 테스트 결과 첨부


<img width="759" height="814" alt="image" src="https://github.com/user-attachments/assets/eb7f8f01-291d-46a4-9c01-7651b9f596bc" />













