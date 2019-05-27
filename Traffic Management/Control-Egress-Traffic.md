## Control Egress Traffic

Istio내의 pod으로부터 나가는 모든 outbound 트래픽은 기본적으로 sidecar proxy로 redirecdt된다. cluster 외부인 URL로의 접근성은 proxy의 설정에 따라 달라질 것 이다. 기본적으로 Istio는 Envoy proxy가 unknown service에 대한 요청을 그냥 흘려보낸다. 하지만 이런 방법이 편할 수는 있지만, 좀더 strict한 설정이 보통 사용될 것이다.

이번 문서에서는 external service에 접근하는 3가지 방법에 대해서 살펴볼 것이다.
1. Envoy proxy가 mesh 내부에 설정하지 않은 service로의 접근을 허용하는 방법
2. [Service entries](https://istio.io/docs/reference/config/networking/v1alpha3/service-entry/)를 설정하여 external service로의 접근을 제어하는 방법
3. 특정 IP 대역에 대해서 Envoy proxy가 bypass하는 방법


### 준비 
- Istio 설치 : https://github.com/grepsean/study-istio/blob/master/setup.md
- requests를 전송하기 위한 test source인 [sleep](https://github.com/istio/istio/tree/release-1.1/samples/sleep)이라는 앱 배포하기
  - autumatic sidecar injection이 설정되어 있다면,
  ```console
  $ kubectl apply -f samples/sleep/sleep.yaml
  ```
  - autumatic sidecar injection이 설정되어 있지 않다면,
  ```console
  $ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
  ```
  - 위에서 만든 sorce pod에 대한 `SOURCE_POD`이라는 환경변수 설정하기 
  ```console
  $ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
  ```

### Envoy passthrough to external services
Istio의 [installation option](https://istio.io/docs/reference/config/installation-options/)을 보면, ` global.outboundTrafficPolicy.mode`라는 옵션이 있는데, 이는 sidecar가 external service에 대한 handling에 대한 설정이다. 
만약 이 옵션이 `ALLOW_ANY`로 설정되어 있다면, Istio proxy는 unknown service에 대해서 passthrough하도록 한다. 
`REGISTRY_ONLY`라면, Istio proxy가 HTTP service가 아니거나 mesh에 정의되어 있지 않은 service entry에 대해서는 block한다.
`ALLOW_ANY`의 경우는, 기본값으로, external service에 대한 접근을 따로 제어하지 않는다. 

1.1.4 이전 버전에서는, `ALLOW_ANY`의 경우 HTTP service가 아니거나 mesh에 정의되어 있는 service에서만 동작한다. 내부 HTTP servie들과 동일한 port를 사용하는 External hosts의 경우는 기본적으로 blocking되는 동작으로 fallback된다. 왜냐면 80과 같이 포트의 경우 Istio내부의 HTTP service에서 기본적으로 사용되기 때문이다. 1.1.3버전 이전에서는 External service를 호출할때 이런 포트들을 사용하지 못했다.

1. 아래의 방법은 우선 `global.outboundTrafficPolicy.mode = ALLOW_ANY`로 설정되어 있어야한다. Istio 설치할때 `REGISTRY_ONLY`로 설정되어 있지 않다면, 기본적으로 설정되어 있을것이다.
  - 아래 커맨드를 실행시키고 제대로 실행되었는지 확인해보자.
```console
$ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY"
mode: ALLOW_ANY
```
  - `mode: ALLOW_ANY`라는 실행 결과가 확인될 것이다.
  - 만약 `REGISTER_ONLY` 모드로 설정되어 있다면, 아래 커맨드로 변경해줘야한다.  
```console
$ kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: REGISTRY_ONLY/mode: ALLOW_ANY/g' | kubectl replace -n istio-system -f -
configmap "istio" replaced
```

2. `SOURCE_POD`에서 external HTTPS services에 대해서 두번의 request를 날려서 200 OK가 오는지 확인해보자.
```console
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.google.com | grep  "HTTP/"; kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://edition.cnn.com | grep "HTTP/"
HTTP/2 200
HTTP/2 200
```

이제 egress traffic이 정상적으로 나가는것을 확인할 수 있을 것이다.

위의 간단한 방법은 문제점이 있는데, Istio의 monitoring과 external service에 대한 제어를 하지 못한다는 것이다. 예를 들면, external service로의 요청이 Mixer log에 보이지 않을 것이다. 다음장에서는 external service로의 접근에서 어떻게 monitor하거나 제어하는지 살펴볼 것이다.


### Controlled access to external services
Istio의 `ServiceEntry`설정을 이용해서, Istio cluster 내부에서 어떠한 service로든 접근이 가능하다. 이 장에서는 어떻게 httpbin.org와 같은 external HTTP service로 접근할 수 있게 설정하는지 살펴볼것이다. www.google.com 같은 HTTPS service도 마찬가지이며, monitoring과 control과 같은 기능도 가능하게 할것이다.

#### blocking-by-default policy 변경
external service로의 접근을 제어하는 방법을 사용하려면, `global.outboundTrafficPolicy.mode` 옵션을 `ALLOW_ANY`에서 `REGISTRY_ONLY`로 변경해야한다.

1. `global.outboundTrafficPolicy.mode` 옵션을 `REGISTRY_ONLY`로 변경하려면 아래와 같은 커맨드를 실행하자.
```console
$ kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
configmap "istio" replaced
```

2. `SOURCE_POD`에서 external HTTPS service로 2번의 requests를 전송해서 block되었는지 확인해보자.
```console
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.google.com | grep  "HTTP/"; kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://edition.cnn.com | grep "HTTP/"
command terminated with exit code 35
command terminated with exit code 35
```
- 위의 설정이 적용되는데까지는 몇초정도 걸릴 수 있다.

#### external HTTP service에 접근하기
1. `ServiceEntry`를 만들어서 external HTTP service로 접근할 수 있게하자.
```console
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

2. `SOURCE_POD`에서 external HTTP service로 요청을 보내보자.
```console
$ kubectl exec -it $SOURCE_POD -c sleep -- curl http://httpbin.org/headers
{
  "headers": {
  "Accept": "*/*",
  "Connection": "close",
  "Host": "httpbin.org",
  "User-Agent": "curl/7.60.0",
  ...
  "X-Envoy-Decorator-Operation": "httpbin.org:80/*",
  }
}
```
  - Istio의 sidecar proxy가 `X-Envoy-Decorator-Operation`라는 헤더를 추가한것을 확인할 수 있다.
  
3. `SOURCE_POD`의 sidecar proxy에서의 로그를 확인해보자.
```console
$ kubectl logs $SOURCE_POD -c istio-proxy | tail
[2019-01-24T12:17:11.640Z] "GET /headers HTTP/1.1" 200 - 0 599 214 214 "-" "curl/7.60.0" "17fde8f7-fa62-9b39-8999-302324e6def2" "httpbin.org" "35.173.6.94:80" outbound|80||httpbin.org - 35.173.6.94:80 172.30.109.82:55314
```
  - `destinationServiceHost`라는 attribute는 `httpbin.org`와 동일할 것이다. 또한 HTTP와 관련된 atrribute인 `method`, `url`, `responseCode`와 같은 것들도 마찬가지이다. Istio의 egress 트래픽 제어를 사용한다면, external service에 대한 monitoring이 가능하며, HTTP-related 정보들에 대해서도 접근이 가능하다.
  
#### external HTTPS service에 접근하기
1. `ServiceEntry`를 생성해서 external HTTPS service에 접근해보자.
```console
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

2. `SOURCE_POD`에서 external HTTPS service로 요청을 보내보자.
```console
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.google.com | grep  "HTTP/"
HTTP/2 200
```

3. `SOURCE_POD`에 있는 sidecar proxy의 로그를 확인해보자.
```console
$ kubectl logs $SOURCE_POD -c istio-proxy | tail
[2019-01-24T12:48:54.977Z] "- - -" 0 - 601 17766 1289 - "-" "-" "-" "-" "172.217.161.36:443" outbound|443||www.google.com 172.30.109.82:59480 172.217.161.36:443 172.30.109.82:59478 www.google.com
```

4. Mixer 로그를 확인하자. 
```console
$ kubectl -n istio-system logs -l istio-mixer-type=telemetry -c mixer | grep 'www.google.com'
{"level":"info","time":"2019-01-24T12:48:56.266553Z","instance":"tcpaccesslog.logentry.istio-system","connectionDuration":"1.289085134s","connectionEvent":"close","connection_security_policy":"unknown","destinationApp":"","destinationIp":"rNmhJA==","destinationName":"unknown","destinationNamespace":"default","destinationOwner":"unknown","destinationPrincipal":"","destinationServiceHost":"www.google.com","destinationWorkload":"unknown","protocol":"tcp","receivedBytes":601,"reporter":"source","requestedServerName":"www.google.com","sentBytes":17766,"sourceApp":"sleep","sourceIp":"rB5tUg==","sourceName":"sleep-88ddbcfdd-rgk77","sourceNamespace":"default","sourceOwner":"kubernetes://apis/apps/v1/namespaces/default/deployments/sleep","sourcePrincipal":"","sourceWorkload":"sleep","totalReceivedBytes":601,"totalSentBytes":17766}
```
  - `requestedServerName` attribute는 `www.google.com`일 것이다. Istio의 egress 제어를 사용한다면, external HTTPS services에 접근을 모니터링할 수 있는데, 특정 SNI나 보내거나 받은 bytes같은것들이다. HTTP와 관련된 atrribute인 `method`, `url`, `responseCode`와 같은 것들은 암호화되기때문에 Istio가 볼 수 없고, HTTPS에서의 정보는 모니터링할 수 없다. 만약 external HTTPS service로 접근시 HTTP관련 정보들을 모니터링하고 싶다면, application이 HTTP 요청을 만들고 [Istio TLS origination를 할 수 있도록 설정](https://istio.io/docs/examples/advanced-gateways/egress-tls-origination/)하게 할것이다.


#### external services로의 트래픽을 관리하기
Cluster간의 요청과 유사하게, Istio의 [routing rules](https://istio.io/docs/concepts/traffic-management/#rule-configuration)은 `ServiceEntry`를 설정해서 external serivces로 접근할 수 있게할 것이다. 이 예제에서는 `httpbin.org` 서비스를 호출할때 timeout rule을 설정한다. 

1. Pod내부에서 test source로 사용하기위해서, curl 요청을 통해서 `/delay` endpoint로 요청을 보내보자.
```console
$ kubectl exec -it $SOURCE_POD -c sleep sh
$ time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
200

real    0m5.024s
user    0m0.003s
sys     0m0.003s
```
  - `200 OK`를 받기까지는 약 5초의 시간이 소요된다.
  
2. source pod에서 나와서, `kubectl`을 사용하여 `httpbin.or`라는 external service에 대한 timeout을 `3초`로 설정해보자.
```console
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
EOF
```

3. 적용되기 위해 몇초정도 기다렸다가, *curl* 요청을 다시 보내보자.
```console
$ kubectl exec -it $SOURCE_POD -c sleep sh
$ time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
504

real    0m3.149s
user    0m0.004s
sys     0m0.004s
```
  - 이번에는 3초후에 `504 Gateway Timeout`을 받게 될 것이다. httpbin.org의 경우 5초를 기다리겠지만, Istio는 3초에 request를 끊어버릴 것이다.
  
#### cleanup
```console
$ kubectl delete serviceentry httpbin-ext google
$ kubectl delete virtualservice httpbin-ext --ignore-not-found=true
```

### External service로 바로 접근하기
특정 IP대역에 대해서는 Istio가 bypass하길 원한다면, Envoy sidecars가 external 요청에 대해서 [intercepting](https://istio.io/docs/concepts/traffic-management/#communication-between-services)하지 않게해야한다. 이 bypass를 설정하기 위해서는, `global.proxy.includeIPRanges`나 `global.proxy.excludeIPRanges` 설정을 변경하면된다. 그리고 `istio-sidecar-injector` 설정을 `kubectl apply`를 이용해서 update하면 된다. `istio-sidecar-injector` 설정을 업데이트하면 이후의 모든 pod deployments에 영향을 미친다.

위에서의 [Envoy passthrough](https://istio.io/docs/tasks/traffic-management/egress/#envoy-passthrough-to-external-services)와는 다르게 `ALLOW_ANY`를 사용해서 Istio sidecar proxy가 unknown serivce로의 요청을 passthroug하게 한다. 이 방법은 sidecar를 bypass하는데 기본적으로 명시된 IPs에 대한 Istio의 모든 기능은 비활성화된다. 따라서 `ALLOW_ANY`를 사용한다면 특정 destination에 대해서 service 추가할 수 없게된다. 
그러므로 이 설정 방법은 sidecar를 사용해서 external 접속을 설정할 수 없는 경우에서 성능이나 다른 이유일때 사용하는 것을 권장한다. 

Sidecar proxy로 redirect시킬 외부 IPs를 제외하는 간단한 방법은 `global.proxy.includeIPRanges` 옵션을 IP range나 internal cluster service를 위한 range로 설정하는 것이다. 이러한 IP range 값은 cluster가 실행되는 platform에 종속적이다. 

#### Determine the internal IP ranges for your platform
`global.proxy.includeIPRanges`옵션 값을 cluster provider에 따라서 설정해보자.

##### Minikube, Docker For Desktop, Bare Metal
기본 값은 `10.96.0.0/12`로 설정되어 있으며, 변경할 수 없다. 아래 커맨드를 이용해서 실제 값을 가져오자.
```console
$ kubectl describe pod kube-apiserver -n kube-system | grep 'service-cluster-ip-range'
      --service-cluster-ip-range=10.96.0.0/12
```

### Configuring the proxy bypass
`이전 사용했던 service entry와 virtual service를 제거하고 진행하자`

Platform에 맞는 IP range를 명시해서 `istio-sidecar-injector` 설정을 업데이트하자.
```console
helm template install/kubernetes/helm/istio <the flags you used to install Istio> --set global.proxy.includeIPRanges="10.0.0.1/24" -x templates/sidecar-injector-configmap.yaml | kubectl apply -f -
```
Istio 설치할때와 동일한 Helm 커맨드를 사용한다. 특히 `--namespace` flag와 `--set global.proxy.includeIPRanges="10.0.0.1/24" -x templates/sidecar-injector-configmap.yaml` flags를 사용하는 것에 유의하자.


#### 외부 서비스에 접근하기
이 bypass 설정은 새로운 deployments일 경우에만 영향을 미치기 때문에, 기존의 `sleep` application을 재배포할 필요가 있다. 

`istio-sidecar-injector` configmap을 업데이트한 후에, `sleep` application을 재배포한다. Istio의 sidecar는 internal request에 대해서만 intercept하거나 제어할 것이다. 그리고 external request에 대해서는 bypass할 것이다. 
```console
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
$ kubectl exec -it $SOURCE_POD -c sleep curl http://httpbin.org/headers
{
  "headers": {
    "Accept": "*/*",
    "Connection": "close",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.60.0"
  }
}
```
HTTP나 HTTPS를 통해서 external serivce로 접근하는것과 다르게, Istio sidecar와 관련된 어떤 header도 확인할 수 없을 것이다. external service로 전송되는 모든 요청은 sidecar의 로그 혹은 Mixer의 로그에서 확인할 수 없을 것이다. Istio의 sidecar로 Bypassing한다는 것은 external service로 접근하는것에 대해서 더 이상 monitor하지 않겠다는 의미를 가진다.

#### cleanup
모든 outbound 요청에 대해서 sidecar proxy로 redirect하도록 `istio-sidecar-injector.configmap.yaml`설정을 변경하자. 
```console
$ helm template install/kubernetes/helm/istio <the flags you used to install Istio> -x templates/sidecar-injector-configmap.yaml | kubectl apply -f -
```

### Understanding what happened
이번 장에서는 external service에 접근하기 위한 3가지 방법에 대해서 살펴보았다. 
1. Envoy proxy가 mesh 내부에 설정하지 않은 service로의 접근을 허용하는 방법
2. [Service entries](https://istio.io/docs/reference/config/networking/v1alpha3/service-entry/)를 설정하여 external service로의 접근을 제어하는 방법, **이 방법이 추천하는 방법이다.**
3. 특정 IP 대역에 대해서 Envoy proxy가 bypass하는 방법

첫번째 방법은 mesh내부에서 unknwon service에 접근하기위해서 Istio sidecar proxy에 바로 트래픽을 전달하는 방법이다. 이 방법을 사용할때는 external serivce로 접근할때 monitor할 수 없으며, Istio의 트래픽 제어하는 기능을 제대로 사용하지 못하게 된다. 특정 services에 대해서 두번째방법으로 빠르게 전환하기 위해서 external services에 대해서 service entries를 생성해야한다. 이 과정은 모든 external serivce에 대해서 일단 접근이 가능하게 하고 나중에 접근을 제어할지 결정할 수 있다. 또한 트래픽에 대해서 monitoring이나 제어 기능을 사용할 수 있다.

두번째 방법은 service가 내부에 있건 외부에 있건, Istio service mesh 기능을 동일하게 사용하게 하는 방법이다. 위에서는 external service에 대한 접근을 어떻게 monitor하거나 timeout 설정을 하는지 알아보았다.

세번째 방법은 external server로 바로 접근할 수 있도록 Istio sidecar proxy로 bypass하는 방법이다. 하지만 이 방법은 cluster provider에 따라서 추가적인 설정 필요로 한다. 첫번째 방법과 비슷하게, 모니터링이나 트래픽제어와 같은 Istio의 기능을 제대로 사용할 수 없게된다.


### Security note
본 예제에서는 **secure egress**를 제공하지는 않는다. 어떤 악의적인 application이 Istio sidecar proxy를 bypass해서 Istio의 제어없이 external service로의 접근을 가능하게 할 수 있다.


### Cleanup
`sleep` 서비스를 내리자.
```console
$ kubectl delete -f samples/sleep/sleep.yaml
```
