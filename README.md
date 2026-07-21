# self-heal-infra

인프라 장애가 발생했을 때 사람이 개입하지 않고 서버·네트워크·보안 장비까지 **홀리스틱하게 진단하고 자율적으로 복구**를 시도하는 에이전트 프로젝트.

> 개인 역량 향상을 위한 **사이드 프로젝트**입니다. 특정 조직 도입이 목적이 아니라 최신 LLM 에이전트 스택(LangGraph, MCP, DSPy, Langfuse 등)을 실제 인프라 자동화 문제 위에 얹어보는 실험 트랙입니다. 자세한 스택·학습 매핑은 [`docs/tech-stack.md`](docs/tech-stack.md) 참조.

## 배경

기존 인프라 자동화 도구는 대체로 **HITL(Human-in-the-Loop)** 구조 — 사람이 승인해야 실행된다. 안정성·감사 요건 때문에 HITL이 필수인 경우가 많지만 다음과 같은 한계가 있다.

- 새벽·주말 등 응답 지연 상황에서 MTTR(평균 복구 시간)이 사람 반응속도에 묶임
- 단일 서버 문제로 보이지만 실은 상단 스위치·L4·방화벽에서 원인이 발생한 케이스에서, 사람이 여러 장비를 hop-by-hop 조사하는 시간이 그대로 손실
- 반복적으로 나타나는 정형 장애(포트 다운, LB 헬스체크 실패, 세션 테이블 포화 등)조차 매번 사람 승인 필요

`self-heal-infra`는 이 한계를 넘기 위해 **HITL을 제거한 완전 자율 에이전트**를 실험한다.

## 컨셉

한 서버에 장애 신호가 들어오면, 그 서버에만 접속해서 끝나지 않는다. 트래픽 경로상의 **모든 관련 장비**를 스스로 파악하고 병렬 조사한 뒤, 원인을 판별하고 조치까지 수행한다.

```
[알람: server-A ping timeout]
        │
        ▼
[Topology Resolver]  ──>  server-A 를 지나는 경로 파악
        │                 (Firewall > L4 > Uplink SW > server-A)
        ▼
[Parallel Diagnosis]
   ├─ server-A     : SSH, syslog, dmesg, systemd, GPU 상태
   ├─ Uplink SW    : 포트 상태, CRC, LLDP, MAC 테이블
   ├─ L4           : VIP 헬스체크, 세션 테이블, 서버풀 상태
   └─ Firewall     : 정책 히트, 세션, drop 로그
        │
        ▼
[Root Cause Reasoner]  ──>  가장 상류의 근본 원인 판정
        │
        ▼
[Autonomous Action]   ──>  안전 가드 통과 시 자동 실행
        │                 (포트 리셋 / 서버풀 재활성화 / 세션 flush 등)
        ▼
[Post-check & Report] ──>  결과 검증, 실패 시 재진단 루프
```

핵심 원칙:

1. **홀리스틱 진단** — 단일 장비가 아니라 경로 전체를 본다.
2. **완전 자율** — 사람 승인 없이 판정과 조치를 수행한다.
3. **안전 불변식** — 자율이라도 파괴적 조치(재부팅·config 대량 변경 등)는 policy로 제한. 자율 실행 가능 조치는 화이트리스트.
4. **자기검증 루프** — 조치 후 반드시 재점검하고, 실패 시 다른 가설로 재진단.

## 진단 대상 범위 (Phase 1)

| 계층 | 대상 예시 | 접근 방식 |
|---|---|---|
| Compute | Linux 서버 (GPU 워크로드 포함) / Windows Server | SSH, WinRM, Prometheus, GPU 메트릭 |
| Uplink Switch | Juniper 계열 (vJunos → 물리 EX) | SSH CLI (netmiko/scrapli), LLDP |
| L4 | HAProxy (F5 옵션) | Unix socket / REST |
| Firewall | FortiGate / SECUI 엑스게이트 | REST / SSH CLI |

장비가 존재하지 않으면(단독 서버 등) 해당 계층은 자동으로 스킵한다.

## 테스트 환경

- 실험용으로 구축한 **격리된 GPU + 네트워크 테스트베드**
- 초기에는 GPU 노드 + containerlab 기반 가상 스위치로 시작 → L4·방화벽 순차 편입
- 장애 시나리오는 chaos 스크립트로 주입 (포트 셧, 정책 오설정, GPU hang 등)
- 상세: [`docs/testbed.md`](docs/testbed.md)

## 실행 형태

- `docker compose up` 한 방으로 기동
- 인벤토리(YAML) + 시크릿(SOPS)만 주입하면 즉시 사용
- 프로덕션 배포(Helm chart, systemd bundle, air-gap 오프라인 번들 등)는 이 프로젝트 스코프 밖

## 학습 목표

이 프로젝트로 실습하려는 최신 스택 (자세한 스토리 매핑: [`docs/tech-stack.md`](docs/tech-stack.md) §11):

- **LangGraph** — 다층 에이전트 그래프
- **MCP (Model Context Protocol)** — 인프라 진단 툴을 LLM-agnostic 표준으로 노출
- **DSPy** — 프롬프트 엔지니어링을 데이터-드리븐 최적화로
- **Langfuse + Ragas** — LLM trace 관측 + golden set 자동 평가
- **n8n** — Alert 정규화·통지 파이프라인의 코드-노코드 하이브리드
- **uv + Ruff + Typer + Rich** — 모던 Python 툴체인

## 상태 & 로드맵

- **Phase 0 (현재)**: 설계 문서화 완료
- Phase 1: 코드 스켈레톤 (models / adapters / reasoner / executor)
- Phase 2: LangGraph orchestrator + testbed 연결 + chaos 시나리오 CX-01~06
- Phase 3: MCP 서버 노출 + DSPy 프롬프트 최적화 + Langfuse trace + Ragas 평가
- Phase 4: n8n 통합 (Alert Ingress + 통지 fan-out)
- Phase 5: 리팩터·문서 정비 + 데모 영상·블로그 포스트

## 문서

| 문서 | 내용 |
|---|---|
| [architecture](docs/architecture.md) | 컴포넌트·데이터 흐름·로드맵 |
| [tech-stack](docs/tech-stack.md) | 기술 스택·앱 인터페이스·학습 매핑 |
| [testbed](docs/testbed.md) | 하드웨어·토폴로지·chaos 시나리오 |
| [data-model](docs/data-model.md) | I/O 스키마 (pydantic) |
| [diagnosis-toolkit](docs/diagnosis-toolkit.md) | 어댑터별 read-only 진단 명령 카탈로그 |
| [reasoner-prompt](docs/reasoner-prompt.md) | LLM 프롬프트 v0·평가 방법론 |
| [autonomous-actions](docs/autonomous-actions.md) | 자율 조치 화이트리스트·Safety Gate |

## 라이선스

TBD (오픈소스 방향으로 검토 중)
