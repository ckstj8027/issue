초기 예약 시스템의 동시성 문제를 해결하기 위해 가장 먼저 시도한 방식은 자바 언어의 synchronized 키워드였습니다. meetingroom/application-api 모듈 내 예약 검증 로직에 이를 적용해 단일 인스턴스 환경에서 100개의 스레드로 테스트한 결과 데이터 정합성은 확보되었으나, 어떤 회의실을 예약하든 모든 사용자가 순서를 기다려야 하는 구조가 되어 예약 처리 속도가 현저히 느려지는 문제가 있었습니다. 이를 개선하고자 자바의 ConcurrentHashMap을 도입하여 회의실 ID별로 동기화 객체를 분리했습니다. 이 방식을 통해 A 회의실 예약자가 B 회의실 예약자의 대기를 기다릴 필요가 없게 되어 여러 회의실 환경에서 독립적인 동시 처리가 가능해졌고, 단일 JVM 내에서는 효율적인 해결책임을 확인했습니다. 그러나 포트 8081과 8082로 이중화된 분산 환경을 시뮬레이션하여 테스트한 결과, 인스턴스마다 메모리 영역이 독립적으로 존재하여 서로 다른 서버에서 동시에 같은 회의실을 예약할 경우, 각 서버가 예약 가능한 빈 좌석으로 판단해 중복 예약을 승인하는 현상을 확인했습니다.

이후 분산 환경에서의 정합성을 확보하기 위해 데이터베이스 레벨의 락 방식을 검토했습니다.

비관적 락(Pessimistic Lock)은 ReservationServiceImpl.kt의 36라인에서 roomRepository.findByIdWithLock(roomId)를 호출하여, 예약하려는 특정 회의실 데이터에 미리 잠금을 거는 방식입니다. 테스트 결과, 동일한 회의실에 예약이 몰릴 경우 뒤따르는 요청들은 앞선 트랜잭션의 작업이 끝날 때까지 대기해야 했고, 이로 인해 전체 소요 시간이 442ms~462ms까지 늘어나는 병목 현상을 측정했습니다. 이는 데이터베이스 수준에서 특정 자원을 점유하려는 트랜잭션들이 순차적으로 처리되면서 발생하는 대기 시간이 전체 시스템의 응답 속도 저하로 이어진 결과임을 직접 확인했습니다.

낙관적 락(Optimistic Lock)은 잠금을 걸지 않으므로, 충돌이 적은 본 서비스의 환경에서는 락 대기 시간이 발생하지 않아 높은 처리량을 확보할 수 있습니다. 이는 물리적으로 데이터를 잠그고 해제하는 과정에서 발생하는 데이터베이스의 대기 시간과 스레드 문맥 교환 비용이 발생하지 않기 때문입니다. 네트워크 지연을 모의하기 위해 검증과 저장 로직 사이에 Thread.sleep(5000)을 삽입해 테스트한 결과, 선행 트랜잭션이 성공하는 순간 후행 트랜잭션들은 즉시 ReservationConflictException을 발생시켰습니다. 대기 과정이 없어 전체 소요 시간은 198ms로 측정되었습니다. 저는 여기서 발생하는 예외를 실패로 간주하고 try-catch로 처리하여, 대기 없이 즉각 사용자에게 결과를 알려주어 다른 시간대나 다른 회의실을 선택하도록 유도하는 방식을 택했습니다.

저는 이 과정에서 두 가지 대안을 기각했습니다. 첫째, 메모리 기반 동기화(synchronized, ConcurrentHashMap)는 이중화 서버 환경에서 인스턴스 간 상태를 공유할 수 없어 분산 환경에서의 데이터 무결성을 보장할 수 없기에 기각했습니다. 둘째, 비관적 락은 동일 자원에 대한 트래픽 집중 시 대기로 인해 전체 처리량(Throughput)을 저하시키고 본 도메인의 확장성 요구사항과 맞지 않아 기각했습니다.

결론적으로 저는 낙관적 락을 최종 선택했습니다. 제가 낙관적 락을 선택한 결정적인 근거 중 하나는 회의실 예약 도메인의 자원 특성입니다. 본 시스템은 고정된 회의실 한 곳만을 사용하는 것이 아니라, POST 요청을 통해 회의실을 동적으로 추가할 수 있도록 설계되었습니다. 따라서 예약 요청이 여러 회의실로 분산되기 때문에 특정 자원에 대한 경합 빈도가 상대적으로 낮습니다. 이러한 환경에서는 비관적 락을 통해 유저를 대기(Block)시키며 커넥션을 점유하게 만드는 것보다, 낙관적 락을 통해 빠른 응답을 제공하는 것이 전체 Throughput 및 사용자 경험 관점에서 훨씬 적합하다고 생각합니다.




---
<img width="905" height="524" alt="Image" src="https://github.com/user-attachments/assets/f8fa6a5d-fd8d-4389-bc36-19efe688519a" />

<img width="904" height="417" alt="Image" src="https://github.com/user-attachments/assets/0d555212-5e2b-410b-9e0f-b37dddcbb28f" />

<img width="901" height="368" alt="Image" src="https://github.com/user-attachments/assets/7a0fa85f-fbbd-48d5-b61f-598052fa5961" />

<img width="902" height="293" alt="Image" src="https://github.com/user-attachments/assets/4096e328-3d2e-4c95-a140-5e5609d20afa" />

<img width="901" height="648" alt="Image" src="https://github.com/user-attachments/assets/6677ab7a-74b7-42c6-b7cc-c80675ce8350" />


<img width="904" height="453" alt="Image" src="https://github.com/user-attachments/assets/a77adf31-2447-4236-b027-6994b74a93e4" />

<img width="903" height="734" alt="Image" src="https://github.com/user-attachments/assets/9cfe3820-19e3-480d-a14b-65d592a517c5" />

<img width="911" height="616" alt="Image" src="https://github.com/user-attachments/assets/242e3635-b558-49ba-89a1-df7fca53625c" />

<img width="906" height="325" alt="Image" src="https://github.com/user-attachments/assets/9931f630-610b-41fb-ab6f-a7a4b1a89a63" />

<img width="905" height="713" alt="Image" src="https://github.com/user-attachments/assets/d41d3e25-1521-494e-8045-07afef9b6979" />

<img width="811" height="795" alt="Image" src="https://github.com/user-attachments/assets/90b4a420-5bca-48c0-8473-6e7cdf18a824" />








