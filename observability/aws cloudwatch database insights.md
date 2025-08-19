## 0. TL;DR

### **1. CloudWatch Database Insights 란?**

| 기능                          | 설명 |
|-----------------------------|------|
| 📊 CloudWatch Database Insights | - 전체 DB Fleet(여러 DB 인스턴스를 한 눈에 보는 Fleet 수준 뷰)에 대한 성능 분석<br>- Performance Insights API 기반의 모니터링 솔루션<br>- 요금은 CloudWatch 기준으로 청구<br>- 표준 모드: 기본 성능 모니터링 (보존 기간 7일, 일부 기능 제한)<br>- 고급 모드: 확장된 기능 (실행 계획, 온디맨드 분석 등) |
| 💰 요금 정보                    | - Performance Insights API: 기존 요금 유지<br>- CloudWatch Database Insights 요금 : CloudWatch 요금 페이지 |


### **2. AWS Performance Insights 중단**
| 항목                      | 내용                                                                                                                                               |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 🔚 **지원 종료일**           | 2025년 11월 30일                                                                                                                                    |
| 📌 **지원 종료 영향**         | - Amazon RDS Performance Insights 콘솔 환경 종료<br>- 유연한 보존 기간(1\~24개월) 종료<br>- 관련 요금 청구 종료                                                           |
| ✅ **계속 제공되는 기능**        | - Performance Insights API는 유지<br>- Performance Insights API 요금은 CloudWatch Database Insights와 함께 청구됨<br>- 기존 요금 변동 없음                           |
| 🛠️ **권장 조치**           | 2025년 11월 30일 전까지 모든 유료 Performance Insights 계층 DB 인스턴스를 CloudWatch **Database Insights**의 고급 모드로 업그레이드                                          |
| ⚠️ **조치 미실시 시**         | - Performance Insights 사용 DB는 CloudWatch Database Insights의 표준 모드로 자동 전환<br>- 7일 이상 성능 데이터 기록에 접근 불가<br>- Amazon RDS 콘솔에서 실행 계획 및 온디맨드 분석 기능 미지원 |
| 🏁 **2025년 11월 30일 이후** | 실행 계획 및 온디맨드 분석은 CloudWatch Database Insights 고급 모드에서만 가능                                                                                        |

## **1. AWS Performance Insights vs CloudWatch Database Insights**

- **AWS Performance Insights**
    - AWS Performance Insights는 Amazon RDS 및 Aurora Database 성능을 시간 흐름에 따라 시각화하고 분석할 수 있는 도구입니다.
        - 주요 기능
            - DB Load를 기준으로 성능 병목 지점 분석
            - Top SQL 쿼리, Top Wait Event, Top Users, Top Host 등 시각화 제공
            - Execution Plan 분석 (RDS for Oracle, RDS for SQL Server)
            - On-Demand Analysis (Aurora PostgreSQL만 지원)
            - 데이터 보존: 기본 7일, 최대 24개월 (유료 티어 기준)
        - 공식 문서
            - https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html
- **CloudWatch Database Insights**
    - CloudWatch Database Insights는 AWS CloudWatch와 통합되어 전반적인 인프라 / DB 상태를 통합적으로 모니터링하고 분석할 수 있는 도구입니다.
    - PI보다 더 광범위한 메트릭, 로그, 이벤트를 통합해서 관리합니다.
        - 주요 기능
            - Fleet 수준(전체 인스턴스 그룹)의 성능 시각화
            - PI 모든 데이터 (Top SQL 쿼리, Top Wait Event, Top Users, Top Host 등 시각화 제공) + 추가적인 I/O, 메모리, OS 수준 메트릭 등
            - CloudWatch Application Signals 통합으로 애플리케이션 관점 성능 추적
            - Lock contention, Query-level diagnostics, On-demand Analysis 지원
            - Execution Plan 분석
            - Standard 모드: Performance Insights 활성화 시 7일간 카운터 메트릭 유지, 7일이 지난 성능 데이터는 삭제
            - Advanced 모드: 15개월간 모든 수집 지표 자동 보존
        - 특이사항
            - Database Insights는 동일한 AWS 계정 내에서만 워크로드 모니터링을 지원
        - 공식 문서
            - https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html
- **정리**
    
    
    | 항목 | Performance Insights (PI) | CloudWatch Database Insights (CDI) |
    | --- | --- | --- |
    | 기능 범위 | 데이터베이스 성능 분석에 특화 (DB 내부 대기 이벤트 중심 분석) | CloudWatch 기반의 종합 모니터링 도구 (PI 데이터 포함 + 인프라 레벨 메트릭 추가 분석) |
    | 시각화 | 단일 대시보드, DB Load 중심 시각화 | CloudWatch 내 대시보드에서 시각화 가능, 대시보드 커스터마이징 지원 |
    | 연동 서비스 | 독립적이며 느슨한 AWS 서비스 연동 | CloudWatch Logs, Metrics, Alarms, Dashboards와 완전 통합 |
    | 분석 대상 | Wait Event, SQL, User, Host 중심 | PI 데이터 포함 + 인프라 레벨 메트릭 (I/O, CPU, RAM, 네트워크 등) 포함 |
    | 확장성 | PI 내 탐색은 제한적 (고정된 시각화 및 필터링 수준) | CloudWatch Logs Insights, Metric Math 등을 통한 확장 분석 가능 |
    | 알람 설정 | PI 자체로는 알람 기능 제한적 | CloudWatch 알람 기능 활용 가능 (Threshold, Anomaly Detection 등 모두 지원) |
    | 지원 DB | RDS, Aurora (MySQL, PostgreSQL, MariaDB 등) | 동일 – Aurora 및 RDS 주요 엔진 지원 |
    | 요금 | PI는 별도 과금 (샘플링 빈도와 저장 용량 기준) | CloudWatch 요금 체계 (지표 수집 및 저장, 대시보드 등 사용량 기반) |
    | 데이터 보존 | 기본 7일, 최대 24개월까지 (유료) | Standard 모드: 7일 후 삭제Advanced 모드: 최대 15개월 보존 (모든 지표 포함) |

