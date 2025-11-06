# IMDS(Instance Metadata Service)

## 1) IMDS(Instance Metadata Service)란 무엇인가?

- **정의**: IMDS는 EC2 인스턴스 자신에 대한 정보(인스턴스 ID, 호스트네임, 네트워크 정보, 사용자 데이터, 그리고 **EC2 인스턴스에 연결된 IAM 역할의 임시 자격증명** 등)를 로컬(링크-로컬 주소)에서 HTTP로 조회할 수 있게 해주는 서비스입니다. IP는 `http://169.254.169.254` (IPv4) 또는 Nitro 인스턴스의 IPv6 주소를 사용합니다.

- **왜 중요한가?**: IMDS를 통해 애플리케이션이나 스크립트가 인스턴스 메타데이터와 IAM 역할 자격증명을 자동으로 조회해 AWS API에 접근할 수 있게 되므로(즉, 애플리케이션에 자격증명을 하드코딩할 필요 없음) 편리하지만, 동시에 **인스턴스 내부의 민감한 자격증명이 노출될 경우 보안 위험**으로 직결됩니다.

## 2) IMDSv1 vs IMDSv2 — 핵심 차이점 (요약)

1. **인증 방식**

   - **IMDSv1**: 단순 HTTP GET 방식
        - `http://169.254.169.254/latest/meta-data/...`
        - 별도의 토큰 / 세션 없음
   - **IMDSv2**:
        - 세션 기반(token) 방식
            - 먼저 `PUT /latest/api/token`로 토큰을 발급받고, 이후 요청에 `X-aws-ec2-metadata-token` 헤더로 토큰을 포함해 GET 요청을 해야함
        - 세션 TTL(최대 6시간) 설정 가능

2. **SSRF(서버사이드 요청 위조)과의 관계**

   - IMDSv1은 SSRF 취약점을 통해 외부에서 메타데이터(특히 IAM 자격증명)를 가져갈 위험성이 있습니다.
   - IMDSv2는 토큰(세션)을 요구함으로써 SSRF 타입 공격에 대한 방어력을 크게 향상시킵니다.
        - 토큰을 발급하려면 인스턴스 내부에서 토큰 요청이 필요하고, 토큰은 세션에 묶여 있어 무작위 접근으로 쉽게 획득/재사용되기 어려움

3. **동작 예제 (간단)**

   - IMDSv2:
        - `PUT http://169.254.169.254/latest/api/token` (헤더: `X-aws-ec2-metadata-token-ttl-seconds`)
        - 발급된 토큰으로 `GET http://169.254.169.254/latest/meta-data/...` (헤더: `X-aws-ec2-metadata-token: <token>`) 요청

4. **기본값 변화**

   - 최근(신규 AMI/인스턴스 기준)에는 IMDSv2를 기본으로 하거나 IMDSv1이 비활성화된 케이스가 많아졌습니다. (특히 `Amazon Linux 2023` 등)
   - 하지만 기존 인스턴스는 여전히 IMDSv1/IMDSv2 둘 다 사용 가능한 `optional` 모드로 설정할 수 있습니다.

## 3) 왜 IMDSv2로 전환해야 하나? (보안적 근거)

- **SSRF 완화**: IMDSv2의 토큰/세션 모델은 기본적으로 SSRF 공격자가 메타데이터를 쉽게 훔쳐갈 가능성을 줄입니다.
- **정책·컴플라이언스**: AWS는 IMDSv2 사용을 권장하며, 조직 정책(SCP / IAM condition keys)을 통해 IMDSv1을 차단하도록 설정할 수 있습니다.

## 4) IMDSv1 → IMDSv2 마이그레이션 시 고려해야 할 사항 (체크리스트)

아래는 AWS 권장 절차와 실무에서 문제를 일으키는 포인트를 합친 항목입니다.

### A. 사전 조사 — 누가/무엇이 IMDS를 호출하는지 파악

- **`IMDS Packet Analyzer`** (AWS에서 제공하는 오픈소스 도구)로 인스턴스에서 발생하는 IMDSv1 호출을 찾아 어떤 프로세스(앱/스크립트/툴)가 호출하는지 확인할 수 있습니다.
- **CloudWatch 메트릭**: `MetadataNoToken`(IMDSv1 호출 수) 메트릭을 모니터링하여 전환 진행 상황을 추적할 수 있습니다.
    - `MetadataNoTokenRejected`로 v1 차단 시 시도 횟수도 확인 가능

### B. SDK / 도구 업데이트

- 최신 **AWS SDK / CLI**는 IMDSv2를 기본으로 지원합니다. (예: AWS CLI v2, 최신 boto3/Java SDK 등)

### C. 애플리케이션/스크립트 수정

- **직접 IMDSv1 엔드포인트를 호출하는 코드**(curl, custom HTTP 등)가 있다면 IMDSv2의 토큰 발급(flow)을 구현하도록 변경해야 합니다.
    - 예: token 발급 PUT → GET에 토큰 헤더 포함

### D. 컨테이너/프록시 환경의 ‘hop limit’ 확인

- **컨테이너 환경**(특히 sidecar 또는 프록시가 있는 경우)은 IMDS 토큰 요청에 네트워크 홉이 추가될 수 있고, 기본 hop limit(=1) 때문에 응답이 차단될 수 있습니다.
- 필요하면 **PUT 응답 hop limit**을 `2`로 설정하거나(인스턴스 레벨 `MetadataOptions`의 `HttpPutResponseHopLimit`), 컨테이너 구성에서 메타데이터 접근을 대체(환경변수 전달 등)해야 합니다.

### E. 자동화/AMI/ASG 설정

