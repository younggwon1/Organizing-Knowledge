# DevOps Engineer로써 발전을 위해 해야할 것들
> 여러 의견에 휘둘려 DevOps 역할이 흔들리게 되면 서비스의 안정성 및 신뢰성이 떨어진다고 생각합니다. 따라서 DevOps Engineer로써 업무를 수행할 시 업무 기준을 세워두고 요구 사항이 기준에 적합한지 파악하여 업무를 수행합니다. 하지만 여러 상황들로 인해 요구 사항들이 업무 기준에 부합할 수는 없습니다. 이때는 ‘기준에 부합하지 않으니 안됀다.’ 가 아닌 유연하게 업무를 대응하여 개발자의 편의성도 확보하는 것이 중요합니다.


* 신기술 지식 습득을 꾸준히 하며 이를 팀 내에 적극적으로 공유하실 수 있으신 분
* 장애 발생의 원인을 딥다이브 하여 연구하며 먼저 나서서 개선하실 수 있으신 분
* 서비스의 안정성을 높이기 위해 문제를 정의하고 대비하기 위해 노력하실 수 있는 분


Infra
- 서비스 인프라를 안정적으로 운영하고, 대규모 트래픽 처리가 가능한 서비스 인프라 구성 및 개선
- Zero Downtime 달성하기 위한 인프라 운영 및 자동화를 통해 보다 더 효율적인 인프라 환경 구축
- 인프라, 시스템, 어플리케이션 장애 발생에 대한 이벤트 탐지, 분석, 복구 자동화 시스템 개발
- 컨테이너의 자원 할당량 최적화 및 최적화된 상태 유지
- 서비스 구성 및 아키텍쳐에 대한 이해가 높으신 분 (HTTP, DNS, LB, SSL, CI/CD, TCP/IP, Kernel, Web, WAS, DB, Network)
- Application Troubleshooting
- Kubernetes
- Istio

Kubernetes
- Kubernetes 아키텍처, 컨테이너 런타임 및 Linux 컨테이너 내부에 대한 확실한 이해
- 일관된 사용자 환경 제공을 위한 정책 operator 개발
- 대규모 Managed ​Multi-tenant Kubernetes ​클러스터 운영/개발
- Kubernetes 형상 관리 및 life cycle management 개발
- Kubernetes 클러스터의 시스템 모니터링 지표와 알림 개발
- Kubernetes 계정/권한 관리 시스템 개발
- Kubernetes 런타임(CRI) / 네트워크 (CNI) / 스토리지 (CSI) 관련 문제 해결
- Kubernetes 관련 시스템 컴포넌트/플러그인 개발 (커스텀 스케쥴러 등)
- Kubernetes Ecosystem에 관심이 많고, Container와 Kubernetes에 대한 깊은 이해와 이를 활용한 서비스 개발 경험이 있으신 분
- Kubernetes, Container 기반 Cloud-native 환경에서의 DevOps 파이프라인 구축 및 운영 실무 경험이 있으신 분
- Kubernetes Controller와 Operator에 대한 실무적인 이해와 적용 경험이 있으신 분
- Kubebuilder, Operator SDK 또는 Informer를 사용하여 Kubernetes Operator를 작성한 경험이 있으신 분
- 다양한 오픈소스 기반 플랫폼 구축 및 운영해 보신 분 (K8S, ElasticSearch, OpenSearch, Kafka, Nagios, prometheus, grafana, RabbitMQ, Nginx, etc)

Network
- L2/L3 네트워크부터 BGP, OSPF 같은 라우팅 프로토콜, EVPN/VXLAN 기반 오버레이 네트워크, 방화벽, ADC
- EVPN 등의 데이터센터 패브릭 네트워킹을 이해하고 구축, 운영 역량을 가진 분
- TCP/IP 프로토콜에 대한 이해도가 높고 트러블슈팅을 위한 패킷 분석 경험
- 멀티 데이터센터 연동을 위한 동적라우팅 L3프로토콜 (BGP, OSPF 등) 운영하신 경험
- L1~L7 계층의 다양한 네트워크 프로토콜(IP, MAC, TCP, HTTP, SSL, mTLS 등) 높은 이해
- NETWORK OVERLAY 기술(ACI, VXLAN, EVPN, MP-BGP, Fabric 등) 대해 높은 이해도를 갖추고, 구축 및 운영 경험
- L7 SLB 및 GSLB(Global Server Load Balancing) 구축 또는 운영 경험
- Network Troubleshooting
- Kernel
- eBPF

Linux
- 리눅스/윈도우 OS 환경 표준화 구축과 서비스를 운영해 보신 분
- 리눅스 및 윈도우 OS 시스템을 기반으로 시스템의 성능 개선 및 지표 관리 시스템(모니터링) 구축 및 운영
- 리눅스/윈도우 계열 환경에서 프로그래밍 개발 및 스크립트 작성이 가능하신 분 (Bash, Python, Go, Ansible, etc)
- Linux 커널 및 분산 시스템 등 저수준 시스템 개발 경험

GPU
- 다양한 GPU 서버 구축 및 GPU 플랫폼 구축 및 운영 (Nvidia, DGX, Slurm, CUDA, etc)
- MIG 지원 GPU를 위한 동적 GPU 스케줄러 작성 경험
- NUMA 또는 PCIE 토폴로지 또는 NVLink/NVSwitch에 대한 하드웨어 토폴로지 관리 및 최적화에 대한 지식
- GPU 팜이나 GPU 클러스터를 구축하고 운영하는 데 있어 실질적인 실무 경험
- Kubernetes, Ray, Modal, RunPod, LambdaLabs 등을 활용한 GPU 오케스트레이션 경험

Observability
- 대규모 서비스 대상의 Observability (Metrics/Events/Logs/Traces 와 Alert, APM) 시스템 개발 / 운영
- 장애 발생 시 근본적인 원인을 파악할 수 있는 직관적인 Observability를 도출하여, 모든 개발자가 신속하게 장애에 대응할 수 있는 시스템 구축
- 인프라 및 서비스의 이상 징후(Anomaly Detection)를 빠르게 탐지하는 시스템 구축

CI / CD
- DevSecOps
    - Security + Compliance를 준수
    - 보안 취약점 탐지 및 개선
- 코드 품질 향상을 위한 환경 및 서비스 구축

개발 (자동화 Tool)
- DevOps
- Infra
- ...

비용 절감 (FinOps)
