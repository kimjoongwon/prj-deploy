# prj-deploy — 배포 상태(이미지 태그) 전용 저장소

GitOps 배포 기록 저장소. **이미지 태그만** 여기에 있고, 차트·환경설정·Application 정의는
[prj-devops](https://github.com/kimjoongwon/prj-devops)에서 관리한다.

아래 내용은 2026-10-22 원격 `main`의 태그 파일, ArgoCD Application,
Jenkinsfile·잡 정의를 대조한 구성 기준이다. 실행 중인 이미지·파드·알림 상태는 별도 운영 검증 대상이다.

## 연결 저장소와 책임

| 저장소 | 소유하는 내용 | 배포에서 읽는 위치 |
|---|---|---|
| [prj-core](https://github.com/kimjoongwon/prj-core) | 앱 소스, Dockerfile, Jenkinsfile | `main`은 prod 빌드, `stg`는 stg 빌드 |
| [prj-devops](https://github.com/kimjoongwon/prj-devops) | Helm 차트·환경 values, ArgoCD Application, Jenkins 잡·범프·롤백 스크립트 | `main` |
| `prj-deploy` | 환경별 앱 이미지 태그 | `main`의 `prod/`와 `stg/` |

`prod`와 `stg`는 이 저장소의 **디렉터리**다. 앱 소스 브랜치를 동기화해도 두 환경의
이미지 태그가 같아지는 것은 아니다. 범프 잡은 해당 환경 빌드가 push한 태그만 기록한다.

## 구조

```
prod/<앱이름>.yaml   # <앱>.image.tag를 가진 Helm values
stg/<앱이름>.yaml    # 같은 형식, stg 대상 앱만 존재
```

| 앱 | prod | stg |
|---|---|---|
| `core-api`, `admin-web`, `idp-api`, `idp-web` | 지원 | 지원 |
| `proposal-web`, `design-web`, `live-web`, `tool-storybook` | 지원 | 없음 |
| `dolmen-api`, `dolmen-web` | 없음 | 지원 |

현재 태그 파일은 prod 8개, stg 6개다. `prj-devops/environments/argocd/apps/`의
대응 Application이 `$values/<환경>/<앱>.yaml`을 참조한다. `srs-server` 같은
upstream 이미지와 인프라·개발 도구 이미지는 이 저장소의 태그 범프 대상이 아니다.

## 동작 방식

1. `prj-core`의 `main` push는 Jenkins `push-router`가 변경 경로에 맞는 앱 빌드를
   호출한다. `stg` 빌드는 앱 잡을 `BRANCH_NAME=stg`로 수동 실행한다.
   각 앱 빌드는 Harbor의 `<환경>/<앱>:<빌드 커밋 SHA 앞 12자>`로 push한다.
2. Harbor push 성공 후 `gitops-prod-image-bump` 잡이
   대응하는 `prod/<앱>.yaml`의 태그를 새 SHA로 커밋/푸시한다 (커밋 메시지:
   `ci(gitops): bump <앱> image to <태그>`). stg 빌드(BRANCH_NAME=stg)는
   `stg/<앱>.yaml`을 갱신하고 커밋 메시지는 `ci(gitops): bump stg/<앱> image to <태그>`.
3. GitHub 웹훅 → ArgoCD가 각 앱 Application(multi-source)에서
   `$values/{prod,stg}/<앱>.yaml`을 추가 values로 읽어 배포한다.
4. 태그 파일의 git log가 **앱 이미지 배포 희망 상태의 변경 기록**이다.
   앱별 기록은 `git log --oneline -- prod/<앱>.yaml` (stg는 `stg/<앱>.yaml`).
   실제 배포 완료는 ArgoCD `Synced`·`Healthy`, 해당 이미지 파드·env,
   Buzz `#cicd` 알림으로 확인한다.

`tool-storybook`은 `push-router` 자동 대상에서 제외되어 앱 잡을 수동 실행한다.

현재 `push-router`에는 Dolmen 잡도 `main` 자동 대상으로 들어 있지만 prod 태그 파일과
Application은 없다. Dolmen 운영 배포를 가정하지 말고 stg 잡·태그·Application을 사용한다.
이 구성 차이는 [빌드·배포 런북](https://github.com/kimjoongwon/prj-core/blob/main/docs/build-deploy-runbook.md)에서 함께 추적한다.

## 롤백

`prj-devops`의 `scripts/rollback.sh --app <앱>`은 최신 범프 커밋을 revert해
이전 이미지로 되돌린다. `--steps N`으로 N단계 롤백, `--dry-run`으로 결과 확인,
`--env stg`로 stg 롤백을 선택한다. 기본 대상 브랜치는 환경과 관계없이 `prj-deploy/main`이다.

현재 이 스크립트의 앱 허용 목록은 `core-api`, `admin-web`, `idp-api`, `idp-web`,
`proposal-web`, `tool-storybook`이다. 이 중 뒤 두 앱은 prod만 지원한다.
`design-web`, `live-web`, `dolmen-api`, `dolmen-web`은 범프를 지원하지만
롤백 스크립트는 아직 지원하지 않는다. 이 앱들은 해당 환경 파일의 최신 범프 커밋을
확인해 `git revert <범프 커밋>`으로 되돌린 뒤, revert 커밋을 원격 `prj-deploy/main`에
반영해야 ArgoCD가 이전 이미지를 배포한다. 먼저 되돌아갈 이미지가 Harbor에 있는지 확인한다.

새 앱은 태그 파일·차트·Application·잡·범프 허용 목록을 함께 준비해야 한다.
범프 스크립트는 없는 태그 파일을 자동 생성하지 않는다.

## 관련 문서

- [prj-devops — Jenkins GitOps 이미지 범프 가이드](https://github.com/kimjoongwon/prj-devops/blob/main/docs/jenkins-gitops-image-bump.md)
- [prj-devops — ArgoCD 웹훅/운영 가이드](https://github.com/kimjoongwon/prj-devops/blob/main/docs/argocd-prod-only-webhook-manual.md)
- [prj-devops — 배포 파이프라인](https://github.com/kimjoongwon/prj-devops/blob/main/docs/ai-autonomous-pipeline.md)
- [prj-core — 빌드·배포 런북](https://github.com/kimjoongwon/prj-core/blob/main/docs/build-deploy-runbook.md)

## 원칙

- 배포 데이터는 **이미지 태그만** 관리한다. README·AGENTS는 저장소 운영 문서이며, 차트·환경설정은 prj-devops가 소유한다.
- 일반 태그 변경은 Jenkins 범프 잡이 수행한다. 사람의 변경은 롤백 또는 사유를 남긴 긴급 복구·신규 앱 초기 등록으로 한정한다.
- 문서 갱신은 앱 태그를 바꾸지 않는다. 문서 커밋과 이미지 배포 완료를 구분한다.
