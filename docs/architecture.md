# Architecture

`self-heal-infra`의 컴포넌트 구조와 데이터 흐름 정의. 세부 스택·모델·어댑터·프롬프트·조치·배포는 별도 문서로 분리.

**연관 문서**
- 기술 스택 + 앱 인터페이스 + 배포: [`tech-stack.md`](tech-stack.md)
- I/O 데이터 모델: [`data-model.md`](data-model.md)
- 어댑터 진단 명령 카탈로그: [`diagnosis-toolkit.md`](diagnosis-toolkit.md)
- LLM 프롬프트 & 평가: [`reasoner-prompt.md`](reasoner-prompt.md)
- 자율 조치 화이트리스트 & Safety Gate: [`autonomous-actions.md`](autonomous-actions.md)
- 테스트 환경 (하드웨어·토폴로지): [`testbed.md`](testbed.md)

## 1. 전체 구조

```
┌────────────────────────────────────────────────────────────────┐
│                     Alert Ingress                              │
│   (Prometheus Alertmanager / Zabbix / Syslog / Webhook)        │
│   (n8n 워크플로우로 정규화 옵션 — tech-stack.md §7.2)          │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────┐
│                     LangGraph Orchestrator                     │
│                                                                │
│  ┌────────────┐   ┌────────────┐   ┌──────────────────────┐    │
│  │ Topology   │──>│ Parallel   │──>│ Root-Cause Reasoner  │    │
│  │ Resolver   │   │ Diagnostor │   │ (LLM)                │    │
│  └────────────┘   └────────────┘   └──────────┬───────────┘    │
│                                               │                │
│                    ┌──────────────────────────┤                │
│                    ▼                          ▼                │
│              ┌───────────┐              ┌───────────┐          │
│              │  Action   │              │ Reporter  │          │
│              │ Executor  │              │ (Notify)  │          │
│              │(Safe-gate)│              └───────────┘          │
│              └─────┬─────┘                                     │
│                    ▼                                           │
│              ┌───────────┐                                     │
│              │ Post-check│                                     │
│              │  & Retry  │                                     │
│              └───────────┘                                     │
│                                                                │
│  (on failure: re-invoke from Diagnostor with new hypothesis)   │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────┐
│                Device Access Layer (Adapters)                  │
│   Read-only 진단 툴킷 — MCP 서버로도 노출                      │
│   SSH  |  WinRM  |  Junos CLI  |  REST(FortiGate)  |  Prom API │
└────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────┐
│   Servers  │  Switches  │  L4  │  Firewalls  │  Observability  │
└────────────────────────────────────────────────────────────────┘
```

## 2. 컴포넌트

### 2.1 Alert Ingress
- **역할**: 외부 모니터링 시스템에서 알람을 받아 그래프의 시작점으로 주입.
- **입력 채널**: Prometheus Alertmanager webhook, Zabbix action, syslog(HTTP relay), 임의 REST webhook.
- **정규화**: 서로 다른 소스의 payload를 공통 `Incident` 스키마로 변환 ([`data-model.md`](data-model.md) §2.1).
- **옵션**: 정규화 계층을 n8n 워크플로우로 대체 가능 ([`tech-stack.md`](tech-stack.md) §7.2).

### 2.2 Topology Resolver
- **역할**: 알람 대상 노드가 소속된 경로(서버 → 상단 SW → L4 → FW) 파악.
- **입력**: 인벤토리(YAML), 실시간 LLDP/CDP, ARP·MAC 테이블.
- **출력**: `TopologyView` — 조사 대상 노드 + credential ref ([`data-model.md`](data-model.md) §3.1).

### 2.3 Parallel Diagnostor
- **역할**: 각 계층에서 정해진 read-only 진단 명령을 병렬 실행하고 raw 결과 수집.
- **구성**: 계층별 sub-graph. 재사용 가능한 "진단 툴킷".
- **원칙**: 이 단계에서는 절대 상태를 변경하지 않는다(show/get만).
- **표준화 옵션**: 각 어댑터를 MCP 서버로 노출 → LLM 외 도구(Claude/Cursor 등)에서도 재활용.
- 상세: [`diagnosis-toolkit.md`](diagnosis-toolkit.md).

### 2.4 Root-Cause Reasoner (LLM)
- **역할**: 수집된 팩트를 LLM에 넣고 근본 원인 판정.
- **프롬프트 설계**: chain-of-cause 구조 (가장 상류 계층부터 원인 후보 열거 → 하위 계층 증거로 반박·지지 → 최종 판정).
- **출력**: `ReasonerVerdict` — `{root_cause, affected_layer, confidence, suggested_actions[]}` ([`data-model.md`](data-model.md) §3.3).
- **최적화**: DSPy 기반 프롬프트 자동 튜닝 ([`tech-stack.md`](tech-stack.md) §7.3).
- 상세: [`reasoner-prompt.md`](reasoner-prompt.md).

