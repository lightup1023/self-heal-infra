# Architecture

`self-heal-infra`의 내부 구성, 기술 스택, 외부 의존 서비스를 정리한다. 아직 설계 단계이며, PoC 진행 중 갱신된다.

## 1. 전체 구조

```
┌────────────────────────────────────────────────────────────────┐
│                     Alert Ingress                              │
│   (Prometheus Alertmanager / Zabbix / Syslog / Webhook)        │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────┐
│                     LangGraph Orchestrator                     │
│                                                                │
│  ┌────────────┐   ┌────────────┐   ┌──────────────────────┐    │
│  │ Topology   │──▶│ Parallel   │──▶│ Root-Cause Reasoner  │    │
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
│   SSH  |  SNMP  |  REST(F5/Fortigate/PA)  |  Prometheus API    │
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
- **입력 채널**: Prometheus Alertmanager webhook, Zabbix action, syslog(rsyslog → HTTP relay), 임의 REST webhook.
- **정규화**: 서로 다른 소스의 payload를 공통 `Incident` 스키마로 변환.

### 2.2 Topology Resolver
- **역할**: 알람 대상 노드가 소속된 경로(서버 → 상단 SW → L4 → FW) 파악.
- **입력**: 인벤토리(YAML), 실시간 LLDP/CDP, ARP·MAC 테이블.
- **출력**: 조사 대상 장비 리스트 + 접근 credential reference.

### 2.3 Parallel Diagnostor
- **역할**: 각 계층에서 정해진 read-only 진단 명령을 병렬 실행하고 raw 결과 수집.
- **구성**: 계층별 sub-graph. 각 sub-graph는 재사용 가능한 "진단 툴킷"으로 정의.
- **원칙**: 이 단계에서는 절대 상태를 변경하지 않는다(show/get만).

### 2.4 Root-Cause Reasoner (LLM)
- **역할**: 수집된 raw 결과를 LLM에 넣고 근본 원인 판정.
- **프롬프트 설계**: chain-of-cause 구조 (가장 상류 계층부터 원인 후보 열거 → 하위 계층 증거로 반박·지지 → 최종 판정).
- **출력**: `{root_cause, affected_layer, confidence, suggested_actions[]}`.

### 2.5 Action Executor (Safe-gated)
- **역할**: 자율 실행이 승인된 조치를 수행.
- **안전 가드**:
  - **화이트리스트**: `autonomous_actions.yaml`에 등재된 명령만 자동 실행.
  - **Blast-radius 상한**: 예) "포트 리셋"은 access-port만, uplink/trunk 금지.
  - **레이트 리밋**: 동일 대상 N분 내 M회 초과 조치 금지.
  - **Confidence 임계**: reasoner의 confidence가 threshold 미달이면 조치 스킵 → notify only.
  - **Kill-switch**: 환경변수 하나로 전체 자율 조치 off.

### 2.6 Post-check & Retry
- **역할**: 조치 직후 원래 알람의 지표가 정상화됐는지 재검증.
- **실패 시**: 다른 원인 가설로 그래프 재invoke (읽기전용 재점검부터).
- **상한**: 재시도 횟수 제한.

### 2.7 Reporter
- **역할**: 판정·조치·검증 결과를 채널로 통지 (Mattermost / Slack / Email).
- **HITL 없음**: 통지는 사후 보고 목적이지 승인 요청이 아니다.

## 3. 기술 스택

### 3.1 Core
| 항목 | 선정 | 비고 |
|---|---|---|
| 언어 | Python 3.11+ | asyncio 기반 병렬 진단 |
| 오케스트레이션 | LangGraph | 사내 프로젝트와 동일, 노하우 재사용 |
| LLM 프레임워크 | LangChain (필요한 범위만) | 프롬프트/파서 유틸 |
| 설정 | pydantic-settings + YAML | 인벤토리, 정책, 화이트리스트 |
| 로깅/트레이싱 | structlog + OpenTelemetry | Langfuse 연동 옵션 |

### 3.2 LLM
| 옵션 | 용도 | 비고 |
|---|---|---|
| Ollama (local) | 기본, 온프렘 GPU 활용 | qwen2.5:14b / 32b 계열 검토 |
| Anthropic Claude | 고난도 케이스 fallback | API 키 있는 환경에서만 |
| OpenAI-호환 endpoint | 벤더 중립성 확보 | vLLM, TGI 등도 연결 가능 |

라우팅 정책은 config로 스위칭 (default local → confidence 낮으면 remote).

### 3.3 Device Access
| 대상 | 라이브러리 | 프로토콜 |
|---|---|---|
| Linux 서버 | `asyncssh`, `paramiko` | SSH |
| Cisco/Juniper/Arista | `netmiko`, `scrapli` | SSH CLI |
| SNMP 폴링 | `pysnmp` | SNMP v2c/v3 |
| F5 | `f5-sdk` or REST | iControl REST |
| FortiGate | `fortiosapi` or REST | REST |
| Palo Alto | REST/XML API | HTTPS |
| iptables/nftables | SSH + parser | SSH |

### 3.4 Observability / 데이터 소스
| 소스 | 용도 |
|---|---|
| Prometheus | 메트릭 시계열 조회 (`PromQL` via HTTP API) |
| Loki / Elasticsearch | 로그 조회 (선택) |
| NVIDIA DCGM Exporter | GPU 상태 (테스트베드가 GPU 인프라이므로 필수) |
| node_exporter | 서버 기본 리소스 |
| snmp_exporter | 네트워크 장비 메트릭 (옵션) |

### 3.5 지식 베이스 (RAG)
| 항목 | 선정 | 비고 |
|---|---|---|
| Vector DB | Chroma (기본) / Qdrant (스케일업 시) | 장비 매뉴얼, 런북, 과거 인시던트 |
| 임베딩 | BGE-M3 / e5-mistral | 한국어 혼용 문서 지원 |
| 문서 소스 | 벤더 CLI reference, 사내 런북 md, 과거 티켓 |

토폴로지는 Neo4j까지 도입하지 않고 초기에는 YAML + in-memory graph(`networkx`)로 시작.

### 3.6 시크릿 관리
| 옵션 | 비고 |
|---|---|
| SOPS + age | 초기 default (git-friendly) |
| HashiCorp Vault | 프로덕션급 환경에서 옵션 |
| 로컬 개발 | `.env` (git ignore) |

### 3.7 패키징 / 배포
| 형태 | 대상 | 상태 |
|---|---|---|
| Docker image | 표준 배포 단위 | Phase 1 필수 |
| docker-compose | 단일 노드 설치 | Phase 1 필수 |
| Helm chart | K8s 환경 | Phase 4 |
| systemd unit | bare-metal 설치 | Phase 4 |
| pip / uv | 개발/CLI 사용 | 상시 |

### 3.8 테스트
| 종류 | 도구 |
|---|---|
| 유닛 | `pytest`, `pytest-asyncio` |
| 장비 시뮬레이션 | `containerlab` (Cisco/Juniper/Arista/SR Linux 컨테이너 이미지) |
| Chaos 주입 | `chaos-mesh` 또는 자체 스크립트 (포트 셧, 링크 다운, LB drain) |
| E2E | 실제 GPU 테스트베드 대상 시나리오 스위트 |

## 4. 외부 서비스 의존

| 서비스 | 필수/선택 | 비고 |
|---|---|---|
| Prometheus | 필수 | 알람 소스 & 사후검증 지표 |
| Alertmanager | 필수 | Ingress webhook |
| Ollama 또는 LLM 엔드포인트 | 필수 | 없으면 reasoner 미동작 |
| Mattermost/Slack | 선택 | 통지 채널 |
| Loki/ES | 선택 | 로그 상관분석 강화 |
| Vault | 선택 | 시크릿 (없으면 SOPS) |
| Langfuse | 선택 | LLM trace 관측 |

## 5. 데이터 모델 (초안)

```yaml
# incident (내부 정규화)
id: inc-20260720-0001
source: alertmanager
severity: critical
target:
  hostname: gpu-node-03
  ip: 10.10.20.13
