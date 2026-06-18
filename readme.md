회의실 좌석 예약 서비스에서는 동시성 처리를 고려해야 합니다. 
좌석 예매 오픈 시 여러 사용자가 동시에 예약을 시도하면, 자원이 안전하게 관리되지 않을 경우 예약이 초과되거나 데이터 일관성이 깨질 위험이 있습니다. 
코드 적용 및 테스트를 통해 각 동시성 처리 방식의 장단점을 검토하고, 최종적으로 처리 방식을 선정해보겠습니다.


먼저 ~ 하겠습니다 .


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
