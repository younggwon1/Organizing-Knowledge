# EKS Cluster 삭제 방지 기능

EKS Cluster 운영 시 예기치 못하게 EKS Cluster를 삭제하게 될 경우 EKS Cluster에서 운영 중인 서비스의 불능으로 인해 장애가 발생합니다.

이를 방지하기 위해 EKS Cluster를 삭제하지 않도록 보호할 수 있습니다. 

만약 클러스터에 삭제 보호를 활성화한 경우, 클러스터를 삭제하기 전에 먼저 삭제 보호를 비활성화해야 합니다. 혹여나 삭제 보호가 활성화된 상태에서 EKS Cluster의 삭제를 요청할 경우 **InvalidRequestException** 에러가 발생합니다.

아직 Console에는 반영되지 않으나, aws cli에서는 설정 가능합니다.

중요
삭제 보호를 켜고 끄려면 해당 클러스터에 대한 `UpdateClusterConfig` 권한이 있어야 하고, 최종 삭제를 위해서는 `DeleteCluster` 권한도 필요합니다.

기존 클러스터에 대한 삭제 보호를 활성화하려면...
```
$ aws eks update-cluster-config --name <cluster-name> --region <aws-region> --deletion-protection
```

기존 클러스터에 대한 삭제 보호를 비활성화하려면...
```
$ aws eks update-cluster-config --name <cluster-name> --region <aws-region> --no-deletion-protection
```

참고 문서

[1]. [Amazon EKS, 실수로 인한 클러스터 삭제 방지를 위해 안전 제어 기능 추가](https://docs.aws.amazon.com/eks/latest/userguide/deletion-protection.html)