signal:
  type: ping_timeout          # ping_timeout | http_5xx | gpu_hang | ...
  metric: probe_success
  value: 0
  window: 2m
received_at: 2026-07-20T09:41:12Z
```

```yaml
# inventory (YAML, git-managed)
sites:
  gpu-lab-1:
    servers:
      - name: gpu-node-03
        ip: 10.10.20.13
        access: {type: ssh, cred: gpu_ssh}
        uplink: sw-tor-1/xe-0/0/13
    switches:
      - name: sw-tor-1
        vendor: juniper
        ip: 10.10.0.11
        access: {type: ssh, cred: net_ssh}
    l4:
      - name: f5-a
        ip: 10.10.0.21
        access: {type: rest, cred: f5_api}
    firewalls:
      - name: fw-a
        vendor: fortigate
        ip: 10.10.0.31
        access: {type: rest, cred: fw_api}
```

```yaml
# autonomous_actions.yaml (자율 실행 화이트리스트)
- id: switch.port.no_shut
  vendor: [juniper, cisco, arista]
  scope: access_port_only
  rate_limit: {per_target: 3, window: 10m}
  requires_confidence: 0.8

- id: l4.pool_member.enable
  vendor: [f5]
  requires_confidence: 0.7

- id: server.service.restart
  scope: unit_whitelist   # 별도 unit 목록으로 재한정
  requires_confidence: 0.9
```

## 6. 오픈 이슈

- **자율 조치의 안전 경계**: "화이트리스트만"으로 커버 못하는 케이스(예: 조합 조치)에서 어떻게 확장할지.
- **크레덴셜 회전**: 다수 장비 접근에서 키 관리 운영 부담.
- **다중 알람 상관**: 동시에 여러 알람이 들어올 때 하나의 인시던트로 병합하는 정책.
- **LLM 비결정성**: 동일 증거에서 서로 다른 판정이 나오는 문제. self-consistency 샘플링 또는 rules-first fallback 검토.
- **네트워크 벤더 다양성**: 벤더 CLI 편차를 adapter 계층에서 얼마나 흡수할지.

## 7. 다음 단계

1. GPU 테스트베드 인벤토리 확정
2. Alert Ingress + Topology Resolver + Diagnostor(서버 계층만) 최소 스켈레톤
3. Reasoner LLM 프롬프트 v0
4. Post-check 루프
5. 스위치/L4/방화벽 adapter 순차 추가
