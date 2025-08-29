# Istio 를 활용하여 특정 path 에 대한 api 호출을 막기 위한 방법 2가지
## 상황
> Public 으로 열려있는 특정 API 에 보안 조치가 필요한 상황. 이를 막는 작업이 필요. Load Balancer를 Private 으로 변경하게 되면 다른 서비스에도 문제가 되어 이슈가 되는 특정 API를 특정 Path 로 묶어 막는 작업을 진행.

## **이슈**
> Istio의 Virtual Service 에 각각의 설정을 반영해보았지만, 의도한대로 동작하지 않았음.

## **파악**
> Istio의 Virtual Service에서 특정 경로를 차단하고 다른 경로를 허용하는 설정이 제대로 반영되지 않는 이유는 **규칙의 순서와 일치 조건** 때문일 가능성이 크다. VirtualService의 규칙은 순차적으로 적용되기 때문에, 보다 구체적인 경로를 먼저 정의해야 한다.

아래는 /internal/* 경로에 대한 요청을 400 상태 코드로 차단하고, 나머지 경로 (/ 포함)에 대해서는 요청을 원래의 서비스로 라우팅하는 올바른 예제.

### EX 1) 
```yaml
- fault:
    abort:
      httpStatus: 400
      percentage:
        value: 100
  match:
  - uri:
      prefix: /internal
  route:
  - destination:
      host: {service name}.{namespace}.svc.cluster.local
      port:
        number: 8080
```

### EX 2) 
```yaml
- match:
  - uri:
      prefix: /internal
  directResponse:
    status: 403
    body:
      string: "does not allow access to internal endpoints"
```

## 참고 문서
(0) [Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/#virtual-services)

(1) [HTTPDirectResponse](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPDirectResponse)

(2) [HTTPFaultInjection](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection)
