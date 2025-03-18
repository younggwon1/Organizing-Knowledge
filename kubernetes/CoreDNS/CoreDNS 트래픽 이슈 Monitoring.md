# CoreDNS 트래픽의 DNS 제한 문제 모니터링

## 상황

> 운영 중인 Application에서 도메인 질의에 실패한 에러가 발생하여 CoreDNS 로그를 확인해보니 i/o timeout이 발생한 상황입니다.

## 대응

대응 방안으로 CoreDNS의 Replica 개수를 늘렸습니다. i/o timeout의 빈도는 줄은 것 같으나 간헐적으로 발생하는 상황입니다.

무작정 Pod 개수를 늘리는 것이 이슈 대응에 필요한 부분이기는 하지만, 결국 원인을 명확히 파악해야한다고 생각하여 모니터링을 시도하였고, 모니터링 시스템 구축은 다음의 문서를 기반으로 진행했습니다.

- https://docs.aws.amazon.com/ko_kr/eks/latest/best-practices/monitoring_eks_workloads_for_network_performance_issues.html

### 사전 작업

모니터링할 NODE에 **Elastic Network Adapter** 활성화가 되어 있어야 모니터링이 가능합니다.

```
$ ethtool -i eth0
driver: ena <- 이렇게 되어있으면 ENA 활성화가 된 것입니다.
version: 2.12.3g
firmware-version:
expansion-rom-version:
bus-info: 0000:00:05.0
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: yes
```

### 이슈 사항

문서 기반으로 모니터링 시스템을 구축하던 중 다음의 문제가 있었습니다.

