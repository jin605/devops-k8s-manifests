# Argo CD Runbook V3 (Argo Rollouts + Blue/Green 전략)

이 문서는 V2(Upstream Trigger 기반) 파이프라인 흐름은 유지하면서, **Kubernetes 리소스를 Deployment에서 Argo Rollouts(`kind: Rollout`)로 전환**하고 **Blue/Green 배포 전략**을 적용한 구성을 정리한다.

현재 manifest 저장소(`devops-k8s-manifests`) 기준으로:

- `university-vue/rollout.yaml` : Blue/Green + **Auto Promote(자동 승격)**
- `department-api/rollout.yaml` : Blue/Green + **Manual Promote(수동 승격)**
- 각 앱은 `*-active`, `*-preview` Service를 사용한다.

---

# Argo Rollouts 개요 (수업자료 보강)

## 1) Argo Rollouts

- Argo Rollouts는 Kubernetes에서 **Canary**, **Blue-Green**과 같은 점진적 배포 전략을 지원하는 컨트롤러/CRD 기반 도구이다.
- 배포 중 문제 발생 시 **자동 또는 수동 롤백(undo)** 을 지원한다.

## 2) 주요 배포 전략

### 2.1) Blue-Green

- 기존 버전(Stable)과 새 버전(Preview)을 동시에 운영하다가 준비가 완료되면 **트래픽을 한 번에 전환**하는 방식이다.
- Argo Rollouts Blue/Green은 보통 아래 2개 Service로 트래픽을 분리한다.
  - `activeService`: 운영 트래픽
  - `previewService`: 새 버전 검증 트래픽

### 2.2) Canary

- 새 버전을 일부 트래픽부터 점진적으로 늘려가며 배포하는 방식이다.

## 3) Argo Rollouts Controller 설치

Argo Rollouts Controller는 Rollout 리소스를 감시하며 정의된 배포 전략(Canary, Blue-Green 등)에 따라 트래픽 전환과 배포 과정을 제어한다.

> 아래는 Windows PowerShell 기준 예시다.

네임스페이스 생성:

```powershell
kubectl create namespace argo-rollouts
```

설치 매니페스트 다운로드:

```powershell
Invoke-WebRequest -Uri "https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml" -OutFile "install.yaml"
```

다운로드한 `install.yaml`에서 아래 값을 `false`로 수정:

```yaml
securityContext:
  runAsNonRoot: false
```

설치 적용:

```powershell
kubectl apply -n argo-rollouts -f install.yaml
```

설치 확인:

```powershell
kubectl get all -n argo-rollouts
```

## 4) `kubectl argo rollouts` 설치 (Windows)

`kubectl argo rollouts`는 Rollout 상태 조회, promote/rollback 등 운영 명령을 제공하는 kubectl 플러그인이다.

다운로드:

```powershell
Invoke-WebRequest -Uri "https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-windows-amd64" -OutFile "kubectl-argo-rollouts.exe"
```

플러그인 경로 생성:

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.argorollouts\bin"
```

이동:

```powershell
Move-Item -Path ".\kubectl-argo-rollouts.exe" -Destination "$env:USERPROFILE\.argorollouts\bin\" -Force
```

PATH 등록:

```powershell
setx PATH "$($env:Path);$env:USERPROFILE\.argorollouts\bin"
```

버전 확인:

```powershell
kubectl argo rollouts version
```

---

# Blue-Green 배포 실습 (현재 repo 매핑)

## 1) 서비스(Service) 생성

Blue-Green 배포에서는 `Active`와 `Preview` 두 개의 서비스(Service)를 사용하여 트래픽 전환과 사전 검증을 수행한다.

### 1.1) Active 서비스

- 실제 사용자 요청을 처리하는 서비스(Service)이다.
- 현재 운영 중인(Stable) 버전의 파드(Pod)로 트래픽이 전달된다.

예시 템플릿:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: active-service
spec:
  type: <Service Type>
  selector:
    app: <Pod 선택기>
  ports:
  - port: <서비스의 Port>
    targetPort: <Pod에서 사용중인 Port>
```

