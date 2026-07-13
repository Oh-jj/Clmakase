# KEDA Cascade 에러 트러블슈팅

```
KEDAScalerFailed: kafka: tried to use a client that was closed
KEDAScalerFailed: scaler with id 0 not found. Len = 0
KEDAScalerFailed: error parsing Datadog metadata: missing AppKey
```

## 원인

k6 부하테스트 중 Pod 수가 전혀 늘지 않는 문제가 발생해 `kubectl get events`로 확인해보니, 아래 순서로 에러가 반복되고 있었습니다.

| 순서 | 에러 | 의미 |
|---|---|---|
| ① | `kafka: tried to use a client that was closed` | Kafka 연결 실패 |
| ② | `scaler with id 0 not found. Len = 0` | ScaledObject 전체 재초기화 신호 |
| ③ | `error parsing Datadog metadata: missing AppKey` | 정상 동작 중이던 Datadog 스케일러까지 재초기화 도중 연쇄 실패 |
| ④ | ①번 에러 반복 | 무한 반복 |

`Len = 0`은 특정 스케일러 하나의 에러가 아니라 **KEDA가 ScaledObject 전체를 재초기화한다는 신호**였습니다. Kafka 트리거가 실패할 때마다 KEDA가 Datadog을 포함한 모든 스케일러를 통째로 리셋했고, 그 과정에서 정상이던 Datadog 스케일러까지 함께 실패하는 cascade 구조였습니다.

근본 원인은 **Kafka consumer group이 아직 생성되지 않은 상태**에서 KEDA가 존재하지 않는 group에 반복적으로 연결을 시도한 것이었습니다.

## 행동

ScaledObject에서 Kafka 트리거를 임시로 제거하기로 했습니다.

먼저 `kubectl patch`로 클러스터에 바로 반영을 시도했지만, ArgoCD의 `selfHeal: true` 설정이 몇 초 만에 Git에 있는 원본 상태로 되돌려버렸습니다. GitOps 환경에서는 클러스터를 직접 수정해도 곧바로 무효화되고, Git에 있는 소스 자체를 바꿔야 한다는 것을 이 과정에서 확인했습니다.

그래서 `scaledobject.yaml`에서 Kafka 트리거를 제거하도록 수정한 뒤 커밋·푸시했고, ArgoCD가 변경된 버전을 감지해 동기화하도록 처리했습니다.

## 결과

- Cascade 에러가 해소되고, Datadog RPS 트리거 기반 스케일링이 정상 동작
- Kafka는 consumer group을 완전히 구성한 뒤 트리거로 다시 추가할 예정
