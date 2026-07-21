# Autonomous Actions v1

자율 실행 가능한 조치 카탈로그(`autonomous_actions.yaml`)와 Safety Gate 상세 정책 정의. Reasoner가 제안한 `action_id`는 반드시 이 문서·YAML에 등재된 것만 실행 대상이 된다. 스키마는 `data-model.md` §5.1.

## 1. 원칙

- **Safety first, coverage second**: v1은 커버리지보다 안전 여백을 우선한다. 4~5개 검증된 action만.
- **Reversibility 우선**: 되돌릴 수 있는 조치만 낮은 confidence로 허용. 비가역 조치는 0.85+.
- **Idempotency**: 동일 조치의 반복 실행이 상태를 더 악화시키지 말아야 함 (또는 rate-limit으로 반복 억제).
- **Explicit scope**: 대상 자체를 문법(규정 없는 wildcard)으로 넓히지 않는다. 인벤토리 상 정확 매칭 or 명시된 whitelist.
- **Whitelist ⊆ Testbed**: v1의 모든 action은 testbed chaos 시나리오 CX-01 ~ CX-06 중 최소 하나와 매핑되어야 한다.
- **Confidence 임계는 action별 override**: 전역 default 위에 action이 더 높은 값을 요구할 수 있음(더 낮출 수는 없음).

## 2. Action ID 명명 규칙

`<layer>.<vendor|technology>.<object>.<verb>` (dot notation).

예:
- `switch.junos.port.no_shut`
- `server.linux.service.restart`
- `server.win.service.restart`
- `lb.haproxy.server.enable`

향후 vendor 확장이 자유로우려면 verb는 조치가 최종적으로 만드는 상태를 표현 (`enable`, `restart`, `no_shut`).

## 3. Whitelist v1

### 3.1 `switch.junos.port.no_shut`

**목적**: Juniper 접근포트가 `admin=up/oper=down` 또는 `admin=down`인데 근본 원인 판정이 config-driven port down인 경우 enable.

```yaml
- id: switch.junos.port.no_shut
  description: "Juniper 접근포트 셧다운 해제"
  adapter: junos_cli
  command_template: |
    configure private
    set interfaces {port} enable
    commit and-quit
  params_schema:
    port: {type: string, pattern: "^(ge|xe|et)-\\d+/\\d+/\\d+$"}
  scope:
    vendor: [juniper]
    port_type: [access]           # inventory에서 uplink/trunk 표기된 포트 차단
  rate_limit:
    per_target: 3
    window: 10m
  requires_confidence: 0.8
  reversible: true
  rollback_command_template: |
    configure private
    set interfaces {port} disable
    commit and-quit
  post_check_ref: ping_recovery
  maps_to_chaos: [CX-03]
```

### 3.2 `server.linux.service.restart`

**목적**: 특정 systemd unit이 `failed` 상태이거나 프로세스 이슈로 서비스 응답 실패일 때 재기동.

```yaml
- id: server.linux.service.restart
  description: "Linux systemd unit 재기동"
  adapter: linux_ssh
  command_template: |
    systemctl restart {unit} && sleep 2 && systemctl is-active {unit}
  params_schema:
    unit: {type: string, enum: [nginx, apache2, httpd, gunicorn, docker, dcgm-exporter, node_exporter]}
  scope:
    os: [linux]
  rate_limit:
    per_target: 2
    window: 15m
  requires_confidence: 0.9
  reversible: false               # 재기동은 되돌릴 수 없음
  post_check_ref: http_5xx_ratio  # 웹 서비스 대상 시. unit별 매핑은 §6 참고
  maps_to_chaos: [CX-02]
```

### 3.3 `server.win.service.restart`

**목적**: Windows 서비스가 `Stopped` 상태(StartType=Automatic)일 때 재기동.

```yaml
- id: server.win.service.restart
  description: "Windows 서비스 재기동"
  adapter: winrm
  command_template: |
    Restart-Service -Name '{service_name}' -Force
    Start-Sleep -Seconds 2
    (Get-Service -Name '{service_name}').Status
  params_schema:
    service_name: {type: string, enum: [W3SVC, IISADMIN, MSMQ]}
  scope:
    os: [windows]
  rate_limit:
    per_target: 2
    window: 15m
  requires_confidence: 0.9
  reversible: false
  post_check_ref: http_5xx_ratio  # IIS(W3SVC) 대상
  maps_to_chaos: [CX-02]
  notes: |
    WinRM 자체가 다운되면(CX-05) 이 action은 실행 불가. 
    CX-05는 v1에서 notify-only로 종결. 향후 fallback 채널 필요.
```

### 3.4 `lb.haproxy.server.enable`

**목적**: HAProxy 백엔드 서버가 관리적으로 disabled 상태일 때 다시 enable. 헬스체크 이후 자동 UP으로 승격되도록.

