# Argo CD Runbook V2 (Webhook + Upstream Trigger 기반)

이 문서는 `devops-university-app` 저장소에서 빌드가 발생했을 때, Jenkins가 Docker 이미지를 빌드/푸시하고 **manifest 저장소(`devops-k8s-manifests`)의 image tag를 갱신**한 뒤, Argo CD가 Git 변경을 감지해 Kubernetes에 자동 배포하는 흐름을 정리한 것이다.

V1과 가장 큰 차이는 **manifest 저장소 Jenkins job이 “웹훅으로 직접 시작”되는 구조가 아니라**, `devops-university-app` 파이프라인에서 **`university-k8s-manifests` job을 upstream으로 트리거**하도록 구성했다는 점이다(특히 CD 구간).

참고: 기존 문서 형식/배경은 `ArgocdRunbook.md`를 기반으로 하고, 본 문서는 V2(Upstream Trigger 기반) 변경점 위주로 정리한다.

## 큰 그림 (CI/CD)

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
  | update deploy.yaml image tag
  | commit & push (GitHub)
  v
GitHub (devops-k8s-manifests)
  |
  | watched by Argo CD
  v
Argo CD Application (Auto Sync)
  |
  | sync/apply
  v
Kubernetes (rollout -> node pulls image)
```

핵심은 V2에서도 동일하게:

- Argo CD는 **DockerHub를 감시하지 않고**, **Git(Manifest Repo) 변경만 감시**한다.
- Kubernetes가 새 Pod를 만들 때 **노드가 DockerHub에서 이미지를 pull**한다.

## Jenkins 구성 (devops-university-app)

`devops-university-app` Jenkinsfile은 다음 역할을 한다.

1. 변경 감지: `university-vue/`, `department-api/` 변경 여부로 빌드 대상 결정
2. DockerHub 로그인
3. APP/API 이미지 빌드 & 푸시 (tag: `BUILD_NUMBER`)
4. downstream job(`university-k8s-manifests`) 트리거

downstream 트리거 파라미터:

```text
DOCKER_IMAGE_VERSION = BUILD_NUMBER
DID_BUILD_APP        = SHOULD_BUILD_APP (true/false)
DID_BUILD_API        = SHOULD_BUILD_API (true/false)
```

## Jenkins 구성 (devops-k8s-manifests / university-k8s-manifests)

`devops-k8s-manifests` 저장소의 Jenkinsfile(이 저장소의 `Jenkinsfile`)은 **manifest repo의 deploy.yaml에서 image tag를 바꾸고 commit/push**한다.

파라미터:

- `DOCKER_IMAGE_VERSION` (string): 반영할 이미지 태그 (예: Jenkins `BUILD_NUMBER`)
- `DID_BUILD_APP` (string, "true"/"false"): Vue 배포 manifest 갱신 여부
- `DID_BUILD_API` (string, "true"/"false"): API 배포 manifest 갱신 여부

동작:

1. `main` 체크아웃
2. (조건부) `university-vue/deploy.yaml`의 image tag 변경
3. (조건부) `department-api/deploy.yaml`의 image tag 변경
4. (조건부) git commit/push

현재 Jenkinsfile 기준으로 실제 치환되는 패턴:

```text
university-vue/deploy.yaml
  jin604/university-vue:<tag> -> jin604/university-vue:${DOCKER_IMAGE_VERSION}

department-api/deploy.yaml
  jin604/department-service:<tag> -> jin604/department-service:${DOCKER_IMAGE_VERSION}
```

## university-k8s-manifests Job 설정 요약

`university-k8s-manifests` Jenkins job은 “Pipeline script from SCM”으로 구성되어 있다.

- GitHub project URL: `git@github.com:jin605/devops-k8s-manifests.git/`
- Pipeline Definition: `Pipeline script from SCM`
  - SCM: Git
  - Repository URL: `git@github.com:jin605/devops-k8s-manifests.git`
  - Credentials: `github-k8s-manifests` (SSH private key)
  - Script Path: `Jenkinsfile`
  - Lightweight checkout: 사용
- Build Parameters:
  - `DOCKER_IMAGE_VERSION` (String Parameter)
  - (현재 저장소 Jenkinsfile에는 `DID_BUILD_APP`, `DID_BUILD_API`도 parameter로 정의되어 있으므로 job에도 동일하게 추가되어야 함)

## Argo CD (CD 구간)

V2에서도 Argo CD의 CD 책임은 동일하다.

- Argo CD Application은 **manifest repo의 변경을 감지**하고
- Git 상태를 기준으로 **Kubernetes에 sync/apply**한다

따라서 “배포가 안 된다”의 1차 원인 대부분은 아래 둘 중 하나다.

1) manifest repo에 기대한 변경 커밋이 안 올라감 (Jenkins downstream job 확인)
2) Argo CD가 아직 reconcile 전이거나, Sync 실패/권한/리소스 오류 (Argo CD Application 이벤트 확인)

## 트러블슈팅 체크리스트 (현 구조 기준)

- `devops-university-app` job에서 `university-k8s-manifests`가 실제로 트리거 되었는지 확인
- `university-k8s-manifests` job 콘솔에서 `DOCKER_IMAGE_VERSION`, `DID_BUILD_APP/API` 값이 기대대로 들어왔는지 확인
- `devops-k8s-manifests` GitHub에 `Update Image Version <tag>` 커밋이 올라왔는지 확인
- Argo CD Application 상태가 `Synced/Healthy`로 회복되는지, `OutOfSync`가 유지되는지 확인
