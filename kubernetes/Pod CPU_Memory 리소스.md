# Pod CPU/Memory 리소스

> [파드](https://kubernetes.io/ko/docs/concepts/workloads/pods/)를 지정할 때, [컨테이너](https://kubernetes.io/ko/docs/concepts/containers/)에 필요한 각 리소스의 양을 선택적으로 지정할 수 있다. 지정할 가장 일반적인 리소스는 CPU와 메모리(RAM) 그리고 다른 것들이 있다.
>
> 컨테이너에 대한 리소스 *요청(**request**)* 을 지정하면, [kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)는 이 정보를 사용하여 파드가 배치될 노드를 결정한다. 또한 kubelet은 request에 명시한 resource만큼을 해당 Container가 사용할 수 있도록 미리 예약을 해둔다.
>
> 컨테이너에 대한 리소스 *제한(**limit**)* 을 지정하면, kubelet은 실행 중인 컨테이너가 설정한 제한보다 많은 리소스를 사용할 수 없도록 해당 제한을 적용한다.



### Request & Limit

파드가 실행 중인 노드에 사용 가능한 리소스가 충분(남은 리소스가 충분하다면..)하면, 컨테이너가 해당 리소스에 지정한 `request` 보다 더 많은 리소스를 사용할 수 있도록 허용된다. 그러나, 컨테이너는 리소스 `limit` 보다 더 많은 리소스를 사용할 수는 없다.

- 예를 들어, 컨테이너에 대해 256MiB의 `memory` 요청을 설정하고, 해당 컨테이너가 8GiB의 메모리를 가진 노드로 스케줄된 파드에 있고 다른 파드는 없는 경우, 컨테이너는 더 많은 RAM을 사용할 수 있다.

해당 컨테이너에 대해 4GiB의 `memory` 제한을 설정하면, kubelet(그리고 [컨테이너 런타임](https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/))이 제한을 적용한다. 런타임은 컨테이너가 구성된 리소스 제한을 초과하여 사용하지 못하게 한다. 

- 예를 들어, 컨테이너의 프로세스가 허용된 양보다 많은 메모리를 사용하려고 하면, 시스템 커널은 메모리 부족(out of memory, OOM) 오류와 함께 할당을 시도한 프로세스를 종료한다.

`request`는 반응적(시스템이 위반을 감지한 후에 개입)으로 또는 강제적(시스템이 컨테이너가 제한을 초과하지 않도록 방지)으로 구현할 수 있다. 런타임마다 다른 방식으로 동일한 제약을 구현할 수 있다.

만약 Container에 메모리 limit은 설정했는데 request 값을 설정하지 않았다면, Kubernetes는 자동으로 limit에 설정한 값만큼 request 값을 설정한다. 이는 CPU에 대해서도 마찬가지이다.



### Resources in Kubernetes

**CPU**

CPU에 대한 request와 limit 값은 cpu unit이라는 단위를 가진다. 

1 cpu는 Kubernetes에서 1vCPU/Core로 생각할 수 있다. 

만약 0.5라는 값으로 설정한다면 이는 500m와 같은 값을 나타내며 500 밀리 cpu를 가진다고 생각할 수 있다.

그리고 이 값은 상대적인 값이 아니다. 무슨 말이냐면 single-core나 dual-core를 가진 node에 0.5이라는 CPU request, limit 값을 설정한다면 코어의 수에 따라 비례하는 값이 아니라 두 개의 node에 대해서 동일한 CPU를 할당한다는 뜻이다.


**Memory**

Memory 대한 request와 limit 값은 바이트 단위를 가진다. 

이 값은 정수로 나타내야하며 숫자 뒤에 E, P, T, G, M, K와 같은 접미어가 붙을 수도 있다. 

각각은 Ei, Pi, Ti, Gi, Mi, Ki와 동일한 의미를 가진다.


각 컨테이너에 대해, 다음과 같은 리소스 제한(limit) 및 요청(request)을 지정할 수 있다.

- `spec.containers[].resources.limits.cpu`
- `spec.containers[].resources.limits.memory`
- `spec.containers[].resources.requests.cpu`
- `spec.containers[].resources.requests.memory`



### How Pods with resource limits are run

Pod에 설정된 resource request와 limit이 어떻게 영향을 미치는지를 알아보기 위해서는 **Compressible/Incompressible resource**에 대해서 아는 것이 도움이 된다.

Kubernetes에서 Comressible한 resource 중 대표적인 것은 바로 CPU이다. Compressible resource는 Pod이 그 값을 초과하려고 하면 자신이 사용할 수 있는 resource에 한계가 있기 때문에 병목이 걸린다. 왜냐하면 필요한 값보다 자신이 사용할 수 있는 resource가 적기 때문이다.

반면에 Incompressible resource에 대표적인 것 중 하나는 메모리가 있는데 Incompressible resource의 경우는 Pod이 그 사용량을 초과하려고하면 해당 Pod 내부 컨테이너의 커널에 의해 killed된다.

그래서 이를 정리하면 아래와 같은 상황이 나타날 수 있다.

- 만약 Container가 자신에게 설정된 메모리 limit보다 그 사용량을 초과했을 때 해당 Container는 종료될 수 있다. 만약 다시 시작할 수 있다면 kubelet은 해당 Container를 다시 실행시킨다.
- 만약 Container가 자신에게 설정된 메모리 request보다 그 사용량을 초과했을 때 Container는 자신이 배정된 node의 메모리가 부족하면 evicted된다.
- 만약 Container 자신에게 설정된 CPU limit을 특정 시간 이상 동안 넘는다하더라도 해당 Container는 죽지 않는다.



# QoS

**Pod Quality of Service**

- Pod의 Limit와 Request에 따라서 **QoS 클래스를** 지정한다.

- 노드의 리소스가 부족하면 kubernetes는 QoS 클래스를 사용하여 **먼저 죽일 Pod를 결정**한다.

- **QoS클래스의 종류**

  - **BestEffort**

    - request, limit을 명시하지 않은 Pod은 BestEffort QoS를 가진다. 

    - request, limit을 명시하지 않으면 scheduler는 해당 Pod을 어떤 node에 scheduling 해야할지 알지 못한다. scheduler는 추측해서 해당 Pod을 띄울 수밖에 없다.

    - 즉, scheduler가 자신이 해당 Pod을 어디에 schedule 해야할지 잘 알진 못하지만 알아서 최선의 선택을 해보겠다는 전략이다.

    - 문제점

      - request, limit이 설정되지 않은 Pod을 새로 띄우려고 할 때, 스케쥴러는 일단 이 Pod을 띄울 node에 BestEffort로 Pod을 띄우려고 한다.

        하지만 새롭게 들어온 Pod은 자신이 얼마나 resource를 사용할 수 있을지 명시를 하지 않았기 때문에 해당 Pod이 resource를 어디까지 사용할 수 있는지 Kubernetes는 알지 못하고 **해당 Pod은 자신이 필요한대로 무한정으로 node의 resource를 추가적으로 요구하고 사용하게 된다.**

        그 결과 기존에 떠 있던 Pod들을 자원을 계속적으로 할당받지 못하는 상태인 resource starvation 상태로 만든다.

        더 문제가 심해지면 Kubernetes를 동작할 수 있게 해주는 kubelet과 같은 중요한 서비스들도 resource가 부족해져 제대로 동작할 수 없는 문제가 발생하기도 한다.

  - **Burstable**

    - Burstable QoS는 request, limit resource 값을 설정했지만 limit 값이 request 값보다 높을 때 설정된다.
    - 이는 어떤 서비스가 피크 타임이 존재해서 특정 시간동안 더 많은 resource를 쓰게하고 싶다거나 어플리케이션이 최초 부팅시 더 많은 resource를 필요로 한다거나 할 때 유용하게 사용할 수 있다.
    - Burstable은 BestEffort보다 resource starvation 문제를 줄일 수는 있지만 완전히 없어지는 것은 아니다. 위에서 말했듯이 기본적으로 Pod이 node에 스케쥴링될 때에는 resource request을 통해서 스케쥴링되기 때문에 어떤 서비스가 트래픽 피크 타임 때 더 많은 resource를 사용한다면 다른 서비스가 사용해야할 resource를 잡아먹을 수도 있다.

  - **Guranteed**

    - Guranteed QoS는 request, limit resource 값을 설정했고 limit 값이 request 값이 같을 때 설정된다.
    - 가장 경제적 비용이 크지만 가장 안정적으로 서비스를 운영할 수 있는 방법이다. 
    - request, limit 값이 같기 때문에 피크 타임 때도 어떤 서비스가 최대 얼마만큼 resource를 사용할지에 대한 정확한 예측이 가능하다. 또한 다른 서비스의 resource starvation 문제도 완전히 없애준다.
    - 한 가지 주의해야할 점이 있다면 Kubernetes 컨텍스트 바깥에서 동작하는 서비스의 resource는 고려되지 않는 다는 것이다. 어떤 말이냐면 Compute Engine 자체에서 돌고 있는 모니터링 툴이라던가 native하게 돌고 있는 프로그램이 사용하는 resource 자원은 Kubernetes 클러스터 단에서 알 수가 없다.

|                  | Pod Resource 설정  | **우선순위**(높을수록 오래 살아남음) |
| ---------------- | ---------------- | ---------------------- |
| BestEffort Class | 설정하지 않음          | 3                      |
| Burstable Class  | Request < Limit  | 2                      |
| Guranteed Class  | Request == Limit | 1                      |







참고 문서
[Kubernetes Resource and QoS Concept](https://www.getoutsidedoor.com/2020/11/15/kubernetes-resource-and-qos/)

[리소스 요청이 포함된 파드를 스케줄링하는 방법](https://kubernetes.io/ko/docs/concepts/configuration/manage-resources-containers/#%EB%A6%AC%EC%86%8C%EC%8A%A4-%EC%9A%94%EC%B2%AD%EC%9D%B4-%ED%8F%AC%ED%95%A8%EB%90%9C-%ED%8C%8C%EB%93%9C%EB%A5%BC-%EC%8A%A4%EC%BC%80%EC%A4%84%EB%A7%81%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95)

[쿠버네티스가 리소스 요청 및 제한을 적용하는 방법](https://kubernetes.io/ko/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)

[Pod CPU/Memory 리소스 최적화하기 (VPA 및 Kubecost 추천로직 분석)](https://devocean.sk.com/blog/techBoardDetail.do?ID=164786&boardType=techBlog#none)