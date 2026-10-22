# prj-deploy — AI 작업 규칙

이미지 태그 상태 저장소. ArgoCD가 각 앱 Application에서 `prod|stg/<앱>.yaml`의 `tag:`를 읽는다 (multi-source).

## 연결 범위 (2026-10-22 형상 기준)

- 앱 소스·빌드는 `prj-core`, 차트·Application·Jenkins 잡·범프/롤백 스크립트는 `prj-devops`가 소유한다.
- ArgoCD는 `prj-devops/main`과 `prj-deploy/main`을 읽는다. 이 저장소의 환경은 브랜치가 아닌 `prod/`, `stg/` 디렉터리로 구분한다.
- prod 8앱: core-api, admin-web, idp-api, idp-web, proposal-web, design-web, live-web, tool-storybook.
- stg 6앱: core-api, admin-web, idp-api, idp-web, dolmen-api, dolmen-web.
- 지원 환경·롤백 스크립트의 앱 제한·연결 문서는 `README.md`를 확인한다. 태그 파일 존재만으로 실제 이미지·파드가 정상이라고 판단하지 않는다.

> 진입점: `~/dev/AGENTS.md`(워크스페이스 지도) · 시크릿 지식: OpenBao `secret/docs/*` (열쇠: `~/dev/onjitda-credentials.md`)

## 규칙

1. **범프(태그 변경)는 Jenkins `gitops-prod-image-bump` 잡 전용** — 일반적으로 직접 커밋 금지.
   잡이 "ci(gitops): bump …" 커밋을 자동 생성·푸시한다.
2. 태그 = 빌드 커밋 SHA 앞 12자. 태그에 해당하는 이미지가 Harbor에 반드시 존재해야
   (없으면 ImagePullBackOff — Synced여도 Healthy 안 됨).
3. 긴급 수동 개입(잡 장애 시 등): 해당 앱 yaml의 tag 값을 유효한 기존 태그로 직접 수정 + 커밋 메시지에 사유 명시.
4. **롤백 = 이전 태그로 revert** — ArgoCD가 자동 반영.
5. 태그 변경 커밋은 ArgoCD 자동 배포로 이어짐 — 실수 여파가 곧 prod/stg 반영이라는 점 유의.
6. 문서 최신화는 허용하며 태그 YAML 변경과 구분한다. 문서 작업에서 환경 간 태그 복사·새 태그 추정·이미지 빌드/범프를 함께 수행하지 않는다.
