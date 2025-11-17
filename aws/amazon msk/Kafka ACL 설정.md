# Kafka ACL 설정

### 문제 상황

> 개발 환경에 구성된 DMS 작업 중 MSK(Kafka) 를 Target으로 하는 모든 태스크들이 실패하는 상황.



### 문제 파악

> DMS 설정 시 IAM Role 기반 아닌 SASL/SCRAM 방식으로 접근, 이때 ACL 을 통해 액세스를 제어하는 것으로 파악하여 권한 에러가 날 것으로 예상.



### 문제 해결

<https://docs.aws.amazon.com/ko_kr/msk/latest/developerguide/msk-acls.html>

~~~
$ ./kafka-acls.sh \
--bootstrap-server {broker endpoints}:9096 \
--command-config client.properties \
--add \
--allow-principal "User:*" \
--operation All \
--topic '*' \
--group '*'
~~~

-> 모든 User, Topic, Group 에 대해서 권한을 열었지만, 타이트하게 걸 수도 있을 것으로 보인다.



### 참고 

**[port 정보](https://docs.aws.amazon.com/msk/latest/developerguide/port-info.html)**

- 일반 텍스트로 브로커와 통신하려면 포트 9092를 사용하십시오.
- TLS 암호화를 사용하여 브로커와 통신하려면 AWS 내 액세스에는 포트 9094를 사용하고 퍼블릭 액세스에는 포트 9194를 사용하십시오.
- SASL/SCRAM을 사용하여 브로커와 통신하려면 AWS 내에서 액세스하려면 포트 9096을 사용하고 퍼블릭 액세스에는 포트 9196을 사용하십시오.
- [IAM 액세스 제어를](https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html) 사용하도록 설정된 클러스터의 브로커와 통신하려면 AWS 내 액세스에는 포트 9098을 사용하고 퍼블릭 액세스에는 포트 9198을 사용하십시오.
- TLS 암호화를 사용하여 Apache ZooKeeper와 통신하려면 포트 2182를 사용하십시오. Apache ZooKeeper 노드는 기본적으로 포트 2181을 사용합니다.