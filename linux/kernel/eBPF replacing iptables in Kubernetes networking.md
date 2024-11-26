# eBPF replacing iptables in Kubernetes networking

### 1. 서론

기본 kubernetes의 네트워킹 모델은 kube-proxy 및 iptables을 사용하여 구현한다.

Application에 대해 접근을 위해서는 Endpoint가 필요한데, 이러한 Endpoint를 제공하는 것이 Kubernetes의 Service라는 Object가 제공한다.

실제 요청이 Endpoint로 전달되었을 때, 요청에 대한 Target Pod로 Routing 하기 위해서 각 Node에 배포 되어 있는 kube-proxy가 이를 대신 수행한다. 즉, kube-proxy가 생성한 iptables를 기반으로 pod 과 container로 트래픽이 올바르게 라우팅되도록 한다.

`but,,, iptables 사용 시 서비스가 확장됨에 따라 문제가 발생했다.`

- iptables 업데이트는 단일 트랜잭션에서 모든 규칙을 다시 만들고 업데이트하여 수행된다.
- iptables는 연결 목록의 규칙 체인으로 구현되므로 모든 작업은 O(n)이다.
- iptables는 순차적인 규칙 목록(O(n))으로 액세스 제어를 구현한다.
- 새로운 IP나 포트가 생길 때마다 규칙을 추가하고 체인을 변경해야한다.
- Kubernetes에서 리소스 소모량이 높다.

위를 다음과 같이 정리해 볼 수 있다.

1. Performance

   > 사용자에게 서비스를 제공할 때, 가장 중요한 부분인 Latency 이다.
   > iptables 기반의 환경의 가장 큰 문제점은 iptables의 수가 많아지는 만큼 Delay가 발생한다는 점이다. iptables는 Packet이 iptables의 규칙에 일치할 때까지 모든 규칙을 평가하게 되는데 이러한 규칙이 많아 질수록 전달되는 시간과 처리 시간이 그만큼 지연된다.

2. Time

   > 또한, iptables의 가장 큰 단점은 "Incremental Update"를 지원하지 않는다는 것이다.
   > 따라서 새로운 Service가 생성 되면서 추가되는 규칙을 적용하기 위해서 전체 iptables list를 교체 해야 한다.
   >
   > - EX), 5000개의 Service(40,000개의 규칙)가 있고 하나의 Rule을 추가하는데 11분 정도가 소요된다.
   >   따라서, Service를 생성하면 기본적으로 생성되는 규칙이 Service의 수만큼 늘어나기 때문에 많은 Service를 제공하는 환경에서는 적합하지 않다.

### 2. 본론

이러한 iptables의 문제점으로 인한 대안으로 eBPF가 있다. 처리량, CPU 사용량, 지연 시간에 대해서도 iptables에 비해 더 확장이 잘된다.

1. Performance

   > eBPF는 iptables를 사용하지 않고 eBPF를 통해 Packet을 처리하여 앞에서 언급한 문제점을 제거할 수 있다.
   > eBPF는 근본적으로 iptables와 달리 커널의 네트워크 스택을 타지 않고 drop / redirect 등이 가능하기 때문에, 그리고 iptables 처럼 규칙을 찾는데 시간이 걸리는 것이 아닌 이미 로드된 eBPF 프로그램이 패킷을 처리하기 때문에 iptables에 비해 성능이 비약적으로 향상된다.

2. Tracing

   > BPF를 사용하면 Pod와 Container-level 의 Packet 추적과 네트워크 통계가 가능하다.

3. https://github.com/cilium/ebpf/tree/main

4. https://ebpf.io/ko-kr/applications/

![스크린샷 2024-11-24 144052](https://github.com/user-attachments/assets/23826768-53fd-4899-929e-be4549108965)

### 3. 결론

kubernetes의 네트워킹 모델인 kube-proxy 및 iptables 대신 eBPF를 활용하여 네트워킹을 구현하려면 Cilium을 활용하는 것이 좋다.

왜냐하면, Cilium은 eBPF (extended Berkeley Packet Filter) 기술을 활용하여 현대적인 클라우드 네이티브 환경을 제공하는 강력한 네트워킹 솔루션이기 때문이다.

Cilium을 helm charts 형태로 배포하고 싶다면, 다음의 링크를 참고.

- https://github.com/cilium/cilium/tree/main/install/kubernetes/cilium

![스크린샷 2024-11-24 143558](https://github.com/user-attachments/assets/35371f92-36d0-45cb-85ec-97bcd6484589)

참고 문서

- [Leveraging eBPF in the Kubernetes Networking Model](https://sookocheff.com/post/kubernetes/leveraging-ebpf-in-the-kubernetes-networking-model/)
- [What is Kube-Proxy and why move from iptables to eBPF? ](https://isovalent.com/blog/post/why-replace-iptables-with-ebpf/)
- [[Kubernetes/Networking] eBPF Basic](https://blog.naver.com/kangdorr/222593265958)
- [eBPF 기반의 강력한 쿠버네티스 네트워킹: Cilium CNI 소개](https://tech.ktcloud.com/250)
