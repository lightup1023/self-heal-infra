# Deployment

`self-heal-infra`를 **설치 가능한 앱**으로 배포하기 위한 아티팩트, 파일 트리, 설정 인터페이스, 운영 절차 정의. 코드 착수 전에 못 박아야 다음 단계에서 파일 경로·env 이름·CLI가 코드 곳곳에 하드코드로 스며드는 걸 방지한다.

## 1. 원칙

- **12-Factor 준수**: config는 파일/env로만, secret은 실행 시 볼륨 마운트, 코드에 하드코드 금지. (12-Factor = 클라우드/컨테이너 시대 앱 개발 관례. 요약하면 "이미지는 어디서 돌든 같은 동작, 환경 차이는 config로만 흡수".)
- **운영자 접촉면 = semver 대상**: 파일 경로, env 이름, CLI 서브커맨드는 한 번 정하면 마이너 버전 안에서 breaking change 금지.
- **Air-gap 가능**: 사내 폐쇄망 사이트가 대상이 되므로 오프라인 설치 절차가 처음부터 존재해야 한다.
- **OS/런타임 종속성 최소화**: 앱 자체는 Linux + 컨테이너 런타임만 있으면 동작.
- **State 위치 단일화**: 상태는 한 곳(`SELFHEAL_STATE_DIR`)에만. 여러 경로에 흩어지지 않게.

## 2. 배포 아티팩트 매트릭스

| 형태 | Phase | 대상 | 특징 |
|---|---|---|---|
| **Docker image** (multi-arch) | 1 | 표준 배포 단위 | 앱 프로세스 하나. LLM/DB는 외부 |
| **docker-compose bundle** | 1 | 단일 노드 사이트 (S~M 규모) | 앱 + Chroma + (선택) Prom sidecar |
| **pip / uv package** | 상시 | 개발자, 컨트리뷰터 | 로컬 실행/디버깅 |
| **Helm chart** | 4 | K8s 도입 사이트 | Postgres/Redis 연결 표준화 |
| **systemd bundle (tar)** | 4 | 컨테이너 금지 사이트 (일부 금융/공공) | binary + unit file + 설정 skeleton |
| **오프라인 번들** | 1 (병행) | Air-gap | Docker image tar + LLM weight + wheels + Chroma seed |

Phase 1은 **Docker image + compose + 오프라인 번들** 3종만. Helm/systemd는 실 요구가 오면 그때.

## 3. 파일 트리 표준

설치 후 시스템에서 만나는 표준 경로. **FHS(Filesystem Hierarchy Standard)** 관례를 따른다.

```
/etc/self-heal/                     # 운영자가 수정하는 config (읽기 전용 마운트 권장)
├── policy.yaml
├── autonomous_actions.yaml
├── post_checks.yaml
├── prom_queries.yaml
├── inventory/
│   └── <site>.yaml
├── credentials/
│   ├── refs.yaml                   # 메타데이터만
│   └── *.enc.yaml                  # SOPS 암호화된 실제 시크릿
└── prompts/
    └── reasoner/
        ├── system_v0.md
        └── user_v0.md

/var/lib/self-heal/                 # 앱이 쓰는 상태 (백업 대상)
├── state.sqlite                    # 감사 로그, rate-limit, incident 이력
├── chroma/                         # 벡터 DB
└── cache/

/var/log/self-heal/                 # 로그 (파일로 남기는 경우만; 기본은 stdout)
├── app.log                         # structlog JSON
└── actions.log                     # 자율 조치 audit trail (append-only)
```

> 왜 이렇게 나누냐: `/etc`는 설정, `/var/lib`는 상태, `/var/log`는 로그로 관례가 굳어 있어 백업·감사 스크립트·SELinux 정책 등이 이 구조를 전제로 동작. 앱이 임의 경로를 쓰면 사이트 인프라 도구가 다 안 맞음.

## 4. 환경변수 규격

접두어 `SELFHEAL_` 고정. Boolean은 `on/off` (대소문자 무시).