### 2.5 Action Executor (Safe-gated)
- **역할**: 자율 실행이 승인된 조치를 수행.
- **안전 가드 8단계**: whitelist → params schema → blast-radius → rate limit → confidence 임계 → kill-switch → reversibility → audit.
- **Kill-switch**: 환경변수 `SELFHEAL_AUTONOMOUS=off` 하나로 전체 자율 조치 off.
- 상세: [`autonomous-actions.md`](autonomous-actions.md).

### 2.6 Post-check & Retry
- **역할**: 조치 직후 원래 알람의 지표가 정상화됐는지 재검증.
- **실패 시**: 다른 원인 가설로 그래프 재invoke (읽기전용 재점검부터). 이전 verdict을 prior_iterations로 프롬프트에 주입 → LLM이 같은 가설 반복 억제.
- **상한**: `policy.yaml` 의 `max_iterations`.

### 2.7 Reporter
- **역할**: 판정·조치·검증 결과를 채널로 통지.
- **HITL 없음**: 통지는 사후 보고 목적이지 승인 요청이 아니다.
- **채널 확장**: n8n webhook 경유로 Mattermost/Slack/JIRA/Email fan-out ([`tech-stack.md`](tech-stack.md) §7.2).

## 3. 데이터 흐름

```
Alert
  → Incident              (Alert Ingress, 정규화)
  → TopologyView          (Topology Resolver)
  → DiagnosisResult×N     (Parallel Diagnostor, 노드별)
  → ReasonerVerdict       (Root-Cause Reasoner LLM)
  → ActionPlan → GateCheckResult
  → ActionRecord          (Action Executor)
  → PostCheckResult       (Post-check)
  → (실패 시 Diagnostor로 재invoke; iteration++)
  → 통지 (Reporter)
```

모든 스키마 정의는 [`data-model.md`](data-model.md).

## 4. 외부 서비스 의존

| 서비스 | 필수/선택 | 비고 |
|---|---|---|
| Prometheus | 필수 | 알람 소스 & 사후검증 지표 |
| Alertmanager | 필수 | Ingress webhook |
| Ollama 또는 LLM endpoint | 필수 | 없으면 Reasoner 미동작 |
| Langfuse | 선택 | LLM trace 관측 |
| n8n | 선택 | Alert Ingress / 통지 프론트 |
| Mattermost/Slack | 선택 | 통지 채널 |
| Loki/ES | 선택 | 로그 상관분석 강화 |
| Vault | 선택 | 시크릿 (없으면 SOPS) |

## 5. 오픈 이슈

- **자율 조치의 안전 경계**: "화이트리스트만"으로 커버 못하는 케이스(조합 조치)에서 확장 방법.
- **크레덴셜 회전**: 다수 장비 접근 시 키 관리 부담.
- **다중 알람 상관**: 동시 다발 알람을 하나의 인시던트로 병합하는 정책.
- **LLM 비결정성**: 동일 증거에서 다른 판정. self-consistency 샘플링 or rules-first fallback ([`reasoner-prompt.md`](reasoner-prompt.md) §7).
- **네트워크 벤더 다양성**: 벤더 CLI 편차를 adapter 계층에서 얼마나 흡수할지.
- **MCP 서버 실행 방식**: 별도 프로세스 vs 앱 내부 라이브러리 ([`tech-stack.md`](tech-stack.md) §13).

## 6. 로드맵 (사이드 프로젝트 스코프)

- **Phase 0 (현재)**: 설계 문서화 (완료)
- **Phase 1**: 코드 스켈레톤 — 모델(pydantic) → 어댑터 3종(Linux/Juniper/WinRM) → Reasoner + Executor
- **Phase 2**: LangGraph orchestrator 조립 + testbed 연결 + chaos 시나리오 CX-01~06
- **Phase 3**: MCP 서버 노출 + DSPy 프롬프트 최적화 + Langfuse trace + Ragas 평가
- **Phase 4**: n8n 통합 (Alert Ingress + 통지 fan-out)
- **Phase 5**: 리팩터·문서 정비 + 데모 영상·블로그 포스트

프로덕션 이관(Helm/systemd/air-gap)은 이 프로젝트 스코프 밖. 필요해지면 별도 트랙.

## 7. 다음 단계

1. `pyproject.toml` (uv 기반) + Ruff 세팅
2. `src/self_heal_infra/models/` — pydantic 모델 ([`data-model.md`](data-model.md))
3. `src/self_heal_infra/adapters/` — Linux/Juniper/WinRM 어댑터 스켈레톤
4. LangGraph 그래프 정의
5. Testbed 인벤토리 샘플 YAML
