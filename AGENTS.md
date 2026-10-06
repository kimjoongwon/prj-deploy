# prj-deploy — AI 작업 규칙

이미지 태그 상태 저장소. ArgoCD가 각 앱 Application에서 `prod|stg/<앱>.yaml`의 `tag:`를 읽는다 (multi-source).

> 진입점: `~/dev/AGENTS.md`(워크스페이스 지도) · 시크릿 지식: OpenBao `secret/docs/*` (열쇠: `~/dev/onjitda-credentials.md`)

## 규칙

1. **범프(태그 변경)는 Jenkins `gitops-prod-image-bump` 잡 전용** — 일반적으로 직접 커밋 금지.
   잡이 "ci(gitops): bump …" 커밋을 자동 생성·푸시한다.
2. 태그 = 빌드 커밋 SHA 앞 12자. 태그에 해당하는 이미지가 Harbor에 반드시 존재해야
   (없으면 ImagePullBackOff — Synced여도 Healthy 안 됨).
3. 긴급 수동 개입(잡 장애 시 등): 해당 앱 yaml의 tag 값을 유효한 기존 태그로 직접 수정 + 커밋 메시지에 사유 명시.
4. **롤백 = 이전 태그로 revert** — ArgoCD가 자동 반영.
5. 이 저장소 커밋은 즉시 배포로 이어짐 — 실수 여파가 곧 prod/stg 반영이라는 점 유의.
