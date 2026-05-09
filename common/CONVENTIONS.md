# 공통 규칙

## Decision ID 형식

```
D-{세자리 숫자}
예: D-001, D-002, D-015
```

프로젝트 내에서 순차 증가. 프로젝트별로 독립적으로 카운트한다.

## Review ID 형식

```
R-{세자리 숫자}
예: R-001, R-002
```

## Status 값

| 값 | 의미 |
|----|------|
| `confirmed` | 확정 완료 |
| `in-progress` | 작업 중 |
| `done` | 구현 완료 |
| `pending` | 검토/결정 대기 |
| `resolved` | 검토 후 해결 완료 |
| `out-of-scope` | 범위 외로 제외 |

## 커밋 메시지 형식

```
state: {동작} - {대상}

예:
state: add decision - sliding window size
state: update implementation status - preprocessing pipeline
state: add review notes - model.py
```

## 날짜 형식

`YYYY-MM-DD` 통일

## Pending Items 체크박스

```markdown
- [ ] 미완료 항목
- [x] 완료된 항목
```