| Env | 기본값 | 설명 |
|---|---|---|
| `SELFHEAL_CONFIG_DIR` | `/etc/self-heal` | Config 루트 |
| `SELFHEAL_STATE_DIR` | `/var/lib/self-heal` | 상태 루트 |
| `SELFHEAL_LOG_DIR` | `/var/log/self-heal` | 파일 로그 루트 (사용 시) |
| `SELFHEAL_LOG_LEVEL` | `INFO` | `DEBUG`/`INFO`/`WARNING`/`ERROR` |
| `SELFHEAL_LOG_FORMAT` | `json` | `json` or `console` |
| `SELFHEAL_AUTONOMOUS` | `on` | Kill-switch. `off` = 자율 조치 봉인 |
| `SELFHEAL_LLM_PRIMARY` | `ollama` | `ollama` or `claude` or `openai_compatible` |
| `SELFHEAL_LLM_OLLAMA_URL` | `http://localhost:11434` | Ollama endpoint |
| `SELFHEAL_LLM_FALLBACK` | (unset) | `claude` 등 |
| `SELFHEAL_CLAUDE_API_KEY` | (unset) | fallback 사용 시 |
| `SELFHEAL_PROM_URL` | (unset) | Prometheus HTTP API |
| `SELFHEAL_ALERTMANAGER_URL` | (unset) | Alertmanager webhook 검증용 |
| `SELFHEAL_DB_URL` | `sqlite:///var/lib/self-heal/state.sqlite` | 상태 DB URL. Postgres 시 `postgresql://...` |
| `SELFHEAL_SITE` | (unset) | 사이트 식별자 (다중 사이트 대비) |
| `SELFHEAL_OTEL_ENDPOINT` | (unset) | OpenTelemetry OTLP endpoint (선택) |
| `SELFHEAL_LANGFUSE_HOST` | (unset) | Langfuse endpoint (선택) |

우선순위: **env > runtime API > policy.yaml > default**.

## 5. Bootstrap CLI

앱 실행 파일 `selfheal`. 운영자가 마주치는 **첫 인터페이스**.

### 5.1 서브커맨드

| Command | 목적 |
|---|---|
| `selfheal init` | 처음 설치 시 config 스켈레톤 생성 |
| `selfheal validate` | 인벤토리·크레덴셜·정책의 스키마·참조 유효성 검증 (실행 전 필수) |
| `selfheal doctor` | 외부 시스템 연결·크레덴셜 도달 스모크 테스트 |
| `selfheal migrate` | 상태 DB 스키마 마이그레이션 (`--to 0.2`) |
| `selfheal run` | 데몬 실행 (docker CMD 기본값) |
| `selfheal killswitch` | 런타임 kill-switch 토글 (`--on/--off/--status`) |
| `selfheal --version` | 앱 + 스키마 버전 출력 |

### 5.2 예시 흐름

```
$ selfheal init --site gpu-lab-1 --path /etc/self-heal
Created /etc/self-heal/
  policy.yaml (default)
  autonomous_actions.yaml (v1 whitelist, disabled by default)
  post_checks.yaml (v1)
  inventory/gpu-lab-1.yaml (empty template with comments)
  credentials/refs.yaml (empty template)
Next steps:
  1. Fill inventory/gpu-lab-1.yaml
  2. Encrypt secrets: sops --encrypt credentials/juniper.raw.yaml > credentials/juniper.enc.yaml
  3. Run: selfheal validate

$ selfheal validate
✓ policy.yaml schema OK
✓ inventory/gpu-lab-1.yaml schema OK (6 nodes: 2 servers, 3 switches, 1 lb)
✓ credentials/refs.yaml: all 4 refs resolvable
✓ autonomous_actions.yaml: 4 entries, all with post_check_ref
✓ prompts/reasoner/system_v0.md exists
✗ post_checks.yaml: 'pool_member_up' referenced by lb.haproxy.server.enable but not defined

$ selfheal doctor
Reachability:
  Ollama http://10.10.0.5:11434         ✓ (qwen2.5:14b loaded)
  Prometheus http://10.10.0.10:9090     ✓
  Alertmanager http://10.10.0.10:9093   ✓
Adapters:
  linux_ssh   node-gpu-03               ✓
  junos_cli   sw-tor-1                  ✗ auth_failed (check cred_net_ssh)
  winrm       win-relay-01              ✓
Autonomous: on
DB migration: state.sqlite schema_version=0.1 (latest)
```

