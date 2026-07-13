# Karpenter CrashLoopBackOff 트러블슈팅

```
karpenter     karpenter-5bf9b94cc-m5k6h   0/1   CrashLoopBackOff   5 (9s ago)    6m54s
karpenter     karpenter-5d7dd7d87-fpbsc   0/1   CrashLoopBackOff   5 (49s ago)   6m54s
karpenter     karpenter-5d7dd7d87-pxv66   0/1   CrashLoopBackOff   5 (29s ago)   6m54s
```

## 원인

Karpenter를 배포한 초기 상태를 점검해보니, 서로 다른 지점에서 생긴 설정 문제 여러 개가 동시에 겹쳐 있었습니다.

| 영역 | 문제 |
|---|---|
| API 버전 | `node-class.yaml`, `node-pool.yaml`이 `karpenter.sh/v1beta1`로 작성됨 — 실제 설치된 Karpenter(1.0.6)의 storage version은 `v1` |
| nodeClassRef | `group`/`kind` 없이 `name: default`만 존재 — v1 스펙과 맞지 않음 |
| Discovery 태그 | EC2NodeClass가 `cloudwave-dev-vpc`를 참조하는데, 실제 서브넷/보안그룹 태그는 `cloudwave-eks` — 노드 배치 대상 자체를 찾지 못하는 상태 |
| Helm values | 값 파일 없이 CLI `--set` 옵션으로 수동 설치 — Controller Role ARN 오탈자 가능성 |
| clusterEndpoint | 미설정 |
| AMI | 구버전(EKS 1.30용) AMI ID로 하드코딩 |
| aws-auth | ConfigMap 관련 설정 자체가 없음 |
| Fallback 노드 | Managed Node Group이 삭제된 상태 — Karpenter가 실패하면 노드를 공급할 수단이 전혀 없는 구조 |

이 문제들이 동시에 존재한 상태로 Karpenter를 설치하니, **인증(Role) 실패 → cert/webhook 실패 → leader election 실패**, **CRD conversion 실패(v1beta1)**, **discovery 실패**가 한꺼번에 발생했고, 그 결과가 Karpenter Pod의 CrashLoopBackOff로 나타났습니다.

## 행동

문제를 원인별로 나눠 하나씩 해결했습니다.

**1. API 버전 불일치 해소**
`node-class.yaml`, `node-pool.yaml`의 `apiVersion`을 `v1beta1` → `v1`으로 전체 교체하고, `nodeClassRef`에 누락되어 있던 `group`/`kind`를 명시적으로 추가했습니다.

**2. Discovery 태그 정정**
EC2NodeClass가 참조하던 `cloudwave-dev-vpc` 태그를, 실제 서브넷/보안그룹에 붙어있는 `cloudwave-eks`로 되돌려 Karpenter가 노드를 배치할 서브넷/SG를 정상적으로 찾도록 했습니다.

**3. AMI 최신화**
하드코딩되어 있던 구버전(EKS 1.30용) AMI ID를 현재 클러스터 버전(EKS 1.32)에 맞는 AMI로 교체했습니다.

**4. Helm values 코드화**
`karpenter-values-fixed.yaml`을 새로 작성해 Controller Role ARN과 `clusterEndpoint`를 명시적으로 지정, CLI `--set` 방식의 수동 설치를 없애고 값을 코드로 관리하도록 바꿨습니다.

**5. aws-auth ConfigMap 정비**
기존 워커 노드용 Role과 Karpenter가 생성하는 노드용 Role을 모두 `aws-auth` ConfigMap에 등록해, 어떤 경로로 생성된 노드든 클러스터에 정상적으로 join되도록 했습니다.

## 결과

```
NAME                         READY   STATUS    RESTARTS   AGE
karpenter-7cd48fd548-699rr   1/1     Running   0          ~
karpenter-7cd48fd548-fflnx   1/1     Running   0          ~

NAME                      NODECLASS   NODES   READY   AGE
security-optimized-pool   default     0       True    ✅

NAME      READY   AGE
default   True    ✅
```

- Karpenter Pod가 `1/1 Running`으로 안정화되었고, NodePool/EC2NodeClass가 `Ready=True` 상태로 전환됨
- Managed Node Group 없이 Karpenter 단독으로 노드를 프로비저닝하는 구조가 정상 동작함을 확인
- 이후 Fargate Profile 구성, SQS 정리, `eks:DescribeCluster` 권한 추가 등 AWS 콘솔 상의 후속 조치가 이어졌으나, 이는 git에 남지 않는 수동 작업이라 별도로 관리

**한 줄 요약**: 문제는 단일 실수가 아니라 "구버전 API로 설계 → 세부 설정 축적 → 안전망(Managed Node Group) 제거와 동시에 태그 설정 회귀 발생"이 겹친 누적 구조였고, 이를 영역별로 나눠 하나씩 원복·수정해 해결했습니다.
