## 이슈
nginx ingress contoller pod 이 죽을 경우 nginx ingress controller 를 통해 트래픽이 들어오는 a 라는 서비스에 접속 시 502 error 가 발생
( ALB -> (Target Group Binding) -> nginx ingress controller -> ingress -> service -> pod )

## 해결
> nginx ingress controller 설정에 minReadySeconds , preStop , terminationGracePeriodSeconds 를 활용하여 해결

minReadySeconds
> pod의 준비 상태를 확인하는 데 사용되는 설정
- pod이 Ready 상태가 되기까지 최소 기다리는 시간을 지정해주며, 설정된 시간동안 트래픽을 받지 않는다.
- minReadySeconds 는 readinessProbe 와 동일한 시간으로 맞춰 설정한다.
- ex) readinessProbe : 10s 이면 minReadySeconds: 10s
- why?
- readinessProbe가 완료되면 .spec.minReadySeconds에 설정된 시간이 아직 남아 있더라고도 무시되고 트래픽을 보냄

preStop
> 컨테이너가 종료되기 직전에 실행되는 Hook
- terminationGracePeriodSeconds만으로 Graceful shotdown 을 보장하기 어렵다. 따라서 preStop 기능을 같이 사용한다.
- terminationGracePeriodSeconds은 preStop과 병렬이기 때문에 최소한 preStop 소요 시간보다 더 길게 설정해야 한다. ( terminationGracePeriodSeconds > preStop )
- 예를들어, preStop hook이 60초 동안 실행되는데 terminationGracePeriodSeconds을 30초로 설정하면 30초 후 SIGKILL 신호가 전달되어 컨테이너가 강제 종료되기 때문이다.
- 추가로, AWS ALB는 idle timeout 시간이 디폴트가 60초이다. 그러므로 AWS ALB를 사용한다면 terminationGracePeriodSeconds을 60 이상으로 설정해야 504 에러를 피할 수 있다. 
- [공식 문서](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#connection-idle-timeout)에서도 애플리케이션의 유휴 시간제한을 로드 밸런서에 대해 구성된 유휴 시간제한보다 크게 구성하는 것이 좋다고 말하고 있다.
