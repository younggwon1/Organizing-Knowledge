# DNS Query Failed 이슈

운영 중인 서비스의 Logs 에 `io.netty.resolver.dns.DnsResolveContext$SearchDomainUnknownHostException` 에러가 발생하였고,
원인을 추측해보았을 때 다음과 같이 정리하였습니다.

1. 해당 Node 에는 CoreDNS 가 존재하지 않음
   - 다른 Node 의 CoreDNS 로 DNS Query 요청을 보내야한다는 의미
2. `io.netty.resolver.dns.DnsResolveContext$SearchDomainUnknownHostException` 해당 이슈는 운영 서비스 -> kube-dns (service) 로 Domain Name Resolving을 시도하는 부분에서 문제가 발생했을 것으로 추측
   - 어떤 Pod가 DNS Query를 수행하기 위해서는 CoreDNS Pod로 요청을 전송해야하는데 해당 Node 에는 CoreDNS 가 존재하지 않기 때문에 다른 Node 의 CoreDNS 로 DNS Query 요청을 보내야 함
   - 이때 너무 많은 DNS Query 를 수행하게 될 경우 Node 내 패킷 구성에 문제가 발생하여 위와 같은 이슈가 발생할 수 있음

따라서 DNS Query 요청을 줄이기 위한 방안을 고려해보았습니다.

아래의 글부터 어떻게 접근하였는지를 보여줍니다.

### /etc/resolv.conf가 생성되는 원리

각 Node 마다 동작하고 있는 kubelet은 pod가 실행될 때, /etc/resolv.conf 파일 안에 CoreDNS의 Service IP를 Nameserver로 등록합니다.
/etc/resolv.conf 설정 파일은 DNS를 질의하는 클라이언트가 사용할 Nameserver를 지정해둔 파일입니다.

대부분의 워크로드의 경우 ndots를 기본값 5가 아닌 2로 설정하면 충분하고, [EKS Best Practice 문서](https://aws.github.io/aws-eks-best-practices/scalability/docs/cluster-services/#scale-coredns)에도 권장합니다.

#### /etc/resolv.conf 구성

```bash
$ cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
nameserver 172.20.0.10
options ndots:5
```

설명

- search default.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
  - search 도메인 목록
- nameserver
  - dns server ip
- options ndots:5
  - FDQN으로 인지하기 시작하는 .(dot)의 개수
  - 기본값은 5

### ndots 의 개수에 따른 질의 방식

> 도메인의 . 개수가 /etc/resolv.conf 설정되어 있는 ndots 개수보다 작으면 search 도메인을 추가하여 순차적으로 질의하고, ndots 개수보다 크면 FQDN으로 간주하여 바로 질의합니다.

EX) ndots 2 로 설정할 경우

- 해당 질의에서 도메인의 . 개수가 설정되어있는 ndots(2)보다 작으면 search 도메인을 추가하여 순차적으로 질의합니다.

- 해당 질의에서 도메인의 . 개수가 설정되어있는 ndots(2)보다 크거나 같으면 FQDN으로 간주하여 바로 질의합니다.

#### 테스트

> nslookup 과 tcpdump 를 사용할 수 있도록 설치합니다.

우분투(Ubuntu)를 포함안 데비안(Debian)계열의 리눅스 환경에서 테스트하는 것이기에 다음의 명령어를 수행하여 nslookup, tcpdump를 설치합니다.

```bash
$ apt-get update
$ apt-get install dnsutils -y
$ apt-get install tcpdump -y
```

nslookup 시 UDP 포트 53(=DNS 요청/응답) 트래픽을 숫자로 변환된 형태(IP 주소, 포트 번호)로 캡처하여 출력하기 위해 다음의 명령어를 클라이언트 쪽에 수행합니다.

```bash
$ tcpdump -nn udp port 53
```

또는, 다음의 명령어를 수행해도 좋습니다.

