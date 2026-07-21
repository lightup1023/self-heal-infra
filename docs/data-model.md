# Data Model

`self-heal-infra` 컴포넌트 간 계약(I/O 스키마)과 저장·설정 데이터 모델 정의. `architecture.md` §5의 초안을 pydantic 수준까지 확장한다. 실제 구현은 pydantic v2 기반, YAML/JSON 왕복 가능한 것을 전제로 한다.

## 1. 원칙

- **Domain 모델**(변하지 않는 도메인 개념)과 **Runtime 모델**(그래프 실행 상태) 분리.
- 크레덴셜·시크릿은 절대 값으로 임베드하지 않고 **opaque ref**로만 참조.
- 시간은 전부 UTC ISO8601 (`datetime` with tz).
- ID 규칙: `<도메인>-<yyyymmdd>-<seq>` (예: `inc-20260721-0001`, `plan-20260721-0007`).
- 스키마 파일은 최상단에 `schema_version` (semver) 필수.
- 어댑터별 자유 필드는 `facts.<adapter>` 하위로 격리 (§3.2 참고).

## 2. Core Domain Models

### 2.1 Incident

Alert Ingress가 외부 페이로드를 정규화한 결과. LangGraph 실행의 진입점.

```python
class Incident:
    id: str                          # inc-yyyymmdd-nnnn
    source: IncidentSource           # alertmanager | zabbix | syslog | webhook
    severity: Severity               # info | warning | critical
    target: TargetRef
    signal: Signal
    received_at: datetime
    labels: dict[str, str]           # 원본 소스 라벨 승계
    raw_payload: dict                # 감사용 원문
    correlation_id: str | None       # 다중 알람 병합 시 링크

class TargetRef:
    hostname: str | None
    ip: str | None
    node_id: str | None              # inventory 매칭 후 채워짐
    labels: dict[str, str]

class Signal:
    type: SignalType                 # ping_timeout | http_5xx | gpu_hang | port_down | ...
    metric: str
    value: float | str
    window: str                      # "2m", "5m"
    threshold: str | None
```

### 2.2 Inventory

YAML로 git 관리. 사이트/노드/링크를 담는다. 노드는 공통 필드(`id`, `credential_ref`, `labels`)를 갖고 `role`에 따라 조건부 필드를 확장.

```yaml
schema_version: "0.1"

sites:
  gpu-lab-1:
    servers:
      - id: node-gpu-03
        hostname: gpu-node-03
        os: linux
        ip: 10.10.20.13
        credential_ref: cred_gpu_ssh
        uplink:
          switch: sw-tor-1
          port: xe-0/0/13
        labels: {env: testbed, role: gpu_worker}

      - id: node-win-01
        hostname: win-relay-01
        os: windows
        ip: 10.10.20.50
        credential_ref: cred_win_winrm
        uplink:
          switch: sw-tor-1
          port: xe-0/0/20

    switches:
      - id: sw-tor-1
        vendor: juniper
        model: EX3400
        ip: 10.10.0.11
        credential_ref: cred_net_ssh
        role: tor

    l4:
      - id: lb-a
        vendor: haproxy
        ip: 10.10.0.21
        credential_ref: cred_lb_ssh

    firewalls:
      - id: fw-a
        vendor: fortigate
        ip: 10.10.0.31
        credential_ref: cred_fw_api
```

### 2.3 Credential Reference

인벤토리에는 **ref만** 저장. 실제 값은 SOPS / Vault / env에서 로드.

```yaml
# credentials/refs.yaml  (metadata only)
schema_version: "0.1"

refs:
  cred_gpu_ssh:
    backend: sops
    path: secrets/ssh/gpu.enc.yaml
    fields: [username, private_key]

  cred_net_ssh:
    backend: sops
    path: secrets/ssh/juniper.enc.yaml
    fields: [username, password]

  cred_fw_api:
    backend: vault
    path: kv/self-heal/fortigate
    fields: [api_token]

  cred_win_winrm:
    backend: sops
    path: secrets/winrm/relay.enc.yaml
    fields: [username, password]
```

## 3. Runtime Models (그래프 실행 상태)

### 3.1 TopologyView

Topology Resolver 출력. Incident 대상이 관통하는 경로 상의 노드·엣지.

