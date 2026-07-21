# Diagnosis Toolkit

Parallel Diagnostor가 각 노드에서 실행하는 **read-only 진단 명령 카탈로그**와 어댑터별 `facts` 표준 스키마 정의. 여기서 정한 스키마가 `data-model.md` §3.2의 `DiagnosisResult.facts`로 그대로 채워진다.

## 1. 원칙

- **읽기 전용**: `show / get / read` 계열만. 어댑터 구현에 파괴적 명령 화이트리스트 차단 로직 필수.
- **결정 가능한 출력**: 자유 텍스트를 그대로 넘기지 않고 어댑터가 파싱해 정형화. LLM 프롬프트 안정성이 여기서 결판남.
- **계층별 표준 facts**: 어댑터 다양화에도 상위 코드가 안 깨지도록 계층(Layer)별 상위 스키마를 강제하고, 벤더별 상세는 하위 필드로.
- **타임아웃 명시**: 명령별 초 단위 타임아웃. 무한 대기 금지.
- **실패도 팩트**: SSH timeout = 서버 hang 후보. 성공/실패 모두 `CommandRun`에 기록되고 Reasoner에 신호로 전달.

## 2. 공통 계약

### 2.1 계층별 facts 상위 스키마

`DiagnosisResult.facts`는 어댑터별 자유 dict이지만 Layer 단위로 다음 최상위 키를 강제한다.

| Layer | 최상위 키 |
|---|---|
| compute | `host_reachable`, `os`, `system`, `services`, `network`, `gpu?` |
| l2 / l3 (switch·router) | `device_reachable`, `device`, `interfaces`, `neighbors`, `mac_table?`, `routes?`, `alarms` |
| l4 (loadbalancer) | `device_reachable`, `device`, `frontends?`, `backends`, `virtual_servers?` |
| firewall | `device_reachable`, `device`, `system`, `interfaces`, `policies`, `drops`, `alarms` |

벤더별 확장은 각 스키마 하위에 `vendor_specific: {...}`로 격리.

### 2.2 타임아웃 정책

| 항목 | 기본값 | 최대 |
|---|---|---|
| 단일 명령 timeout | 10s | 60s |
| 어댑터 세션 총 timeout | 90s | 180s |
| SSH connect timeout | 5s | 10s |
| HTTP request timeout | 10s | 30s |

### 2.3 재시도

**진단 단계에서는 재시도하지 않는다.** flapping을 은폐하고 latency만 늘리기 때문. 실패는 `DiagnosisError`로 그대로 남기고 Reasoner에 신호로 전달.

## 3. Linux 어댑터 (asyncssh)

### 3.1 명령 세트 (Phase 1)

| 카테고리 | 명령 | 목적 | timeout |
|---|---|---|---|
| Reachability | `uname -a && uptime` | SSH 도달 + 기본 정보 | 5s |
| Systemd | `systemctl --failed --no-legend --output=json` | 실패 유닛 | 5s |
| Journal | `journalctl -p err -n 100 --since '10min ago' --no-pager` | 최근 error 로그 | 10s |
| Kernel | `dmesg -T --level=err,crit,alert,emerg \| tail -200` | 커널 에러 | 10s |
| Process | `ps -eo pid,pcpu,pmem,comm --sort=-pcpu \| head -20` | CPU top 20 | 5s |
| Memory | `free -m` | 메모리 요약 | 5s |
| Disk | `df -hT` | 파일시스템 | 5s |
| Net link | `ip -json link show && ip -json addr show` | 인터페이스 (JSON) | 5s |
| Net route | `ip -json route show` | 라우팅 (JSON) | 5s |
| Net conn | `ss -tan state established \| wc -l && ss -s` | 커넥션 요약 | 5s |
| GPU state | `nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,memory.used,memory.total,pstate --format=csv,noheader` | GPU 상태 | 10s |
| GPU Xid | `dmesg -T \| grep -i xid \| tail -50` | Xid 에러 | 10s |
| Container | `docker ps --format '{{.Names}}\t{{.Status}}'` (있으면) | 컨테이너 상태 | 5s |

### 3.2 facts 스키마

```yaml
facts:
  host_reachable: true
  os:
    kernel: "6.8.0-...-generic"
    distro: "Ubuntu 22.04"
    uptime_seconds: 12345
  system:
    load_avg: [1.2, 1.1, 0.9]
    mem_total_mb: 65536
    mem_available_mb: 4096
    disk:
      - {mount: "/", used_pct: 87, fs: "ext4"}
  services:
    failed_units: ["nginx.service"]
    critical_errors_recent:
      - {ts: "...", unit: "nginx.service", msg: "..."}
  network:
    interfaces:
      - {name: "eno1", state: "UP", addrs: ["10.10.20.13/24"]}
    established_conns: 1523
    default_gw: "10.10.20.1"
  gpu:                              # optional (GPU 없는 노드는 생략)
    devices:
      - {index: 0, name: "RTX 4090", temp_c: 78, util_pct: 12,
         mem_used_mb: 2048, mem_total_mb: 24564, pstate: "P0"}
    xid_recent:
      - {ts: "...", xid: 79, device: 0}
  vendor_specific: {}
```

