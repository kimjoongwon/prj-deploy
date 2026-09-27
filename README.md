# prj-deploy — 배포 상태(이미지 태그) 전용 저장소

GitOps 배포 기록 저장소. **이미지 태그만** 여기에 있고, 차트·환경설정·Application 정의는
[prj-devops](https://github.com/kimjoongwon/prj-devops)에서 관리한다.

## 구조

```
prod/<앱이름>.yaml   # 각 파일은 <앱>.image.tag 한 줄이 전부
```

지원 앱: idp-api, idp-web, core-api, admin-web, proposal-web, tool-storybook

## 동작 방식

1. Jenkins가 앱을 빌드해 Harbor에 push하면 `gitops-prod-image-bump` 잡이
   대응하는 `prod/<앱>.yaml`의 태그를 새 SHA로 커밋/푸시한다 (커밋 메시지:
   `ci(gitops): bump <앱> image to <태그>`).
2. GitHub 웹훅 → ArgoCD가 각 앱 Application(multi-source)에서
   `$values/prod/<앱>.yaml`을 추가 values로 읽어 배포한다.
3. 이 저장소의 git log가 곧 **플랫폼 전체의 순수 배포 기록**이다.
   앱별 기록은 `git log --oneline -- prod/<앱>.yaml`.

## 롤백

prj-devops의 `scripts/rollback.sh --app <앱>` 을 실행하면 이 저장소에서
최신 범프 커밋을 revert해 이전 이미지로 되돌린다 — `--steps N`으로 N단계 롤백,
`--dry-run`으로 결과 시뮬레이션. 수동 롤백은 `git revert <범프 커밋>` 후 push.

## 관련 문서

- [prj-devops — Jenkins GitOps 이미지 범프 가이드](https://github.com/kimjoongwon/prj-devops/blob/main/docs/jenkins-gitops-image-bump.md)
- [prj-devops — ArgoCD 웹훅/운영 가이드](https://github.com/kimjoongwon/prj-devops/blob/main/docs/argocd-prod-only-webhook-manual.md)

## 원칙

- 이 저장소에는 **태그 외에는 아무 것도 넣지 않는다** (운영 설정은 prj-devops).
- 사람이 직접 커밋하는 경우는 롤백뿐이라고 가정한다.