```python
class TopologyView:
    incident_ref: str
    nodes: list[NodeRef]
    edges: list[Edge]
    resolved_at: datetime

class NodeRef:
    node_id: str                     # inventory id
    layer: Layer                     # compute | l2 | l3 | l4 | firewall
    credential_ref: str

class Edge:
    src: str                         # node_id
    dst: str
    relation: EdgeRelation           # uplink | pool_member | routes_through | protects
    metadata: dict
```

### 3.2 DiagnosisResult

Parallel Diagnostor가 노드마다 생성. `facts`는 어댑터별로 구조가 다르며, 계층별 표준 스키마는 진단 툴킷 명세(별도 문서)에서 정의.

```python
class DiagnosisResult:
    node_id: str
    adapter: AdapterKind             # ssh | winrm | junos_cli | fortigate_rest | prom
    started_at: datetime
    finished_at: datetime
    commands: list[CommandRun]
    facts: dict                      # adapter-specific 정형 데이터
    errors: list[DiagnosisError]

class CommandRun:
    cmd: str
    stdout: str
    stderr: str
    exit_code: int | None
    duration_ms: int
```

`facts` 예시:
- Linux: `{systemd_failed_units: [...], dmesg_errors: [...], gpu_state: {...}}`
- Juniper: `{interfaces: {port: {status, errors, ...}}, lldp: [...], mac_table: [...]}`
- Windows: `{event_log_errors: [...], services_stopped: [...]}`
- FortiGate: `{policy_hits: [...], sessions: {...}, drop_log: [...]}`

### 3.3 ReasonerVerdict

Root-Cause Reasoner LLM 출력. `suggested_actions`는 반드시 `autonomous_actions.yaml` 등재 ID만 허용.

```python
class ReasonerVerdict:
    verdict_id: str
    root_cause: str                  # 자연어 요약 1~2문장
    affected_layer: Layer
    confidence: float                # [0.0, 1.0]
    suggested_actions: list[SuggestedAction]
    evidence: list[EvidenceRef]      # 근거 fact 참조
    reasoning_summary: str
    generated_at: datetime

class SuggestedAction:
    action_id: str                   # autonomous_actions.yaml ID
    target_node_id: str
    params: dict
    rationale: str
    expected_effect: str             # "포트 up 후 30초 내 ping 회복"
```

### 3.4 ActionPlan / ActionRecord

Safety gate 통과 후 실제 실행 계획 + 실행 로그.

```python
class ActionPlan:
    plan_id: str
    action_id: str
    target_node_id: str
    params: dict
    confidence_at_decision: float
    gate_check: GateCheckResult
    approved: bool
    reason_if_denied: str | None

class GateCheckResult:
    whitelist_ok: bool
    blast_radius_ok: bool
    rate_limit_ok: bool
    confidence_ok: bool
    kill_switch_off: bool
    notes: list[str]

class ActionRecord:
    plan_ref: str
    executed_at: datetime
    finished_at: datetime
    output: str
    exit_status: ExecStatus          # success | fail | timeout
    rollback_used: bool
```

### 3.5 PostCheckResult

조치 후 재검증.

```python
class PostCheckResult:
    action_record_ref: str
    checks: list[CheckOutcome]
    verdict: PostCheckVerdict        # recovered | still_broken | inconclusive
    checked_at: datetime

class CheckOutcome:
    spec_ref: str                    # PostCheckSpec 참조
    observed: str
    passed: bool
```

## 4. LangGraph Shared State

노드 간 흐르는 통합 상태. 재진단 루프는 `iteration` 증가로 표현.

```python
class GraphState:
    # 진입
    incident: Incident

    # 단계별 결과
    topology: TopologyView | None
    diagnoses: dict[str, DiagnosisResult]     # key: node_id
    verdict: ReasonerVerdict | None
    action_plans: list[ActionPlan]
    action_records: list[ActionRecord]
    post_check: PostCheckResult | None

    # 제어
    iteration: int
    max_iterations: int
    terminal_state: TerminalState | None
    # resolved | failed | notify_only | max_retry_exceeded | kill_switched

    # 관측
    trace_id: str                    # OTel / Langfuse
    started_at: datetime
    updated_at: datetime
```

