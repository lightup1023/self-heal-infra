# Contributing

`self-heal-infra`는 사이드 프로젝트지만 자율 조치를 다루는 만큼 최소한의 규율은 유지한다.

## 브랜치
- `feature/<간단설명>` — 새 기능
- `fix/<간단설명>` — 버그 수정
- `docs/<간단설명>` — 문서만
- `refactor/<간단설명>` — 동작 유지 개선

`main`은 직접 push 금지 (branch protection). 모든 변경은 PR 경유.

## 커밋 메시지

```
<type>: <짧은 요약, 명령형>

<필요시 본문: 왜 이 변경이 필요한지>
```

`type`: `docs`, `feat`, `fix`, `refactor`, `chore`, `test`

예:
```
feat: add SNMP adapter for Juniper switches
docs: expand safety guardrails section
```

## PR 절차
1. 이슈에서 스코프 합의 (설계 제안이 필요한 규모면 먼저 issue)
2. 브랜치 만들어 작업
3. Draft PR → 스스로 리뷰 → Ready for review
4. 1명 이상 승인 → **Squash and merge** (권장)
5. Merge 후 브랜치 삭제

## 이슈 종류
- **설계 제안 (Design Proposal)**: 구현 전 합의가 필요한 안건
- **작업 (Task)**: 이미 합의된 실행 단위
- **버그 (Bug)**: 예상과 다른 동작

빈 이슈는 비활성화되어 있으니 위 세 템플릿 중 선택.

## Milestone
로드맵 Phase 0~4와 매핑되어 있음. 이슈·PR에 반드시 하나 붙일 것.

## 안전 원칙 (읽기 필수)
자율 조치를 다루는 프로젝트이므로 다음은 절대적:
- 새 조치는 화이트리스트(`autonomous_actions.yaml`) 등재 없이 자율 실행 금지
- Blast-radius가 큰 조치는 rate-limit 및 confidence 임계 필수
- Kill-switch(env)로 전체 자율 조치 off 가능해야 함

자세한 내용은 [`docs/architecture.md`](docs/architecture.md) §2.5 참조.
