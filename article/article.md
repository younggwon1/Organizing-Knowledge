# 읽어볼만한 Article 모음

### [0]. AWS
- [혹시 S3에 ‘연/월/일’로 나누어 저장하고 있나요? 파티셔닝 방식의 함정!](https://kr.linkedin.com/posts/hyunsoo-ryan-lee_%ED%98%B9%EC%8B%9C-s3%EC%97%90-%EC%97%B0%EC%9B%94%EC%9D%BC%EB%A1%9C-%EB%82%98%EB%88%84%EC%96%B4-%EC%A0%80%EC%9E%A5%ED%95%98%EA%B3%A0-%EC%9E%88%EB%82%98%EC%9A%94-%ED%8C%8C%ED%8B%B0%EC%85%94%EB%8B%9D-%EB%B0%A9%EC%8B%9D%EC%9D%98-activity-7382318323545468928-vM1Q)

### [1]. Kubernetes
- [Container의 격리된 환경의 의미](https://velog.io/@rookie0031/Container%EB%9E%80-%EB%8F%84%EB%8C%80%EC%B2%B4-%EB%AC%B4%EC%97%87%EC%9D%BC%EA%B9%8C)
- [Pod와 Container의 차이](https://velog.io/@rookie0031/Pod%EC%99%80-Container%EC%9D%98-%EC%B0%A8%EC%9D%B4-8mt17k1x)
- [쿠버네티스가 쉬워지는 컨테이너 이야기 — cgroup, cpu편](https://medium.com/@7424069/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EA%B0%80-%EC%89%AC%EC%9B%8C%EC%A7%80%EB%8A%94-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EC%9D%B4%EC%95%BC%EA%B8%B0-cgroup-cpu%ED%8E%B8-c8f1e2208168)
- [쿠버네티스가 쉬워지는 컨테이너 이야기 — memory편](https://medium.com/@7424069/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EA%B0%80-%EC%89%AC%EC%9B%8C%EC%A7%80%EB%8A%94-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EC%9D%B4%EC%95%BC%EA%B8%B0-memory%ED%8E%B8-62cafabfd160)
- [쿠버네티스가 쉬워지는 컨테이너 이야기 — cpuset.cpu편](https://medium.com/@7424069/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EA%B0%80-%EC%89%AC%EC%9B%8C%EC%A7%80%EB%8A%94-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EC%9D%B4%EC%95%BC%EA%B8%B0-cpuset-cpu%ED%8E%B8-3ff523ea298b)
- [쿠버네티스가 쉬워지는 컨테이너 이야기 — cpuset.mem편](https://medium.com/@7424069/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EA%B0%80-%EC%89%AC%EC%9B%8C%EC%A7%80%EB%8A%94-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EC%9D%B4%EC%95%BC%EA%B8%B0-cpuset-mem%ED%8E%B8-f3a46268328e)
- [Kubernetes Pod Gracefully Termination](https://ssup2.github.io/blog-software/docs/theory-analysis/kubernetes-pod-gracefully-termination/)
- [Istio Pod Gracefully Termination](https://ssup2.github.io/blog-software/docs/theory-analysis/istio-pod-gracefully-termination/)
- [Kubernetes Pod with Linux OOM Killer](https://ssup2.github.io/blog-software/docs/theory-analysis/kubernetes-pod-linux-oom-killer/)
- [Kubernetes 환경에서 발생하는 DNS Query Failed 이슈와 NodeLocal DNSCache를 이용한 해결](https://nangman14.tistory.com/108)
- [Linux UDP Packet Drop with conntrack Race Condition](https://ssup2.github.io/blog-software/docs/issue/linux-udp-packet-drop-conntrack-race-condition/)
- [Kubernetes Pod 메모리 누수 – 표준 모니터링을 넘어 eBPF로 파고들다](https://kr.linkedin.com/posts/victor-maltsev_kubernetes-ebpf-devops-activity-7353735799185502208-AVJy)
- [Istio 도입후 503 conenction refused 에러 트러블 슈팅](https://velog.io/@rookie0031/Istio-%EB%8F%84%EC%9E%85%ED%9B%84-503-conenction-refused-%EC%97%90%EB%9F%AC-%ED%8A%B8%EB%9F%AC%EB%B8%94-%EC%8A%88%ED%8C%85)
- [Kubernetes DNS: FQDN과 ndots의 동작 방식 정리](https://lakescript.net/entry/Kubernetes-DNS-FQDN%EA%B3%BC-ndots%EC%9D%98-%EB%8F%99%EC%9E%91-%EB%B0%A9%EC%8B%9D-%EC%A0%95%EB%A6%AC)
- [Kubernetes Troubleshooting: Common Errors and Fixes](https://www.linkedin.com/posts/kunal-dhole-32a11a112_kubernetes-devops-cloudcomputing-activity-7376334663533006850-6zsj)
- [[if(kakaoAI)2024] 카카오페이증권의 Kubernetes 지능형 리소스 최적화 (feat. Dr.Pym Project 공유)](https://tech.kakaopay.com/post/ifkakao2024-dr-pym-project/)
- [KEP-5295: Introducing KYAML, a safer, less ambiguous YAML subset / encoding](https://github.com/kubernetes/enhancements/blob/master/keps/sig-cli/5295-kyaml/README.md?trk=public_post_comment-text)
- [클라우드 네이티브 보안 개요](https://kubernetes.io/ko/docs/concepts/security/overview/)
- [내부 플랫폼에 AIOps 도입기: 알람 홍수에서 예측형 복구까지, MTTR 31% 단축](https://kr.linkedin.com/posts/victor-maltsev_devops-aiops-platformengineering-activity-7359468223748022273-vvs3)
- [토스증권의 수 천개 실시간 데이터 파이프라인 운영방법 #2: MSA 환경 Observability 높이기](https://toss.tech/article/MSA-observability)
- [Amazon EKS 모범 사례 가이드](https://docs.aws.amazon.com/ko_kr/eks/latest/best-practices/introduction.html)
- [AWS Prescriptive Guidance Patterns](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/welcome.html)
- [Amazon EKS, 클러스터 업그레이드의 일부로 업그레이드 인사이트 검사 시행](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html#update-cluster-control-plane)

### [2]. Docker Image
- [컨테이너가 뜨는 순간, 이미지가 “내가 만든 것”인지 커널이 직접 증명한다.](https://kr.linkedin.com/posts/victor-maltsev_slsa-sigstore-sbom-activity-7368817518855077888-PT8t)
- [Docker 실행 시, Entrypoint에서 exec 명령어를 사용하는 이유](https://kr.linkedin.com/posts/gwonsoolee_entrypoint%EC%97%90%EC%84%9C-exec-%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94-%EC%9D%B4%EC%9C%A0-activity-7203018882406600704-fi6f)

### [3]. GitOps
- [GitHub Actions 도입기(1): ARC 구축 및 Container Jobs 지원](https://channel.io/ko/team/blog/articles/GitHub-Actions-%EB%8F%84%EC%9E%85%EA%B8%B01-ARC-%EA%B5%AC%EC%B6%95-%EB%B0%8F-Container-Jobs-%EC%A7%80%EC%9B%90-55fa0670)
- [GitHub Actions 도입기(2): 실제 운영 및 트러블슈팅](https://channel.io/ko/team/blog/articles/GitHub-Actions-%EB%8F%84%EC%9E%85%EA%B8%B02-%EC%8B%A4%EC%A0%9C-%EC%9A%B4%EC%98%81-%EB%B0%8F-%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85-29c9a273)

### [4]. Network
- [Apache HttpClient Connection 관리하기 (TIME_WAIT & CLOSE_WAIT 문제)](https://velog.io/@jundragon/JAVA-Apache-HttpClient-Connection-Management)
- [TCP/HTTP 타임아웃(Timeout), 이 글 하나로 개념 완전 정복](https://devpanpan.tistory.com/118)

### [5]. Observability
- [Key metrics for CoreDNS monitoring](https://www.datadoghq.com/blog/coredns-metrics/#throughput-metrics)

### [6]. FinOps
- [Amazon EKS 클러스터를 비용 효율적으로 오토스케일링하기](https://aws.amazon.com/ko/blogs/tech/amazon-eks-cluster-auto-scaling-karpenter-bp/)
- [우아한 Cloud FinOps 여정](https://techblog.woowahan.com/22855/)