```bash
$ tcpdump -i eth0 src {client ip} and udp port 53
$ tcpdump -i eth0 -vv -nn host {dns service ip}
```

그리고 나서 클라이언트 쪽에 nslookup 명령어를 수행하여 패킷을 확인해봅니다.

```bash
$ nslookup {Domain} -debug | grep QUESTIONS -A 1
```

결과

```
# nslookup google.com 시 tcpdump 결과 -> 5번 질의
08:58:46.168032 IP 10.119.39.47.51789 > 172.20.0.10.53: 20000+ A? google.com.pfops.svc.cluster.local. (52)
08:58:46.169232 IP 172.20.0.10.53 > 10.119.39.47.51789: 20000 NXDomain*- 0/1/0 (145)
08:58:46.169379 IP 10.119.39.47.47823 > 172.20.0.10.53: 45767+ A? google.com.svc.cluster.local. (46)
08:58:46.170410 IP 172.20.0.10.53 > 10.119.39.47.47823: 45767 NXDomain*- 0/1/0 (139)
08:58:46.170505 IP 10.119.39.47.37861 > 172.20.0.10.53: 2008+ A? google.com.cluster.local. (42)
08:58:46.171557 IP 172.20.0.10.53 > 10.119.39.47.37861: 2008 NXDomain*- 0/1/0 (135)
08:58:46.171661 IP 10.119.39.47.35307 > 172.20.0.10.53: 17957+ A? google.com.ap-northeast-2.compute.internal. (60)
08:58:46.172824 IP 172.20.0.10.53 > 10.119.39.47.35307: 17957 NXDomain 0/1/0 (183)
08:58:46.172901 IP 10.119.39.47.36808 > 172.20.0.10.53: 23432+ A? google.com. (28)
08:58:46.173821 IP 172.20.0.10.53 > 10.119.39.47.36808: 23432 1/0/0 A 142.250.206.238 (54)
08:58:46.173927 IP 10.119.39.47.44292 > 172.20.0.10.53: 39520+ AAAA? google.com. (28)
08:58:46.175105 IP 172.20.0.10.53 > 10.119.39.47.44292: 39520 1/0/0 AAAA 2404:6800:400a:804::200e (66)

# nslookup www.google.com 시 tcpdump 결과 -> 1번 질의
08:58:52.501675 IP 10.119.39.47.55615 > 172.20.0.10.53: 13178+ A? www.google.com. (32)
08:58:52.502510 IP 172.20.0.10.53 > 10.119.39.47.55615: 13178 1/0/0 A 172.217.161.228 (62)
08:58:52.502744 IP 10.119.39.47.51748 > 172.20.0.10.53: 43126+ AAAA? www.google.com. (32)
08:58:52.504900 IP 172.20.0.10.53 > 10.119.39.47.51748: 43126 1/0/0 AAAA 2404:6800:400a:804::2004 (74)
```

최대 5번의 질의를 1번으로 줄임으로써 불필요한 DNS 질의를 줄이고 클러스터 내부의 통신 지연을 줄일 수 있습니다.

또한, **Kubernetes** 내에서의 호출도 마찬가지입니다.

`ndots(2)` 로 설정할 경우

- kubernetes
  - 동일 네임스페이스에 있을 경우
    - {서비스 이름} -> 1번 질의
    - {서비스 이름}.{네임스페이스} -> 2번 질의
  - 다른 네임스페이스에 있는 경우
    - {서비스 이름}.{네임스페이스}.svc.cluster.local -> 1번 질의
    - {서비스 이름}.{네임스페이스} -> 2번 질의

`ndots(5)` 로 설정할 경우

- kubernetes
  - 동일 네임스페이스에 있을 경우
    - {서비스 이름} -> 1번 질의
    - {서비스 이름}.{네임스페이스} -> 2번 질의
  - 다른 네임스페이스에 있는 경우
    - {서비스 이름}.{네임스페이스}.svc.cluster.local -> 5번 질의
    - {서비스 이름}.{네임스페이스} -> 2번 질의

