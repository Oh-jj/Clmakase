# OliveYoung 세일 이벤트 대응 인프라

![OliveYoung x AWS](docs/images/cover.png)

> 담당 파트: 네트워크(VPC) / 컨테이너(EKS, Karpenter)

---

## 1. 프로젝트 개요 & 인프라 핵심 성과

### 프로젝트 배경

저희가 선택한 서비스는 올리브영의 대규모 세일 이벤트, '올영세일'입니다. 올리브영은 연 4회 이상 정기 세일을 진행하는데, 세일 시작 시점에 대량의 트래픽이 단시간에 집중되는 특성이 있습니다. 이 순간 트래픽을 감당하지 못하면 아래와 같은 서비스 지연·접속 장애로 이어져 매출에 직접적인 타격을 입습니다.

![올리브영 온라인몰 서비스 지연 안내](docs/images/service-delay-notice.png)

이러한 문제 인식을 바탕으로 **"올영세일 대응 AWS 클라우드 인프라 구축"**을 프로젝트 목표로 삼았습니다.

### 담당 범위

올리브영 정기 세일의 순간 대량 트래픽에 대응하는 AWS EKS 기반 인프라 중, **VPC 네트워크 설계와 EKS 클러스터/노드 오토스케일링(Karpenter)** 파트를 담당했습니다.

---

## 2. 시스템 아키텍처 구조도 & 트래픽 흐름

![OliveYoung Architecture](docs/images/Oliveyoung-Architecture.png)

사용자 요청은 Route53 → CloudFront/S3(프론트엔드 정적 호스팅) → WAF → ALB를 거쳐 VPC 내부로 유입됩니다. VPC는 ap-northeast-2 리전에 2개 AZ(2a/2c)로 이중화되어 있으며, Public/Private App/Private Data 3-tier 서브넷으로 분리했습니다.

- **Public Subnet**: ALB, NAT Gateway 배치 — 인터넷 인그레스와 아웃바운드 경로만 노출
- **Private App Subnet**: EKS 워커 노드 배치. Karpenter가 EC2NodeClass/NodePool 설정에 따라 트래픽 양에 맞춰 노드를 자동 프로비저닝하며, ALB는 target-type: ip로 Pod에 직접 라우팅
- **Private Data Subnet**: Aurora MySQL(Multi-AZ), ElastiCache Redis 배치 — App 계층과 물리적으로 분리해 데이터 계층 보안 경계 확보
- AZ별로 NAT Gateway를 이중화해 한쪽 AZ 장애 시에도 아웃바운드 경로 유지

---

## 3. IaC 디렉토리 구조

```
terraform/
├── main.tf                 # 전체 모듈 오케스트레이션
├── karpenter_iam.tf         # Karpenter IAM (OIDC Trust, Node KMS)
├── variables.tf / outputs.tf / terraform.tfvars
└── modules/
    ├── vpc/                 # VPC, 3-tier 서브넷, NAT, 라우팅
    ├── security-groups/     # 7개 SG
    ├── eks/                 # 클러스터 + OIDC
    ├── ecr/                 # 컨테이너 이미지 레지스트리
    ├── rds/                 # Aurora MySQL Multi-AZ
    ├── elasticache/         # Redis
    ├── alb-controller/      # ALB Controller Helm 릴리스
    ├── argocd/              # ArgoCD Helm 릴리스
    ├── cli/                 # SSM 기반 Bastion
    ├── secrets/             # Secrets Manager
    ├── waf/                 # WAF
    ├── kms/                 # KMS
    ├── s3/                  # 프론트엔드 정적 호스팅
    ├── route53/             # DNS
    ├── acm/                 # SSL 인증서
    └── cloudfront/          # CDN
```

---

## 4. 파이프라인 흐름

```
git push main
  │
  ├─ [1] test             Gradle JUnit (allow_failure)
  ├─ [2] build            Docker build → ECR push (commit SHA 태그)
  ├─ [3] trivy-scan       CVE 취약점 스캔 (CRITICAL 시 실패)
  ├─ [4] update-manifest  k8s/deployment.yaml 이미지 태그 교체 → git push [skip ci]
  ├─ [5] deploy-secrets   EKS에 KEDA용 Secret 주입 (git 미포함, kubectl apply)
  ├─ [6] deploy-frontend  npm build → S3 sync → CloudFront 캐시 무효화
  └─ [7] load-test        k6 (수동 트리거, main 브랜치 한정)
       │
       ▼
   ArgoCD가 manifest 변경 감지 → EKS 롤링 배포 (selfHeal/prune)
```

- 이미지 태그를 `latest`가 아닌 commit SHA로 고정 → manifest 변경분이 생겨야 ArgoCD가 배포를 감지
- `update-manifest` 커밋에 `[skip ci]`를 붙여 파이프라인 무한 루프 방지
- Secret은 파이프라인 단계에서 `kubectl apply`로 직접 주입, git 리포지토리에는 값 자체를 남기지 않음