### 3.3 파싱 노트

- `systemctl --output=json`, `ip -json`으로 파싱 부담 최소화.
- `nvidia-smi --format=csv,noheader`로 CSV 파싱.
- `journalctl --output=json`도 가능하지만 라인당 JSON이 나오므로 line-parse.

## 4. Windows 어댑터 (pywinrm)

### 4.1 명령 세트

전부 PowerShell. 결과를 `ConvertTo-Json`으로 감싸 파싱 부담 최소화.

| 카테고리 | 명령 | 목적 | timeout |
|---|---|---|---|
| Reachability | `hostname; (Get-CimInstance Win32_OperatingSystem).Caption` | 도달 확인 | 5s |
| Uptime | `(Get-CimInstance Win32_OperatingSystem).LastBootUpTime` | 부팅 시간 | 5s |
| Services (stopped-auto) | `Get-Service \| Where {$_.Status -eq 'Stopped' -and $_.StartType -eq 'Automatic'} \| Select Name,DisplayName \| ConvertTo-Json` | 자동시작인데 멈춘 서비스 | 5s |
| System Event | `Get-WinEvent -FilterHashtable @{LogName='System';Level=1,2;StartTime=(Get-Date).AddMinutes(-30)} -MaxEvents 50 \| Select TimeCreated,Id,ProviderName,Message \| ConvertTo-Json` | 최근 System 오류 | 15s |
| Application Event | `Get-WinEvent -FilterHashtable @{LogName='Application';Level=1,2;StartTime=(Get-Date).AddMinutes(-30)} -MaxEvents 50 \| ConvertTo-Json` | 앱 오류 | 15s |
| CPU/Mem | `Get-Counter '\Processor(_Total)\% Processor Time','\Memory\Available MBytes' -SampleInterval 1 -MaxSamples 3 \| ConvertTo-Json -Depth 4` | 성능 카운터 | 10s |
| Disk | `Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3" \| Select DeviceID,FreeSpace,Size \| ConvertTo-Json` | 로컬 디스크 | 5s |
| Network | `Get-NetAdapter \| Select Name,Status,LinkSpeed,MacAddress \| ConvertTo-Json` | 인터페이스 | 5s |
| IIS Site | `Import-Module WebAdministration; Get-Website \| Select Name,State,PhysicalPath \| ConvertTo-Json` | IIS 사이트 | 10s |
| IIS App Pool | `Get-IISAppPool \| Select Name,State \| ConvertTo-Json` | 앱풀 상태 | 5s |

### 4.2 facts 스키마

```yaml
facts:
  host_reachable: true
  os:
    caption: "Windows Server 2022 Standard"
    last_boot: "2026-07-15T09:00:00Z"
  system:
    cpu_pct_avg: 12.4
    mem_available_mb: 5120
    disk:
      - {drive: "C:", free_gb: 45, total_gb: 100}
  services:
    stopped_auto: ["W3SVC"]
    critical_events_recent:
      - {ts, source, id, message}
  network:
    adapters:
      - {name, status, link_speed, mac}
  iis:                              # optional
    sites:
      - {name, state, physical_path}
    app_pools:
      - {name, state}
  vendor_specific: {}
```

### 4.3 파싱 노트

- WinRM UTF-16 인코딩 이슈 잦음. 세션 프리앰블 필수:
  ```
  [Console]::OutputEncoding = [Text.Encoding]::UTF8
  $OutputEncoding = [Text.Encoding]::UTF8
  ```
- `ConvertTo-Json -Depth 3` 이상 필요 시 명시 (기본 2).
- AD 도메인 미참여 stand-alone 가정 (인증은 로컬 계정, testbed.md §7 참고).

## 5. Juniper 어댑터 (netmiko / scrapli, `juniper_junos`)

### 5.1 명령 세트

Junos는 `| display json` 붙이면 정형 출력. JSON 우선.

