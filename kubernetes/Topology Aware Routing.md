# Topology Aware Routing

## 개요

> AWS에서 복원력 있는 시스템을 구축하는 가장 좋은 방법은 여러 가용 영역(AZ)을 활용하는 것입니다. 이러한 전략은 안정성을 향상시키지만, 특히 가용 영역 간 트래픽으로 인해 비용이 증가하게 됩니다.

> `Topology Aware Routing`은 트래픽을 동일한 AZ 내 또는 인근 위치 등, 원점에 더 가깝게 유지하는 것을 목표로 합니다. `Topology Aware Routing`을 통해 Multi AZ 에 대한 통신을 최소화하여 AWS DataTransfer 비용을 절약할 수 있고, 네트워크 홉도 줄일 수 있습니다. 이를 반영해보기 위해 `Topology Aware Routing`에 대해서 조사합니다.

## Topology Aware Routing 란?

`Topology Aware Routing`은 네트워크 트래픽이 발생한 Zone 내에 머무를 수 있도록 돕는 메커니즘을 제공합니다. 

클러스터 내에서 Pod 간 트래픽이 동일한 존 안에서 이루어지도록 우선 처리하면, 신뢰성, 성능(네트워크 지연 시간 및 처리량) 또는 비용 측면에서 이점을 얻을 수 있습니다.

Kubernetes `1.27` 이전에는 이 기능을 `Topology Aware Hint` 라고 불렀습니다.

## Topology Aware Routing 동작 방식

"Auto" 휴리스틱(heuristic)은 각 가용 영역(Zone)마다 엔드포인트를 비례적으로 할당하려고 시도합니다.
단, 이 휴리스틱은 엔드포인트 수가 충분히 많은 서비스에서 가장 잘 작동합니다.

### EndpointSlice Controller
EndpointSlice Controller는 이 휴리스틱이 활성화된 경우, EndpointSlice에 Hints를 설정하는 역할을 담당합니다.

EndpointSlice Controller는 각 Zone에 비례적으로 엔드포인트를 할당합니다.
이 비율은 각 Zone에 존재하는 노드들의 할당 가능한(Allocatable) CPU 코어 수를 기반으로 계산됩니다.
- EX)
  - 어떤 Zone이 2개의 CPU 코어를 가지고 있고, 다른 Zone이 1개의 CPU 코어만 가지고 있다면, EndpointSlice Controller는 CPU 코어가 2개인 Zone에 2배 많은 엔드포인트를 할당하게 됩니다.

### kube-proxy
kube-proxy는 EndpointSlice Controller가 설정한 Hints를 기반으로, 트래픽을 라우팅할 엔드포인트를 필터링합니다.

이로 인해 kube-proxy는 같은 Zone에 있는 엔드포인트로 트래픽을 라우팅할 수 있게 됩니다.

하지만 경우에 따라, EndpointSlice Controller는 Zone 간 엔드포인트 분포를 보다 고르게 유지하기 위해, 다른 Zone에 있는 엔드포인트를 할당할 수도 있습니다.

이러한 경우에는, 일부 트래픽이 다른 Zone으로 라우팅되는 결과가 발생할 수 있습니다.


## Topology Aware Routing 반영 방법

- service.kubernetes.io/topology-mode: Auto (기존에 Topology Aware Hint에서부터 지원하던 방식) 
- trafficDistribution: PreferClose (1.33 부터 공식지원하는 방식)

`1.33 이전 버전일 경우` Service에 대해 Topology Aware Routing을 활성화하려면, service.kubernetes.io/topology-mode annotation을 Auto로 설정하면 됩니다.

```yaml
metadata:
  annotations:
    service.kubernetes.io/topology-mode: Auto
```

EndpointSlice 예시
```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-hints
  labels:
    kubernetes.io/service-name: example-svc
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    zone: zone-a
    hints:
      forZones:
        - name: "zone-a"
```

`1.33 이후 버전일 경우` Service에 대해 Topology Aware Routing을 활성화하려면, trafficDistribution: PreferClose를 설정하면 됩니다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-svc
  namespace: example-test
spec:
  type: ClusterIP
  selector:
    app: example-app
  ports:
  - port: 80
    targetPort: 5678
  trafficDistribution: PreferClose