따라서 결론은 다음과 같습니다.
1. `ndots(2)` 로 설정
2. 동일 네임스페이스에 있을 경우
    - {서비스 이름} -> 1번 질의
3. 다른 네임스페이스에 있는 경우
    - {서비스 이름}.{네임스페이스}.svc.cluster.local -> 1번 질의

### Kubernetes Workloads 에 ndots를 적용하는 방식

> Workloads 에서 `spec.template.spec` 에 dnsConfig 를 설정합니다.

> 또한, 스펙이 변경되는 작업이므로 해당 Workloads 에 속한 모든 파드가 재시작됩니다.

```diff
kind: Deployment
spec:
  template:
    spec:
+     dnsConfig:
+       options:
+         - name: ndots
+           value: "2"
      containers:
        - name: application
```

ndots를 설정한 서비스에 접근해 `/etc/resolv.conf` 파일을 확인해보면 `ndots` 값이 5 -> 2로 변경된 것을 확인할 수 있습니다.

```bash
$ kubectl exec -it <POD_NAME> \
    -n <NAMESPACE> \
    -- cat /etc/resolv.conf

search default.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
nameserver 172.20.0.10
options ndots:2
```

### 결과
#### ndots 2
1. CoreDNS로의 요청량이 줄었습니다.

   - 특히 A, AAAA 레코드에 대한 요청이 줄었고, 이는 search 도메인 목록 순차대로 요청하지 않고 FQDN 요청이 많아졌다는 것으로 판단됩니다.
   - 즉, 질의하는 수 자체가 줄었다는 의미입니다.

2. NXDomain 응답이 줄었습니다.

   - search 도메인 목록의 요청이 아닌 FQDN 요청이 증가한 결과입니다.
   - 그에 따라 NOERROR 응답이 소폭 상승하였습니다.

3. CoreDNS 의 CPU 사용량 및 Latency 도 줄었습니다.
   - CoreDNS 로의 요청량이 줄다보니 그에 따라 CPU 사용량 및 Latency도 같이 줄었습니다.

![ndots 2 반영 후 결과](https://github.com/user-attachments/assets/6b480c1b-cd9e-48f1-b7a8-e5d1544a2332)

#### CoreDNS 개수 증가
1. Request & Response 지표 증가

   - CoreDNS 대수 증가에 따른 자연스러운 현상으로 판단됩니다.

2. conntrack_allowance_exceeded 메트릭이 '0'
 CoreDNS Latency 감소

3. bw_in_allowance_exceeded 와 bw_out_allowance_exceeded 지표는 CoreDNS 대수 증가에 따라 metric이 변경되지는 않은 것으로 보임

    - 상위 5개 metric 을 확인해보았을 때, 서비스 자체가 데이터가 왔다갔다 하는 로직이 많아 해당 지표가 계속적으로 보이는 것으로 판단됩니다.
    - 네트워크 대역이 큰 인스턴스로 변경하거나 더 분산시켜 위치시키는 방안으로 가야할 듯 보입니다.

참고 문서

1. [Pod's DNS Config](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config)
2. [EKS Best Practice 문서](https://aws.github.io/aws-eks-best-practices/scalability/docs/cluster-services/#scale-coredns)
3. [의도하지 않는 pod의 DNS 요청](https://malwareanalysis.tistory.com/749)
4. [Kubernetes DNS: FQDN과 ndots의 동작 방식 정리](https://lakescript.net/entry/Kubernetes-DNS-FQDN%EA%B3%BC-ndots%EC%9D%98-%EB%8F%99%EC%9E%91-%EB%B0%A9%EC%8B%9D-%EC%A0%95%EB%A6%AC#ndots2)
5. [Kubernetes 환경에서 발생하는 DNS Query Failed 이슈와 NodeLocal DNSCache를 이용한 해결](https://nangman14.tistory.com/108)