## 5. Policy / Config Models

### 5.1 Autonomous Action (whitelist entry)

`autonomous_actions.yaml`. Reasoner가 제안한 `action_id`는 여기 등재된 것만 실행 가능.

```yaml
schema_version: "0.1"

actions:
  - id: switch.port.no_shut
    description: "Juniper 접근포트 셧다운 해제"
    adapter: junos_cli
    command_template: |
      configure private
      set interfaces {port} enable
      commit
    scope:
      vendor: [juniper]
      port_type: access_only
    rate_limit:
      per_target: 3
      window: 10m
    requires_confidence: 0.8
    reversible: true
    rollback_command_template: |
      configure private
      set interfaces {port} disable
      commit

  - id: l4.pool_member.enable
    adapter: haproxy_socket
    command_template: "enable server {backend}/{server}"
    requires_confidence: 0.7
    reversible: true

  - id: server.service.restart
    adapter: ssh
    command_template: "systemctl restart {unit}"
    scope:
      unit_whitelist: [nginx, httpd, gunicorn]
    requires_confidence: 0.9
    reversible: false
```

### 5.2 PostCheckSpec

알람 유형별 사후검증 정의.

```yaml
schema_version: "0.1"

post_checks:
  - id: ping_recovery
    applies_to: [ping_timeout]
    indicator: prom
    expression: 'probe_success{instance="{ip}"}'
    expected:
      op: "=="
      value: 1
      samples: 3
      interval: 10s
      timeout: 60s

  - id: http_5xx_ratio
    applies_to: [http_5xx]
    indicator: prom
    expression: |
      sum(rate(http_requests_total{status=~"5..",host="{hostname}"}[1m]))
      / sum(rate(http_requests_total{host="{hostname}"}[1m]))
    expected:
      op: "<"
      value: 0.01
      timeout: 120s
```

### 5.3 Global Policy / Kill-switch

```yaml
# config/policy.yaml
schema_version: "0.1"

autonomous:
  enabled: true                      # env override: SELFHEAL_AUTONOMOUS=off
  max_iterations: 3
  default_confidence_threshold: 0.7
  notify_channels: [mattermost_ops]

llm:
  primary: ollama_local
  fallback: claude_api
  fallback_trigger: confidence_below_0.5
```

## 6. 직렬화 & 버전 관리

- 모든 스키마 파일 최상단에 `schema_version` (semver).
- **마이너 변경**: 옵셔널 필드 추가만 허용. 기존 필드 의미 변경 금지.
- **메이저 변경**: `docs/data-model-migration.md`에 마이그레이션 노트 추가.
- Runtime `GraphState`는 JSON 직렬화하여 Langfuse/OTel span에 첨부. 감사 목적으로 90일 보관 (초안).

## 7. 오픈 이슈

- **Incident correlation 정책**: 짧은 시간 내 동일 target 다중 알람을 하나의 incident로 병합할지, `correlation_id` 링크만 두고 개별 처리할지.
- **Inventory drift**: 실시간 LLDP/ARP로 관측된 노드가 인벤토리에 없을 때 자동 등록 vs 무시 vs 알람.
- **크레덴셜 캐싱**: 매 조사마다 SOPS/Vault를 뚫으면 지연이 큼. 프로세스 메모리 캐시 TTL 정책.
- **Facts 스키마 표준화**: 어댑터별 자유 dict을 계속 두면 Reasoner 프롬프트가 어댑터 다양화에 취약. 계층별 표준 스키마를 별도 문서로 확정할지.
- **재진단 루프 종료 조건**: `max_iterations` 외에 "동일 verdict 반복 감지" 같은 추가 종료 조건 필요.
- **크레덴셜 로테이션 이벤트**: rotation 발생 시 in-flight 그래프 실행의 credential ref 무효화 처리.

## 8. 다음 단계

1. 본 문서 스키마를 pydantic 모델로 구현 (`src/self_heal_infra/models/`)
2. 인벤토리 샘플 YAML 작성 (testbed 4~5대 매핑)
3. Credential ref + SOPS 파일 초안 (개발용 dummy)
4. 어댑터별 `facts` 표준 스키마 상세화 → **진단 툴킷 명세** 문서로 연결 (Option B)
