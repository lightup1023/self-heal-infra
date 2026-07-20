# self-heal-infra

인프라 장애가 발생했을 때 사람이 개입하지 않고 서버·네트워크 장비·보안 장비까지 **홀리스틱하게 진단하고 자율적으로 복구**를 시도하는 에이전트 프로젝트.

## 배경

사내에서 운영 중인 `ChatOps 고도화` 프로젝트는 LangGraph 기반의 진단·조치 에이전트를 Mattermost에 결합해, **사람(운영자)이 카드 UI로 승인·실행**하는 HITL(Human-in-the-Loop) 구조로 동작한다. 안정성·감사 요건 때문에 HITL은 필수지만, 다음과 같은 한계가 있다.

- 새벽·주말 등 응답 지연 상황에서 MTTR(평균 복구 시간)이 사람 반응속도에 묶임
- 단일 서버 문제로 보이지만 실은 상단 스위치·L4·방화벽에서 원인이 발생한 케이스에서, 사람이 여러 장비를 hop-by-hop 조사하는 시간이 그대로 손실
- 반복적으로 나타나는 정형 장애(포트 다운, LB 헬스체크 실패, 세션 테이블 포화 등)조차 매번 사람 승인 필요

`self-heal-infra`는 이 한계를 넘기 위해 **HITL을 제거한 완전 자율 에이전트**를 별도 트랙으로 실험하는 사이드 프로젝트다. 사내 프로덕션과는 격리된 GPU 인프라를 테스트베드로 사용한다.

## 컨셉

한 서버에 장애 신호가 들어오면, 그 서버에만 접속해서 끝나지 않는다. 트래픽 경로상의 **모든 관련 장비**를 스스로 파악하고 병렬 조사한 뒤, 원인을 판별하고 조치까지 수행한다.

```
[알람: server-A ping timeout]
        │
        ▼
[Topology Resolver]  ──▶  server-A 를 지나는 경로 파악
        │                 (Firewall → L4 → Uplink SW → server-A)
        ▼
[Parallel Diagnosis]
   ├─ server-A     : SSH, syslog, dmesg, systemd, GPU 상태
   ├─ Uplink SW    : 포트 상태, CRC, LLDP, MAC 테이블
   ├─ L4           : VIP 헬스체크, 세션 테이블, 서버풀 상태
   └─ Firewall     : 정책 히트, 세션, drop 로그
        │
        ▼
[Root Cause Reasoner]  ──▶  가장 상류의 근본 원인 판정
        │
        ▼
[Autonomous Action]   ──▶  안전 가드 통과 시 자동 실행
        │                 (포트 리셋 / 서버풀 재활성화 / 세션 flush 등)
        ▼
[Post-check & Report] ──▶  결과 검증, 실패 시 재진단 루프
```

핵심 원칙:

1. **홀리스틱 진단** — 단일 장비가 아니라 경로 전체를 본다.
2. **완전 자율** — 사람 승인 없이 판정과 조치를 수행한다.
3. **안전 불변식** — 자율이라도 파괴적 조치(재부팅·config 대량 변경 등)는 policy로 제한. 자율 실행 가능 조치 목록은 화이트리스트.
4. **자기검증 루프** — 조치 후 반드시 재점검하고, 실패 시 다른 가설로 재진단.
5. **설치형 앱** — 특정 환경에 종속되지 않도록 컨테이너/패키지 형태로 배포.

## 진단 대상 범위 (Phase 1)

| 계층 | 대상 예시 | 접근 방식 |
|---|---|---|
| Compute | Linux 서버 (GPU 워크로드 포함) | SSH, Prometheus, GPU 메트릭 |
| Uplink Switch | Juniper / Cisco / Arista | SSH (netmiko), SNMP, LLDP |
| L4 | F5 / HAProxy / Envoy | REST API, SSH |
| Firewall | FortiGate / Palo Alto / iptables | REST API, syslog |

장비가 존재하지 않으면(단독 서버 등) 해당 계층은 자동으로 스킵한다.

## 테스트 환경

- 사내 프로덕션과 **격리된 GPU 인프라** (별도 구축 중)
- 초기에는 GPU 노드 + 상단 스위치만으로 시작 → 이후 L4·방화벽 순차 편입
- 장애 시나리오는 chaos 스크립트로 주입 (포트 셧, 정책 오설정, GPU hang 등)

## 배포 형태

가능한 한 **설치 가능한 앱** 형태로 제공하는 것을 목표로 한다.

- `docker compose up` 한 방으로 기동
- 인벤토리(YAML) + 시크릿(Vault or SOPS)만 주입하면 즉시 사용
- 향후 Helm chart / systemd bundle 로 확장

자세한 컴포넌트·기술 스택은 [`docs/architecture.md`](docs/architecture.md) 참조.

## 상태

- **Phase 0 (현재)**: 설계 문서화, 테스트베드 구축 대기
- Phase 1: 단일 서버 + 스위치 진단 PoC
- Phase 2: L4 / 방화벽 편입, 자율 조치 화이트리스트 v1
- Phase 3: 조치-재점검 루프, 실패 시 자기수정
- Phase 4: 설치형 앱 배포 (Docker → Helm)

## 라이선스

TBD (사내 자산 여부 확인 후 결정)

## 관련 프로젝트

- 사내 `ChatOps 고도화` (HITL 버전, private) — 본 프로젝트의 원류
