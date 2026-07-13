# KEDA Cascade 에러 트러블슈팅

KEDA 설치 후 첫 부하테스트에서 Pod 스케일링이 전혀 동작하지 않았습니다. `kubectl get events`에서 `KEDAScalerFailed`가 3,000번 이상(x3027) 반복 출력되고 있었습니다.

```
Warning  KEDAScalerFailed  x3027  keda-operator
error parsing Datadog metadata: missing AppKey
```

메시지만 보면 Datadog AppKey 설정 오류처럼 보였지만, 실제 Secret은 정상적으로 존재(`DATA: 2`)했습니다. 단순 설정 오류가 아니라는 판단하에 더 깊게 들어갔습니다.

## 원인

이벤트 타임스탬프를 시간순으로 정렬해 패턴을 확인했습니다.

```
T+0   Kafka 연결 실패 (kafka-1.kafka-headless not found)
T+1   KEDA operator가 전체 ScaledObject 재초기화 시도
T+2   scaler with id 0 not found. Len = 0   ← 모든 Scaler가 초기화 중 상태
T+3   재초기화 중 Datadog AppKey 읽기 실패 → missing AppKey 재발
      ↩ T+0으로 반복
```

`Len = 0`은 특정 스케일러 하나의 에러가 아니라 **KEDA가 ScaledObject 전체를 재초기화한다는 신호**였습니다. Kafka 트리거 1개의 연결 실패가 ScaledObject 전체를 재초기화시켰고, 그 과정에서 정상 동작 중이던 Datadog 트리거까지 함께 실패하는 cascade 구조였습니다.

근본 원인은 두 가지가 겹쳐 있었습니다. `bootstrapServers`가 StatefulSet 헤드리스 DNS 형식(`kafka-1.kafka-headless`)으로 설정되어 있었는데, 실제 환경의 Kafka는 아직 consumer group(`queue-processor-group`)이 생성되지 않은 상태였습니다. KEDA가 존재하지 않는 그룹에 반복적으로 연결을 시도하면서 위 cascade가 계속 재발했습니다.

## 행동

ScaledObject에서 Kafka 트리거를 임시로 제거하기로 했습니다.

먼저 `kubectl patch`로 클러스터에 바로 반영을 시도했습니다.

```
kubectl patch scaledobject oliveyoung-api-scaledobject \
  -n oliveyoung --type=json \
  -p='[{"op":"remove","path":"/spec/triggers/0"}]'
```

패치는 성공했지만, ArgoCD의 `selfHeal: true` 설정이 몇 초 만에 Git에 있는 원본 상태로 되돌려버렸습니다. GitOps 환경(`selfHeal: true`, `prune: true`)에서는 클러스터를 직접 수정해도 곧바로 무효화되고, Git에 있는 소스 자체를 바꿔야 한다는 것을 이 과정에서 확인했습니다.

그래서 `scaledobject.yaml`에서 Kafka 트리거를 제거하도록 수정한 뒤 커밋·푸시했습니다.

```
git add k8s/keda/scaledobject.yaml
git commit -m "fix: Kafka scaler 임시 비활성화"
git push origin main
```

ArgoCD가 변경된 버전을 감지해 동기화하도록 처리했습니다.

## 결과

Kafka 트리거 제거 후 `KEDAScalerFailed` 이벤트가 즉시 소멸했고, ScaledObject 상태가 아래와 같이 정상화되었습니다.

```
READY: True  /  ACTIVE: True  /  FALLBACK: False
```

- Datadog 트리거가 안정적으로 동작하며 HPA 메트릭이 정상 수집되는 것을 확인
- 이후 부하테스트를 진행할 수 있는 상태를 확보
- Kafka는 consumer group을 완전히 구성한 뒤 트리거로 다시 추가할 예정