```yaml
- id: lb.haproxy.server.enable
  description: "HAProxy 백엔드 서버 enable"
  adapter: haproxy_socket
  command_template: "enable server {backend}/{server}"
  params_schema:
    backend: {type: string, pattern: "^[a-zA-Z0-9_-]+$"}
    server:  {type: string, pattern: "^[a-zA-Z0-9_-]+$"}
  scope:
    vendor: [haproxy]
  rate_limit:
    per_target: 5
    window: 10m
  requires_confidence: 0.7        # reversible + 저위험이라 낮게
  reversible: true
  rollback_command_template: "disable server {backend}/{server}"
  post_check_ref: pool_member_up
  maps_to_chaos: [CX-06]
```

### 3.5 (참고) v1에서 **제외된** 후보

명시적으로 v1에서 뺀 항목. Phase 2+ 재검토.

| 후보 | 제외 이유 |
|---|---|
| `server.linux.gpu.reset` (nvidia-smi -r) | 런중 워크로드 중단. confidence 0.95+ 필요, chaos 시나리오와 안전 검증 후 편입 |
| `network.junos.mac_table.clear` | 전면 clear 시 짧은 flood. blast-radius 크고 원인 판정 오지 않는 대상 |
| `firewall.fortigate.session.clear` | 정상 트래픽 중단 위험 |
| `server.linux.iptables.rule.remove` | 규칙 조작은 대부분 근본 원인이 아니라 증상 유발자 |
| `switch.junos.config.rollback` | commit rollback은 blast-radius 매우 큼. HITL 필수 판단 |

## 4. Safety Gate 상세 정책

Action Executor가 Reasoner의 `SuggestedAction`을 실행 전 다음 순서로 검증. 하나라도 실패하면 실행 skip, `ActionPlan.approved=false` 기록.

### 4.1 Whitelist check
- `action_id`가 `autonomous_actions.yaml`에 등재됐는가.
- 존재하지 않으면 즉시 reject + warning 로그.

### 4.2 Params schema
- `params`가 action의 `params_schema` (JSON Schema) 통과.
- 특히 문자열 파라미터의 정규식 매칭 강제 (인젝션 방지). 예: 포트명은 `^(ge|xe|et)-\d+/\d+/\d+$`만 허용.

### 4.3 Blast-radius
- `target_node_id` → inventory 매핑. 해당 노드의 필드가 action의 `scope`와 부합하는가.
- 예: `switch.junos.port.no_shut`은 `port_type=access`인 포트만. Uplink/trunk 표기된 포트는 reject.
- 예: `server.linux.service.restart`은 `os=linux`인 노드만.
- Wildcard 대상 금지. 정확 매칭만.

### 4.4 Rate limit
- 최근 `window` 시간 내 동일 (`action_id`, `target_node_id`) 조합 실행 횟수가 `per_target` 미만이어야 함.
- **전역 rate limit**: policy.yaml `autonomous.global_rate_limit`(예: 20/hour) 초과 시 전면 봉인.
- 저장소: 초기에는 프로세스 in-memory (재기동 시 리셋). 프로덕션은 Redis.

### 4.5 Confidence threshold
- `Reasoner verdict.confidence ≥ max(policy.default_confidence_threshold, action.requires_confidence)`
- 미달 시 reject. `notify_only` 종결로 전환.

### 4.6 Kill-switch
- 다음 중 하나라도 truthy면 즉시 전면 봉인:
  - 환경변수: `SELFHEAL_AUTONOMOUS=off` (또는 `false`, `0`)
  - `policy.yaml`: `autonomous.enabled: false`
  - 런타임 API 상태: `/admin/killswitch` PUT으로 세팅된 값
- 우선순위: **env > runtime > config**. env가 세팅됐으면 다른 값을 무시.
- Kill-switch 활성 시에도 진단·리포트는 정상 동작 (조치만 skip).

### 4.7 Reversibility 강제
- `reversible: false` action은 `requires_confidence ≥ 0.85` 등재 필수 (본 문서 CI 룰).
- Executor는 이 조건을 실행 전 재확인.

### 4.8 GateCheckResult 기록
모든 결과는 `data-model.md` §3.4의 `GateCheckResult`에 필드별 boolean으로 저장. `notes`에 reject 사유 명시.

## 5. Execution Semantics

### 5.1 실행 순서 (Reasoner가 여러 개 제안 시)
- Reasoner가 `suggested_actions`를 여러 개 반환할 수 있으나 **v1에서는 첫 번째 하나만** 실행.
- 이유: 다중 조치 조합의 부작용을 미리 모델링하지 못함. 재진단 루프에서 순차 진행.

### 5.2 Timeout
- Action별 실행 timeout 기본 30s. 명시 시 최대 120s.
- Timeout 발생 시 `exit_status=timeout`, `rollback_used=false`. Post-check는 정상 진행 (조치가 부분 반영됐을 수 있음).

### 5.3 Output capture
- stdout/stderr 전체 capture 후 `ActionRecord.output`에 저장.
- 민감 정보 마스킹은 어댑터 단에서 (예: `password=...` 패턴).

