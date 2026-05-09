# 세션 프로토콜

## 역할 분담

| AI | 역할 | GitHub 쓰기 권한 |
|----|------|-----------------|
| Claude AI | 시스템 설계, 개념/이론, 의사결정 | 없음 |
| Claude Code | 코드 작성 및 구현 | 있음 |
| Codex | 코드 검토, 평가, 개선 방안 제시 | 있음 |

## 작업 순서

```
Claude AI (설계/결정) → Claude Code (구현) → Codex (검토/개선)
```

---

## Claude AI 세션 프로토콜

### 세션 시작
1. `projects/{project}/STATE.md` 읽기
2. Decisions Log 최신 항목 확인
3. Pending Items 확인
4. 설계/이론 작업 진행

### 세션 종료
- STATE.md에 직접 쓰기 불가 → 아래 형식으로 업데이트 블록 출력
- 사용자가 직접 커밋하거나 다음 Claude Code 세션에 위임

```
--- STATE.md UPDATE BLOCK ---
섹션: Decisions Log
위치: 마지막 Decision 블록 아래에 추가

### [D-00X] {결정 제목}
- **Date:** YYYY-MM-DD
- **Decided by:** claude-ai
- **Content:** ...
- **Ref:** ...
- **Status:** confirmed
--- END UPDATE BLOCK ---
```

---

## Claude Code 세션 프로토콜

### 세션 시작
1. `projects/{project}/STATE.md` 읽기
2. Implementation Status에서 현재 작업 대상 확인
3. Decisions Log 최신 항목 확인 후 구현 시작

### 세션 종료
1. STATE.md Implementation Status 업데이트 (직접 커밋)
2. Claude AI가 남긴 UPDATE BLOCK이 있으면 함께 반영
3. 커밋 메시지 형식: `state: update implementation status - {모듈명}`

---

## Codex 세션 프로토콜

### 세션 시작
1. `projects/{project}/STATE.md` 읽기
2. Implementation Status에서 `done` 항목 확인
3. 검토 대상 코드 확인

### 세션 종료
1. STATE.md Review Notes 추가 (직접 커밋)
2. 개선 필요 사항은 Pending Items에 추가
3. 커밋 메시지 형식: `state: add review notes - {모듈명}`

---

## STATE.md 업데이트 원칙

- 결정사항 하나가 확정될 때마다 즉시 업데이트
- 각 AI는 자신의 담당 섹션만 수정
- 기존 내용 수정 최소화, 원칙적으로 append only
