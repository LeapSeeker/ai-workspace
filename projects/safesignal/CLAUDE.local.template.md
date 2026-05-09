# SafeSignal — CLAUDE.local.md (개인 환경 전용, .gitignore 등록)

이 파일은 팀 공용 `CLAUDE.md`와 함께 Claude Code가 자동 로드한다.
git에 추적되지 않으므로 본인 작업 흐름·도구 연결 정보만 둔다.

---

## ai-workspace 협업 프로토콜

협업 상태 파일은 별도 레포에서 관리:

- 로컬 경로: `C:\Project\LastProject\ai-workspace\`
- 원격: https://github.com/LeapSeeker/ai-workspace
- 이 환경의 역할: **Claude Code** (STATE.md 직접 커밋 권한 있음)

### 세션 시작 시 (필수)

1. `git -C C:/Project/LastProject/ai-workspace pull --ff-only` 로 최신화
2. `projects/safesignal/STATE.md` 읽고 Decisions Log / Implementation Status / Pending Items 파악
3. 이미 confirmed 상태인 D-XXX 결정사항은 그대로 따른다 (재논의 금지)
4. 필요 시 `common/PROTOCOL.md` 의 Claude Code 섹션 재확인

### 세션 중

- 결정·구현 진행이 생기면 STATE.md 갱신을 미루지 말고 즉시 반영
- 사용자가 `--- STATE.md UPDATE BLOCK ---` 형식 텍스트를 붙여 넣으면 지정 섹션 위치에 그대로 반영

### 세션 종료 시 (필수)

1. STATE.md `Implementation Status` 표 갱신 — 이번 세션에 건드린 모듈의 status / branch / last updated
2. 헤더 `_Last updated: YYYY-MM-DD | Updated by: claude-code_` 갱신
3. ai-workspace 에 커밋·푸시
4. wifi-csi-fall-detection 의 코드 커밋과 ai-workspace 의 STATE 커밋은 **분리**

### 커밋 메시지 규칙 (ai-workspace 전용)

- `state: add decision - {제목}`
- `state: update implementation status - {모듈}`
- `state: add review notes - {모듈}`
- `docs: ...` (메타 정보 정정 등)

---

## 새 머신에서 복원하는 방법

이 파일 자체는 추적되지 않으므로 새 환경에서는 ai-workspace 에 보관된 템플릿으로 복원한다:

```powershell
git clone https://github.com/LeapSeeker/ai-workspace.git C:\Project\LastProject\ai-workspace
Copy-Item C:\Project\LastProject\ai-workspace\projects\safesignal\CLAUDE.local.template.md `
          C:\Project\LastProject\wifi-csi-fall-detection\CLAUDE.local.md
```

이후 `git -C C:/Project/LastProject/ai-workspace pull` 로 STATE 동기화 후 작업 시작.
