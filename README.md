# ai-workspace

개인 AI 협업 워크스페이스. Claude AI, Claude Code, Codex 간 순차 작업 시 공유 상태를 관리한다.

## 구조

```
ai-workspace/
├── _common/                  # 모든 프로젝트 공통 프로토콜
│   ├── PROTOCOL.md           # 세션 시작/종료 프로토콜
│   ├── PROMPT_TEMPLATES.md   # AI별 세션 시작 프롬프트 템플릿
│   └── CONVENTIONS.md        # 공통 규칙 및 포맷
└── projects/
    └── {project-name}/
        └── STATE.md          # 프로젝트별 상태 파일
```

## 세션 시작 전 필수 확인

1. `_common/PROTOCOL.md` — 현재 역할에 맞는 프로토콜 확인
2. `projects/{project}/STATE.md` — 최신 결정사항 및 구현 현황 확인

## 새 프로젝트 추가 시

`projects/` 아래 프로젝트 폴더를 만들고 `STATE.md`를 생성한다.
`_common/PROMPT_TEMPLATES.md`의 템플릿을 복사해서 프로젝트에 맞게 수정한다.