```

## Topology Aware Routing 반영 시 고려 사항
1. Zone 간 Pod 개수가 불균등할 경우에 트래픽의 불균형이 발생할 수 있습니다.


Zone 간 균등 배분을 하기 위한 설정

Zone 간 균등 배분을 하기 위해 다음의 설정들을 수행했다.

1. Topology Spread Constraints

Topology Spread Constraints는 DoNotSchedule (충족되지 않을 경우 Pod를 Pending state로 만들기) , zone 설정을 추가해줬다.

2. Scale Down 시에 Topology Spread Constraint 설정을 무시하며 Random하게 Pod를 제거하기에 이를 다시 맞춰주기 위한 Descheduler

Descheduler는 문제가 터지고나서 도입한건데, Scheduling의 반대인 De-Scheduling을  수행해주는 오픈소스이다.

Descheduler를 써야 하는 이유는 Pod가 스케일 다운 될 때 AZ간 균등 배분 설정을 무시하기 때문이다. (중요)

따라서 A 존에 3대, C 존에 2대 였던 파드가 Scale down 되어서 1대가 줄어든다면 A존의 Pod가 줄어들지, C 존의 Pod가 줄어들지 모른다는 의미이다.

따라서 A 존의 Pod가 3대, C 존의 Pod가 1대라면 Descheduler가 이를 인지하고 A 존의 Pod 1대를 C 존으로 옮겨줘야 Zone 간 균등 배포 설정이 맞춰질것이다.


3. Service 리소스의 internalTrafficPolicy가 Local인 경우 Topology Aware Routing은 동작하지 않으므로 주의합니다. 기본값은 Cluster 입니다


## AZ 간 트래픽 모니터링을 위한 방안
Multi AZ 간 트래픽을 모니터링 하기 위해 Microsoft 에서 제공하는 [`Retina`](https://github.com/microsoft/retina/tree/main) 를 활용합니다.

### Retina 란?
> Retina는 기본 OS나 CNI에 관계없이 Hubble을 제어 평면으로 사용할 수 있는 클라우드 독립적인 오픈 소스 Kubernetes 네트워크 관찰 플랫폼 입니다. 풀어서 이야기하면, Retina는 애플리케이션 및 네트워크 상태를 모니터링하는 중앙 허브를 제공합니다.

Retina는 사용자 정의 가능한 원격 측정 데이터를 수집하여 여러 스토리지 옵션 (예: Prometheus, Azure Monitor 등) 으로 내보내고, 다양한 방식 (예: Grafana, Azure Log Analytics 등)으로 시각화할 수 있습니다.

Retina를 사용하면 필요에 따라 네트워크 문제를 조사하고 클러스터를 지속적으로 모니터링 할 수 있습니다.

레티나 측정항목은 다음 사항에 대한 지속적인 관찰을 제공합니다.
1. 유입/유출 트래픽
2. 삭제된 패킷
3. TCP/UDP
4. DNS
5. API 서버 대기 시간
6. 노드/인터페이스 통계

Retina는 다음 두 가지를 모두 제공합니다.
1. 기본 메트릭 - 노드 수준(기본값)
2. 고급 메트릭 - Pod 수준(활성화된 경우)

자세한 내용은 [캡처](https://retina.sh/docs/Metrics/modes/)를 참조하시면 됩니다.

### Retina Architecture

#### Data Plane
- Plugin Lifecycle


#### Control Plane
- Hubble Control Plane
- Standard Control Plane



### Retina를 활용한 AZ 간 통신 메트릭 확인
Retina를 활용하여 Kubernetes 에서의 Pod 간 통신 시, AZ 간 통신에 대한 메트릭을 확인해봅니다.
이때 Retina에서 두가지의 메트릭([Hubble Metrics](https://retina.sh/docs/Metrics/hubble_metrics), [Standard Metric Modes](https://retina.sh/docs/Metrics/modes/))을 제공하는데, AZ 간 통신 메트릭을 확인하기 위해 Hubble Metrics를 활용하기로 했습니다.

Retina의 Hubble Metrics 을 활용하기 위해 [Helm Charts](https://github.com/microsoft/retina/tree/main/deploy/hubble/manifests/controller/helm/retina) 를 사용했습니다.

#### Retina 배포

Values File 수정
1. Image Registry 및 Tags 변경 
- acndev.azurecr.io -> ghcr.io 로 변경
- latest -> v0.0.31

2. Config 변경
- remoteContext: false -> true
- bypassLookupIPOfInterest: true -> false
- enableConntrackMetrics: true 추가
- dataAggregationLevel: "high" -> "low"

3. Hubble Metrics 수정
```diff
hubble:
  metrics:
    enabled:
