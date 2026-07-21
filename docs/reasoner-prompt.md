# Reasoner Prompt

Root-Cause Reasoner LLM의 프롬프트 v0, 입력 구성, 출력 스키마, confidence 산출 정책 정의. 입력은 `diagnosis-toolkit.md`의 facts, 출력은 `data-model.md` §3.3의 `ReasonerVerdict`.

## 1. 원칙

- **Chain-of-cause 강제**: 최상류(firewall) → 하류(compute) 순으로 원인 후보를 열거하고, 각 계층의 fact로 반박·지지. 임의의 계층에서 정지 가능하되 반드시 근거 인용.
- **Structured Output만 허용**: 자유 텍스트 응답 금지. 반드시 JSON schema에 맞는 응답, JSON 외 프로즈 금지. 파싱 실패는 재샘플링.
- **Fact fabrication 금지**: 입력에 없는 사실을 만들어내면 안 됨. 누락된 fact는 `missing_fact`로 명시하고 confidence 하향.
- **Whitelist 강제**: `suggested_actions[].action_id`는 반드시 `available_actions`(autonomous_actions.yaml) 내에서.
- **Safety gate와 분리**: Reasoner는 제안만. 실행 가능 여부는 Action Executor가 gate로 판정 (`data-model.md` §3.4).
- **재현성**: 동일 입력 → 동일 verdict을 목표로 self-consistency 샘플링 (§7).

## 2. 입력 구성 (Prompt Builder 책임)

Prompt Builder는 `GraphState`에서 다음을 추출·가공하여 프롬프트로 조립.

### 2.1 트리밍 규칙

Raw facts를 그대로 넣으면 컨텍스트 폭발. 다음 규칙으로 축약:

- **인터페이스**: `oper == "down"` 또는 `crc_errors > 0` 또는 `input_errors > 0` 만 포함. 정상 포트 전체 제외.
- **로그**: 최근 10분 이내 + level=error 이상. 라인 상한 50줄. 동일 메시지 반복은 count만 남기고 병합.
- **서비스**: `failed_units`, `stopped_auto`만. 정상 실행 서비스 제외.
- **세션/카운트**: 요약값만 (`total`, `top-N`).
- **GPU**: Xid 발생 or `util_pct > 95` or `temp_c > 85` 인 device만.
- **알람**: 활성만, `severity in [major, critical]`.
- **시간**: ISO8601 유지, 상대시간은 계산해서 함께 삽입 (e.g., `"3분 전"`).

트리밍 후 프롬프트가 여전히 12k 토큰 초과 시 `partial_input=true` 플래그를 붙이고 오래된 로그부터 절삭.

### 2.2 available_actions 추림

Reasoner에게 전체 whitelist를 다 주지 않고, incident target/topology와 관련된 계층의 action만 제공.

## 3. Chain-of-cause 추론 규칙

Reasoner는 다음 순서로 사고해야 함 (프롬프트에 명시):

```
1. Firewall 계층: policy 히트/deny 로그가 incident 시점과 상관? 
   → 있으면 후보 채택, 하위 계층으로 진행하되 upstream 우세
   → 없으면 firewall 후보 기각, 하위로 이동
2. L4 계층: pool member down / VS 헬스체크 실패 / 세션 포화?
   → 있으면 후보 채택. server 계층 fact가 이를 설명하는지 확인.
3. Switch (l2/l3): 대상 uplink 포트 down / CRC / LLDP 인접 loss?
   → 있으면 강력 후보. 하위 server의 network fact와 일치 여부 확인.
4. Compute (server): systemd/journal/dmesg/GPU 이상?
   → 상위 계층이 clean일 때만 최유력 후보로 승격.
5. 없음: 모든 계층 clean인데 알람이 왔다면 관측 지표 자체 문제.
   → affected_layer는 원 알람의 target layer로, confidence 0.4 이하.
```

## 4. System Prompt (v0)

