# tcl_gateway

<h3>서비스 시나리오</h3>

  > <h4>기능적 요구사항</h4>
  1. 고객이 주문을 하면 주문정보를 바탕으로 요리가 시작된다.
  2. 요리가 완료되면 배달이 시작된다. 
  3. 고객이 주문취소를 하게 되면 요리가 취소된다.
  4. 고객이 주문 시 제고가 없을 경우 주문이 취소된다. 
  5. 고객은 Mypage를 통해, 주문과 요리, 배달의 전체 상황을 조회할수 있다.


  > <h4>비기능적 요구사항</h4>
  1. 트랜잭션
    : 주문 취소시 요리취소가 함께 처리되도록 한다. Sync 호출(Saga Pattern).
  2. 장애격리
    : 주문 시스템에 장애가 발생하면 주문은 잠시뒤에 처리될 수 있도록 한다(Circuit breaker).
  3. 성능 
    : 고객이 mypage를 통해 주문과 요리, 배달의 전체 상황을 조회할수 있다(CQRS).


  > <h4>MSAEz 로 모델링한 이벤트스토밍 결과</h4>  
http://labs.msaez.io/#/storming/t5Z5EXdDP0UOZDvGzeNH61hF8qG3/share/52e31337a76ddeacc1d288ea11e24158/-MH4jm58lJNE_9tgT82F
![EventStorming_Restaurant](https://user-images.githubusercontent.com/54210936/93165941-4bda4580-f758-11ea-9195-bc577796b8d0.png)


  > <h4>이벤트도출</h4>
  1. 주문됨
  2. 주문취소됨
  3. 요리재고체크됨
  4. 요리시작됨
  5. 배달시작됨

<h3>체크포인트 구현</h3>

><h4>1. SAGA</h4>
>+ 주문(Order) 후 요리(Coook) 시점에 재고가 없을 경우 요리가 취소 됌.
>+ 요리가 취소되는 경우 주문도 함께 취소 처리.

><h4>2. CQRS</h4>
>+ 주문(Order) / 요리(Cook) / 개발(Delivery) 현황을 모두 Mypage에서 조회 가능


><h4>3. Correlation</h4>
>+ 주문(Order) > 요리(Cook) : menu
>+ 요리(Cook) > 배달(Delivery) : cook


><h4>4. Request/Response</h4>
>+ 주문(Order) 취소시 Req/Res 형태로 연결


><h4>5. Gateway</h4>
>+ Gateway 접속으로 각 Microservice의 접근 루트를 통일
![gateway_LoadBalancer](https://user-images.githubusercontent.com/54210936/93172197-5b13c000-f765-11ea-9e31-aeb17c091f42.png)
![gateway_LoadBalancer_delivery](https://user-images.githubusercontent.com/54210936/93172200-5bac5680-f765-11ea-906f-d6edb1c8ec94.png)


 ><h4>6. Deploy / Pipeline</h4>
>+ AWS 코드빌더를 통한 CI/CD 구축.
Github 소스 수정 시 자동으로 MVN 컴파일 --> DockerBuild --> ECR 업로드 --> Deploy 적용
![Deploy, Pipeline  AWS_CodeBuild](https://user-images.githubusercontent.com/54210936/93167299-50ecc400-f75b-11ea-9568-331955fb320d.jpg)
![Deploy, Pipeline  buildspec yaml](https://user-images.githubusercontent.com/54210936/93167305-52b68780-f75b-11ea-8d55-33f3a9f6e9e8.jpg)


  ><h4>7, 8. CircuitBreaker / Autoscale(HPA)</h4>
>+ CircuitBreaker 설정 값
![Circuit Breaker  yaml Setting](https://user-images.githubusercontent.com/54210936/93168671-68797c00-f75e-11ea-926d-6de0dd8acffd.jpg)

>+ 부하적용 전
![ZeroDownTime_HPA  AS-IS STATUS](https://user-images.githubusercontent.com/54210936/93167881-8d6cef80-f75c-11ea-853b-a3734f7af356.jpg)

>+ CircuitBreaker 적용되어 Availability 70% 에서 정지
![HPA, Circuit Breaker  SEIGE_STATUS](https://user-images.githubusercontent.com/54210936/93168766-9ced3800-f75e-11ea-9d6b-fdf37591b97a.jpg)

>+ AutoscaleUp 적용됨
![HPA  TOBE_STATUS](https://user-images.githubusercontent.com/54210936/93167897-95c52a80-f75c-11ea-8f0e-51a94332141b.jpg)


><h4> 9. Zero-down Time Deploy</h4>
>+ Zero down Time setting
![ZeroDownTime  yaml Setting](https://user-images.githubusercontent.com/54210936/93168241-59de9500-f75d-11ea-85b6-1b87b09359ab.jpg)

>+ Image upload
![ZeroDownTime  AWS - Image info](https://user-images.githubusercontent.com/54210936/93168819-baba9d00-f75e-11ea-8b92-54db92767163.jpg)

>+ Image Change
![ZeroDownTime  console - pod change status](https://user-images.githubusercontent.com/54210936/93168822-bbebca00-f75e-11ea-8cf0-ab28fbddf6dd.jpg)

>+ Image 변경 중 부하 발생                                                                                                                                                      
![ZeroDownTime  SEIGE_STATUS](https://user-images.githubusercontent.com/54210936/93168826-bd1cf700-f75e-11ea-801d-c83912df06b4.jpg)

>+ Image 변경적용 확인
![ZeroDownTime  console - pod describe](https://user-images.githubusercontent.com/54210936/93168825-bc846080-f75e-11ea-91d8-bd8e9aa9dadd.jpg)


><h4> 10. PersistenceVolume</h4>
>+ PVC 사용을 위한 yaml 세팅
![PVC  yaml Setting](https://user-images.githubusercontent.com/54210936/93169153-711e8200-f75f-11ea-901d-d168a01284a3.jpg)
>+ Application에서 EFS에 기록한 Log 내역
![PVC  console - log file test](https://user-images.githubusercontent.com/54210936/93169149-6f54be80-f75f-11ea-8d97-28e3720c82e1.jpg)


><h4> 12. SelfHealing</h4>
>+ SelfHealing 적용을 위한 Replica와 liveness 세팅 값                                                                                     
![KakaoTalk_20200915_150627075](https://user-images.githubusercontent.com/54210936/93172478-e68d5100-f765-11ea-9321-9f960f245d83.jpg)
![KakaoTalk_20200915_150634479](https://user-images.githubusercontent.com/54210936/93172487-e7be7e00-f765-11ea-9e33-eb6c8fb5875c.jpg)

>+ Pod kill 적용 후 다시 기동되는 내역 확인
![Self-healing  console test](https://user-images.githubusercontent.com/54210936/93169273-b93da480-f75f-11ea-939e-925352bc13bd.jpg)