Prometheus ethtool Exporter 배포 시 [prometheus-ethtool-exporter](https://raw.githubusercontent.com/Showmax/prometheus-ethtool-exporter/master/deploy/k8s-daemonset.yaml) 이라는 exporter를 사용하여 ethtool 관련 Metrics 를 노출하는데 해당 exporter가 Archive 된 상태였습니다.

- https://github.com/Showmax/prometheus-ethtool-exporter

따라서 이를 활용하여 운영하기에는 문제가 있다고 판단하여, prometheus node exporter에 [Prometheus Node Exporter 1.2.0 버전](https://github.com/prometheus/node_exporter/tree/v1.2.0/collector)부터 ethtool collector가 추가된 것을 확인하여 이를 활용하기로 하였습니다.

- ethtool 외의 collector, [prometheus-node-exporter collector](https://www.mankier.com/1/prometheus-node-exporter) 에 관심이 있으시면 해당 링크로 접속하셔서 확인하면 됩니다.

작업 방법은 다음과 같습니다.

데몬셋으로 배포된 prometheus-node-exporter에서 ethtool collector를 사용하기 위해서는 args 인자에 2가지를 추가해야합니다.

- **--collector.ethtool**
  - ethtool collector를 활성화
- **--collector.ethtool.metrics-include=^(bw_in_allowance_exceeded|bw_out_allowance_exceeded|conntrack_allowance_exceeded|linklocal_allowance_exceeded|pps_allowance_exceeded)$**
  - ethtool collector가 수집할 메트릭을 지정하는 정규표현식

```
spec:
  template:
    spec:
      containers:
      - args:
        - ...
        - --collector.ethtool
        - --collector.ethtool.metrics-include=^(bw_in_allowance_exceeded|bw_out_allowance_exceeded|conntrack_allowance_exceeded|linklocal_allowance_exceeded|pps_allowance_exceeded)$
```

Prometheus node exporter 파드가 구동될 때 아래 로그가 남으면 ethtool collector가 잡힌 것입니다.

```
ts=2025-02-19T09:36:06.553Z caller=node_exporter.go:118 level=info collector=ethtool
```

Prometheus node exporter에 ethtool collector 활성화 후 아래의 쿼리를 활용하여 Grafana 로 조회하면 됩니다.

```
rate(node_ethtool_linklocal_allowance_exceeded{device="eth0"}[5m])
rate(node_ethtool_conntrack_allowance_exceeded{device="eth0"}[5m]) > 0
rate(node_ethtool_bw_in_allowance_exceeded{device="eth0"}[5m]) > 0
rate(node_ethtool_bw_out_allowance_exceeded{device="eth0"}[5m]) > 0
rate(node_ethtool_pps_allowance_exceeded{device="eth0"}[5m]) > 0
```

구축 후 다음의 지표를 활용하여 CoreDNS i/o timeout 트러블슈팅을 어떻게 할 지 모호한 상황입니다.
(다음의 지표가 '0'을 넘으면 critical 하다고 가이드 문서에 작성되어 있는데 특정 시간에 '0' 넘는다고해서 CoreDNS i/o timeout 이 발생하는 것은 또 아닌 것 같습니다..)

- linklocal_allowance_exceeded
- conntrack_allowance_available
- conntrack_allowance_exceeded
- bw_in_allowance_exceeded
- bw_out_allowance_exceeded
- pps_allowance_exceeded

위의 지표와 관련하여 잘 작성된 문서가 있습니다. 이를 같이 참고하여 파악해보면 좋을 것 같습니다.

- [문서](https://engineering.doit.com/troubleshooting-aws-network-throttling-a-comprehensive-guide-368811424148)

## 요청

> 위의 지표를 활용하여 어떻게 CoreDNS i/o timeout 트러블슈팅을 하면 좋을지 AWS 측에 문의하였습니다.

## 정리

- linklocal_allowance_exceeded

  > 로컬 서비스에 대한 PPS(초당 패킷) 비율 허용치를 초과하여 형태가 변경되거나 삭제된 패킷의 수

- conntrack_allowance_exceeded

  > 인스턴스의 연결 추적이 최대치를 초과하여 새로운 연결을 설정할 수 없게 되어 드롭된 패킷의 수

- bw_in_allowance_exceeded

  > 인스턴스의 최대 인바운드 대역폭 초과로 인해 대기열에 추가되거나 삭제된 패킷의 수

- bw_out_allowance_exceeded

  > 인스턴스의 최대 아웃바운드 대역폭 초과로 인해 대기열에 추가되거나 삭제된 패킷의 수

- pps_allowance_exceeded
  > 인스턴스의 최대 양방향 PPS(초당 패킷 수) 초과로 인해 대기열에 추가되거나 삭제된 패킷의 수

각 지표들이 0 이상을 나타내게 되면, 네트워크 패킷이 드롭/삭제된 것을 의미하기 때문에 해당 지표들이 0 이상의 값을 나타나게 되면 네트워크 성능에 영향이 있을 수 있습니다.

다만, 해당 지표는 인스턴스 및 ENI 레벨에서 수집된 결과들이기에 0 이상으로 나타나진다고 하여 CoreDNS 파드에서 i/o timeout 이 항상 발생하는 것은 아닐 수 있습니다.

즉, 드롭 / 삭제된 패킷은 CoreDNS <-> AmazonProvidedDNS 통신간 발생한 패킷이 아닐 수 있고, 해당 노드에서 실행중인 다른 파드들의 패킷일 수도 있다는 의미입니다.

만약, 위 지표 값들이 주기적으로 발생한다면 네트워크가 불안정하다는 것을 의미하며 각 지표에 따라 아래와 같이 해석 및 해결을 할 수 있습니다.

1. linklocal_allowance_exceeded

   > 인스턴스에서 허용 가능한 초당 패킷 수가 초과된 결과이므로 CoreDNS 서비스를 수를 증가(다른 인스턴스에서 실행)시키고, 트래픽을 부하 분산하여 PPS에 대한 부하를 감소 할 수 있습니다.

   > 인스턴스 대비 서비스 파드 수를 줄여 단일 인스턴스에 대한 PPS 부하를 감소 시킬 수 있습니다.

   > 예를 들어, CoreDNS 파드를 2대로 운영하게 되면 클러스터 전체에서 DNS 리졸브 요청을 두 대의 파드에서 처리하게 되며 트래픽이 증가합니다. 이는 pps_allowance_exceeded 지표 초과를 유발할 수 있습니다. CoreDNS를 4대로 운영하며 각 다른 인스턴스에 배치시키고 트래픽을 분산시켜 pps_allowance_exceeded 지표를 낮출 수 있습니다.

2. conntrack_allowance_exceeded

   > 서비스 분산 및 단일 인스턴스 대비 서비스 수를 줄이고, 추가적으로 인스턴스에서 커넥션을 과도하게 많이 사용하는 파드가 있는지에 대한 확인을 합니다.

3. bw_in_allowance_exceeded, bw_out_allowance_exceeded
   > 서비스 분산 및 단일 인스턴스 대비 서비스 수를 줄이고 또한, 인스턴스는 타입 별로 네트워크 대역폭이 다르기 때문에, 대역폭 관련 지표들이 0 이상의 값들을 주기적으로 초과한다면 네트워크 대역폭을 고려하여 인스턴스 타입 변경을 고려해볼 수 있습니다.

## 결론

따라서 결론으로는 다음과 같습니다.

CoreDNS i/o timeout 은 i/o 작업이 시간 내에 완료되지 않았음을 의미하며 네트워크 연결문제, 지연, 리소스 부족으로 인한 처리 지연 등 다양한 이유에 의해 시간내 작업이 완료되지 않았을 때 발생합니다.

Amazon EKS 클러스터에서는 DNS 서버로 CoreDNS를 사용(클러스터 도메인에 대한 리졸브 처리)하며 CoreDNS 는 업스트림 네임 서버로 VPC DNS(클러스터 외부 도메인에 대한 리졸브 처리)를 사용합니다.

CoreDNS 에서 VPC DNS 요청에 대해 i/o timeout 이 발생한 것은 외부도메인 리졸브 요청을 하였으나 응답이 시간 내 처리되지 않았음을 의미합니다.

여기서도 마찬가지로 원인은 다양할 수 있으며 가장 가능성이 높은 원인으로는 CoreDNS <-> VPC DNS 구간 사이에서 네트워크 이슈입니다.

CoreDNS i/o timeout 의 원인을 추적하기 위해 5가지의 지표들이 네트워크 구간 사이 이슈를 확인하는 지표로 유용하게 활용됩니다.

따라서 다음의 방안으로 진행하면 각 지표를 낮출 수 있습니다.

1. linklocal_allowance_exceeded, pps_allowance_exceeded 지표는 CoreDNS 파드 대수를 늘려 많은 DNS 리졸브 요청을 분산시켜 지표를 낮추는 방안으로 진행합니다.

2. conntrack_allowance_exceeded 지표는 단일 인스턴스 대비 서비스 수를 줄이거나 커넥션을 과도하게 많이 사용하는 파드가 있는지 파악하여 지표를 낮추는 방안으로 진행합니다.

3. bw_in_allowance_exceeded, bw_out_allowance_exceeded 지표는 단일 인스턴스 대비 서비스 수를 줄이거나 네트워크 대역폭을 고려하여 인스턴스 타입 변경을 고려하여 지표를 낮추는 방안으로 진행합니다.

위의 방안으로 진행하기 위해서 결국은 운영 중인 환경에 맞게 서비스를 분산하고 단일 인스턴스 대비 워크로드를 줄이는 방안으로 고려해야합니다. (그만큼 리소스 사용량이 늘어날 수 있어 비용이 증가할 수 있습니다.)

추가로 확인할 수 있는 내용으로 VPC DNS 의 리졸버 쿼리로그를 수집하여 VPC DNS 까지 정상적으로 요청이 수신되고 처리하였는지 확인할 수 있습니다.

- [Log your VPC DNS queries with Route 53 Resolver Query Logs | Amazon Web Services](https://aws.amazon.com/ko/blogs/aws/log-your-vpc-dns-queries-with-route-53-resolver-query-logs/)

**참고 문서**

- [네트워크 성능 문제에 대한 EKS 워크로드 모니터링](https://docs.aws.amazon.com/ko_kr/eks/latest/best-practices/monitoring_eks_workloads_for_network_performance_issues.html)

- [AWS_VPC_K8S_CNI_EXTERNALSNAT](https://github.com/aws/amazon-vpc-cni-k8s?tab=readme-ov-file#aws_vpc_k8s_cni_externalsnat)

- [maxPods](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)

- [Amazon EC2 인스턴스 유형](https://aws.amazon.com/ko/ec2/instance-types/)

- [Troubleshooting AWS network throttling: A Comprehensive Guide](https://engineering.doit.com/troubleshooting-aws-network-throttling-a-comprehensive-guide-368811424148)
