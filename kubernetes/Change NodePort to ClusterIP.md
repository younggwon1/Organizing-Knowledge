# Change Service Type : NodePort to ClusterIP

### 0. 개요
> 현재 많은 서비스들이 NodePort로 구성되어 있습니다.
> 하지만 이 구성은 보안상 취약하고, 관리가 어렵습니다.
> 따라서 NodePort를 사용하지 않는 ClusterIP 구성을 진행합니다.

### 1. 인스턴스 vs IP
**대상 유형: `인스턴스`**
> 대상 유형을 인스턴스 로 ALB / NLB를 구성하는 경우, 트래픽은 인스턴스의 네트워크 인터페이스(eth0)의 기본 IP를 활용하여 POD를 호스팅하는 EC2 인스턴스로 라우팅됩니다.

장점
1. 단순성: EC2 인스턴스의 기본 IP를 사용하므로 설정이 간단합니다.
2. 호환성: 새로운 인스턴스가 자동으로 등록되므로 자동 확장 그룹과 원활하게 작동합니다.
3. 리소스 활용도: 확장 및 모니터링을 위해 인스턴스 수준 메트릭을 일관되게 사용합니다.

단점
1. 지연 시간: 트래픽이 인스턴스를 통해 추가 홉을 거치므로 지연 시간이 증가할 가능성이 있습니다.
2. 확장성: 인스턴스의 IP 주소 수에 따라 제한되므로 트래픽이 많은 상황에서는 병목 현상이 발생할 수 있습니다.
3. 오버헤드: 인스턴스가 중개자 역할을 하므로 네트워크 및 리소스 관리 측면에서 오버헤드가 증가합니다.

**대상 유형: `IP`**
> 대상 유형을 IP를 사용하면 ALB / NLB가 AWS VPC CNI에서 할당한 개별 IP 주소를 활용하여 트래픽을 POD로 직접 라우팅합니다. 이 설정은 특히 동적이고 확장 가능한 환경에 효율적입니다.

장점
1. 직접 라우팅: 트래픽이 포드에 직접 라우팅되어 대기 시간이 줄어들고 응답 시간이 향상됩니다.
2. 유연성: 포드가 자주 생성되고 파괴되는 동적 환경에 더 적합합니다.
3. 효율성: 인스턴스를 통한 라우팅의 중간 단계를 제거하여 전체 처리량을 향상시킬 수 있습니다.
4. 세분성: 포드 수준에서 보다 정확한 확장 및 트래픽 관리가 가능합니다.

단점
1. 복잡성: 특히 AWS VPC CNI의 경우 더 복잡한 설정 및 구성이 필요합니다.
2. 네트워킹 제약: 충돌을 피하고 확장성을 보장하려면 VPC 내에서 적절한 IP 주소 관리가 필요합니다.
3. 유지관리: 네트워크 구성을 유지 관리하고 문제를 해결하는 데 있어 복잡성이 증가했습니다.

