## Control Ingress Traffic
Kubernetes환경에서 [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)라는 리소스는 cluster 외부에서 노출할 서비스를 지정하는데 사용한다. Istio에서는 좀더 낳은 접근(Kubernetes나 다른 환경 모두에서 동작하는) 방법을 사용할 수 있다. 그것이 바로 [Istio Gateway](https://istio.io/docs/reference/config/networking/v1alpha3/gateway/)라는 것이다. `Gateway`라는 것을 통해서 cluster로 들어오는 트래픽에 대한 monitoring이나 route rule과 같은 피쳐들을 가능하게 해준다.

여기에서는 istio가 `Gateway`를 이용해서 서비스를 mesh 외부에 어떻게 노출시키는지 살펴볼 것이다.

### Ingress Concept
#### Ingress란?
Networking community에서 나온 용어로 외부로부터 내부 네트워크의 외부에서 안쪽으로 들어오는 트래픽을 의미하며, 트래픽이 내부 네트워크 상에서 최초로 통과하는 부분을 Ingress Point라고 한다. 이 Ingress point를 통하지 않는다면, 내부 네트워크로 트래픽이 인입될 수 없다. 또한 Ingress point는 내부 네트워크의 특정 endpoint로 proxy해주는 역할을 한다.

#### Reverse proxy
Ingress는 reverse proxy와 동일한 역할을 수행하며, 외부의 요청에 대해서 Cluster 내부의 service로 proxy해주며, service 단위의 load balancing 기능도 제공해준다.

#### Istio Gateway
Istio에서는 이런 Ingress 역할을 담당하는 것이 Gateway이다. 즉 Ingress point 역할을 수행해서 cluster 외부에서 내부로 트래픽을 전달하고, load balancing, virtual-host routing 등을 수행한다.

또한 Istio Gateway는 Istio의 control plane 중 하나로, ingress proxy를 구현하는데 Envoy를 사용한다. 
Istio를 설치했다면, 초기화 과정에서 이미 ingress의 구현체가 설치되어 있을 것이다.
```console
$ kubectl gte pod -n istio-system
NAME                                      READY     STATUS      RESTARTS   AGE
istio-ingressgateway-788c96cd5f-lfxk9     1/1       Running     0          28d

$ kubectl get pod istio-ingressgateway-788c96cd5f-lfxk9 -o json -n istio-system
{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "labels": {
            "app": "istio-ingressgateway",
            "istio": "ingressgateway",
            "release": "istio"
            "chart": "gateways",
            "heritage": "Tiller",
            "pod-template-hash": "3447527819",
        }
        ...
    ...
}

$ kubectl get svc -n istio-system
NAME                     TYPE         CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
istio-ingressgateway   LoadBalancer  10.104.143.79  localhost    80:31380/TCP...   28d

$ kubectl get svc istio-ingressgateway -o json -n istio-system
{
    "apiVersion": "v1",
    "kind": "Service",
    "metadata": {
        "selector": {
            "app": "istio-ingressgateway",
            "istio": "ingressgateway",
            "release": "istio"
        },
        ...
    ...
}
```

#### Gateway, VirtualService
- Gateway : 클러스터 외부에 내부로 트래픽을 허용하는 설정
- VirtualService : 클러스터 내부로 들어온 트래픽을 특정 서비스로 라우팅하는 설정

#### 그런데 왜 Kubernetes의 Ingress를 사용하지 않은 것인가?
Istio도 초기에는 Kubernetes의 Ingerss를 사용했지만, 아래과 같은 이유로 Gateway를 만들었다고 한다.

1. Kubernetes의 Ingress는 HTTP/S 트래픽만 전달해주는 너무 심플한 spec이기때문이다. Kafka같은 큐를 사용한다면, broker를 TCP connection으로 제공해줄 수 있지만 Kubernetes Ingress에서는 이를 지원하지 않는다.

2. 또한 Kubernetes Ingress가 특정 패키지에 종속적이기 때문이다. 다르게 말해서 정해진 표준이 없고 구현체(Nginx, HAProxy, Traefik, Ambassodor) 마다 설정 방법이 다르다. 복잡한 routing rule, traffic splitting, shadowing등 상세한 설정을 하기 위한 표준화된 방법이 없다. 그렇다고 Istio도 새로운 구현체를 만들어서 추가한다면 더욱 복잡해질 것이다.


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

### ingress의 IP와 ports 가져오기
external load balancer를 지원해주는 kubernetes cluster 환경에서 동작중이라면 아래 커맨드를 실행시켜보자.
```console
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   172.21.109.129   130.211.10.121  80:31380/TCP,443:31390/TCP,31400:31400/TCP   17h
```
만약 `EXTERNAL-IP` 값이 설정되어 있다면, 현재 실행 환경이 ingress gateway를 사용할 수 있는 external load balancer를 지원하는 환경이라고 보면된다. 
하지만 `EXTERNAL-IP` 값이 `<none>`이라면 (혹은 `<pending>`이라면) 현재 환경이 external load balancer를 지원하지 않는 것이다. 이 경우에는 서비스의 [node port](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)를 이용해서 이용해서 gateway에 접근할 수 있다.


#### external load balancer 사용해서 ingress의 IP, ports 가져오기
external load balancer를 지원한다면, 다음 커맨드를 이용해보자.
```console
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

특정 환경에서, IP를 사용하지 않고 host name을 이용해서 load balancer가 노출될 수 있다. 이 경우에는 `EXTERNAL-IP` 값이 IP address가 아니라 host name이 보여질 것이다. 
따라서 위의 `INGRESS_HOST` 환경변수값이 아무것도 채워져 있지 않을 것이기 때문에, 아래와 같은 커맨드로 host name을 가져와야한다.
```console
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

#### node ports 사용해서 ingress의 IP, ports 가져오기
external load balancer를 가지고 있지 않다면 아래와 같은 커맨드를 사용해보자.
```console
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```

만약 docker for Desktop을 사용한다면, 다음과 같은 커맨드로 host를 가져와도 된다.
```console
export INGRESS_HOST=127.0.0.1
```

### Istio Gateway를 이용해서 Ingress 설정하기
Ingress [Gateway](https://istio.io/docs/reference/config/networking/v1alpha3/gateway/)는 mesh의 종단에서 HTTP/TCP connection을 받아주는 load balancer를 나타낸다. 따라서 노출할 ports와 protocol등을 설정해야한다. 하지만 [Kubernetes Ingress Resource](https://kubernetes.io/docs/concepts/services-networking/ingress/)와 다르게 어떠한 routing rules도 포함하지 않는다. Ingress 트래픽에 대한 routing은 Istio의 routing rules를 사용해서 설정하며, 내부 서비스간의 reqeust일때와 동일하게 설정하면 된다.

80번 port와 HTTP 트래픽을 사용하는 `Gateway`에 대한 설정은 다음과 같을 것이다.
1. `Gateway`를 생성한다.
```console
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF
```

2. `Gateway`를 통해서 인입되는 트래픽에 대한 routing을 설정한다. 
```console
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```
  - 위처럼 `/status`와 `/delay`라는 두개의 path에 대해서 routing rules를 가지는 `httpbin` 서비스에 대해서 `virtual service`를 설정하였다. 
  - 위의 `gateways` 설정에서 `httpbin-gateway`라고 설정하면 해당 gateway에서만 오는 트래픽만 허용하는 것이다. 그 이외의 트래픽은 모두 404로 응답한다.
  
  - 만약 다른 서비스로부터 `internal 요청`이 들어오게 되면, 위에서 설정한 rules이 적용되지 않는다. 대신에 default round-robin routing이 실행될 것이다. 따라서 위에서 설정한 rules에 대해서 internal 요청에도 적용하려면, special value인 `mesh`라는 값을 `gateways`리스트에 추가해주면 된다. 만약 Internal hostname이 External일 경우와 다르다면(예를 들면, `httpbin.default.svc.cluster.local`), 이 호스트도 `hosts` 리스트에 추가해야한다. 더 자세한 사항은 [troubleshooting guid](https://istio.io/help/ops/traffic-management/troubleshooting/#route-rules-have-no-effect-on-ingress-gateway-requests)를 참고하자.
  
3. curl을 이용해서 httpbin 서비스에 접근해보자.
```console
$ curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
HTTP/1.1 200 OK
server: envoy
date: Mon, 29 Jan 2018 04:45:49 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 48
```
  - 위에서 `-H` 옵션은 요청시 host를 지정하는 것이다. 또한 이 옵션을 사용하는 이유는, ingress `Gateway`가 "httpbin.example.com"라는 호스트에 대해서 설정해놨기 때문이다. 하지만 test환경에서는 이 hsot에 대한 DNS 바인딩되어 있지않을 것이기 때문에 그냥 ingress IP로 요청을 날려도 된다.
  
4. 명시적으로 노출하지 않은 다른 URL에 접근해보자. 이 경우는 당연히 404를 수신할 것이다.
```console
$ curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/headers
HTTP/1.1 404 Not Found
date: Mon, 29 Jan 2018 04:45:49 GMT
server: envoy
content-length: 0
```

### 브라우저를 이용해서 ingress service에 접근해보자
브라우저에서 `httpbin`서비스 URL로 접속해보면 동작하지 않을텐데 이건 브라우저가 위의 curl 명령처럼 host가 `httpbin.example.com`인것처럼 동작하지 않기때문이다. 하지만 real world에서는 host를 정확히 입력할뿐이나라 DNS에서도 resolve될것이기 때문에 문제가 되지 않을것이다. 그러므로 `https://httpbin.example.com/status/200`처럼 host domain name을 사용하면 된다.
테스트나 데모를 위해서 이 문제를 work around로 처리하려면, `Gateway`와 `VirtualService`설정에 있는 `host`부분에 `*`(wildcard)로 처리하면된다. 
```console
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

그리고나서 브라우저에서 `$INGRESS_HOST:$INGRESS_PORT` URL로 접속해보면 된다. 예를 들어, `http://192.168.99.100:31380/headers`로 접속해보면 브라우저에서 보내는 header 정보를 확인할 수 있다.

### 무슨일이 일어난걸까?
`Gateway` 리소스는 external 트패픽에 대해서 Istio mesh 내부로 들어올 수 있게했으며, edge 서비스들이 Istio의 traffic management와 policy와 같은 피처를 사용할 수 있게 한것이다. 
이번 문서에서 한일은 mesh 안쪽에서 서비스르 만들고 external 트래픽을 받을 서비스의 HTTP endpoint를 노출했다.

### Troubleshooting
1. 아래 커맨드를 이용해서 환경 변수 `INGRESS_HOST`, `INGRESS_PORT`값이 정상적인지 확인해보자.
```console
$ kubectl get svc -n istio-system
$ echo INGRESS_HOST=$INGRESS_HOST, INGRESS_PORT=$INGRESS_PORT
```

2. 생성되어 있는 다른 ingress gateway가 같은 port를 사용하고 있는지 확인한다.
```console
$ kubectl get gateway --all-namespaces
```

3. 같은 IP와 port로 설정한 kubernetes의 ingress resource가 있는지 확인한다.
```console
$ kubectl get ingress --all-namespaces
```

4. 만약 external load balancer를 가지고 있지만 사용하지 않는다면, 서비스의 [node port](https://istio.io/docs/tasks/traffic-management/ingress/#determining-the-ingress-ip-and-ports-when-using-a-node-port)를 이용해서 접근해보자.


### Cleanup
`Gateway`와 `VirtualService` 설정을 지우고, `httpbin` 서비스를 내리면 된다.
```console
$ kubectl delete gateway httpbin-gateway
$ kubectl delete virtualservice httpbin
$ kubectl delete --ignore-not-found=true -f samples/httpbin/httpbin.yaml
```