### 1.2) Preview 서비스

- 새로 배포된 버전을 미리 검증하기 위한 서비스(Service)이다.
- 운영 트래픽에는 노출되지 않고 테스트 용도로 사용된다.

예시 템플릿:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: preview-service
spec:
  type: <Service Type>
  selector:
    app: <Pod 선택기>
  ports:
  - port: <서비스의 Port>
    targetPort: <Pod에서 사용중인 Port>
```

현재 repo 매핑:

- `university-vue/services.yaml`
  - `university-vue-active`, `university-vue-preview`
- `department-api/services.yaml`
  - `department-api-active`, `department-api-preview`

## 2) 롤아웃(Rollout) 생성

- Rollout은 Kubernetes 기본 Deployment를 대체/확장해 사용하며, 고급 배포 전략과 롤백을 지원하는 리소스이다.
- Argo Rollouts에서는 Deployment 대신 Rollout 리소스를 사용해 배포를 관리한다.
- Rollouts controller가 Rollout 리소스를 감시하면서 정의된 전략에 따라 배포를 수행한다.

예시 템플릿:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: <롤아웃 이름>
spec:
  replicas: <Pod 유지 개수>
  selector:
    matchLabels:
      app: <관리할 Pod를 찾는 선택기>
  template:
    # 이하 생략
  strategy:
    blueGreen:
      activeService: active-service
      previewService: preview-service
      autoPromotionEnabled: false
```

현재 repo 매핑:

- `university-vue/rollout.yaml` (autoPromotionEnabled: `true`)
- `department-api/rollout.yaml` (autoPromotionEnabled: `false`)

## 3) Argo Rollouts 명령어

### 3.1) Rollout 목록 조회

```bash
kubectl argo rollouts list rollouts
```

### 3.2) Rollout 상태 조회/모니터링

```bash
kubectl argo rollouts get rollout myapp-rollout
```

### 3.3) 새 버전으로 승격(Promote)

```bash
kubectl argo rollouts promote myapp-rollout
```

### 3.4) 롤백(Undo)

```bash
kubectl argo rollouts undo myapp-rollout
```

### 3.5) 대시보드(Dashboard)

```bash
kubectl argo rollouts dashboard
```

---

## 큰 그림 (CI/CD + Rollout)

```text
GitHub (devops-university-app)
  |
  | webhook (push)
  v
Jenkins job: devops-university-app
  |
  | docker build & push (DockerHub)
  v
DockerHub
  - <image>:<BUILD_NUMBER>

Jenkins job: devops-university-app
  |
  | trigger downstream with params
  v
Jenkins job: university-k8s-manifests (devops-k8s-manifests)
  |
  | update rollout.yaml image tag
  | commit & push
  v
GitHub (devops-k8s-manifests)
  |
  | watched by Argo CD
  v
Argo CD Application (Auto Sync)
  |
  | sync/apply
  v
Kubernetes
  - Rollout Controller가 새 ReplicaSet 생성 (Preview)
  - Promote(자동/수동)에 따라 Active Service 전환
```

핵심은 V3에서도 동일하게:

- Argo CD는 **DockerHub를 감시하지 않고**, **Git(Manifest Repo) 변경만 감시**한다.
- Git 변경이 적용되면, **Argo Rollouts Controller**가 Rollout 리소스를 기반으로 Blue/Green 전환을 수행한다.

## Manifest 구성 (Blue/Green)

### Rollout 리소스

두 애플리케이션은 Deployment 대신 `kind: Rollout`을 사용한다.

- `university-vue/rollout.yaml`
  - `strategy.blueGreen.activeService: university-vue-active`
  - `strategy.blueGreen.previewService: university-vue-preview`
  - `autoPromotionEnabled: true`
  - `autoPromotionSeconds: 30` (Preview로 뜬 뒤 30초 후 자동으로 Active 전환)
  - `scaleDownDelaySeconds: 30` (이전 버전 RS를 잠시 유지)
