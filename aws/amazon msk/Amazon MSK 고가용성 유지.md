# Amazon MSK 고가용성 유지

### 1. Amazon MSK Version Upgrade 시 고가용성 유지

[고가용성 모범 사례](https://docs.aws.amazon.com/ko_kr/msk/latest/developerguide/bestpractices.html#ensure-high-availability)

> Amazon MSK 버전 업그레이드는 한 번에 하나의 브로커를 오프라인으로 전환하는 Rolling 업데이트 프로세스를 통해 배포하므로 MSK의 가용성과 내구성을 유지합니다. 
>
> 리더 파티션을 사용하는 브로커가 오프라인 상태가 되면 Apache Kafka는 파티션 리더십을 재할당하여 클러스터의 다른 브로커에게 작업을 재분배합니다. 하지만 특정 시간에는 브로커를 이용할 수 없게 되기 때문에 생산자/소비자에게 영향을 미칠 수 있습니다. 클라이언트는 연결 거부/타임아웃 오류 또는 리더가 이전과 같지 않다는 메시지와 함께 연결 오류가 발생할 수 있지만, 올바른 리더에 대한 메타데이터를 다시 요청하고 사용 가능한 다른 브로커에 대해 자동으로 작업을 재시도합니다. 이는 클라이언트측 지연으로 나타날 수 있지만 클러스터/토픽의 복제 인자가 1보다 크게 설정되어 있는 한 클라이언트의 기능에는 영향을 미치지 않습니다.

##### Amazon MSK Version Upgrade 시 Pub 에서 발생한 Warning

~~~
[2024-02-06 10:47:03,491] WARN [Producer clientId=console-producer] Got error produce response with correlation id 328 on topic-partition mskversionupgradetest-0, retrying (2 attempts left). Error: NOT_LEADER_OR_FOLLOWER (org.apache.kafka.clients.producer.internals.Sender)
[2024-02-06 10:47:03,492] WARN [Producer clientId=console-producer] Received invalid metadata error in produce request on partition mskversionupgradetest-0 due to org.apache.kafka.common.errors.NotLeaderOrFollowerException: For requests intended only for the leader, this error indicates that the broker is not the current leader. For requests intended for any replica, this error indicates that the broker is not a replica of the topic partition.. Going to request metadata update now (org.apache.kafka.clients.producer.internals.Sender)
~~~

##### Amazon MSK Version Upgrade 시 Sub에서 발생한 Warning

~~~
[2024-02-06 11:23:01,627] WARN [Consumer clientId=consumer-console-consumer-51435-1, groupId=console-consumer-51435] Connection to node 2147483645(broker..) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
~~~

고가용성 클러스터 빌드의 권장 사항은 **RF를 3개와 minISR은 2를 갖는 3-AZ 클러스터를 배포하는 것**이다.

**RF**는 고가용성을 위해 클러스터의 브로커 수보다 작거나 같아야 한다. Topic을 생성할 때 지정하지 않으면 기본값(AZ 3개 클러스터의 경우 3, AZ 2개에 있는 클러스터의 경우 2)이 사용된다.

**MinISR**은 최소 동기화 내 복제본 수로 쓰기가 성공한 것으로 간주되기 위해 만족해야 하는 복제본의 수이다. 이는 최대 RF - 1로 설정한다. 클러스터가 2-AZ로만 배포되므로 이 값을 1 (RF-1, 즉 2-1 = 1) 로 설정할 수 있다.

> 런타임에서 RF와 minISR 변경은 Downtime이 없다. 그러나 [클러스터 전체 구성 변경](https://docs.aws.amazon.com/ko_kr/msk/latest/developerguide/msk-configuration-properties.html)을 진행하는 경우 Rolling Update로 인한 일부 성능 저하가 발생할 수 있다.

가용 영역에 장애가 발생 시 다운 타임과 가용성 측면에서 2-AZ(RF : 2)는 3-AZ(RF-3) 보다 데이터 손실 가능성이 높다. 

그리고 RF가 3인 토픽이 2-AZ 클러스터에 있는 경우, 3개의 파티션 복제본이 각각 서로 다른 가용 영역에 있을 수 없기 때문에 가용 영역에 장애 발생 시 가용성에 도움이 되지 않는다. 

브로커의 수보다 RF가 크다면 하나의 브로커에 복제본이 두 개 이상이 되는 경우가 발생하기 때문에 이 또한 가용성을 높여주지 않는다.

### 2. Turn on Multi-VPC Connectivity

> Multi-VPC Connectivity 활성화는 무중단으로 진행되며 Downtime은 없다.
>
> Multi-VPC Connectivity는 활성화하여도 MSK Connector에 끼치는 영향은 없다.
>
> Multi-VPC Connectivity는 활성화 후 Cluster Policy를 수정 즉, 특정 클라이언트에만 권한을 추가 하더라도 기존에 사용하시던 기존에 클러스터에 접근 가능했던 클라이언트에는 영향이 없다.

#### 참고 문서

1. [Amazon MSK 클러스터의 버전 업데이트 시](https://docs.aws.amazon.com/ko_kr/msk/latest/developerguide/version-support.html#version-upgrades-best-practices)
2. [다중 VPC 프라이빗 연결에 대한 권한](https://docs.aws.amazon.com/ko_kr/msk/latest/developerguide/mvpc-cross-account-permissions.html)
3. [Connect Kafka client applications securely to your Amazon MSK cluster from different VPCs and AWS accounts](https://aws.amazon.com/blogs/big-data/connect-kafka-client-applications-securely-to-your-amazon-msk-cluster-from-different-vpcs-and-aws-accounts)
4. [단일 리전에서의 Amazon MSK 다중 VPC 프라이빗 연결](https://docs.aws.amazon.com/ko_kr/msk/latest/developerguide/aws-access-mult-vpc.html#mvpc-requirements)