```
You are an autonomous infrastructure incident analyst.

Given a normalized incident and per-node diagnostic facts, identify the most 
likely root cause using chain-of-cause reasoning from upstream (firewall) to 
downstream (compute).

CORE RULES
1. Reason top-down: firewall → l4 → switch → compute. Justify moving down.
2. For each candidate cause, cite EXACTLY which fact supports or refutes it.
   Use the format {"node_id": "...", "fact_path": "interfaces[0].oper", 
   "value": "down", "supports_or_refutes": "supports"}.
3. Never invent facts. If a fact is missing, note it and lower confidence.
4. Only suggest actions from the provided `available_actions` whitelist.
   Never fabricate action_id.
5. Output MUST be valid JSON matching the schema. No prose outside JSON.
6. If any adapter returned `auth_failed` or `unauthorized`, set 
   `suggested_actions` to [] and describe the credential issue in 
   `reasoning_summary`.

CONFIDENCE GUIDANCE
- 0.9+  : multiple independent facts converge on one cause; no missing upstream.
- 0.7-0.9 : clear cause but some upstream layer only partially observed.
- 0.5-0.7 : plausible but competing hypotheses remain.
- < 0.5 : too many gaps; caller should treat as notify-only.

RETRY MODE (only when `prior_iterations` present)
- Do NOT repeat the previous verdict's root_cause unless new evidence explicitly 
  supports it and refutes the prior post-check failure signal.
- Explore alternative hypotheses at other layers first.

OUTPUT SCHEMA
{
  "root_cause": string,              // 1-2 sentences, actionable
  "affected_layer": "compute"|"l2"|"l3"|"l4"|"firewall",
  "confidence": number,              // 0.0-1.0
  "reasoning_summary": string,       // 3-6 lines, chain-of-cause narrative
  "evidence": [
    {
      "node_id": string,
      "fact_path": string,           // dotted path into facts
      "value": string,               // observed value (stringified)
      "supports_or_refutes": "supports" | "refutes"
    }
  ],
  "missing_facts": [                 // optional, list of gaps that lowered confidence
    {"node_id": string, "expected_fact": string, "why_matters": string}
  ],
  "suggested_actions": [
    {
      "action_id": string,           // MUST match available_actions[].id
      "target_node_id": string,
      "params": object,
      "rationale": string,
      "expected_effect": string
    }
  ]
}
```

## 5. User Prompt Template

```
# Incident
- id: {incident.id}
- source: {incident.source}
- severity: {incident.severity}
- target: {target.hostname} ({target.ip}) node_id={target.node_id}
- signal: {signal.type} value={signal.value} window={signal.window}
- received_at: {incident.received_at}

# Topology (path relevant to target)
nodes:
{yaml_dump(topology.nodes)}
edges:
{yaml_dump(topology.edges)}

# Diagnostic Facts (trimmed)

## {node_id} [{layer}] adapter={adapter}
errors:
{yaml_dump(errors)}
facts:
{yaml_dump(trimmed_facts)}

## ...

# Available Actions (whitelist)
{yaml_dump([
  {id, description, scope, requires_confidence, reversible}
  for a in filtered_available_actions
])}

# Prior Iterations
{yaml_dump(prior_verdicts_summary)}   # empty on first pass

# Task
Perform chain-of-cause reasoning from upstream (firewall) to downstream 
(compute). Return the JSON verdict only.
```

## 6. Output Post-processing

Reasoner 응답을 그대로 `ReasonerVerdict`로 신뢰하지 않고 다음 검증 후 채택:

1. **JSON parse**: 실패 시 최대 2회 재샘플링 (temperature 낮춤).
2. **Schema validation**: pydantic 스트릭트 파싱. 필드 누락/타입 불일치 → 재샘플링.
3. **action_id whitelist 재검증**: 응답의 action_id가 available_actions에 실제 존재하는지. 없으면 해당 action 제거하고 warning 로그.
4. **evidence 검증**: `fact_path`가 실제 fact dict에 존재하는지 dot-path resolver로 확인. 존재하지 않으면 그 evidence 제거 + confidence 페널티 (-0.1).
5. **layer 정합성**: `affected_layer`가 `suggested_actions[].target_node_id`의 layer와 일치하는지 (예: affected_layer=switch인데 action이 server 대상이면 warning).
6. **Confidence 보정**: 최종 confidence = model_confidence − (missing evidence penalty) − (partial_input penalty).

## 7. Self-consistency & Fallback

### 7.1 Self-consistency

- 프로덕션은 `n=5, temperature=0.3` 샘플링. 반복 5회.
- 합의 판정:
  - `affected_layer` 다수결 ≥ 3표 → 채택
  - `root_cause` 텍스트는 임베딩 유사도 > 0.85 인 클러스터의 대표 선택
  - `confidence`는 샘플 평균 − (샘플 표준편차)
- 다수결 미달 (≤2표) → notify-only 종결 or LLM fallback.

### 7.2 LLM fallback (policy.yaml §5.3)

- Primary: Ollama local (`qwen2.5:14b` 기본)
- Trigger: primary consensus 실패 또는 최종 confidence < 0.5
- Fallback: Claude API (환경에 API 키 있을 때만)
- Fallback verdict은 별도 label(`llm_source: "claude"`) 부착. 감사 필수.

### 7.3 Cost 관리

- Fallback 트리거 rate가 10%/hour 초과 시 primary 프롬프트 회귀 신호 → 알람.
- Fallback verdict의 confidence 상한 0.85 (외부 모델이라도 사내 컨텍스트 부족).

## 8. Retry Mode (재진단 루프)

Post-check 실패로 재진단 시:

