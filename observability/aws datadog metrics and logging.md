## **1. 개요**
- 어떤 AWS 리소스의 메트릭 / 로그를 수집할 지?
- 수집 대상은 어떻게 할 것인지?
- 실시간성으로 수집을 원하는지? 어느정도 비 실시간으로도 운영도 가능한지?
- 중앙 집중형?
    - audit 계정을 생성하여 중앙 집중식 로깅을 통해?

## **2. AWS 리소스 모니터링을 위한 메트릭 선정**

1. Amazon EC2
    - ⭐ Metrics 우선 순위
        | 우선순위 | 메트릭 이름 | 설명 |
        | --- | --- | --- |
        | ★★★ | `CPUUtilization` | 인스턴스 과부하 여부 파악의 핵심 지표 |
        | ★★★ | `StatusCheckFailed` | 인스턴스/시스템 상태 이상 여부 (즉시 장애 감지) |
        | ★★☆ | `NetworkIn`, `NetworkOut` | 네트워크 트래픽 수준 (비정상 spike 탐지) |
        | ★★☆ | `NetworkPacketsIn`, `NetworkPacketsOut` | 패킷 수 기준으로 네트워크 상태 판단 가능 (DDoS 징후 등 탐지) |
        | ★★☆ | `VolumeReadOps`, `VolumeWriteOps` | 디스크 I/O 수 (I/OPS) 확인 가능 |
        | ★★☆ | `VolumeQueueLength` | I/O 큐가 길어지면 디스크 병목 가능성 |
2. RDS (AWS PostgreSQL)
    - ⭐ Metrics 우선 순위
        | 우선순위 | 메트릭 이름 | 설명 |
        | --- | --- | --- |
        | ★★★ | `CPUUtilization` | DB 부하 주요 지표 |
        | ★★★ | `FreeableMemory` | DB 인스턴스의 여유 메모리. 메모리 부족/OOM 위험 탐지에 매우 중요 |
        | ★★★ | `SwapUsage` | 메모리 부족 시 스왑 활성화 → 성능 급락 원인 |
        | ★★★ | `DatabaseConnections` | 연결 수가 포화되는지 감지 |
        | ★★☆ | `FreeStorageSpace` | 디스크 공간 부족 여부 감지 |
        | ★★☆ | `WriteLatency`, `ReadLatency` | 스토리지 I/O 지연 (성능 병목 여부) |
        | ★★☆ | `DiskQueueDepth` | 디스크 I/O가 대기 상태일 때 병목 가능성 진단 |
        | ★★☆ | `NetworkReceiveThroughput` | 네트워크 수신량 → 비정상 트래픽, 복제, ETL 감지 가능 |
        | ★★☆ | `NetworkTransmitThroughput` | 네트워크 송신량 → 외부 전송 과부하 탐지 가능 |
        | ★★☆ | `MaximumUsedTransactionIDs` | PostgreSQL에서 transaction wraparound 감지 (wraparound = DB 정지 위험) |
        | ★☆☆ | `ReadIOPS`, `WriteIOPS` | 상세한 I/O 분석 필요할 때만 |
3. AWS Lambda
    - ⭐ Metrics 우선 순위
        | 우선순위 | 메트릭 이름 | 설명 |
        | --- | --- | --- |
        | ★★★ | `Invocations` | 호출 수 (정상 동작 여부 확인) |
        | ★★★ | `Errors` | 실패한 호출 (장애/예외 감지) |
        | ★★★ | `Duration` | 실행 시간 (비정상적 증가 여부 확인) |
        | ★★☆ | `Throttles` | 호출 제한 발생 시 성능 문제 가능성 |
        | ★☆☆ | `IteratorAge` | 스트리밍 기반 트리거(Kinesis 등)에서만 유의미 |
4. AWS ELB
    - ⭐ Metrics 우선 순위
        | 우선순위 | 메트릭 이름 | 설명 |
        | --- | --- | --- |
        | ★★★ | `HTTPCode_ELB_5XX_Count` | LoadBalancer 수준 5XX 오류 |
        | ★★★ | `HTTPCode_Target_5XX_Count` | 백엔드 서버의 5xx 오류 응답 수 |
        | ★★★ | `Latency` | 전체 지연 시간 |
        | ★★☆ | `RequestCount` | 트래픽 규모 파악 |
        | ★★☆ | `HealthyHostCount`, `UnHealthyHostCount` | 백엔드 상태 추적 |
        | ★★☆ | `SurgeQueueLength` | 백엔드 병목 시 대기열 길이 증가 → 병목 전조 |
        | ★★☆ | `SpilloverCount` | 큐 넘칠 때 버려지는 요청 수 → 사용량 초과 탐지 |
        | ★★☆ | `BackendConnectionErrors` | ALB ↔ 백엔드 간 네트워크 오류 |
        | ★☆☆ | `HTTPCode_Target_4XX_Count`, `2XX_Count`, `HTTPCode_ELB_4XX_Count`, `HTTPCode_Target_3XX_Count` | 상세 오류 분포가 필요할 때만 |