-      - flow:sourceEgressContext=pod;destinationIngressContext=pod
+      - flow:sourceEgressContext=pod;destinationIngressContext=pod;labelsContext=source_pod,source_ip,source_namespace,destination_pod,destination_ip,destination_namespace,traffic_direction
      - tcp:sourceEgressContext=pod;destinationIngressContext=pod
-      - dns:query;sourceEgressContext=pod;destinationIngressContext=pod
+      - dns:query;sourceEgressContext=pod;destinationIngressContext=pod;labelsContext=source_pod,destination_pod,traffic_direction
      - drop:sourceEgressContext=pod;destinationIngressContext=pod 
```

#### Prometheus를 통한 Retina 메트릭 수집
`kube-prometheus-stack` 을 기반으로 메트릭을 수집하고 있어, 이를 활용하여 Retina 메트릭을 수집합니다.

additionalScrapeConfigs 에 다음을 추가하여 메트릭을 활성화시킵니다.
```yaml
additionalScrapeConfigs:
    - job_name: networkobservability-hubble
    kubernetes_sd_configs:
        - role: pod
    relabel_configs:
        - target_label: cluster
        replacement: myCluster
        action: replace
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_pod_label_k8s_app]
        regex: kube-system;(retina)
        action: keep
        - source_labels: [__address__]
        action: replace
        regex: ([^:]+)(?::\d+)?
        replacement: $1:9965
        target_label: __address__
        - source_labels: [__meta_kubernetes_pod_node_name]
        target_label: instance
        action: replace
    - job_name: retina-pods
    kubernetes_sd_configs:
        - role: pod
    relabel_configs:
        - source_labels: [__meta_kubernetes_pod_container_name]
        action: keep
        regex: retina
        - source_labels:
            [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        separator: ":"
        regex: ([^:]+)(?::\d+)?
        target_label: __address__
        replacement: ${1}:${2}
        action: replace
        - source_labels: [__meta_kubernetes_pod_node_name]
        action: replace
        target_label: instance
```

#### 결과
> Retina Hubble Metrics 를 활성화하게 되면 아래의 4개에 대한 promql 을 조회할 수 있습니다.
- hubble_dns_queries_total
- hubble_dns_responses_total
- hubble_flows_processed_total
- hubble_tcp_flags_total

참고)
많은 양의 메트릭을 Grafana 대시보드에서 보여주기 무리가 된다면, Prometheus 의 `Recoding Rules` 를 활용하면 됩니다.

```yaml
additionalPrometheusRulesMap:
 inter-az-traffic-analysis:
   groups:
   - name: inter_az_traffic_analysis.rules
     interval: 1m
     rules:
     - record: pod:source_kube_pod_info_mapping
       expr: |
        label_replace(
          label_replace(
            kube_pod_info{pod_ip!=""}, 
            "source_pod", "$1", "pod", "(.*)"
          ),
          "source_ip", "$1", "pod_ip", "(.*)"
        )
     - record: pod:destination_kube_pod_info_mapping
       expr: |
        label_replace(
          label_replace(
            kube_pod_info{pod_ip!=""}, 
            "destination_pod", "$1", "pod", "(.*)"
          ),
          "destination_ip", "$1", "pod_ip", "(.*)"
        )
     - record: node:kube_node_info_mapping
       expr: |
        label_replace(
          sum by (node, provider_id) (kube_node_info),
          "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
        )
     - record: node:source_kube_node_info_mapping
       expr: |
        label_replace(
          sum by (node, provider_id) (kube_node_info),
          "source_zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
        )
     - record: node:destination_kube_node_info_mapping
       expr: |
        label_replace(
          sum by (node, provider_id) (kube_node_info),
          "destination_zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
        )
     - record: traffic:hubble_flows_processed_total:irate5m
       expr: |
        irate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[5m])