참고 문서
- [Choosing between IP and Instance target type in AWS Load Balancers for EKS](https://medium.com/@kespineira/choosing-between-ip-and-instance-target-type-in-aws-load-balancers-for-eks-065fb9b548a7)

### 2. Service 의 Type 을 NodePort -> ClusterIP 로 전환하는 과정 (ALB 의 TargetType 을 Instance -> IP 로 전환)

1. Ingress 에 다음의 Annotations 이고,

   - `alb.ingress.kubernetes.io/target-type: instance`

2. Service 의 type 이 NodePort 인 경우에,

   - `type: NodePort`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:0000000000000:certificate/xxxxxxx-bbbb-aaaaa-cccccc-dddddddd
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/security-groups: sg-xxxxxxxxxxxxxxxxx
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/target-type: instance
    ...
spec:
  ingressClassName: alb
  rules:
  - host: x.y.z.com
    http:
      paths:
      - backend:
          service:
            name: x-y-z
            port:
              number: 80
        path: /*
        pathType: ImplementationSpecific
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: x-y-z
    app.kubernetes.io/name: x-y-z
    ...
  name: x-y-z
  namespace: default
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/instance: x-y-z
    app.kubernetes.io/name: x-y-z
  type: NodePort
```

3. Service 의 Annotations 이 다음과 같이 설정할 경우,

   - `alb.ingress.kubernetes.io/target-type: ip`
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    ...
  labels:
    app.kubernetes.io/instance: x-y-z
    app.kubernetes.io/name: x-y-z
    ...
  name: x-y-z
  namespace: default
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/instance: x-y-z
    app.kubernetes.io/name: x-y-z
  type: NodePort
```

4. TargetGroupBinding 은 TARGET-TYPE 은 자동으로 `Instance -> IP` 로 변경되고,

5. ALB 의 타겟그룹, 대상 유형도 자동으로 `인스턴스 -> IP`로 변경된다.

6. 중요한 점은 이 상황에서 에러가 발생할 것으로 추측했지만, 계속적으로 `200` 응답을 받는다.


#### 참고
> ingress.yaml의 alb.ingress.kubernetes.io/target-type : ip or instance  - service.yaml의 spec.type : ClusterIP or NodePort    

> Service 의 type 이 NodePort인 경우 ingress.yaml의 target-type을 ip로 설정하더라도 동작은 Instance Mode처럼 동작합니다.

> 따라서 ingress.yaml의 target-type을 ip로 설정하는 목적을 달성하려면 Service 의 type을 ClusterIP 로 설정해야합니다.

AS-IS
```
- ELBSecurityPolicy-2016-08
- alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'  (aws-load-balancer-controller version =< 2.2)
- alb.ingress.kubernetes.io/target-type: instance
```

TO-BE
```
- ELBSecurityPolicy-TLS13-1-2-2021-06
- alb.ingress.kubernetes.io/ssl-redirect: '443' (aws-load-balancer-controller version >= 2.3)
- alb.ingress.kubernetes.io/target-type: ip
```

참고 문서

[1]. https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/tasks/ssl_redirect/

[2]. https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html

`-> 
Service 의 Annotation 에 alb.ingress.kubernetes.io/target-type: ip를 추가하면, ALB 와 TargetGroupBinding의 타켓 타입이 Instance -> IP 로 전환되는 것을 확인했으니, 이를 활용하여 Service 의 Type 까지 ClusterIP 로 바꿔보자.`

### 3. 테스트 (Kubernetes 내 / 외부 테스트)
```bash
# Kubernetes 외부
for i in {1..10000}; do curl -I http://x.y.z.com | grep "HTTP/1.1"; done

# Kubernetes 내부
for i in {1..10000}; do curl -I http://x-y-z.default | grep "HTTP/1.1"; done
```

#### 가설
1. ALB의 타켓 그룹 타입이 인스턴스, IP 둘 다 있는 경우의 동작?
   - Ingress 에 설정된 Service 중 NodePort, ClusterIP가 공존할 경우.?

2. Service Annotations에 추가 후 (TargetGroupBinding TargetType이 IP 로 바뀐 후) , Service Type 이 NodePort -> ClusterIP로 변경될 경우 동작.?

3. 만약 롤백이 필요할 경우, 어떻게 해야 하는가.? (to NodePort, 인스턴스 타입)

##### 결과
1. 
- ALB의 타켓 그룹 타입이 인스턴스, IP 둘 다 공존해도 문제 없음 (동작 등등...)
- 같은 ALB 의 TargetGroupBinding 에서의 TargetType이 인스턴스, IP 둘 다 공존 가능

2. 
- Service Annotations 추가 후 (`alb.ingress.kubernetes.io/target-type: ip`), Service Type 을 NodePort -> ClusterIP로 변경할 경우에도 문제 없이 동작
- 계속적으로 `200` 응답을 받는다.

3.
- Service Type 을 `ClusterIP -> NodePort` 로 변경
- Service Annotations 제거 (`alb.ingress.kubernetes.io/target-type: ip`)
- 계속적으로 `200` 응답을 받는다.

### 4. 결론
Service Type을 NodePort 에서 ClusterIP 로 변경하기 위해서는 다음과 같이 정리할 수 있습니다.

1. Service 의 Annotations 이 다음과 같이 설정

   - `alb.ingress.kubernetes.io/target-type: ip`

2. TargetGroupBinding 은 TARGET-TYPE 이 `Instance -> IP` 로 변경되고, ALB 타겟 그룹의 대상 유형도 `인스턴스 -> IP`로 변경되는 것을 확인합니다.

3. Service Type 을 NodePort 에서 ClusterIP로 변경하고 spec.ports[].nodePort를 삭제합니다.

4. Ingress 의 Annotations 에 `alb.ingress.kubernetes.io/target-type: instance` -> `alb.ingress.kubernetes.io/target-type: ip` 로 변경합니다.

5. 변경 후 서비스에 이슈나 장애가 발생하지 않았는지 확인합니다.