5. Amazon CloudFront (Polling 방식만 지원 , Optional)
    - ⭐ Metrics 우선 순위
        | 우선순위 | 메트릭 이름 | 설명 |
        | --- | --- | --- |
        | ★★★ | `4xxErrorRate`, `5xxErrorRate` | 사용자 요청 실패율 감지 |
        | ★★☆ | `Requests` | 전체 요청량 확인 (스파이크 탐지용) |
        | ★★☆ | `BytesDownloaded`, `BytesUploaded` | CDN 트래픽 양 추적 |
        | ★☆☆ | `TotalErrorRate` | 에러율 통합 지표 (상위 4xx/5xx 포함) |
6. AWS WAF (Polling 방식만 지원 , Optional)
    - ⭐ Metrics 우선 순위
        | 우선순위 | 메트릭 이름 | 설명 |
        | --- | --- | --- |
        | ★★★ | `BlockedRequests` | 차단된 공격 시도 감지 |
        | ★★☆ | `AllowedRequests` | 전체 요청 대비 차단 비율 파악 |
        | ★☆☆ | `CountedRequests`, `CaptchaSubmitted` | WAF 동작 테스트 중 또는 고급 설정일 때 사용 |

## **3. 로그 수집이 필요한 대상**

`Datadog Lambda Forwarder`를 활용하여 로그 수집을 진행

**로그 저장 위치 선정**

- S3로 전송되는 로그가 정기적 배치 성격일 경우 (예: VPC Flow Logs) S3가 유리
- Datadog의 경고(Alerts), 실시간 대시보드를 사용하려면 CloudWatch Logs가 유리
- Lambda Forwarder는 S3 이벤트가 발생하지 않으면 실행되지 않기 때문에, S3 이벤트 설정이나 실패 알림 설정을 함께 구성해두는 것이 좋음

| 항목 | CloudWatch Logs | S3 |
|------|------------------|----|
| **로그 수집 방식** | 로그가 CloudWatch에 실시간으로 전송되며, Forwarder는 CloudWatch Logs 구독(subscription)을 통해 실시간 처리 | S3에 저장된 로그 파일(Object)을 이벤트 기반(Put 이벤트 등) 또는 주기적으로 처리 |
| **평균 지연 시간** | 수 초 | 1~3분 (최대 수 분) |
| **실시간 경고/모니터링 적합성** | ✅ 매우 적합 | ⚠️ 경고 기준엔 부적합할 수 있음 |
| **데이터 손실 가능성** | 낮음 | 이벤트 누락 시 일부 손실 가능 |
| **비용** | - CloudWatch 로그 저장 비용<br>- Lambda 실행 비용<br>- Datadog 로그 처리 비용 | - S3 저장 비용<br>- S3 요청 비용<br>- Lambda 실행 비용<br>- Datadog 로그 처리 비용 |
| **재처리 용이성** | ❌ 로그는 일정 기간 지나면 삭제되므로 재처리 어려움 | ✅ S3에 저장된 로그 파일을 다시 처리 가능 (재실행 시 유리) |


### 3.1 **Datadog Lambda Forwarder 설정 방법**

| 리소스 | 로그 저장 위치 | Lambda Forwarder 접근 방법 |
| --- | --- | --- |
| **AWS** **Lambda** | **CloudWatch Logs** | CloudWatch Logs 구독 필터 |
| **AWS** **ELB** | **S3 버킷** | S3 이벤트 트리거 혹은 S3 직접 읽기 |
| **Amazon** **CloudFront** | **S3 버킷** | S3 이벤트 트리거 혹은 S3 직접 읽기 |
| **AWS CloudTrail** | **CloudWatch Logs** | CloudWatch Logs 구독 필터 |
| **RDS (PostgreSQL)** | **CloudWatch Logs** | CloudWatch Logs 구독 필터 |
| **Amazon EC2** | **CloudWatch Logs** | CloudWatch Logs 구독 필터 |

**로그 보관 주기 방안**

