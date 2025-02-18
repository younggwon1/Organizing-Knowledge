# CoreDNS 이슈 정리

## 상황

> Application에서 도메인 질의를 위한 DNS 쿼리 실패 에러가 발생한 상황입니다. (EAI_AGAIN (DNS 조회실패) 발생)

```
response error: ... getaddrinfo EAI_AGAIN {domain}
```

CoreDNS 의 Log에는 다음의 **i/o timeout** ERROR가 발생하였습니다.

```
[ERROR] plugin/errors: 2 g.comail.com. AAAA: read udp {CoreDNS POD IP}:32966->10.17.0.2:53: i/o timeout
[ERROR] plugin/errors: 2 {domain}.ap-northeast-2.compute.internal. AAAA: read udp {CoreDNS POD IP}:37510->10.17.0.2:53: i/o timeout
```

## 원인 추측

> 1. CoreDNS CPU 사용량이 설정된 Request(100m) 보다 많아 CPU Throttling 이 발생하여 지연이 되는 것으로 추측
>
> - https://github.com/coredns/coredns/issues/4544#issuecomment-848751210

> 2. CoreDNS 의 PPS를 넘어가 발생한 문제로 추측
>
> - https://github.com/coredns/deployment/blob/master/kubernetes/Scaling_CoreDNS.md

## 파악

> 1. CoreDNS CPU 사용량이 설정된 Request(100m) 보다 많아 CPU Throttling 이 발생하여 지연이 되는 것으로 추측하였지만, CoreDNS CPU Limit이 설정되어 있지 않아 CPU를 충분히 사용할 수 있기에 CPU Throttling은 발생하지 않았습니다.

```
topk(50, sum(rate(container_cpu_cfs_throttled_seconds_total{pod=~"coredns-.*"}[5m])) by (pod)) > 0
```

```
참고)
CPU Limit이 없는 Pod 끼리의 경합이 있을 경우 CPU Share 비율로 조정이 되어 보통 Request를 사용량보다 너무 낮게 잡게 되면 그만큼 덜 사용하게끔 되기에 Request를 평소 사용량(보장받아야 하는...)으로 맞춰주면 좋습니다.
하지만 이 경우 Pod 끼리의 경합은 발생하지 않은 것으로 보여 해당 이슈에 해당하지는 않는 것으로 보입니다.
```

또한, CoreDNS의 처리에 지연이 있지도 않았습니다.

```
histogram_quantile(0.99, sum(rate(coredns_dns_request_duration_seconds_bucket{instance=~".*"}[2m])) by (server,zone,le,instance))
```

> 2. CoreDNS 의 PPS를 넘어가 발생한 문제로 추측

```
the hard limit of **1024** packets per second (PPS) set at the ENI level
```