- User prompt에 `prior_iterations` 추가:
  ```yaml
  prior_iterations:
    - iteration: 1
      verdict_id: v-...
      root_cause: "..."
      affected_layer: switch
      actions_taken: [switch.port.no_shut]
      post_check_result: still_broken
      post_check_details: "probe_success still 0 after 60s"
  ```
- System prompt의 `RETRY MODE` 섹션이 활성화되어 이전 가설 반복 억제.
- 각 iteration의 verdict는 GraphState에 누적. `max_iterations` 초과 시 `terminal_state=max_retry_exceeded`.

## 9. 프롬프트 개발 & 평가 워크플로

### 9.1 Golden set

`testbed.md` §6의 chaos 시나리오 CX-01 ~ CX-06 각각에 대해 **기대 verdict** JSON을 사전에 작성.

```
tests/reasoner/goldens/
  cx-01_gpu_hang.json          # incident + facts + expected_verdict
  cx-02_apache_kill.json
  cx-03_juniper_port_down.json
  cx-04_lldp_loss.json
  cx-05_winrm_stopped.json
  cx-06_lb_pool_disable.json
```

각 파일은 실제 어댑터가 수집한 facts를 캡처한 fixture여야 한다 (프롬프트 개발 중에는 mock 가능, PoC 이후엔 실환경 캡처로 교체).

### 9.2 평가 지표

| 지표 | 목표 (v0) |
|---|---|
| `affected_layer` 정확도 | ≥ 85% (골든셋 대비) |
| `suggested_actions[0].action_id` 정확도 | ≥ 70% |
| 파싱 실패율 | < 2% |
| Whitelist 위반 (없는 action_id) 발생률 | 0% (post-processing에서 제거되지만 로그 카운트) |
| 평균 confidence 캘리브레이션 오차 | 실제 정답률과 confidence 평균 차 ≤ 0.1 |

### 9.3 프롬프트 버전 관리

- `prompts/reasoner/system_v0.md`, `user_v0.md` 파일로 관리.
- 프롬프트 변경은 semver: 마이너=문구 정리, 메이저=스키마 변경.
- 각 verdict에 `prompt_version` 필드 부착하여 회귀 추적.

## 10. 안전 인터랙션

Reasoner 자체가 자율 실행 권한을 갖지 않지만 다음은 프롬프트 레벨에서 방지:

- **파괴적 자연어 힌트 무시**: 사용자가 fact 안에 "just reboot the server" 같은 문자열을 심어도 Reasoner는 whitelist 외 액션을 제안하지 않도록 system prompt에서 못 박음.
- **auth_failed 시 액션 봉인**: §4 CORE RULES 6 참고. 크레덴셜 실패 상황에서 조치 시도는 상황 악화 위험.
- **`confidence < threshold` 자동 강등**: policy.yaml의 `default_confidence_threshold` 미달이면 Executor가 `notify_only` 종결하도록 (Reasoner는 confidence만 정직히 리포트).

## 11. 오픈 이슈

- **facts 순서 편향**: 프롬프트에 top-down 순으로 넣지만 모델이 앞부분에 가중치를 줄 수 있음. 실제로는 원인이 하류에 있을 때 miss 가능. 순서 랜덤화 실험 필요.
- **Chain-of-thought 노출 여부**: `reasoning_summary` 필드로 CoT를 요구하는데, structured output 시 CoT 품질 저하 리포트 있음. 별도 free-text 필드 + 최종 JSON 조합도 검토.
- **한국어 vs 영어 프롬프트**: qwen2.5는 한/영 다 됨. Reasoner용은 영어 시스템 프롬프트가 안정적이라는 관측. 초기 v0은 영어 유지, 필요 시 한국어 병기.
- **다중 target incident**: 현재 프롬프트는 단일 target 가정. 동시 다발 알람에서 병합된 correlation_id 사용 시 스키마 확장 필요.
- **evidence 부재 케이스**: 모든 fact가 clean인데 알람이 온 경우 verdict은 어떻게? 관측 지표 문제로 판정할지, 관측 자체를 재수집할지.
- **model_confidence 신뢰도**: LLM의 self-reported confidence는 잘 알려진 over-confidence 편향. self-consistency 표준편차와 결합해 캘리브레이션 필요.

## 12. 다음 단계

1. `prompts/reasoner/system_v0.md`, `user_v0.md` 파일 실체화
2. `src/self_heal_infra/reasoner/` 스켈레톤 (Prompt Builder, Post-processor, Self-consistency)
3. Golden set 6종 JSON fixture 작성 (chaos 시나리오와 1:1 매핑)
4. Evaluation harness (`pytest -k reasoner_eval`)
5. 다음 옵션: **Autonomous Action Whitelist v1** (`autonomous_actions.yaml` 실제 등재 목록) — Reasoner 제안을 실행으로 이어주는 마지막 연결고리