| 카테고리 | 명령 | 목적 | timeout |
|---|---|---|---|
| Reachability | `show version \| display json` | 접속 + 버전 | 5s |
| Interfaces summary | `show interfaces terse \| display json` | 전 포트 상태 요약 | 10s |
| Interface detail | `show interfaces {port} extensive \| display json` | 특정 포트 에러/CRC | 15s |
| LLDP | `show lldp neighbors \| display json` | 인접 장비 | 10s |
| Ethernet Switching | `show ethernet-switching table \| display json` | MAC 학습 | 15s |
| Route | `show route summary \| display json` | 라우팅 요약 | 10s |
| System Alarms | `show system alarms \| display json` | 활성 알람 | 5s |
| Chassis Alarms | `show chassis alarms \| display json` | 하드웨어 알람 | 5s |
| Log | `show log messages \| last 200` | 최근 로그 (텍스트 파싱) | 15s |
| Commit history | `show system commit` | 최근 커밋 이력 (회귀 탐지) | 5s |

### 5.2 facts 스키마

```yaml
facts:
  device_reachable: true
  device:
    hostname: "sw-tor-1"
    model: "EX3400-48P"
    junos_version: "21.4R3-S4"
  interfaces:
    - name: "xe-0/0/13"
      admin: "up"
      oper: "down"                  # 곧장 장애 후보
      speed: "10Gbps"
      input_errors: 0
      crc_errors: 152
      output_errors: 0
      last_flap: "2026-07-21T09:41:00Z"
  neighbors:
    - local_port: "xe-0/0/13"
      remote_system: "gpu-node-03"
      remote_port: "eno1"
  mac_table:
    entries_total: 1245
    per_vlan:
      "100": 340
  alarms:
    - {type: "chassis", severity: "major", description: "PEM 1 Not OK"}
  routes:
    inet_total: 245
  recent_commits:
    - {ts, user, comment}
  vendor_specific: {}
```

### 5.3 파싱 노트

- Junos JSON 출력은 최상위가 `{"key": [ ... ]}`로 감싸져 있음. 파서에서 언랩 필요.
- `show interfaces terse | match "^xe|^ge"` 같은 필터는 pipe로 CLI에서 축소.
- SRX 확장 시 `show security policies hit-count`, `show security flow session` 추가.

## 6. FortiGate 어댑터 (REST, `fortiosapi` or `httpx`)

### 6.1 엔드포인트 세트 (Phase 2)

| 카테고리 | Endpoint | 목적 | timeout |
|---|---|---|---|
| Reachability | `GET /api/v2/monitor/system/status` | 시스템 요약 | 5s |
| Interfaces | `GET /api/v2/monitor/system/interface` | 포트 상태 | 10s |
| Policies | `GET /api/v2/cmdb/firewall/policy?with_meta=1` | 정책 목록 | 15s |
| Policy hit | `GET /api/v2/monitor/firewall/policy` | 정책 히트카운트 | 10s |
| Sessions | `GET /api/v2/monitor/firewall/session?count=100` | 세션 샘플 | 15s |
| Deny log | `GET /api/v2/log/traffic/forward?rows=100&filter=action==deny` | 최근 drop | 15s |
| HA | `GET /api/v2/monitor/system/ha-checksums` | HA 상태 | 5s |
| Resource | `GET /api/v2/monitor/system/resource/usage` | CPU/메모리/세션 | 5s |

### 6.2 facts 스키마

```yaml
facts:
  device_reachable: true
  device:
    hostname: "fw-a"
    model: "FGT-VM64"
    fortios_version: "7.4.2"
    ha_role: "master"
  system:
    cpu_pct: 22
    mem_pct: 45
    session_count: 12345
    session_limit: 2000000
  interfaces:
    - {name: "port1", status: "up", speed: "1000full", rx_errors: 0, tx_errors: 0}
  policies:
    total: 128
    top_hits:
      - {id: 12, name: "allow-web", hits: 452301}
  drops:
    recent:
      - {ts, src, dst, action: "deny", policy_id: 0}
  alarms: []
  vendor_specific: {}
```

### 6.3 파싱 노트

- REST 응답이 `{ "results": [...], "status": "success" }`로 감싸짐. 언랩 필수.
- 인증은 헤더 `Authorization: Bearer <token>`. 크레덴셜은 `cred_fw_api` ref로.
- 세션 쿼리는 반드시 `count=`로 상한. 전체 덤프 금지.

## 7. HAProxy 어댑터 (unix socket / stats API)

### 7.1 명령 세트 (Phase 2)

Runtime API via unix socket 또는 stats page.

| 카테고리 | 명령 | 목적 | timeout |
|---|---|---|---|
| Info | `show info` | 프로세스 정보 | 5s |
| Stat | `show stat` (CSV) | 프론트/백엔드/서버 상태 | 10s |
| Servers state | `show servers state` | 서버별 상세 | 10s |
| Errors | `show errors` | 최근 파싱 에러 | 10s |

### 7.2 facts 스키마