- `department-api/rollout.yaml`
  - `strategy.blueGreen.activeService: department-api-active`
  - `strategy.blueGreen.previewService: department-api-preview`
  - `autoPromotionEnabled: false` (Preview 검증 후 수동 Promote 필요)

### Service 리소스

각 앱은 2개의 Service를 가진다.

- Active Service: 운영 트래픽 처리 (`*-active`)
- Preview Service: 새 버전 검증용 (`*-preview`)

서비스는 아래 파일에 선언되어 있다.

- `university-vue/services.yaml`
- `department-api/services.yaml`

주의: Blue/Green에서 Service selector는 Rollouts controller가 전환 과정에서 조정할 수 있다(Active/Preview가 각각 특정 ReplicaSet으로 라우팅되도록).

## Jenkins 구성 (manifest repo 업데이트)

`devops-k8s-manifests` 저장소의 `Jenkinsfile`은 image tag를 **Deployment가 아니라 Rollout manifest**에서 갱신한다.

치환 대상:

```text
university-vue/rollout.yaml
  jin604/university-vue:<tag> -> jin604/university-vue:${DOCKER_IMAGE_VERSION}

department-api/rollout.yaml
  jin604/department-service:<tag> -> jin604/department-service:${DOCKER_IMAGE_VERSION}
```

그 외 V2와 동일하게 commit/push 후 Argo CD가 Git 변경을 감지해 Sync한다.

## 배포/검증/승격(Promote) 흐름

### 1) Argo CD Sync로 Rollout 업데이트 반영

Argo CD가 Git 변경을 sync/apply 하면 Rollout 리소스의 `spec.template.spec.containers[].image`가 변경된다.

그 결과 Rollouts controller가 새 ReplicaSet(새 버전)을 만들고, 기본적으로 **Preview Service로 먼저 트래픽을 보낼 준비**를 한다.

### 2) Preview 검증

Preview로 뜬 버전을 확인하려면 Preview Service를 대상으로 확인한다.

예시(포트포워딩):

```bash
kubectl -n university port-forward svc/department-api-preview 18088:8088
kubectl -n university port-forward svc/university-vue-preview 18080:80
```

### 3) Promote(운영 전환)

#### university-vue (자동 Promote)

`autoPromotionEnabled: true` 이므로 Preview가 준비되면 `autoPromotionSeconds` 이후 자동으로 Active Service가 새 버전으로 전환된다.

#### department-api (수동 Promote)

`autoPromotionEnabled: false` 이므로 운영 전환은 수동으로 수행한다.

```bash
kubectl argo rollouts promote department-api-rollout -n university
```

## 확인 명령어 (Rollouts 관점)

Rollout 목록:

```bash
kubectl get rollouts -n university
```

Rollout 상세(기본 kubectl):

```bash
kubectl get rollout department-api-rollout -n university -o yaml
kubectl get rollout university-vue-rollout -n university -o yaml
```

Rollouts 플러그인(권장)이 설치되어 있다면 상태 확인이 더 편하다:

```bash
kubectl argo rollouts get rollout department-api-rollout -n university
kubectl argo rollouts get rollout university-vue-rollout -n university
```

Active/Preview 서비스 확인:

```bash
kubectl get svc -n university | findstr /i "active preview"
```

## 트러블슈팅 체크리스트 (V3)

- `university-k8s-manifests` job에서 `rollout.yaml`의 image tag가 기대대로 변경/커밋/푸시 되었는지 확인
- Argo CD Application이 `OutOfSync`에서 `Synced`로 전환되는지, Sync 실패 이벤트가 있는지 확인
- Rollout이 Preview 단계에서 멈췄다면:
  - `department-api`는 의도적으로 수동 Promote가 필요함 (`autoPromotionEnabled: false`)
  - readinessProbe / livenessProbe 실패로 새 RS가 Ready가 안 되는지 확인
- Service가 예상과 다르게 라우팅되면:
  - `*-active`, `*-preview` Service 존재 여부 확인
  - Rollouts controller가 Service selector를 갱신했는지(Active/Preview가 다른 ReplicaSet을 가리키는지) 확인
