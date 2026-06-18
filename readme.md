초기 예약 시스템의 동시성 문제를 해결하기 위해 가장 먼저 시도한 방식은 자바 언어의 synchronized 키워드였습니다. meetingroom/application-api 모듈 내 예약 검증 로직에 이를 적용해 단일 인스턴스 환경에서 100개의 스레드로 테스트한 결과 데이터 정합성은 확보되었으나, 어떤 회의실을 예약하든 모든 사용자가 순서를 기다려야 하는 구조가 되어 예약 처리 속도가 현저히 느려지는 문제가 있었습니다. 이를 개선하고자 자바의 ConcurrentHashMap을 도입하여 회의실 ID별로 동기화 객체를 분리했습니다. 이 방식을 통해 A 회의실 예약자가 B 회의실 예약자의 대기를 기다릴 필요가 없게 되어 여러 회의실 환경에서 독립적인 동시 처리가 가능해졌고, 단일 JVM 내에서는 효율적인 해결책임을 확인했습니다. 그러나 포트 8081과 8082로 이중화된 분산 환경을 시뮬레이션하여 테스트한 결과, 인스턴스마다 메모리 영역이 독립적으로 존재하여 서로 다른 서버에서 동시에 같은 회의실을 예약할 경우, 각 서버가 예약 가능한 빈 좌석으로 판단해 중복 예약을 승인하는 현상을 확인했습니다.

이후 분산 환경에서의 정합성을 확보하기 위해 데이터베이스 레벨의 락 방식을 검토했습니다.

비관적 락(Pessimistic Lock)은 ReservationServiceImpl.kt의 36라인에서 roomRepository.findByIdWithLock(roomId)를 호출하여, 예약하려는 특정 회의실 데이터에 미리 잠금을 거는 방식입니다. 테스트 결과, 동일한 회의실에 예약이 몰릴 경우 뒤따르는 요청들은 앞선 트랜잭션의 작업이 끝날 때까지 대기해야 했고, 이로 인해 전체 소요 시간이 442ms~462ms까지 늘어나는 병목 현상을 측정했습니다. 이는 데이터베이스 수준에서 특정 자원을 점유하려는 트랜잭션들이 순차적으로 처리되면서 발생하는 대기 시간이 전체 시스템의 응답 속도 저하로 이어진 결과임을 직접 확인했습니다.

낙관적 락(Optimistic Lock)은 잠금을 걸지 않으므로, 충돌이 적은 본 서비스의 환경에서는 락 대기 시간이 발생하지 않아 높은 처리량을 확보할 수 있습니다. 이는 물리적으로 데이터를 잠그고 해제하는 과정에서 발생하는 데이터베이스의 대기 시간과 스레드 문맥 교환 비용이 발생하지 않기 때문입니다. 네트워크 지연을 모의하기 위해 검증과 저장 로직 사이에 Thread.sleep(5000)을 삽입해 테스트한 결과, 선행 트랜잭션이 성공하는 순간 후행 트랜잭션들은 즉시 ReservationConflictException을 발생시켰습니다. 대기 과정이 없어 전체 소요 시간은 198ms로 측정되었습니다. 저는 여기서 발생하는 예외를 실패로 간주하고 try-catch로 처리하여, 대기 없이 즉각 사용자에게 결과를 알려주어 다른 시간대나 다른 회의실을 선택하도록 유도하는 방식을 택했습니다.

저는 이 과정에서 두 가지 대안을 기각했습니다. 첫째, 메모리 기반 동기화(synchronized, ConcurrentHashMap)는 이중화 서버 환경에서 인스턴스 간 상태를 공유할 수 없어 분산 환경에서의 데이터 무결성을 보장할 수 없기에 기각했습니다. 둘째, 비관적 락은 동일 자원에 대한 트래픽 집중 시 대기로 인해 전체 처리량(Throughput)을 저하시키고 본 도메인의 확장성 요구사항과 맞지 않아 기각했습니다.

결론적으로 저는 낙관적 락을 최종 선택했습니다. 제가 낙관적 락을 선택한 결정적인 근거 중 하나는 회의실 예약 도메인의 자원 특성입니다. 본 시스템은 고정된 회의실 한 곳만을 사용하는 것이 아니라, POST 요청을 통해 회의실을 동적으로 추가할 수 있도록 설계되었습니다. 따라서 예약 요청이 여러 회의실로 분산되기 때문에 특정 자원에 대한 경합 빈도가 상대적으로 낮습니다. 이러한 환경에서는 비관적 락을 통해 유저를 대기(Block)시키며 커넥션을 점유하게 만드는 것보다, 낙관적 락을 통해 빠른 응답을 제공하는 것이 전체 Throughput 및 사용자 경험 관점에서 훨씬 적합하다고 생각합니다.




---
<img width="818" height="471" alt="Image" src="https://github.com/user-attachments/assets/5ad94c73-5c2c-4497-a656-695e46627dbb" />

<img width="796" height="363" alt="Image" src="https://github.com/user-attachments/assets/c773925e-c562-4122-9aaa-3ac3b3dbe9b5" />



<img width="806" height="318" alt="Image" src="https://github.com/user-attachments/assets/869489e5-6d6e-48a3-86da-129ec816570b" />


<img width="815" height="259" alt="Image" src="https://github.com/user-attachments/assets/a191a661-23db-4c73-9ebe-0a7c9098b261" />

<img width="803" height="563" alt="Image" src="https://github.com/user-attachments/assets/2d3579e4-1b29-4c05-8a66-a791845b8789" />


<img width="810" height="396" alt="Image" src="https://github.com/user-attachments/assets/e8c74d48-3fd4-4c6a-a536-8d663f58663d" />


<img width="810" height="663" alt="Image" src="https://github.com/user-attachments/assets/1497932e-e306-4ec4-8dca-5b177a2550ad" />

<img width="813" height="551" alt="Image" src="https://github.com/user-attachments/assets/5d8f765f-f3c0-4d3a-99a1-5fbd2d8bc291" />


<img width="797" height="288" alt="Image" src="https://github.com/user-attachments/assets/5398d7f1-4cbc-429c-8812-1e7c3a8e1273" />

<img width="808" height="635" alt="Image" src="https://github.com/user-attachments/assets/c530b66b-890e-4000-bc7c-4d7a0ce979c8" />

<img width="803" height="476" alt="Image" src="https://github.com/user-attachments/assets/0c1b27bf-81dd-4cc6-a7c8-8d19750927b2" />


<img width="800" height="346" alt="Image" src="https://github.com/user-attachments/assets/2fdb303c-c59f-431b-9e81-7b0cd96b2d52" />