```

#### AS-IS
- Total Traffic AZ
```
sum(
    (
        irate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[5m])
        * on (source_pod, source_ip) group_left(node)
        label_replace(
          label_replace(
            kube_pod_info{pod_ip!=""}, "source_pod", "$1", "pod", "(.*)"
          ),
          "source_ip", "$1", "pod_ip", "(.*)"
        )
        * on (node) group_left(zone)
        label_replace(
            sum by (node, provider_id) (kube_node_info),
            "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
        )
    )
    * on (destination_pod, destination_ip) group_left(node)
    (
        label_replace(
          label_replace(
            kube_pod_info{pod_ip!=""}, "destination_pod", "$1", "pod", "(.*)"
          ),
          "destination_ip", "$1", "pod_ip", "(.*)"
        )
        * on (node) group_left(zone)
        label_replace(
            sum by (node, provider_id) (kube_node_info),
            "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
        )
    )
)OR on() vector(0)
```

- Same AZ
```
sum(
    ceil(
        (
          sum(
              rate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[$__rate_interval]) 
              * on (source_pod, source_ip) group_left(node)
              label_replace(
                label_replace(
                  kube_pod_info{pod_ip!=""}, "source_pod", "$1", "pod", "(.*)"
                ),
                "source_ip", "$1", "pod_ip", "(.*)"
              )
              * on (node) group_left(zone)
              label_replace(
                  sum by (node, provider_id) (kube_node_info), 
                  "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
              )
          ) by (destination_pod, source_pod, zone)
        )
        + on (destination_pod, source_pod, zone) group_left() # By including the zone as a join criterion label, only flows where both the source and destination are in the same zone are summed.
        (
          sum(
              rate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[$__rate_interval]) 
              * on (destination_pod, destination_ip) group_left(node)
                label_replace(
                  label_replace(
                    kube_pod_info{pod_ip!=""}, "destination_pod", "$1", "pod", "(.*)"
                  ),
                  "destination_ip", "$1", "pod_ip", "(.*)"
                )

              * on (node) group_left(zone)
                label_replace(
                    sum by (node, provider_id) (kube_node_info), 
                    "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
                )
          ) by (destination_pod, source_pod, zone)
        )
      ) / 2  # we are divinding because we are summing values in '+on()' function to find the Same AZ traffic
)OR on() vector(0)
```

- Cross AZ
```
(
    sum(
    (
        rate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[$__rate_interval])
        * on (source_pod, source_ip) group_left(node)
        label_replace(
          label_replace(
            kube_pod_info{pod_ip!=""}, "source_pod", "$1", "pod", "(.*)"
          ),
          "source_ip", "$1", "pod_ip", "(.*)"
        )
        * on (node) group_left(zone)
        label_replace(
            sum by (node, provider_id) (kube_node_info),
            "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
        )
    )
    * on (destination_pod, destination_ip) group_left(node)
    (
        label_replace(
          label_replace(
            kube_pod_info{pod_ip!=""}, "destination_pod", "$1", "pod", "(.*)"
          ),
          "destination_ip", "$1", "pod_ip", "(.*)"
        )
        * on (node) group_left(zone)
        label_replace(
            sum by (node, provider_id) (kube_node_info),
            "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
        )
    )
    )OR on() vector(0)
)
-
(
    sum(
        ceil(
            (
            sum(
                rate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[$__rate_interval]) 
                * on (source_pod, source_ip) group_left(node)
                label_replace(
                  label_replace(
                    kube_pod_info{pod_ip!=""}, "source_pod", "$1", "pod", "(.*)"
                  ),
                  "source_ip", "$1", "pod_ip", "(.*)"
                ) 
                * on (node) group_left(zone)
                label_replace(
                    sum by (node, provider_id) (kube_node_info), 
                    "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
                )
            ) by (destination_pod, source_pod, zone)
            )
            + on (destination_pod, source_pod, zone) group_left() # By including the zone as a join criterion label, only flows where both the source and destination are in the same zone are summed.
            (
            sum(
                rate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[$__rate_interval]) 
                * on (destination_pod, destination_ip) group_left(node)
                label_replace(
                  label_replace(
                    kube_pod_info{pod_ip!=""}, "destination_pod", "$1", "pod", "(.*)"
                  ),
                  "destination_ip", "$1", "pod_ip", "(.*)"
                )
                * on (node) group_left(zone)
                    label_replace(
                        sum by (node, provider_id) (kube_node_info), 
                        "zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
                    )
            ) by (destination_pod, source_pod, zone)
            )
        ) / 2  # we are divinding because we are summing values in '+on()' function to find the Same AZ traffic
    )OR on() vector(0)
)
```

- Traffic per AZ
```
sum by (source_zone, destination_zone) (
  rate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[$__rate_interval])
  * on (source_pod, source_ip) group_left(node)
    label_replace(
      label_replace(
        kube_pod_info{pod_ip!=""}, "source_pod", "$1", "pod", "(.*)"
      ),
      "source_ip", "$1", "pod_ip", "(.*)"
    )
  * on (node) group_left(source_zone)
    label_replace(
      sum by (node, provider_id) (kube_node_info),
      "source_zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
    )
  * on (destination_pod, destination_ip) group_left(node)
    label_replace(
      label_replace(
        kube_pod_info{pod_ip!=""}, "destination_pod", "$1", "pod", "(.*)"
      ),
      "destination_ip", "$1", "pod_ip", "(.*)"
    )
  * on (node) group_left(destination_zone)
    label_replace(
      sum by (node, provider_id) (kube_node_info),
      "destination_zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
    )
)
```

- Traffic per AZ (Pod)
```
sum(
  (
    rate(hubble_flows_processed_total{traffic_direction="egress", protocol!="DNS"}[$__rate_interval])
    * on (source_pod, source_ip) group_left(node)
      label_replace(
        label_replace(
          kube_pod_info{pod_ip!=""}, "source_pod", "$1", "pod", "(.*)"
        ),
        "source_ip", "$1", "pod_ip", "(.*)"
      )
    * on (node) group_left(source_zone)
      label_replace(
        sum by (node, provider_id) (kube_node_info), 
        "source_zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
      )
    * on (destination_pod, destination_ip) group_left(node)
      label_replace(
        label_replace(
          kube_pod_info{pod_ip!=""}, "destination_pod", "$1", "pod", "(.*)"
        ),
        "destination_ip", "$1", "pod_ip", "(.*)"
      )
    * on (node) group_left(destination_zone)
      label_replace(
        sum by (node, provider_id) (kube_node_info), 
        "destination_zone", "$1", "provider_id", "aws:///([a-z0-9-]+)/.*"
      )
  )
) 
by (source_pod, destination_pod, source_zone, destination_zone)
```


#### TO-BE
- Total Traffic AZ
```
sum(
    (
      traffic:hubble_flows_processed_total:rate5m
      * on (source_pod, source_ip) group_left(node)
      pod:source_kube_pod_info_mapping
      * on (node) group_left(zone)
      node:kube_node_info_mapping
    )
    * on (destination_pod, destination_ip) group_left(node)
    (
      pod:destination_kube_pod_info_mapping
      * on (node) group_left(zone)
      node:kube_node_info_mapping
    )
) OR on() vector(0)
```

- Same AZ
```
sum(
    ceil(
        (
          sum(
              traffic:hubble_flows_processed_total:rate5m
              * on (source_pod, source_ip) group_left(node)
              pod:source_kube_pod_info_mapping
              * on (node) group_left(zone)
              node:kube_node_info_mapping
          ) by (destination_pod, source_pod, zone)
        )
        + on (destination_pod, source_pod, zone) group_left() # By including the zone as a join criterion label, only flows where both the source and destination are in the same zone are summed.
        (
          sum(
              traffic:hubble_flows_processed_total:rate5m
              * on (destination_pod, destination_ip) group_left(node)
              pod:destination_kube_pod_info_mapping
              * on (node) group_left(zone)
              node:kube_node_info_mapping
          ) by (destination_pod, source_pod, zone)
        )
    ) / 2  # we are divinding because we are summing values in '+on()' function to find the Same AZ traffic
)OR on() vector(0)
```

- Cross AZ
```
(
  sum(
      (
        traffic:hubble_flows_processed_total:rate5m
        * on (source_pod, source_ip) group_left(node)
        pod:source_kube_pod_info_mapping
        * on (node) group_left(zone)
        node:kube_node_info_mapping
      )
      * on (destination_pod, destination_ip) group_left(node)
      (
        pod:destination_kube_pod_info_mapping
        * on (node) group_left(zone)
        node:kube_node_info_mapping
      )
  ) OR on() vector(0)
)
-
(
  sum(
      ceil(
          (
            sum(
                traffic:hubble_flows_processed_total:rate5m
                * on (source_pod, source_ip) group_left(node)
                pod:source_kube_pod_info_mapping
                * on (node) group_left(zone)
                node:kube_node_info_mapping
            ) by (destination_pod, source_pod, zone)
          )
          + on (destination_pod, source_pod, zone) group_left() # By including the zone as a join criterion label, only flows where both the source and destination are in the same zone are summed.
          (
            sum(
                traffic:hubble_flows_processed_total:rate5m
                * on (destination_pod, destination_ip) group_left(node)
                pod:destination_kube_pod_info_mapping
                * on (node) group_left(zone)
                node:kube_node_info_mapping
            ) by (destination_pod, source_pod, zone)
          )
      ) / 2  # we are divinding because we are summing values in '+on()' function to find the Same AZ traffic
  )OR on() vector(0)
)
```

- Traffic per AZ
```
sum by (source_zone, destination_zone) (
  traffic:hubble_flows_processed_total:rate5m
  * on (source_pod, source_ip) group_left(node)
  pod:source_kube_pod_info_mapping
  * on (node) group_left(source_zone)
  node:source_kube_node_info_mapping
  * on (destination_pod, destination_ip) group_left(node)
  pod:destination_kube_pod_info_mapping
  * on (node) group_left(destination_zone)
  node:destination_kube_node_info_mapping
)
```

- Traffic per AZ (Pod)
```
sum(
  (
    traffic:hubble_flows_processed_total:rate5m
    * on (source_pod, source_ip) group_left(node)
    pod:source_kube_pod_info_mapping
    * on (node) group_left(source_zone)
    node:source_kube_node_info_mapping
    * on (destination_pod, destination_ip) group_left(node)
    pod:destination_kube_pod_info_mapping
    * on (node) group_left(destination_zone)
    node:destination_kube_node_info_mapping
  )
)
by (source_pod, destination_pod, source_zone, destination_zone)
```

#### Retina (Hubble Metrics) 구축 시 이슈

retina-agent 중 CPU 를 많이 사용하는 케이스가 있어 확인해보니 로그에 아래와 같이 있었습니다.

```
time=2025-05-09T07:14:09.413Z level=WARN msg="hubble events queue is full: dropping messages; consider increasing the queue size (hubble-event-queue-size) or provisioning more CPU" module=agent.control-plane related-metric=hubble_lost_events_total
```

Hubble Events Queue 가 가득차, Queue Size 를 늘리거나 CPU 를 더 사용할 수 있도록 하라고 되어있습니다.
CPU 를 늘리는 것이 쉬운 방법일 수 있지만, Queue Size 를 늘려보는 방식으로 진행해봅니다.

우선 retina-agent 의 resource 는 다음과 같습니다.

```
Limits:
  cpu:     500m
  memory:  500Mi
