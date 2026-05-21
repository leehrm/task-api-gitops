# task-api-gitops

`task-api-gitops`는 `task-api` 애플리케이션을 Amazon EKS에 FluxCD 기반 GitOps 방식으로 배포하기 위한 저장소입니다.
Kubernetes 클러스터에 배포되어야 하는 **원하는 상태(desired state)** 를 선언하는 저장소입니다.

## 관련 저장소

| 저장소 | 역할 |
|---|---|
| `task-api-platform` | FastAPI 애플리케이션 코드, Dockerfile, Helm Chart 관리 |
| `task-api-gitops` | FluxCD가 감시하는 GitOps 배포 선언 관리 |
| `aws-eks-terraform-lab` | EKS, ECR, VPC, EBS CSI 등 AWS 인프라 관리 |

## 저장소 구조

```text
clusters/
└── dev/
    ├── flux-system/
    │   ├── gotk-components.yaml
    │   ├── gotk-sync.yaml
    │   └── kustomization.yaml
    │
    ├── task-api-source.yaml
    ├── task-api-kustomization.yaml
    │
    └── apps/
        └── task-api/
            ├── kustomization.yaml
            ├── namespace.yaml
            └── helmrelease.yaml
```

## 주요 구성

### 1. Flux bootstrap 리소스

`clusters/dev/flux-system/` 아래 파일들은 `flux bootstrap github` 명령으로 생성된 FluxCD 기본 구성입니다.

- `gotk-components.yaml`: Flux controller 설치 리소스
- `gotk-sync.yaml`: Flux가 이 Git 저장소를 감시하도록 설정
- `kustomization.yaml`: 위 두 파일을 묶어서 적용

Flux bootstrap으로 생성된 `flux-system` GitRepository는 `task-api-gitops` 저장소의 `main` 브랜치와 `./clusters/dev` 경로를 감시합니다.
별도로 `task-api-source.yaml`은 `task-api-platform` 저장소의 `main` 브랜치를 Helm Chart source로 참조합니다.

### 2. GitRepository

`task-api-source.yaml`은 FluxCD가 `task-api-platform` 저장소를 가져오도록 정의합니다.

```yaml
kind: GitRepository
metadata:
  name: task-api-platform
  namespace: flux-system
spec:
  url: https://github.com/leehrm/task-api-platform
  ref:
    branch: main
```

이 설정을 통해 Flux는 `task-api-platform` 저장소 안의 Helm Chart를 참조할 수 있습니다.

### 3. Kustomization (Kubernetes resource)

`task-api-kustomization.yaml`은 FluxCD에게 어떤 경로를 클러스터에 적용할지 알려주는 리소스입니다.

```text
Flux야, 이 GitOps 저장소의 clusters/dev/apps/task-api 경로를 보고
그 안의 YAML들을 EKS 클러스터에 적용해줘.
```

```yaml
kind: Kustomization
metadata:
  name: task-api
  namespace: flux-system
spec:
  path: ./clusters/dev/apps/task-api
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

| 설정 | 의미 |
|---|---|
| `path` | Flux가 적용할 Git 경로 |
| `prune: true` | Git에서 제거된 리소스를 클러스터에서도 제거 |
| `sourceRef` | 어떤 GitRepository를 기준으로 볼지 지정 |
| `wait: true` | 리소스 적용이 완료될 때까지 기다림 |

### 4. kustomization.yaml

`apps/task-api/kustomization.yaml`은 해당 폴더 안에서 어떤 YAML을 묶어서 적용할지 정의합니다.

```yaml
resources:
  - namespace.yaml
  - helmrelease.yaml
```

 `task-api` 배포에는 `namespace.yaml`과 `helmrelease.yaml`이 필요하다는 뜻입니다.

### 5. HelmRelease

`helmrelease.yaml`은 FluxCD의 Helm Controller가 `task-api` Helm Chart를 설치하거나 업데이트하도록 정의합니다.

현재 설정의 핵심은 다음과 같습니다.

```yaml
chart:
  spec:
    chart: ./helm/task-api
    reconcileStrategy: Revision
    sourceRef:
      kind: GitRepository
      name: task-api-platform
      namespace: flux-system

values:
  appVersion: "0.2.2"

  image:
    repository: 519330023984.dkr.ecr.ap-northeast-1.amazonaws.com/task-api
    tag: "0.2.2"
```

이 설정을 통해 `task-api-platform` 저장소의 Helm Chart를 사용하고, ECR에 있는 `task-api:0.2.2` 이미지를 배포합니다.

## FluxCD 동작 흐름

이 저장소 기준의 전체 배포 흐름은 다음과 같습니다.

```text
1. Flux가 task-api-gitops 저장소의 main 브랜치를 감시
2. Kustomization이 clusters/dev/apps/task-api 경로를 적용
3. kustomization.yaml이 namespace.yaml, helmrelease.yaml을 포함
4. HelmRelease가 task-api-platform의 Helm Chart를 참조
5. Helm Controller가 task-api를 EKS에 배포
```

| 리소스 | 역할 |
|---|---|
| `GitRepository` | Flux가 가져올 Git 저장소 정의 |
| `Kustomization` | Git 저장소 안에서 적용할 경로 정의 |
| `kustomization.yaml` | 해당 경로에서 적용할 YAML 목록 정의 |
| `HelmRelease` | Helm Chart 배포 방식 정의 |
| `Helm Chart` | 실제 Deployment, Service, PostgreSQL 리소스 생성 |


## 실습에서 확인한 내용

- FluxCD bootstrap 완료
- `task-api-gitops` 저장소와 FluxCD 연결
- `task-api-platform` 저장소를 Flux GitRepository로 등록
- HelmRelease 기반 `task-api` 배포
- ECR 이미지 `task-api:0.2.2` 배포
- `/version` endpoint에서 `0.2.2` 반환 확인

```bash
kubectl port-forward -n task-api svc/task-api 8000:8000
curl http://localhost:8000/version
```

예상 응답:
```json
{"app":"task-api","version":"0.2.2"}
```

## 자주 사용하는 명령어

Flux 상태 확인:

```bash
flux get sources git
flux get kustomizations
flux get helmreleases -A
```

수동 reconcile:

```bash
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization task-api --with-source
flux reconcile helmrelease task-api -n task-api
```

배포 리소스 확인:

```bash
kubectl get pods -n task-api
kubectl get svc -n task-api
kubectl get pvc -n task-api
```

## FluxCD 기본 개념 정리

클러스터에서 직접 `kubectl edit`으로 수정하더라도 Git에 없는 변경이라면 Flux가 다시 Git 상태로 되돌릴 수 있습니다.

- FluxCD는 Git 저장소를 감시하는 GitOps Controller이다.
- FluxCD는 Git에 반영된 배포 선언을 클러스터에 적용한다.
- Reconcile은 Git 상태와 클러스터 상태를 맞추는 작업이다.
- `GitRepository`는 소스 저장소를 정의한다.
- `Kustomization`은 적용할 Git 경로를 정의한다.
- `HelmRelease`는 Helm Chart 배포를 정의한다.