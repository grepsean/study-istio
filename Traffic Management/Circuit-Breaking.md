## Circuit Breaking
이번 섹션에서는 connection, requests, outlier detection에 대한 circuit breaking 설정하는 방법에 대해서 살펴본다.
Circuit breaking은 resilient한 miscroserice application을 만들기 위한 중요한 패턴중 하나이다. 
따라서 특별한 failure 상황이나, latency가 갑자기 튄다던지하는 원치않는 장애 상황에서 circuit breaker를 사용해서 어떻게 대처하는지 살펴보자.


### 준비 
- Istio 설치 : https://github.com/grepsean/study-istio/blob/master/setup.md
- [httpbin](https://github.com/istio/istio/tree/release-1.1/samples/httpbin) 샘플 설치
  - autumatic sidecar injection이 설정되어 있다면,
  ```console
  $ kubectl apply -f samples/httpbin/httpbin.yaml
  ```
  - autumatic sidecar injection이 설정되어 있지 않다면,
  ```console
  $ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
  ```

### Circuit breaker 설정
1. destination rule을 통해서 circuit breaking 설정을 해보자.
  - 주의 : `mutual TLS Authentication enabled` 환경이라면, DestinationRule을 apply하기전에 TLS traffic policy에 `mode: ISTIO_MUTUAL`와 같은 설정을 추가해야한다.
  - 그렇지 않으면 503 에러가 발생할 것이다. 자세한 내용은 [여기](https://istio.io/help/ops/traffic-management/troubleshooting/#503-errors-after-setting-destination-rule)를 살펴보자.
```console
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

2. 적용된 destination rule을 확인하자.
```console
$ kubectl get destinationrule httpbin -o yaml
```
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
  ...
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
      tcp:
        maxConnections: 1
    outlierDetection:
      baseEjectionTime: 180.000s
      consecutiveErrors: 1
      interval: 1.000s
      maxEjectionPercent: 100
```


### Client 추가하기
이제 `httpbin` 서비스로 트래픽을 보낼 client를 만들어보자. 이 섹션에서는 [fortio](https://github.com/istio/fortio)라는 istio에서 Go언어로 개발한 simple load-testing tool을 사용할 것이다.
Fortio는 nGrinder나 Siege같은 outgoing HTTP call에 대해서 conntions의 개수나, 동시 접속, delay등을 컨트롤할 수 있게 해준다. 
이 Tool을 이용해서 client가 `DestinationRule`에 설정해둔 circuit breaker policy를 _"trip"_ 할 수 있게 해준다.

1. 우선 Istio가 network interaction을 통제할 수 있게 Fortio client를 istio sidecar proxy에 주입해보자.
```bash
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)
```

2. client pod에 로그인후 fortio tool을 이용해서 `httpbin`을 호출해보자. `-curl` 옵션을 전달하면 단 한번만 호출하게 한다.
```console
$ FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
$ kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -curl  http://httpbin:8000/get
HTTP/1.1 200 OK
server: envoy
date: Tue, 16 Jan 2018 23:47:00 GMT
content-type: application/json
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 445
x-envoy-upstream-service-time: 36

{
  "args": {},
  "headers": {
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "istio/fortio-0.6.2",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "824fbd828d809bf4",
    "X-B3-Traceid": "824fbd828d809bf4",
    "X-Ot-Span-Context": "824fbd828d809bf4;824fbd828d809bf4;0000000000000000",
    "X-Request-Id": "1ad2de20-806e-9622-949a-bd1d9735a3f4"
  },
  "origin": "127.0.0.1",
  "url": "http://httpbin:8000/get"
}
```
  - 위처럼 request를 확인했다면 일단 성공한 것이다.
  