`init`은 최소 스켈레톤을 만들되 실제 값은 **비워둔다**. 자동 채움은 실수 유발.

## 6. 12-Factor 규약 (코딩 룰)

배포 이슈의 대부분은 코드 스타일에서 시작된다. 다음은 코드 리뷰 필수 체크:

1. **Config load는 `settings.py` 하나 경유**. 파이썬 코드 안에서 `os.getenv`, YAML 열기 금지.
2. **경로는 `pathlib.Path` + Settings에서 나온 값만**. 하드코드 문자열 금지.
3. **Secret은 코드/이미지에 없음**. 크레덴셜은 실행 시 마운트 되는 파일에서만 로드.
4. **로그는 stdout으로만**. 파일 로그는 사이트 운영자의 수집기(예: filebeat, fluent-bit)가 담당. 앱은 그저 stdout에 structlog JSON.
5. **어댑터 외의 shell out 금지**. `subprocess.run("...")` 직접 호출은 어댑터 계층 안에서만.
6. **Timezone-aware datetime**. `datetime.now()` 금지, `datetime.now(timezone.utc)`.

이 규약은 `docs/coding-guidelines.md` 별도 문서로 승격 예정.

## 7. 상태 저장소

### 7.1 저장 대상

| 데이터 | 목적 | 저장소 |
|---|---|---|
| Incident 이력 | 감사, correlation | RDB |
| Verdict 이력 | 프롬프트 회귀 추적 | RDB |
| ActionPlan / ActionRecord | 감사, rate-limit 산출 | RDB |
| Rate-limit 카운터 | Safety Gate | RDB (S/M) → Redis (L) |
| Correlation window | 다중 알람 병합 | RDB 또는 in-memory |
| 벡터 임베딩 | RAG | Chroma (파일) → Qdrant |
| 프롬프트 trace | 디버깅 | Langfuse (선택) |

### 7.2 RDB 선택

- **개발/S/M 규모**: SQLite (`state.sqlite`). 별도 프로세스 없음.
- **프로덕션/L 규모**: Postgres (`SELFHEAL_DB_URL=postgresql://...`).
- 스키마 마이그레이션: Alembic. 앱 시작 시 자동 `alembic upgrade head` 옵션 (default off, 명시적 `selfheal migrate`).

### 7.3 백업 대상

- `/var/lib/self-heal/state.sqlite` (또는 Postgres 덤프)
- `/etc/self-heal/` 전체 (git 관리 권장)
- Chroma는 재빌드 가능하므로 백업 필수 아님

## 8. 시크릿 관리

### 8.1 기본: SOPS + age

**SOPS**는 YAML/JSON 파일의 값만 암호화하고 키는 평문으로 두는 도구. git에 커밋 가능. **age**는 SSH 키 유사한 대칭키 알고리즘.

절차:
```
$ age-keygen -o /etc/self-heal/keys.txt
$ sops --age <public_key> --encrypt --input-type yaml --output-type yaml \
       credentials/juniper.raw.yaml > credentials/juniper.enc.yaml
$ rm credentials/juniper.raw.yaml
```

`refs.yaml`에서 `backend: sops`로 참조. 앱은 실행 시 자동 복호화.

### 8.2 프로덕션: HashiCorp Vault

`backend: vault`로 세팅 시 앱이 Vault API를 호출해 시크릿 조회. Vault agent sidecar 또는 short-lived token.

### 8.3 개발: `.env`

`.env` 파일은 로컬 개발 전용. **git ignore 필수**. 프로덕션 배포에는 사용 금지.

## 9. 크기 산정

운영자가 "우리 규모에 얼마나?" 판단하는 표.

| 규모 | 관리 노드 | vCPU | RAM | Disk | 배포 형태 | State DB |
|---|---|---|---|---|---|---|
| S | ~50 | 4 | 8GB | 50GB | 단일 컨테이너 | SQLite |
| M | 50~500 | 8 | 16GB | 200GB | docker-compose | SQLite or Postgres |
| L | 500+ | 16+ | 32GB+ | 500GB+ | K8s + Helm | Postgres + Redis |

