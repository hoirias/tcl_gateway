# Team-C

<h3>서비스 시나리오</h3>

<h4>기능적 요구사항></h4>
    1. 고객이 주문을 하면 주문정보를 바탕으로 요리가 시작된다.</br>
    2. 요리가 완료되면 배달이 시작된다.</br>
    3. 고객이 주문취소를 하게 되면 요리가 취소된다.</br>   
    4. 고객이 주문 시 제고가 없을 경우 주문이 취소된다.</br>   
    5. 고객은 Mypage를 통해, 주문과 요리, 배달의 전체 상황을 조회할수 있다.</br> </br>  
    '''

<h4><비기능적 요구사항></h4>
  1. 트랜잭션</br>
    - 주문 취소시 요리취소가 함께 처리되도록 한다.</br>
  2. 장애격리</br>
    - 주문 시스템에 장애가 발생하면 주문은 잠시뒤에 처리될 수 있도록 한다.</br>
  3. 성능</br>
    - 주문(Order) 접수 시 조회 불가 상황이 떨어지면.</br></br>

<h4><이벤트 도출></h4>
  1. 주문됨</br>
  2. 주문취소됨</br>
  3. 요리재고체크됨</br>
  4. 요리시작됨</br>
  5. 배달시작됨</br>
</br>
<h3><개발 현황></h3>
1. 프로그램 개발 후 github에 commit</br>
2. AWS codebuild 세팅에 따라 컴파일 후 Docker build, ECR 저장 및 Deploy(EKS)</br>
3. Matrix-Server를 설치. Autoscale 적용</br>
4. 각 Microservice에서 동작하며 EFS에 log 파일을 기록</br>
5. 각 Microservice는 kafka와 RestAPI로 통신</br>
</br>
* Region : ap-northeast-2</br>
* EKS : TeamC-final</br>
* ECR Image :   
    271153858532.dkr.ecr.ap-northeast-2.amazonaws.com/order</br>
    271153858532.dkr.ecr.ap-northeast-2.amazonaws.com/cook</br>
    271153858532.dkr.ecr.ap-northeast-2.amazonaws.com/delivery</br>
    271153858532.dkr.ecr.ap-northeast-2.amazonaws.com/mypage</br>
    271153858532.dkr.ecr.ap-northeast-2.amazonaws.com/gateway</br>
* EFS : EFS-teamc (fs-96929df7) </br>
* CodeBuild :</br>
    https://github.com/dew0327/final-cna-order/blob/master/cloudbuild.yaml</br>
    https://github.com/dew0327/final-cna-cook/blob/master/cloudbuild.yaml</br>
    https://github.com/dew0327/final-cna-delivery/blob/master/cloudbuild.yaml</br>
    https://github.com/dew0327/final-cna-mypage/blob/master/cloudbuild.yaml</br>
    https://github.com/dew0327/final-cna-gateway/blob/master/cloudbuild.yaml</br>
* github :</br>
    https://github.com/dew0327/final-cna-order/</br>
    https://github.com/dew0327/final-cna-cook/ </br>
    https://github.com/dew0327/final-cna-delivery/</br>
    https://github.com/dew0327/final-cna-mypage/</br>
    https://github.com/dew0327/final-cna-gateway/ </br>
</br>

<h3>체크포인트 구현</h3>

<h4>1. SAGA</h4>
* 주문(Order) 후 요리(Coook) 시점에 재고가 없을 경우 요리가 취소 됌</br>
* 요리가 취소되는 경우 주문도 함께 취소 처리</br>

<h4>2. CQRS</h4>
* 주문(Order) / 요리(Cook) / 개발(Delivery) 현황을 모두 Mypage에서 조회 가능


<h4>3. Correlation</h4>
* 주문(Order) > 요리(Cook) : menu </br>
* 요리(Cook) > 배달(Delivery) : cook</br>


<h4>4. Request/Response</h4>
* 주문(Order) 취소시 Req/Res 형태로 연결</br>


<h4>5. Gateway</h4>
* Gateway 접속으로 각 Microservice의 접근 루트를 통일<br/> 
![gateway_LoadBalancer](https://user-images.githubusercontent.com/54210936/93172197-5b13c000-f765-11ea-9e31-aeb17c091f42.png)
![gateway_LoadBalancer_delivery](https://user-images.githubusercontent.com/54210936/93172200-5bac5680-f765-11ea-906f-d6edb1c8ec94.png)


<h4>6. Deploy / Pipeline</h4>
* AWS 코드빌더를 통한 CI/CD 구축<br/>
Github 소스 수정 시 자동으로 MVN 컴파일 -> DockerBuild -> ECR 업로드 -> Deploy 적용
![Deploy, Pipeline  AWS_CodeBuild](https://user-images.githubusercontent.com/54210936/93167299-50ecc400-f75b-11ea-9568-331955fb320d.jpg)
![Deploy, Pipeline  buildspec yaml](https://user-images.githubusercontent.com/54210936/93167305-52b68780-f75b-11ea-8d55-33f3a9f6e9e8.jpg)


<h4>7, 8. CircuitBreaker / Autoscale(HPA)</h4>
* 안정적인 주문(Order) 서비스를 위해 CircuitBreaker를 설정하여 서비스가 유지될 수 있도록 지정한다. 500 Error가 발생하면 1초마다 스캔하여 10분간 CircuitBreak 처리.</br>
* 주문(Order)에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. CPU 사용률이 10% 초과시 Replica를 5까지 늘리도록 설정 함.</br>

<pre><code>
    ##CircuitBreaker 설정
    http:
        http1MaxPendingRequests: 1  # 연결을 기다리는 request 수를 1개로 제한 (Default 
        maxRequestsPerConnection: 1 # keep alive 기능 disable
    outlierDetection:
      consecutiveErrors: 1          # 5xx 에러가 5번 발생하면
      interval: 1s                  # 1초마다 스캔 하여
      baseEjectionTime: 10m         # 10분 동안 circuit breaking 처리   
      maxEjectionPercent: 100       # 100% 로 차단
</code></pre>

<pre><code>
    ##HPA 설정 설정
    cat <<EOF | kubectl apply -f -
    apiVersion: autoscaling/v1
    kind: HorizontalPodAutoscaler
    metadata:
     name: skcchpa-order
     namespace: teamc
     spec:
      scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: $_PROJECT_NAME           # order (주문) 서비스 HPA 설정
      minReplicas: 3                 # 최소 3개
      maxReplicas: 5                 # 최대 5개
</code></pre>

>+ 부하적용 전
![ZeroDownTime_HPA  AS-IS STATUS](https://user-images.githubusercontent.com/54210936/93167881-8d6cef80-f75c-11ea-853b-a3734f7af356.jpg)

>+ CircuitBreaker 적용되어 Availability 70% 에서 정지됨
![HPA, Circuit Breaker  SEIGE_STATUS](https://user-images.githubusercontent.com/54210936/93168766-9ced3800-f75e-11ea-9d6b-fdf37591b97a.jpg)

>+ AutoscaleUp 적용됨
![HPA  TOBE_STATUS](https://user-images.githubusercontent.com/54210936/93167897-95c52a80-f75c-11ea-8f0e-51a94332141b.jpg)


<h4> 9. Zero-down Time Deploy</h4>
>+ Zero down Time setting
>+ Image 변경 시 
![ZeroDownTime  yaml Setting](https://user-images.githubusercontent.com/54210936/93168241-59de9500-f75d-11ea-85b6-1b87b09359ab.jpg)

>+ Image upload
![ZeroDownTime  AWS - Image info](https://user-images.githubusercontent.com/54210936/93168819-baba9d00-f75e-11ea-8b92-54db92767163.jpg)

>+ Image Change
![ZeroDownTime  console - pod change status](https://user-images.githubusercontent.com/54210936/93168822-bbebca00-f75e-11ea-8cf0-ab28fbddf6dd.jpg)

>+ Image 변경 중 부하 발생                                                                                                                                                      
![ZeroDownTime  SEIGE_STATUS](https://user-images.githubusercontent.com/54210936/93168826-bd1cf700-f75e-11ea-801d-c83912df06b4.jpg)

>+ Image 변경적용 확인
![ZeroDownTime  console - pod describe](https://user-images.githubusercontent.com/54210936/93168825-bc846080-f75e-11ea-91d8-bd8e9aa9dadd.jpg)


<h4> 10. PersistenceVolume</h4>
>+ 각 Microservice의 Log를 기록하기 위해 사용  <br/>
>+ PVC 사용을 위한 yaml 세팅
![PVC  yaml Setting](https://user-images.githubusercontent.com/54210936/93169153-711e8200-f75f-11ea-901d-d168a01284a3.jpg)

>+ 각 Microservice에서 기록한 Log 내역 확인
![PVC  console - log file test](https://user-images.githubusercontent.com/54210936/93169149-6f54be80-f75f-11ea-8d97-28e3720c82e1.jpg)


<h4> 12. SelfHealing</h4>
>+ SelfHealing 적용을 위한 Replica와 liveness 세팅 값                                                                                     
![KakaoTalk_20200915_150627075](https://user-images.githubusercontent.com/54210936/93172478-e68d5100-f765-11ea-9321-9f960f245d83.jpg)
![KakaoTalk_20200915_150634479](https://user-images.githubusercontent.com/54210936/93172487-e7be7e00-f765-11ea-9e33-eb6c8fb5875c.jpg)

>+ Pod kill 적용 후 다시 기동되는 내역 확인
![Self-healing  console test](https://user-images.githubusercontent.com/54210936/93169273-b93da480-f75f-11ea-939e-925352bc13bd.jpg)
