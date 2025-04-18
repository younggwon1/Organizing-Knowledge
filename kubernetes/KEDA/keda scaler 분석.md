# KEDA Scaler 분석

## 분석

> KEDA Memory Scaler 동작을 위해 triggers 를 설정

> 참고 ) Kubernetes Version 이 1.30 이상일 경우 HPAContainerMetrics feature enabled가 기본 설정이기 때문에 containerName을 작성해야합니다. (1.30 이전 버전에서 containerName을 작성하려면 HPAContainerMetrics를 enabled 해야합니다.)

```yaml
triggers:
- type: memory
  metricType: Utilization
  metadata:
    value: "75"
    containerName: "{Container Name}"
```

하지만, Memory Scaler 임계치가 75%로 설정하더라도, 76%인 경우에는 상황에 따라 즉시 스케일링하지는 않아 왜그런지 의아했습니다.

- 정말로 임계치를 넘지 않아서 스케일아웃하지 않았던 거고, 스케일아웃되되면 HPA의 Event에 아래와 같이 정상 기록되었습니다.

```
HPA Events
 Events:
   Type     Reason                   Age                From                       Message
   ----     ------                   ----               ----                       -------
   Normal   SuccessfulRescale        8s                 horizontal-pod-autoscaler  New size: 15; reason: memory container resource utilization (percentage of request) above target
```

결과적으로 HPA는 계산된 퍼센트가 1%라도 넘으면 바로 스케일아웃하지는 않는다고 합니다. 왜 그런지 더 분석해보면 HPA의 알고리즘 동작을 이해해야합니다.

### 동작 원리

컨트롤 플레인이 판단하기에 비율이 1에 가깝다면 스케일링을 넘긴다고 합니다. 이로 인해 76%여도 스케일링이 안되었던 것입니다.

- [알고리즘 세부정보](https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale/#%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%EC%84%B8%EB%B6%80-%EC%A0%95%EB%B3%B4)

이는 Memory 뿐만 아니라 CPU 도 동일합니다.

참고 문서

- https://keda.sh/docs/2.16/scalers/memory/