- [0]. CloudWatch Logs Retension 1 day to 3 day
- [1]. CloudWatch Logs export to S3 → S3 Lifecycle Retention
- [2]. [**Datadog Archive**](https://docs.datadoghq.com/logs/log_configuration/archives/?tab=awss3) Logs

| 리소스 | Datadog (Hot) 로그 보관 주기 | S3 or Datadog Archive (Cold) 로그 보관 주기 |
| --- | --- | --- |
| **AWS** **Lambda** | 30일 | 6개월~1년 |
| **AWS** **ELB** | 15~30일 | 6개월~1년 |
| **Amazon** **CloudFront** | 15일 | 6개월~1년 |
| **AWS CloudTrail** | 7~15일 | 1년~3년 |
| **RDS (PostgreSQL)** | 30일 | 6개월 이상 |
| **Amazon EC2** | 30일 | 6개월 이상 |
- **`로그 수집 활성화`**
    - **CloudWatch Logs**
        - [AWS Lambda](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/monitoring-cloudwatchlogs.html)
        - [AWS CloudTrail](https://docs.aws.amazon.com/ko_kr/awscloudtrail/latest/userguide/send-cloudtrail-events-to-cloudwatch-logs.html)
        - [RDS (PostgreSQL)](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html)
        - [Amazon EC2](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html)
    - **S3 버킷**
        - [AWS ELB](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/enable-access-logs.html)
        - [Amazon CloudFront](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/standard-logging.html)
- **`Datadog에 로그 전송`**
    - **Datadog Lambda Forwarder 배포**
        1. Add New AWS Account
            1. [**`Datadog-AWS integration`**](https://docs.datadoghq.com/integrations/amazon_web_services/)에서 [**`CloudFormation`**](https://docs.datadoghq.com/ko/logs/guide/forwarder/?tab=cloudformation#cloudformation)을 사용해 자동으로 Forwarder를 설치하기를 **권고**합니다. 
                1. Add New AWS Account에서 Automatically using Cloudformation으로 과정에 따라 진행해줍니다.
                    1. [[Datadog] Cloudformation Template](https://github.com/DataDog/cloudformation-template/tree/master/aws)
                2. 물론 [Terraform](https://docs.datadoghq.com/ko/logs/guide/forwarder/?tab=cloudformation#terraform)이나 [수동 작업](https://docs.datadoghq.com/ko/logs/guide/forwarder/?tab=cloudformation#manual)으로 설정 프로세스를 진행할 수도 있습니다.
        2. Already have an AWS Account
            1. Launch Stack - [**`CloudFormation`**](https://docs.datadoghq.com/logs/guide/forwarder/?tab=cloudformation#installation)
            2. Parameter
                1. DdApiKey : {Datadog API Key}
                2. DdApiKeySecretArn : 기본값
                3. DdSite : datadoghq.com
            3. "AWS CloudFormation이 IAM 리소스를 생성할 수 있음을 확인합니다." 를 선택
            4. "스택 생성" 클릭
    - **트리거 자동 설정**
        1. [`Datadog-AWS integration`](https://docs.datadoghq.com/integrations/amazon_web_services/)에서 AWS 계정을 선택해 로그를 수집할 AWS 계정을 선택하고 로그 수집 탭을 클릭합니다.
        2. 이전에 생성한 Lambda Function (Datadog Lambda Forwarder 용)의 ARN을 입력하고 추가를 클릭합니다.
        3. 로그를 수집하려는 서비스를 선택하고 저장을 클릭합니다. 특정 서비스에서 로그 수집을 중지하려면 로그를 선택 해제합니다.
            1. 참고: [Subscription Filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters)는 Datadog Forwarder에 의해 CloudWatch 로그 그룹에 자동으로 생성되며 형식에 따라 이름이 지정됩니다.
        4. 트리거 자동 설정이더라도 Lambda 함슈의 트리거에 수집할 로그 대상을 설정해야합니다.
        5. 이 초기 설정 후 몇 분 내에 Datadog [로그 탐색기](https://app.datadoghq.com/logs)에 AWS 로그가 나타납니다.
        - (참고)
            1. 여러 지역의 로그를 보유하고 있는 경우 해당 지역에 Lambda Function을 추가로 생성해야합니다.
                1. Datadog 서버가 미국 us-east1 리전에 있기 때문에 WAF, CloudFront와 같은 글로벌 서비스에 대한 로그를 수집하기 위해서는 Lambda Function(Datadog Lambda Forwarder 용) 을 us-east1 리전에도 배포가 필요하여 us-east-1, ap-northeast-2 총 2개 리전에 배포가 필요합니다.
            2. 배포 시 DdApiKey, DdApiKeySecretArn, DdSite 가 필수 Parameter로 요구되나 Secret Manager에 이미 Datadog API가 저장되어 있을 경우, DdApiKeySecretArn 만 입력하면 DdApiKey를 자동으로 불러오는 것이 가능합니다.
    - **트리거 수동 설정**
        1. CloudWatch Logs에서 로그를 수집하는 경우
            1. (사전 작업) CloudWatch Logs로 AWS 리소스 로그를 게시합니다.
            2. CloudWatch 로그 그룹에서 로그 수집
                1. 콘솔 또는 CloudFormation 을 통해 수동으로 트리거를 설정합니다.
                2. [Collecting logs from CloudWatch log group](https://docs.datadoghq.com/logs/guide/send-aws-services-logs-with-the-datadog-lambda-function/?tab=awsconsole#collecting-logs-from-cloudwatch-log-group)
        2. S3 버킷에서 로그를 수집하는 경우
            1. [Collecting logs from S3 buckets](https://docs.datadoghq.com/logs/guide/send-aws-services-logs-with-the-datadog-lambda-function/?tab=awsconsole#collecting-logs-from-s3-buckets)

참고 문서

- https://docs.datadoghq.com/ko/logs/guide/forwarder
- https://docs.datadoghq.com/ko/logs/guide/send-aws-services-logs-with-the-datadog-lambda-function/?tab=awsconsole
