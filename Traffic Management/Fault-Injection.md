## Fault Injection

### 준비
- Istio 설치 : https://github.com/grepsean/study-istio/blob/master/setup.md
- Bookinfo Sample Application 배포 : https://github.com/grepsean/study-istio/blob/master/examples.md
- Traffic Management 컨셉 확인 : https://istio.io/docs/concepts/traffic-management/
- [Traffic Routing 섹션](#Configuring-Request-Routing) 에서 살펴본 아래 실행해 보기
```bash
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```
  - 이전 섹션에서의 라우팅 flow에 대한 설정 이해하기
    - `productpage` → `reviews:v2` → `ratings` (jason이라는 사용자만)
    - `productpage` → `reviews:v1` (이외의 모든 사용자)

### Injecting an HTTP delay fault 
Bookinfo microservice에 대한 resiliency를 테스트하기 위해서 `7초` 정도의 딜레이가 생기게 해보자.
  - 단, `reviews:v2`와 `ratings` service 사이의 `json`이라는 사용자에 대해서만
이 테스트에서는 Bookinfo 내부에서 의도적으로 발생시킨 버그에 대해서는 다루지 않는다.

`reviews:v2` service에서는 `ratings` service를 호출시 `10초`의 timeout이 하드코딩되어 있다.
따라서 `7초`의 딜레이를 주더라도, 우선 에러없이 정상적으로 서비스되어야 한다.
