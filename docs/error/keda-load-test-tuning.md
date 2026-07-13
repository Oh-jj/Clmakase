# KEDA 부하테스트 기반 파라미터 튜닝

부하테스트 중 Pod가 생성되자마자 반복 삭제되는 현상이 있었습니다. KEDA 스케일링이 동작하는 것처럼 보이지만, 실제 트래픽 처리에는 전혀 기여하지 못하는 상태였습니다.

## 원인

### 문제 1 — queryValue 오설정으로 HPA thrashing

`kubectl get pods -w`로 모니터링하던 중, Pod가 생성→삭제를 15초 주기로 반복하는 것을 확인했습니다. HPA가 계속 max(20개)를 향해 스케일업을 시도하는 상태였습니다.

HPA 상태를 확인해보니:

```
2348 / 10 (avg)  →  목표 replicas = 2348 ÷ 10 = 234개
                     → max 20개 한도에 막혀 계속 충돌
```

초기 디버깅 편의를 위해 `queryValue`를 `"10"`으로 낮춰뒀던 값이 그대로 운영에 적용되어 있었던 것이 원인이었습니다.

### 문제 2 — readinessProbe 타이밍 불일치로 신규 Pod 미활용

스케일아웃으로 새 Pod가 생성됐지만, 부하테스트 중 실제 트래픽 처리에 기여하지 못하고 테스트 종료 직후 즉시 삭제되는 현상이 있었습니다.

Pod 로그에서 실제 기동 시간을 측정해보니:

```
Pod 생성:  08:32:47
Pod Ready: 08:35:58
           ─────────
           3분 11초 소요
```

부하테스트 지속 시간(3분)과 거의 동일했습니다. 즉 Pod가 Ready 상태가 되는 순간 테스트가 이미 종료되어 트래픽이 사라지고, HPA가 곧바로 scale down을 명령하는 구조였습니다.

타임라인으로 쪼개보면:

```
t=0s    테스트 시작, KEDA 트리거
t=30s   HPA 스케일업 결정 → Pod 생성
t=54s   init container 7개 완료 (~24초)
t=54s   앱 컨테이너 기동 시작
t=144s  readinessProbe 시작 (initialDelaySeconds: 90)
t=180s  ← 테스트 종료 (3분)
t=191s  Pod Ready — 이미 테스트 끝난 상태
```

init container 7개(~24초) + 앱 기동 시간 + `initialDelaySeconds: 90` 설정이 겹치면서, 총 기동 시간이 테스트 시간을 초과하고 있었습니다.

## 행동

**1. queryValue 재계산**

Datadog scaler의 rollup 구조를 파악해 적정값을 직접 계산했습니다.

```
실제 부하: 160 RPS
Datadog rollup(sum, 60) 방식 → 160 × 60 = 9,600
피크에서 목표 6 Pod → 9,600 ÷ 6 = 1,600
```

`queryValue`를 `"10"` → `"1600"`으로 수정한 뒤 git push (ArgoCD 동기화 반영).

**2. readinessProbe 지연 시간 하향 조정**

실제 앱 로그로 Spring Boot 기동 완료 시점을 확인한 뒤 `initialDelaySeconds` 값을 낮춰, 불필요하게 긴 대기를 제거하고 총 Pod 기동 시간을 단축했습니다.

## 결과

| 지표 | 수정 전 | 수정 후 |
|---|---|---|
| p(95) 응답시간 | 58ms | **15.9ms** |
| 에러율 | 2.3% | **0.81%** |

- HPA thrashing이 사라지고 Pod 수가 안정적으로 목표(6개)에 수렴
- 신규 Pod가 테스트 진행 중에 Ready로 전환되는 것을 확인, `SuccessfulRescale` 이벤트 로그로 실제 스케일아웃 동작을 검증