LLM은 앱 노드에 얹지 않는다. 별도 GPU 노드 또는 원격 endpoint. (`testbed.md` §2의 G1과 매칭.)

## 10. Air-gap 설치 절차

사내 폐쇄망(인터넷 접근 불가) 사이트 대응. **초기부터 CI에 포함**해서 배포 형태가 인터넷 전제로 굳지 않도록 한다.

### 10.1 오프라인 번들 구성

```
selfheal-offline-<version>.tar.gz
├── images/
│   ├── selfheal-<version>.tar          # docker save 결과
│   ├── chroma-<version>.tar
│   └── ollama-<version>.tar
├── models/
│   └── qwen2.5-14b-instruct-q4_K_M.gguf
├── wheels/                              # pip 오프라인 설치용
│   └── *.whl
├── scripts/
│   ├── install.sh
│   └── uninstall.sh
└── README-offline.md
```

### 10.2 절차

1. 인터넷 존에서 `scripts/build-offline-bundle.sh` 실행 → `.tar.gz` 생성
2. 사이트로 이동 (USB / 승인된 파일서버)
3. 사이트에서 `install.sh` 실행:
   - `docker load` 로 image 등록
   - Ollama 모델 배치
   - config skeleton 생성
4. `selfheal init --offline` (온라인 체크 skip)

## 11. 업그레이드 & 롤백

### 11.1 앱 semver

- **PATCH**: 버그 수정만. 무중단 rolling 가능.
- **MINOR**: 옵셔널 기능 추가, config 스키마 마이너 변경. Migration 자동.
- **MAJOR**: Breaking. Migration 수동. 최소 마이너 1버전 deprecation window.

### 11.2 스키마 마이그레이션

각 스키마(policy, inventory, autonomous_actions, state DB)는 자체 `schema_version` 필드. 앱이 시작 시:

1. `schema_version` 읽음
2. 앱이 지원하는 최소 버전 미만이면 실행 거부, `selfheal migrate` 안내
3. 지원 범위면 자동 실행

### 11.3 롤백

- Config: git checkout 이전 버전
- State DB: pre-migrate 자동 백업(`state.sqlite.bak-<ts>`)에서 복구, `alembic downgrade`
- 이미지: 이전 태그로 재기동 (`docker compose up -d --force-recreate`)

## 12. 앱 자체 관측성

앱이 곧 관측 대상. 노출 지표·엔드포인트:

| 엔드포인트 | 목적 |
|---|---|
| `/metrics` | Prometheus text format |
| `/healthz` | Liveness (프로세스 살아있음) |
| `/readyz` | Readiness (DB/LLM 도달 가능) |
| `/version` | 앱 + 스키마 버전 |

노출 메트릭 초안:

| 메트릭 | 종류 | 라벨 |
|---|---|---|
| `selfheal_incidents_total` | counter | `source`, `severity`, `signal_type` |
| `selfheal_verdicts_confidence` | histogram | `affected_layer`, `llm_source` |
| `selfheal_actions_total` | counter | `action_id`, `status` |
| `selfheal_gate_denials_total` | counter | `reason` |
| `selfheal_llm_latency_seconds` | histogram | `llm_source`, `model` |
| `selfheal_adapter_errors_total` | counter | `adapter`, `error_type` |
| `selfheal_kill_switch_state` | gauge | (0/1) |

Langfuse/OTel 트레이스는 선택. Env로 세팅되면 활성.

## 13. 보안

- **Non-root 실행**: 컨테이너 내 `selfheal` uid 10001. root 사용 금지.
- **Read-only root filesystem** + 필요 tmpfs 명시.
- **Secret 마운트**: `/etc/self-heal/credentials/*.enc.yaml`은 read-only 볼륨. 이미지에 미포함.
- **Egress TLS 강제**: LLM/REST 대상 아웃바운드는 TLS 검증 on.
- **Audit log append-only**: `/var/log/self-heal/actions.log`는 immutable bit (`chattr +a`) 권장.
- **최소 권한**: 어댑터별 리모트 계정은 진단·조치에 필요한 최소 권한. 예:
  - Linux SSH: 별도 `selfheal` 유저, sudoers는 whitelist 명령만.
  - Juniper: 진단용 read-only 계정 + 조치용 별도 계정 (Configure만).
  - FortiGate: REST API 토큰 최소 스코프.

