---
marp: true
---

# CoreDNS lameduck 정리

> Pod는 Name 확인을 위해 `kube-dns` 서비스를 사용합니다. Kubernetes는 Destination NAT(DNAT)를 사용하여 `kube-dns` 트래픽을 Node에서 CoreDNS Pod로 리디렉션합니다.

> CoreDNS 배포를 확장하면 `kube-proxy` 가 Node에서 iptables 규칙과 체인을 업데이트하여 DNS 트래픽을 CoreDNS Pod로 리디렉션합니다.

> CoreDNS를 확장할 때 새 엔드포인트를 전파하고 축소할 때 규칙을 삭제하는 데는 클러스터 크기에 따라 1~10초가 걸릴 수 있습니다.

> 이 전파 지연은 CoreDNS Pod가 종료되었지만 Node의 iptables 규칙이 업데이트되지 않은 경우 DNS 조회 실패를 일으킬 수 있습니다. 이 시나리오에서 Node는 종료된 CoreDNS Pod에 DNS 쿼리를 계속 보낼 수 있습니다.

> CoreDNS Pod에서 [lameduck](https://coredns.io/plugins/health/) 기간을 설정하여 DNS 조회 실패를 줄일 수 있습니다.
> lameduck 모드에 있는 동안 CoreDNS는 진행 중인 요청에 계속 응답합니다. lameduck 기간을 설정하면 CoreDNS 종료 프로세스가 지연되어 Node가 iptables 규칙과 체인을 업데이트하는 데 필요한 시간을 확보할 수 있습니다.

따라서 CoreDNS lameduck 기간을 30초로 설정하는 것이 좋습니다.

- CoreDNS는 실제로 프로세스가 종료되기 전에 lameduck 기간을 기다립니다.
- [CoreDNS lameduck 기간](https://docs.aws.amazon.com/ko_kr/eks/latest/best-practices/scale-cluster-services.html#_coredns_lameduck_duration)

## Setting

> EKS Add ON으로 CoreDNS를 사용한다면, 다음과 같이 Corefile을 수정합니다. (lameduck을 5s -> 30s 변경)
> 참고 문서

- https://aws.amazon.com/ko/blogs/containers/amazon-eks-add-ons-advanced-configuration/

```
{
    "replicaCount": 2,
    "podDisruptionBudget": {
        "enabled": true,
        "maxUnavailable": 1
    },
    "corefile": ".:53 {\n    errors\n    health {\n        lameduck 30s\n    }\n    ready\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n        pods insecure\n        fallthrough in-addr.arpa ip6.arpa\n    }\n    prometheus :9153\n    forward . /etc/resolv.conf\n    cache 30\n    loop\n    reload\n    loadbalance\n}"
    ...
}
```

```
# AS-IS
apiVersion: v1
data:
  Corefile: |
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
kind: ConfigMap
metadata:
  labels:
    eks.amazonaws.com/component: coredns
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system

---

# TO-BE
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 30s
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
kind: ConfigMap
metadata:
  labels:
    eks.amazonaws.com/component: coredns
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
```

## Test

First Case

> CoreDNS의 Replica 10 -> 0, lameduck : 5s

- UTC 기준 07:14:37 정도에 CoreDNS의 Replica를 0으로 낮춘 후, lameduck에 설정한 5초 정도의 시간 동안은 정상적으로 프로세스가 동작합니다.

```
$ for i in {1..100000} ; do sleep .1 ; echo -n "$(date '+%Y-%m-%d %H:%M:%S') " ; dig +time=1 +tries=1 +short reviews.default.svc.cluster.local | grep -v 10.1 ; done
2025-02-14 07:14:37 172.20.56.223
2025-02-14 07:14:38 172.20.56.223
2025-02-14 07:14:38 172.20.56.223
2025-02-14 07:14:38 172.20.56.223
2025-02-14 07:14:38 172.20.56.223
2025-02-14 07:14:39 172.20.56.223
2025-02-14 07:14:39 172.20.56.223
2025-02-14 07:14:39 172.20.56.223
2025-02-14 07:14:39 172.20.56.223
2025-02-14 07:14:40 172.20.56.223
2025-02-14 07:14:40 172.20.56.223
2025-02-14 07:14:40 172.20.56.223
2025-02-14 07:14:41 172.20.56.223
2025-02-14 07:14:41 172.20.56.223
2025-02-14 07:14:41 172.20.56.223
2025-02-14 07:14:41 172.20.56.223
2025-02-14 07:14:42 172.20.56.223
2025-02-14 07:14:42 172.20.56.223
2025-02-14 07:14:42 ;; communications error to 172.20.0.10#53: timed out
;; no servers could be reached
2025-02-14 07:14:44 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 07:14:44 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 07:14:44 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 07:14:45 ;; communications error to 172.20.0.10#53: timed out
;; no servers could be reached
```

Second Case

> CoreDNS의 Replica 10 -> 0, lameduck : 30s

- UTC 기준 07:19:20 정도에 CoreDNS의 Replica를 0으로 낮춘 후, lameduck에 설정한 30초 정도의 시간 동안은 정상적으로 프로세스가 동작합니다.

```
$ for i in {1..100000} ; do sleep .1 ; echo -n "$(date '+%Y-%m-%d %H:%M:%S') " ; dig +time=1 +tries=1 +short reviews.default.svc.cluster.local | grep -v 10.1 ; done
2025-02-14 07:19:20 172.20.56.223
2025-02-14 07:19:20 172.20.56.223
2025-02-14 07:19:20 172.20.56.223
2025-02-14 07:19:20 172.20.56.223
2025-02-14 07:19:21 172.20.56.223
2025-02-14 07:19:21 172.20.56.223
2025-02-14 07:19:21 172.20.56.223
2025-02-14 07:19:22 172.20.56.223
...
2025-02-14 07:19:48 172.20.56.223
2025-02-14 07:19:49 172.20.56.223
2025-02-14 07:19:49 172.20.56.223
2025-02-14 07:19:49 172.20.56.223
2025-02-14 07:19:49 172.20.56.223
2025-02-14 07:19:50 172.20.56.223
2025-02-14 07:19:50 172.20.56.223
2025-02-14 07:19:50 172.20.56.223
2025-02-14 07:19:51 172.20.56.223
2025-02-14 07:19:51 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 07:19:52 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 07:19:52 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 07:19:52 ;; communications error to 172.20.0.10#53: timed out
;; no servers could be reached
2025-02-14 07:19:53 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
```

Third Case

> CoreDNS의 Replica 10 -> 0, lameduck : 30s > terminationGracePeriodSeconds : 20s

- UTC 기준 07:24:55 정도에 CoreDNS의 Replica를 0으로 낮춘 후, terminationGracePeriodSeconds에 설정한 20초 정도의 시간 동안은 정상적으로 프로세스가 동작합니다.
- 이때 lameduck에 설정한 30s는 무시됩니다.

```
$ for i in {1..100000} ; do sleep .1 ; echo -n "$(date '+%Y-%m-%d %H:%M:%S') " ; dig +time=1 +tries=1 +short reviews.default.svc.cluster.local | grep -v 10.1 ; done
2025-02-14 07:24:55 172.20.56.223
2025-02-14 07:24:55 172.20.56.223
2025-02-14 07:24:56 172.20.56.223
2025-02-14 07:24:56 172.20.56.223
2025-02-14 07:24:56 172.20.56.223
2025-02-14 07:24:57 172.20.56.223
2025-02-14 07:24:57 172.20.56.223
2025-02-14 07:24:57 172.20.56.223
2025-02-14 07:24:57 172.20.56.223
2025-02-14 07:24:57 172.20.56.223
...
2025-02-14 07:25:11 172.20.56.223
2025-02-14 07:25:12 172.20.56.223
2025-02-14 07:25:12 172.20.56.223
2025-02-14 07:25:12 172.20.56.223
2025-02-14 07:25:13 172.20.56.223
2025-02-14 07:25:13 172.20.56.223
2025-02-14 07:25:13 172.20.56.223
2025-02-14 07:25:13 172.20.56.223
2025-02-14 07:25:14 172.20.56.223
2025-02-14 07:25:14 172.20.56.223
2025-02-14 07:25:14 172.20.56.223
2025-02-14 07:25:15 172.20.56.223
2025-02-14 07:25:15 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 07:25:15 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 07:25:16 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
```

Fourth Case

> CoreDNS의 Replica 10 -> 0, lameduck : 30s =< terminationGracePeriodSeconds : 30s

- UTC 기준 08:15:17 정도에 CoreDNS의 Replica를 0으로 낮춘 후, lameduck, terminationGracePeriodSeconds에 설정한 30초 정도의 시간 동안은 정상적으로 프로세스가 동작합니다.

```
$ for i in {1..100000} ; do sleep .1 ; echo -n "$(date '+%Y-%m-%d %H:%M:%S') " ; dig +time=1 +tries=1 +short reviews.default.svc.cluster.local | grep -v 10.1 ; done
2025-02-14 08:15:17 172.20.56.223
2025-02-14 08:15:17 172.20.56.223
2025-02-14 08:15:17 172.20.56.223
2025-02-14 08:15:17 172.20.56.223
2025-02-14 08:15:17 172.20.56.223
2025-02-14 08:15:17 172.20.56.223
2025-02-14 08:15:17 172.20.56.223
2025-02-14 08:15:18 172.20.56.223
2025-02-14 08:15:18 172.20.56.223
2025-02-14 08:15:18 172.20.56.223
2025-02-14 08:15:18 172.20.56.223
...
2025-02-14 08:16:05 172.20.56.223
2025-02-14 08:16:06 172.20.56.223
2025-02-14 08:16:06 172.20.56.223
2025-02-14 08:16:06 172.20.56.223
2025-02-14 08:16:06 172.20.56.223
2025-02-14 08:16:06 172.20.56.223
2025-02-14 08:16:06 172.20.56.223
2025-02-14 08:16:07 172.20.56.223
2025-02-14 08:16:07 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 08:16:07 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 08:16:07 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 08:16:07 ;; communications error to 172.20.0.10#53: timed out
;; no servers could be reached
2025-02-14 08:16:08 s;; communications error to 172.20.0.10#53: timed out
;; no servers could be reached
2025-02-14 08:16:09 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
2025-02-14 08:16:10 ;; communications error to 172.20.0.10#53: timed out
;; no servers could be reached
2025-02-14 08:16:11 ;; communications error to 172.20.0.10#53: connection refused
;; no servers could be reached
```

## 결론

결론적으로, lameduck 설정 시 Pod 단에서 추가 설정 (preStop, terminationGracePeriodSeconds, ...) 다음과 같이 설정하면 안정적으로 운영될 수 있습니다.

- lameduck : 30s
- terminationGracePeriodSeconds : 30s
- preStop : 5s