### 5.4 Rollback 트리거
- Post-check verdict = `still_broken` + action.reversible=true + `policy.auto_rollback: true` → 자동 rollback 실행.
- Rollback도 별도 `ActionRecord`로 기록 (`plan_ref`는 원 plan에 연결, `rollback_used=true`).
- Rollback 실패는 즉시 심각 알람 (사후 수동 개입 필요).

### 5.5 Idempotency
- 각 action은 이미 목표 상태인 경우에도 실패하지 않아야 함 (예: 이미 enabled인 포트에 enable → OK).
- Command template 설계 시 실행 후 상태 확인까지 포함 (§3.2, §3.3 참고: `is-active`, `Get-Service`).

## 6. Post-check 연결

각 action은 `post_check_ref`로 `post_checks.yaml`의 spec을 참조. 실행 완료 직후 Post-check가 트리거.

| Action | post_check_ref | 검증 로직 |
|---|---|---|
| `switch.junos.port.no_shut` | `ping_recovery` | `probe_success{instance={target.ip}} == 1` for 3 samples |
| `server.linux.service.restart` (web) | `http_5xx_ratio` | 5xx 비율 < 1% for 120s |
| `server.win.service.restart` (W3SVC) | `http_5xx_ratio` | 동일 |
| `lb.haproxy.server.enable` | `pool_member_up` | HAProxy stat에서 해당 server `status=UP` for 2 samples |

`pool_member_up`은 `post_checks.yaml`에 추가 필요 (v1과 함께 등재).

## 7. Audit & Logging

각 자율 실행은 다음 필수 항목을 남긴다:

- `ActionRecord` (data-model.md §3.4) 전 필드
- 매칭된 `verdict_id`, `plan_id`, `trace_id`
- `GateCheckResult` 전체
- 원 명령 텍스트 (템플릿 치환 후)
- 실행 컨텍스트: `llm_source`, `prompt_version`, `iteration`
- Kill-switch 상태 스냅샷

저장:
- 단기: 프로세스 로그 (structlog JSON)
- 장기: SQLite (초기) → PostgreSQL (프로덕션). 90일 이상.
- 요약 통지: Reporter를 통해 Mattermost/Slack에 실행 이력 요약 (통지지 승인은 아님).

## 8. 신규 Action 편입 절차

`autonomous_actions.yaml`에 항목을 추가하는 PR은 다음 자료를 반드시 포함:

1. **정당화**: 어떤 장애 패턴에서 발생하고, 왜 자율 실행이 안전한가.
2. **Command 검증**: testbed에서 실제 명령을 실행해 성공한 스크립트/로그.
3. **Rollback 검증**: reversible action은 rollback 실행 결과도 첨부.
4. **Chaos 매핑**: `maps_to_chaos` 필드에 CX-ID. 없으면 새 chaos 시나리오도 함께 제출.
5. **Blast-radius 분석**: 실패 시 최대 피해 범위 (몇 노드? 다운타임 예상?).
6. **Rate limit 근거**: 왜 그 숫자인지 (사고 다중 발생 시나리오 상정).
7. **CODEOWNER 리뷰 필수**: `.github/CODEOWNERS`에 `docs/autonomous-actions.md`와 `autonomous_actions.yaml` 등재 요구.

## 9. 오픈 이슈

- **WinRM 자체 다운(CX-05) 대응**: 현재 v1은 notify-only. 이중 접근 채널(예: 별도 관리 VLAN + SSH agent) 도입 검토.
- **GPU 조치 편입**: `nvidia-smi -r` 또는 `dcgmi diag` 재시작을 언제 자율 화이트리스트에 넣을지. 워크로드 스케줄러와의 인터록 필요.
- **다중 조치 조합**: v1은 첫 하나만 실행. 순차 실행 vs 병렬 실행 모델링 필요 시점.
- **Rate limit 저장소**: 프로세스 재기동 시 카운터 리셋되면 rate 우회. Redis 도입 시점.
- **Rollback 실패 처리**: 자동 rollback이 실패했을 때 두 번째 rollback을 자동 시도할지, 즉시 HITL로 넘길지.
- **Cross-target action**: 예를 들어 "서버 A의 문제 해결을 위해 스위치 X의 포트를 조작"처럼 target이 진단 대상과 다를 때 blast-radius 재정의 필요.
- **Config drift 대응**: `switch.junos.config.rollback` 같은 강한 조치는 안전 판정 불가. `show system commit`으로 회귀만 리포트하고 조치는 skip이 원칙.

## 10. 다음 단계

1. `autonomous_actions.yaml` 실체화 (§3의 4개 항목 그대로)
2. `post_checks.yaml`에 `pool_member_up` spec 추가
3. `src/self_heal_infra/executor/` 스켈레톤 (Gate → Command → Post-check → Rollback)
4. 각 action의 chaos 매핑 결과를 `tests/e2e/`에 시나리오로
5. `.github/CODEOWNERS`에 본 문서·YAML 등재
6. **여기까지 완료되면 설계 단계 종료 → 코드 스켈레톤 착수**