## 14. 지원 매트릭스

### 14.1 호스트 OS (앱을 돌리는 서버)

| OS | 상태 |
|---|---|
| Ubuntu 22.04 LTS | Tier 1 (개발/CI 기준) |
| Ubuntu 24.04 LTS | Tier 1 |
| RHEL 9 / Rocky 9 / AlmaLinux 9 | Tier 2 (docker rootless 이슈 주의) |
| Debian 12 | Tier 2 |
| Windows Server | 미지원 (앱은 Linux only) |

### 14.2 컨테이너 런타임

| 런타임 | 상태 |
|---|---|
| Docker 24+ | Tier 1 |
| Podman 4+ | Tier 2 (rootless는 별도 검증) |
| containerd (K8s Helm 경로) | Tier 1 (Phase 4) |

## 15. 오픈 이슈

- **사이트별 커스텀 어댑터 확장**: 사이트가 벤더 커스텀 어댑터를 넣고 싶을 때 플러그인 로딩 방식(entry point)으로 갈지, 재빌드 방식으로 갈지.
- **LLM weight 배포 정책**: 앱 이미지에 포함할지, Ollama volume 별도로 갈지. 이미지 크기·업데이트 주기 트레이드오프.
- **감사 로그 tamper-evident**: `chattr +a`만으로 부족한 규제 사이트(금융)를 위해 hash chain 또는 외부 WORM 스토리지 옵션 필요할지.
- **다중 사이트 관리**: 사이트 수가 늘면 중앙 관리 콘솔이 필요해짐. Phase 5+ 검토.
- **자동 업그레이드 정책**: 앱이 스스로 최신 버전 pull 할지 말지 (자율 시스템이 자기 자신을 자율 업그레이드 하는 건 위험).
- **라이선스**: README TBD. OSS로 갈지 사내 자산으로 갈지 결정 후 이미지 라벨·SBOM 포함.
- **SBOM (Software Bill of Materials)**: 컴플라이언스 요구가 있는 사이트를 위해 이미지에 SBOM 첨부 표준화 필요.

## 16. 다음 단계

1. `pyproject.toml` — CLI entry-point (`selfheal = self_heal_infra.cli:main`)
2. `src/self_heal_infra/settings.py` — pydantic-settings로 §4 env 로딩
3. `Dockerfile` — multi-stage, non-root, distroless-friendly
4. `docker-compose.yaml` — 앱 + Chroma (선택: Prom/Alertmanager sidecar)
5. `scripts/build-offline-bundle.sh` — §10 오프라인 번들
6. `docs/coding-guidelines.md` — §6 규약 명문화 (별도 문서로 승격)
7. 여기까지 완료되면 **설계 단계 최종 종료 → 코드 스켈레톤 착수**

---

## 부록: 배포 개념이 처음이라면 알아두면 좋을 것

문서 이해 돕기용 짧은 용어 정리.

- **12-Factor App**: 12factor.net. 요약하면 "설정은 코드가 아니라 환경에서, 상태는 프로세스가 아니라 백엔드에, 로그는 파일이 아니라 stdout으로".
- **FHS**: Filesystem Hierarchy Standard. `/etc/설정`, `/var/lib/상태`, `/var/log/로그` 관례.
- **structlog JSON logs**: 로그를 문자열이 아니라 JSON 객체로 남겨서 나중에 Loki/ELK가 자동 파싱 가능하게 함.
- **Semver**: `MAJOR.MINOR.PATCH`. Breaking = MAJOR, feature = MINOR, fix = PATCH.
- **Air-gap**: 인터넷 접근이 물리적/정책적으로 차단된 폐쇄망.
- **SBOM**: 이미지에 포함된 오픈소스 목록·라이선스·버전 리스트. 컴플라이언스용.
- **Helm chart**: K8s 배포 템플릿 묶음. `helm install` 한 방으로 여러 K8s 리소스 배포.
- **Kill-switch**: "전체 자율 조치를 즉시 봉인"할 수 있는 스위치. env 하나로 켤 수 있어야 함.