```yaml
facts:
  device_reachable: true
  device:
    haproxy_version: "2.8.1"
    pid: 12345
    uptime_seconds: 87600
  frontends:
    - {name: "http-in", status: "OPEN", sessions_cur: 234, sessions_max: 2000}
  backends:
    - name: "web-pool"
      status: "UP"
      active_servers: 2
      backup_servers: 0
      servers:
        - {name: "web-1", status: "UP", weight: 100,
           check_status: "L4OK", check_duration_ms: 12}
        - {name: "web-2", status: "DOWN", last_check: "L4CON"}
  vendor_specific: {}
```

## 8. Prometheus 어댑터 (PromQL via httpx)

### 8.1 쿼리 카탈로그

계층별 정합성 확인용 쿼리를 카탈로그로 관리 (`config/prom_queries.yaml`).

```yaml
schema_version: "0.1"

queries:
  ping_success:
    promql: 'probe_success{instance="{ip}"}'
    aggregation: last
  http_5xx_ratio:
    promql: |
      sum(rate(http_requests_total{status=~"5..",host="{hostname}"}[1m]))
      / sum(rate(http_requests_total{host="{hostname}"}[1m]))
    aggregation: last
  node_cpu_load:
    promql: 'node_load1{instance="{ip}:9100"}'
    aggregation: last
  dcgm_gpu_util:
    promql: 'DCGM_FI_DEV_GPU_UTIL{Hostname="{hostname}"}'
    aggregation: last
  dcgm_gpu_temp:
    promql: 'DCGM_FI_DEV_GPU_TEMP{Hostname="{hostname}"}'
    aggregation: max
```

### 8.2 facts 스키마

```yaml
facts:
  prom_reachable: true
  queried_at: "..."
  samples:
    - query_id: ping_success
      value: 1
      labels: {instance: "10.10.20.13"}
    - query_id: dcgm_gpu_util
      value: 0
      labels: {Hostname: "gpu-node-03", gpu: "0"}
  vendor_specific: {}
```

## 9. 에러 처리

`DiagnosisError`는 다음 유형으로 분류하여 Reasoner가 신호로 사용.

| type | 발생 예 | Reasoner 해석 힌트 |
|---|---|---|
| `connect_timeout` | SSH connect 5s 초과 | 대상 서버 hang 또는 네트워크 단절 강한 후보 |
| `auth_failed` | 크레덴셜 오류 | 즉시 알람, 자율 조치 금지 |
| `command_timeout` | 단일 명령 60s 초과 | 부하 or 프로세스 hang |
| `parse_error` | 출력 파싱 실패 | 어댑터 버그 신호, LLM에 raw 힌트 전달 |
| `unauthorized` | REST 401/403 | 즉시 알람 |
| `partial` | 일부 명령만 성공 | 남은 fact로 진단 진행 |

## 10. 어댑터 확장 절차

신규 벤더/장비 추가 시:

1. 본 문서에 명령 세트 + facts 스키마 초안 추가 (PR)
2. `src/self_heal_infra/adapters/<vendor>.py` 구현 (read-only enforcement)
3. `tests/adapters/test_<vendor>.py` — mocked device output으로 파싱 검증
4. Inventory 스키마의 vendor enum 확장
5. `AdapterKind` enum 확장 → `data-model.md` §3.2 업데이트

## 11. 오픈 이슈

- **명령 카탈로그 버저닝**: Junos 버전 간 출력 스키마 미묘한 차이. 카탈로그에 `min_version` 표기 필요할지.
- **facts 스키마 strictness**: pydantic strict로 강제 vs 부분 누락 허용. Reasoner 안정성 vs 실패 대응 트레이드오프.
- **로그 노이즈 억제**: `journalctl -p err`가 수만 줄 나오는 서버 존재. 시간창 + 라인 상한 강제 방침.
- **명령 파라미터 인젝션**: `show interfaces {port}` 템플릿 치환에서 인벤토리 값 검증 필수. 화이트리스트 정규식.
- **SNMP 편입 여부**: 현재 세트에 없음. Phase 2 이후 스위치 폴링용으로 편입할지.
- **Prometheus 접근성**: LX2가 다운되면 사후검증 자체 불가. 다중 Prom endpoint fallback 정책 필요.
- **SECUI 어댑터**: Phase 3 편입. CLI 문법이 벤더 종속적이라 문서 별도 절 필요.

## 12. 다음 단계

1. 어댑터 스켈레톤 3종: Linux SSH / Juniper CLI / WinRM (`src/self_heal_infra/adapters/`)
2. 어댑터 파싱 유닛 테스트 (fixture: 실제 명령 출력 캡처본)
3. `config/prom_queries.yaml` 초안
4. Chaos 시나리오 CX-01~06 각각의 "기대 facts" 문서화 (regression baseline)
5. 다음 옵션: **LLM Reasoner 프롬프트 설계 (Option C)** — 위 facts를 입력으로 하는 chain-of-cause 프롬프트