- [Everything you need to know about monitoring CoreDNS for DNS performance](https://dev.to/aws-builders/everything-you-need-to-know-about-monitoring-coredns-for-dns-performance-5hi9)
- [Monitor CoreDNS traffic for DNS throttling issues](https://aws.github.io/aws-eks-best-practices/ko/networking/monitoring/)
- [Monitoring CoreDNS for DNS throttling issues using AWS Open source monitoring services](https://aws.amazon.com/ko/blogs/mt/monitoring-coredns-for-dns-throttling-issues-using-aws-open-source-monitoring-services/)

```
# AWS 측에서 다음의 답변을 받음
CoreDNS 혹은 파드 자체에서 PPS limit 혹은 비정상적인 CPU / MEM 사용 등으로 인해 정상적으로 DNS 쿼리가 되지 않을 수 있으니 CoreDNS 혹은 파드에서 이슈는 없었는지도 확인해보실 것을 권장합니다.
```

```
# 문서 정리
CoreDNS 배포에는 Kubernetes 스케줄러가 클러스터의 개별 워커 노드에서 CoreDNS 인스턴스를 실행하도록 지시하는 반선호도 정책이 있습니다.
즉, 동일한 워커 노드에 복제본을 같은 위치에 배치하지 않도록 해야 합니다.
이렇게 하면 각 복제본의 트래픽이 다른 ENI를 통해 라우팅되므로 네트워크 인터페이스 당 DNS 쿼리 수가 효과적으로 줄어듭니다.
초당 1024개의 패킷 제한으로 인해 DNS 쿼리가 병목 현상을 겪는 경우 1) 코어 DNS 복제본 수를 늘리거나 2) NodeLocal DNS 캐시를 구현해 볼 수 있습니다.
```

## 조치

1. DNS 쿼리 실패가 재발될 경우 출발지에서 dig 명령어로 트레이싱하여 파악합니다.

```
dig <DOMAIN> @publicHZ NS
dig +trace <DOMAIN>
```

2. CoreDNS의 CPU 리소스를 많이 사용하거나 or PPS를 넘어가 발생하는 문제로 판단하였고 **대수 증설**로 트래픽 분배를 통해 대응하고 설정 등의 개선 작업을 진행합니다.
3. 대수 증설로 부족하다면 **CPU Request** 를 증설합니다.

> 결과

1. 대수를 8대 -> 16대로 늘림에 따라 CPU 사용량이 낮아졌습니다.
2. 트래픽이 고르게 분배되어 요청되는 처리량이 낮아졌습니다.

## 추가 개선 방안

### CoreDNS를 Restart 또는 Scale In 할 경우 DNS 이슈 발생에 대한 개선 방안

1. cache : 30 (default) -> 60

- 너무 짧으면 금방 캐시가 휘발되어 부하를 유발함

2. max_concurrent : none (default) -> 1000

- 다음과 같이 반영

```
## AS-IS
> cache : 30s, max_concurrent : none
.:53 {
    errors
    health {
        lameduck 5s
      }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}

## TO-BE
> cache : 60s, max_concurrent : 1000
.:53 {
    errors
    health {
        lameduck 5s
      }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf {
      max_concurrent 1000
    }
    cache 60
    loop
    reload
    loadbalance
}
```

3. lameduck

- CoreDNS 개수를 줄이거나 Restart 할 때 Graceful Shutdown 하기 위해 CoreDNS configmap 에 lameduck 이라는 설정으로 조절
- https://coredns.io/plugins/health/

### Monitoring 고도화

1. CoreDNS의 Monitoring

- CoreDNS의 중요 Metrics 를 수집하여 Monitoring하고 이슈 발생 시 알람을 통해 대응할 수 있도록 설정
- [CoreDNS Metrics](https://coredns.io/plugins/metrics/)
- 특히 Monitoring 해야할 지표
  - **CPU**
  - **PPS**
  - **QPS**
  - ...
- Monitoring 설정
  - http://docs.aws.amazon.com/ko_kr/eks/latest/best-practices/monitoring_eks_workloads_for_network_performance_issues.html
  - https://aws.github.io/aws-eks-best-practices/ko/networking/monitoring/
  - https://aws.amazon.com/ko/blogs/mt/monitoring-coredns-for-dns-throttling-issues-using-aws-open-source-monitoring-services/
  - https://dev.to/aws-builders/everything-you-need-to-know-about-monitoring-coredns-for-dns-performance-5hi9

> 모니터링 할 EKS NODE(EC2)에 Elastic Network Adapter 활성화가 되어 있어야 모니터링이 가능합니다.

```
$ ethtool -i eth0
driver: ena <- 이렇게 되어있으면 ENA 활성화가 된 것입니다.
version: 2.12.3
firmware-version:
expansion-rom-version:
bus-info: 0000:00:05.0
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: yes
```

### 참고

1. ndots : https://nyyang.tistory.com/161 , https://themapisto.tistory.com/240
2. DNS 응답 코드 : https://hippogrammer.tistory.com/364
3. PPS limit : https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/pps-limit-edit-firewall-policer.html