Requests
  cpu:     500m
  memory:  500Mi
```

이때 최대 500m Core 까지 사용하는 경우가 있었습니다.

Values.yaml 에 보면 Queue Size 를 조정할 수 있는 부분이 있습니다.

```yaml
hubble:
  # -- Enable Hubble (true by default).
  enabled: true

  # -- Annotations to be added to all top-level hubble objects (resources under templates/hubble)
  annotations: {}

  # -- Buffer size of the channel Hubble uses to receive monitor events. If this
  # value is not set, the queue size is set to the default monitor queue size.
  eventQueueSize: "16383"

  # -- Number of recent flows for Hubble to cache. Defaults to 4095.
  # Possible values are:
  #   1, 3, 7, 15, 31, 63, 127, 255, 511, 1023,
  #   2047, 4095, 8191, 16383, 32767, 65535
  eventBufferCapacity: "4095"
```

eventQueueSize와 eventBufferCapacity를 두배로 늘려 대응합니다.
- eventQueueSize: 16383 -> 32767
- eventBufferCapacity: 4095 -> 8191

반영 후 CPU 사용이 낮아 진 것을 확인하였습니다.
- 최대 500m Core -> 100m Core 이하



promql 에서 execution: found duplicate series for the match group 이 발생하여 쿼리 조회가 안돼는 현상 발생이 발생했습니다..

```
{source_pod="loki-ingester-2"} on the right hand-side of the operation: [{__name__="kube_pod_info", container="kube-state-metrics", created_by_kind="StatefulSet", created_by_name="loki-ingester", endpoint="http", host_ip="10.241.61.89", host_network="false", instance="10.241.61.181:8080", job="kube-state-metrics", namespace="loki", node="ip-10-241-61-89.ap-northeast-2.compute.internal", pod="loki-ingester-2", pod_ip="10.241.63.202", service="kube-prometheus-stack-kube-state-metrics", source_pod="loki-ingester-2", uid="f78fa615-919f-45f3-90e2-c63f6cad701d"}, {__name__="kube_pod_info", container="kube-state-metrics", created_by_kind="StatefulSet", created_by_name="loki-ingester", endpoint="http", host_ip="10.241.42.51", host_network="false", instance="10.241.61.181:8080", job="kube-state-metrics", namespace="loki", node="ip-10-241-42-51.ap-northeast-2.compute.internal", pod="loki-ingester-2", pod_ip="10.241.43.180", service="kube-prometheus-stack-kube-state-metrics", source_pod="loki-ingester-2", uid="bb93457f-7143-4e51-aba2-010cc8bb6c4d"}] ;many-to-many matching not allowed: matching labels must be unique on one side
```

### 분석

조인 대상
```
... on (source_pod) 
```
해당 쿼리는 on(source_pod)을 기준으로 조인을 시도하고 있습니다.

하지만 우변(right-hand side)에 해당하는 kube_pod_info metric에는 다음과 같은 source_pod="loki-ingester-2" 값을 가지는 두 개의 시계열이 존재합니다:

```
1. host_ip="10.241.61.89", pod_ip="10.241.63.202", uid="f78f..."
2. host_ip="10.241.42.51", pod_ip="10.241.43.180", uid="bb93..." 
```
즉, source_pod="loki-ingester-2"인 시계열이 2개 이상 존재하는 것이 문제입니다.

문제 이유

Prometheus는 binary operation을 수행할 때 on(source_pod) 라는 조건만으로 두 시계열을 매칭시킵니다.

그러나 지금과 같이 source_pod="loki-ingester-2"로 매칭될 수 있는 시계열이 우변에 2개 이상 존재하면, 좌변의 동일한 source_pod="loki-ingester-2"를 가진 시계열과 다대다(many-to-many) 관계가 형성됩니다.

이것은 Prometheus에서 허용되지 않습니다.

- 좌변의 1개 시계열 : 우변의 정확히 1개 시계열 → ✅ OK

- 좌변의 1개 시계열 : 우변의 2개 이상 시계열 → ❌ 오류 발생

따라서 이러한 중복을 피하기 위하고 유일성을 확보하기 위해 label 을 추가합니다.
```
* on (source_pod, source_ip) group_left(node)
```
단, 조인 대상이 되는 라벨(source_ip 등)이 좌변에도 있어야 합니다. 없을 경우 label_replace() 등을 써서 동일한 라벨을 만들 수 있습니다.


AS-IS
```
* on (source_pod) group_left(node)
...
label_replace(
  kube_pod_info{pod_ip!=""}, "source_pod", "$1", "pod", "(.*)"
),
```


TO-BE
```
* on (source_pod, source_ip) group_left(node)
...
label_replace(
  label_replace(
    kube_pod_info{pod_ip!=""}, "source_pod", "$1", "pod", "(.*)"
  ),
  "source_ip", "$1", "pod_ip", "(.*)"
)
```



**참고 문서**
- [Implementing Topology Aware Routing in Kubernetes](https://medium.com/@j.aslanov94/implementing-topology-aware-routing-in-kubernetes-237abacd70ed)
- [Monitoring Inter-Pod Traffic at the AZ Level with Retina (an eBPF based tool)](https://medium.com/@j.aslanov94/monitoring-inter-pod-traffic-at-the-az-level-with-ebpf-based-tool-retina-7a79818e305b)