- **새 AMI**에 IMDSv2를 기본으로 설정(AMI 등록 시 `ImdsSupport=v2.0`)하거나, 런치 템플릿/ASG의 인스턴스 메타데이터 옵션을 변경하여 신규 인스턴스가 자동으로 IMDSv2로 생성되도록 합니다.
    - 기존 ASG의 런치 템플릿만 변경하면 기존 인스턴스에는 영향을 주지 않으니 이 점 주의

### F. 점진적 전환 → 강제(Require)로 변경

1) SDK/도구와 앱을 업데이트
2) `MetadataNoToken`이 0 임을 확인
3) 인스턴스/계정 기본값을 `IMDSv2 required`로 변경
4) (원하면) 기존 IMDSv1을 차단(Require)

### G. IAM/조직 수준 보호

- IAM condition keys(`ec2:MetadataHttpTokens`, `ec2:MetadataHttpEndpoint`, `ec2:MetadataHttpPutResponseHopLimit`)와 SCP를 사용해 IMDSv1을 비활성화하거나 인스턴스 출시에 제약을 둘 수 있습니다. 
- 또한 `ec2:RoleDelivery` 조건을 사용하면 v1으로 전달된 자격증명을 사용한 API 호출을 거부하도록 정책을 구성할 수 있습니다.

## 5) 마이그레이션 중 자주 발생하는 문제와 해결법

- **문제: 애플리케이션이 갑자기 메타데이터를 못 씀(403/401 또는 연결 실패)**

  - 원인: IMDSv2가 `required`로 변경됐는데 앱이 IMDSv1 로직만 가지고 있음
  - 해결: 해당 프로세스를 식별 → IMDSv2 토큰 로직 또는 SDK 최신화 → 테스트 후 `required` 적용 (CloudWatch `MetadataNoTokenRejected` 확인)

- **문제: 컨테이너에서 메타데이터 호출 지연/실패**

  - 원인: hop limit(기본 1)이 컨테이너 네트워크 경로상 추가 홉으로 인해 차단
  - 해결: `HttpPutResponseHopLimit`을 2로 늘리거나, 컨테이너에 환경변수/시크릿 전달로 IMDS 종속성 제거

- **문제: 너무 잦은 메타데이터 호출 → 쓰로틀(Throttling)**

  - 권장: 자격증명은 로컬에서 캐시(만료 직전까지 재사용)하고, 매번 조회하지 않도록 코드 변경

## 6) 구체적 마이그레이션 단계 (실전 절차 — 요약)

1. **준비**

   - 모든 인스턴스(또는 대표 인스턴스)에 IMDS Packet Analyzer 설치/실행 → IMDSv1 호출 식별
   - AWS CLI/SDK 최신화

2. **코드 수정**

   - 직접 메타데이터 호출하는 스크립트/앱을 IMDSv2 방식(token 요청 + 헤더 포함)으로 변경.
   - SDK 사용 코드는 최신 SDK로 교체

3. **모니터링으로 검증**

   - CloudWatch `MetadataNoToken` → 0 달성 확인

4. **강제 적용**

   - 인스턴스(또는 계정 기본값)에서 `HttpTokens=required`로 설정하여 IMDSv1 차단(또는 AMI/런치 템플릿을 미리 설정)
   - 기존 인스턴스는 `modify-instance-metadata-options`로 변경 가능(재시작 불필요)

5. **후속 모니터링**

   - `MetadataNoTokenRejected` 체크 → 차단된 v1 요청이 있는지 확인하고, 필요 시 문제 프로세스 수정

## 7) 추가 팁 & 모범사례

* **원칙**: 인스턴스가 IMDS에서 값을 읽어야만 하는 경우를 최소화 — 가능한 환경변수/시크릿 매니저/Parameter Store로 대체합니다.
* **IAM 조건 사용**: 조직 정책으로 새 인스턴스에 IMDSv2만 허용하도록 강제하면 실수로 v1을 활성화하는 것을 방지할 수 있습니다.
* **로그·감시 필수**: IMDS 호출 로그(또는 CloudWatch 메트릭)를 주기적으로 점검하여 비정상 접근을 감지합니다.

## 8) 참고(주요 출처)

* AWS — *Transition to using Instance Metadata Service Version 2* (EC2 User Guide). ([AWS Documentation][3])
* AWS — *Access instance metadata for an EC2 instance* (EC2 User Guide — 데이터 조회 예제 포함). ([AWS Documentation][1])
* AWS Security Blog — *Get the full benefits of IMDSv2 and disable IMDSv1 across your AWS infrastructure*. ([Amazon Web Services, Inc.][2])
* Slack Engineering — *Our Journey Migrating to AWS IMDSv2* (실무 마이그레이션 사례). ([slack.engineering][4])
* Datadog Security Lab — *Securing the EC2 Instance Metadata Service* (IMDS 보안 관점 설명). ([securitylabs.datadoghq.com][5])

[1]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html "Access instance metadata for an EC2 instance - Amazon Elastic Compute Cloud"
[2]: https://aws.amazon.com/blogs/security/get-the-full-benefits-of-imdsv2-and-disable-imdsv1-across-your-aws-infrastructure/?utm_source=chatgpt.com "Get the full benefits of IMDSv2 and disable IMDSv1 across ..."
[3]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-transition-to-version-2.html "Transition to using Instance Metadata Service Version 2 - Amazon Elastic Compute Cloud"
[4]: https://slack.engineering/our-journey-migrating-to-aws-imdsv2/?utm_source=chatgpt.com "Our Journey Migrating to AWS IMDSv2"
[5]: https://securitylabs.datadoghq.com/articles/misconfiguration-spotlight-imds/?utm_source=chatgpt.com "Securing the EC2 Instance Metadata Service"