## **2. CloudWatch Database Insights 모드**

| 항목 | Standard 모드 | Advanced 모드 |
| --- | --- | --- |
| 데이터 보존 기간 | Standard: 7일 | Advanced: 최대 15개월 |
| 단일 인스턴스 → Fleet 뷰 | Fleet‑단위 통합 대시보드 가능 | Fleet‑단위 통합 대시보드 가능 |
| SQL 통계 분석 | 일부 기본 제공 | Per‑query 통계, 실행 계획, 실행 잠금 분석 등 고급 기능 제공 |
| 로그 및 이벤트 통합 뷰 | 없음 | Metrics + logs + events + Application Signals 통합된 대시보드 |
| APM 연동 | 없음 | CloudWatch Application Signals와 연계하여 DB‑애플리케이션 관계 분석 가능 |
| 운영체제 프로세스 수준 모니터링 | 없음 | Enhanced Monitoring 연동으로 OS‑level process 분석 |
| 실행 계획(On‑demand analysis) | 미지원 | Aurora, PostgreSQL, Oracle, SQL Server 대상 실행 계획 분석 |
- 기존 Performance Insights → Standard 모드의 CloudWatch Database Insights 전환은 비용 부담 없이 대체 가능
    - → 거의 동일한 수준의 기능을 사용 가능
- 하지만 더 정밀한 모니터링 / 분석이 필요하거나, 운영 자동화(알람/이상탐지) 등을 활용하려면 Advanced 모드로 업그레이드 하는 것이 필요

## **3. CloudWatch Database Insights (**Advanced 모드**) vs Datadog DBM**

- 차이
    
    
    | 항목 | **CloudWatch Database Insights (Advanced Mode)** | **Datadog Database Monitoring (DBM)** |
    | --- | --- | --- |
    | **소속 플랫폼** | AWS | Datadog |
    | **통합성** | AWS Aurora / RDS에 최적화됨 | 멀티클라우드 및 온프레미스 포함 다양한 DB 환경 지원 |
    | **설정 난이도** | Agent 설치 불필요 | Agent 설치 필요 |
    | **알림 / 경보 기능** | CloudWatch Alarms 기반 | Datadog Monitors 기반 (더 유연하고 고급) |
    | **비용 구조** | - CloudWatch 요금 모델 기반 (메트릭 수, 스토리지, API 요청 등)
    - vCPU/ACU‑hours 기반 사용량 요금 | Datadog DBM 요금 별도 (에이전트 및 사용량 기반) |
    - 현재 데이터베이스가 AWS RDS / Aurora로 운영 중이고, AWS 환경 외에 DB가 많지 않다면 CloudWatch Database Insights Advanced Mode가 비용, 기능 측면에서 효율적입니다.
    - AWS 외의 DB도 운영 중이거나 전체 Observability를 Datadog 하나의 통합 플랫폼으로 운영하고 싶다면, 비용과 확장성을 충분히 고려하되 Datadog DBM을 선택해볼 수 있습니다.
- 비용 비교
    - 다만, API 호출 기반 요금, 그리고 CloudWatch의 기타 서비스 비용은 별도 검토가 필요
        
        
        | DB 엔진 / 인스턴스 클래스 | 인스턴스 수 | vCPU 각 | 총 vCPU | 요율 ($0.0125/vCPU‑hour) | 1개월 (720hr) 월 요금 |
        | --- | --- | --- | --- | --- | --- |
        | PostgreSQL (db.m5.12xlarge) | 1대 | 48 | 48 | 0.0125 | 48 × 0.0125 × 720 = **$432** |
        | SQL Server (db.t3.small ×3) | 3대 | 2 | 6 | 0.0125 | 6 × 0.0125 × 720 = **$54** |
        | SQL Server (db.t3.xlarge) | 1대 | 4 | 4 | 0.0125 | 4 × 0.0125 × 720 = **$36** |
        | **총계** | — | — | **58** | — | **$522 /월** |
    - 다만, 로그 집계 / 전송에 따른 비용도 예측이 어려워 검토가 필요
        
        
        | 항목 | 개별 Agent 운영 | 통합 Agent 운영 (1대) |
        | --- | --- | --- |
        | DBM 라이선스 ($70/DB) | $350 | $350 (동일) |
        | EC2 비용 (Agent 용) | $424.80 (m6i.large ×5) | $84.96 (m6i.large ×1) |
        | EBS 비용 (100GB/Agent) | $40.00 | $8.00 |
        | Custom Metrics / Logs | $150~$250 (그 이상도 가능) | $80~$150 (그 이상도 가능) |
        | **총합계 (월)** | **$964.80~$1,064.80** | **$522.96~$592.96** |

## **4. CloudWatch Database Insights 활성화**

1. 콘솔 또는 AWS CLI 사용
    1. 콘솔 기준
        1. 설정 경로
            1. Amazon RDS Console > 데이터베이스 선택 > DB 인스턴스 선택 & 수정 > 모니터링 > Database Insights 활성화
    2. Database Insights의 "표준 모드" 또는 "고급 모드"를 활성화하도록 DB 인스턴스를 수정해도 다운타임이 발생하지 않습니다.
2. 콘솔 가이드
    1. [Standard Mode](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_DatabaseInsights.TurningOnStandard.html)
    2. [Advanced Mode](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_DatabaseInsights.TurningOnAdvanced.html)