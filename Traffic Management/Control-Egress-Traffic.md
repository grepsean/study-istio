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

2. `SOURCE_POD`에서 external HTTPS service


