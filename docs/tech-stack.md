# Tech Stack

`self-heal-infra`의 기술 스택 선정, 최신 스택 additions, 앱 인터페이스, 학습 목표 매핑.

## 1. 스택 선정 기준

- **개인 역량 향상용 사이드 프로젝트**. 학습 밀도 우선.
- 각 기술은 프로젝트 내 명확한 역할 + 포트폴리오 스토리 요건 만족.
- 프로덕션급 안정성보다 **최신성·재현성** 우선. 단, 무작정 다 얹지 않음 (§12 참고).

## 2. Core Stack

| 항목 | 선정 | 역할 |
|---|---|---|
| 언어 | Python 3.11+ | asyncio 병렬 진단 |
| 오케스트레이션 | **LangGraph** | 에이전트 그래프 |
| 데이터 검증 | Pydantic v2 | 모든 스키마 |
| 설정 로딩 | pydantic-settings | env + YAML 통합 |
| 로깅 | structlog | JSON 로그 (stdout) |
| CLI | **Typer + Rich** | 커맨드 + 컬러 출력 |
| 패키징 | **uv** | Rust 기반 초고속 |
| 린팅/포매팅 | **Ruff** | 통합 (black/isort/flake8/pyupgrade 대체) |

## 3. LLM

| 항목 | 선정 | 비고 |
|---|---|---|
| Primary | Ollama (qwen2.5:14b / 32b) | 로컬 GPU (testbed G1) |
| Fallback | Claude API | 고난도 케이스 (env 세팅 시만) |
| OpenAI-호환 endpoint | 옵션 | 벤더 중립 |

라우팅은 `policy.yaml`에서 관리. 기본은 로컬 → confidence < 0.5 시 fallback.

## 4. Device Access (진단 어댑터)

| 대상 | 라이브러리 | 프로토콜 |
|---|---|---|
| Linux 서버 | asyncssh | SSH |
| Juniper (스위치·SRX) | netmiko or scrapli | SSH CLI (`juniper_junos`) |
| Windows | pywinrm | WinRM |
| Prometheus | httpx | HTTP API |
| FortiGate | fortiosapi or httpx | REST |
| HAProxy | asyncssh + stats socket | Unix socket |
| SECUI (Phase 3) | asyncssh + syslog parser | SSH CLI (자체 어댑터) |

전체 명령 카탈로그 → `docs/diagnosis-toolkit.md`.

## 5. Observability

| 소스 | 용도 |
|---|---|
| Prometheus | 메트릭 시계열, 사후검증 지표 |
| Loki | 로그 (선택) |
| NVIDIA DCGM Exporter | GPU 상태 |
| OpenTelemetry | 앱 자체 트레이스 |
| Langfuse | LLM 특화 trace (§7.4) |

## 6. Vector DB / RAG

| 항목 | 선정 | 비고 |
|---|---|---|
| Vector DB | Chroma (기본) / LanceDB (임베디드 실험) | 초기 파일 기반 |
| 임베딩 | BGE-M3 | 한/영 혼용 |
| 문서 | 벤더 CLI reference, 사내 런북, 과거 티켓 |

토폴로지는 초기 YAML + `networkx` 인메모리 그래프. Neo4j 미도입.

## 7. 최신 스택 Additions

이 프로젝트의 **차별화 포인트**. 각 스택은 프로젝트 내 명확한 역할 + 포트폴리오 스토리.

### 7.1 MCP (Model Context Protocol)

- **역할**: 진단 어댑터를 MCP 서버로 노출. LLM-agnostic 재사용.
- **왜**: Anthropic 발 표준, 2024~2026 에이전트 생태계 사실상 표준. Claude/Cursor/기타 IDE에서 즉시 재활용.
- **적용 위치**: `src/self_heal_infra/mcp_servers/` — Linux/Juniper/WinRM 어댑터를 각각 MCP 서버로 wrap.
- **자연스러운 매핑**: `diagnosis-toolkit.md`의 read-only 명령 카탈로그가 이미 MCP tool 개념과 1:1.

### 7.2 n8n — Alert Ingress & 통지 프론트엔드

- **역할 A** (Alert Ingress): Alertmanager/Zabbix/syslog → n8n webhook → 정규화 → self-heal 앱 호출. 코드로 짤 필요 없는 ETL을 노코드로.
- **역할 B** (통지·리포팅): 조치 결과 → n8n → Mattermost/Slack/JIRA/Email fan-out. 채널 확장을 앱 재빌드 없이.
- **왜**: 노코드-코드 하이브리드 스토리. 사이트별 커스터마이즈를 앱 코드 밖으로 뺌.

### 7.3 DSPy — 프롬프트 자동 튜닝

- **역할**: `reasoner-prompt.md`의 chain-of-cause 프롬프트를 DSPy 모듈로 표현. Optimizer가 golden set(CX-01~06) 기반 자동 튜닝.
- **왜**: Stanford 발. 프롬프트 엔지니어링의 다음 패러다임. 손튜닝을 데이터-드리븐 튜닝으로.
- **적용 위치**: `src/self_heal_infra/reasoner/dspy_modules/`
- **주의**: 러닝 커브 있음. golden set 6개면 감당 가능한 규모.

### 7.4 Langfuse + Ragas (or DeepEval)

- **Langfuse**: LLM 호출 전체 trace + 프롬프트 버전 관리
- **Ragas / DeepEval**: golden set 기반 자동 평가 파이프라인
- **왜**: LLM 앱의 "관측+평가"가 없으면 프로덕션급 아님. 이 조합이 그 스토리 완성.
- **자연 통합**: `reasoner-prompt.md` §9의 평가 지표 정의가 이미 있음. 실제 툴로 굴리기만 하면 됨.
- **호스팅**: 초기 SaaS(Langfuse Cloud) 편함. self-host는 학습 별도 트랙.

### 7.5 Dev Toolchain 근대화

| 도구 | 대체 대상 | 학습 시간 |
|---|---|---|
| **uv** | pip / venv / poetry | 30분 |
| **Ruff** | black + isort + flake8 + pyupgrade | 30분 |
| **Typer** | argparse / Click | 1시간 |
| **Rich** | print / logging 포매팅 | 1시간 |

전부 하루면 세팅 완료. 이력서·데모 인상 크게 개선.

## 8. Testing

| 종류 | 도구 |
|---|---|
| 유닛 | pytest, pytest-asyncio |
| 속성 기반 | Hypothesis |
| 컨테이너 통합 | Testcontainers |
| 장비 시뮬레이션 | containerlab (vJunos-switch, cRPD) |
| Chaos 주입 | 자체 스크립트 (`chaos/`) |
| E2E | pytest + testbed |
| LLM 평가 | Ragas / DeepEval + golden set |

## 9. 앱 인터페이스

개발자·사용자 접촉면 정리. 프로덕션 배포 문서가 아니라 **개발용 표준 표면**.

### 9.1 파일 트리

```
config/                          운영자가 수정 (git 관리)
├── policy.yaml
├── autonomous_actions.yaml
├── post_checks.yaml
├── prom_queries.yaml
├── inventory/
│   └── <site>.yaml
└── credentials/
    ├── refs.yaml                메타데이터만
    └── *.enc.yaml               SOPS 암호화된 시크릿

state/                           앱이 쓰는 상태 (백업 대상)
├── state.sqlite
└── chroma/

prompts/reasoner/
├── system_v0.md
└── user_v0.md
```

프로덕션 이관 시에는 FHS 관례에 따라 `/etc/self-heal/`, `/var/lib/self-heal/` 로 마운트. 사이드 프로젝트 스코프에서는 프로젝트 하위 상대 경로로 충분.

### 9.2 환경변수 규격

접두어 `SELFHEAL_` 고정. 핵심만.

| Env | 기본값 | 목적 |
|---|---|---|
| `SELFHEAL_CONFIG_DIR` | `./config` | Config 루트 |
| `SELFHEAL_STATE_DIR` | `./state` | 상태 루트 |
| `SELFHEAL_AUTONOMOUS` | `on` | Kill-switch (`off`=봉인) |
| `SELFHEAL_LLM_OLLAMA_URL` | `http://localhost:11434` | Ollama endpoint |
| `SELFHEAL_LLM_FALLBACK` | (unset) | `claude` 등 |
| `SELFHEAL_CLAUDE_API_KEY` | (unset) | fallback 사용 시 |
| `SELFHEAL_PROM_URL` | (unset) | Prometheus HTTP API |
| `SELFHEAL_DB_URL` | `sqlite:./state/state.sqlite` | 상태 DB |
| `SELFHEAL_LANGFUSE_HOST` | (unset) | Langfuse endpoint |
| `SELFHEAL_LANGFUSE_PUBLIC_KEY` | (unset) | Langfuse auth |
| `SELFHEAL_LOG_LEVEL` | `INFO` | `DEBUG`/`INFO`/`WARNING`/`ERROR` |

우선순위: **env > runtime API > policy.yaml > default**.

### 9.3 CLI 서브커맨드 (Typer + Rich)

| Command | 목적 |
|---|---|
| `selfheal init` | Config 스켈레톤 생성 |
| `selfheal validate` | 스키마·참조·크레덴셜 유효성 검사 |
| `selfheal doctor` | Ollama/Prom/adapter 도달성 스모크 테스트 |
| `selfheal run` | 데몬 실행 |
| `selfheal killswitch --on/--off/--status` | 런타임 kill-switch |
| `selfheal eval` | Golden set 자동 평가 (Ragas 트리거) |
| `selfheal migrate` | 상태 DB 스키마 마이그레이션 |
| `selfheal --version` | 앱 + 스키마 버전 |

`init/validate/doctor`가 UX 핵심. 데모 영상 찍을 때 Rich progress bar·컬러 표로 잘 보이게.

### 9.4 코딩 규약 (12-Factor + α)

배포 문제의 대부분은 코드 스타일에서 시작. 다음은 코드 리뷰 필수 체크:

1. **Config load는 `settings.py` 하나 경유**. `os.getenv` 직접 호출 금지.
2. **경로는 `pathlib.Path` + Settings 값만**. 하드코드 문자열 금지.
3. **Secret은 코드/이미지에 없음**. 실행 시 마운트 되는 파일에서만 로드.
4. **로그는 stdout으로만**. 파일 로그는 사이트 수집기 담당.
5. **Adapter 외의 shell out 금지**. `subprocess.run` 직접 호출은 어댑터 계층 안에서만.
6. **Timezone-aware datetime**. `datetime.now()` 금지, 항상 `datetime.now(timezone.utc)`.

## 10. 배포

**사이드 프로젝트 스코프**: Docker image + docker-compose 만 지원.

```
docker-compose.yaml
├── selfheal              앱 컨테이너
├── chroma                벡터 DB
├── langfuse (선택)       LLM trace
├── n8n (선택)            워크플로우
└── prometheus (선택)     로컬 관측
```

Helm chart / systemd bundle / air-gap 오프라인 번들은 **이 프로젝트에서 다루지 않음**. 필요해지면 그때 확장.

## 11. 학습 목표 매핑

이 프로젝트로 만들 수 있는 이력서·포트폴리오 문장:

| 학습 축 | 스택 | 이력서 문장 예 |
|---|---|---|
| Agent orchestration | LangGraph | "LangGraph 로 다층 인프라 자율 진단·조치 에이전트 설계·구현" |
| LLM 표준화 | MCP | "인프라 진단 툴을 MCP 서버로 표준화, LLM-agnostic 재사용성 확보" |
| Prompt engineering 진화 | DSPy | "프롬프트 엔지니어링을 DSPy 기반 데이터-드리븐 최적화로 대체" |
| LLM 관측·평가 | Langfuse + Ragas | "LLM trace 관측 + golden set 자동 평가 파이프라인 구축" |
| Modern Python | uv + Ruff + Typer | "Rust 기반 툴체인(uv/Ruff)으로 개발 생산성 개선" |
| Workflow 하이브리드 | n8n | "코드-노코드 하이브리드 파이프라인으로 사이트별 확장 흡수" |
| Multi-vendor 자동화 | netmiko/asyncssh/pywinrm/fortiosapi | "다벤더 네트워크·서버·보안 장비 통합 진단 자동화" |
| Chaos engineering | 자체 chaos 스크립트 | "chaos 시나리오 기반 회귀 테스트로 자율 조치 안전성 검증" |
| Structured LLM output | Pydantic v2 + JSON mode | "LLM 응답을 Pydantic strict 스키마로 강제, hallucination 방지 파이프라인" |

## 12. 하지 않기로 한 것

명시적 제외 (사이드 프로젝트 스코프 유지):

- **AutoGen / CrewAI 등 multi-agent 프레임워크 도입**: LangGraph가 이미 그 역할. 중복은 감점.
- **LangChain → LlamaIndex 전환**: 큰 차이 없음. 시간 낭비.
- **Kubernetes 자체 학습 목적의 K8s 배포**: 이 프로젝트에 K8s 스토리 불필요.
- **Air-gap 오프라인 번들 / RHEL·Windows Server 지원 매트릭스 / SBOM**: 사내 도입 준비물. 사이드 프로젝트 스코프 초과.
- **모든 최신 툴 무제한 도입**: §7의 5개 additions로 상한.
- **HITL(Human-in-the-Loop)**: 원류 사내 프로젝트에 이미 있음. 이 프로젝트는 자율 트랙 실험이 목적.

## 13. 오픈 이슈

- **MCP 서버 실행 방식**: 각 어댑터를 별도 프로세스로 띄울지, 앱 내부 라이브러리 호출로 유지할지.
- **DSPy optimizer 선택**: BootstrapFewShot vs MIPROv2 등. golden set 6개 규모에서 어느 optimizer가 안정적인지 실험 필요.
- **n8n 도입 시점**: 초기 코드부터 n8n을 우선 배치할지, 앱 완성 후 n8n으로 wrap 할지.
- **Langfuse SaaS vs self-host**: 사이드 프로젝트는 SaaS 편함. self-host는 학습 별도 트랙.
- **Chroma → LanceDB 스위칭**: 임베디드 벡터DB로 상태 저장 통합 실험 가치.
- **Ollama vs vLLM**: 프로덕션 지향이면 vLLM이 throughput 우세. 사이드 프로젝트 편의는 Ollama.

## 14. 다음 단계

1. `pyproject.toml` — uv 기반, Ruff 설정 포함
2. `src/self_heal_infra/settings.py` — §9.2 env 로딩 (pydantic-settings)
3. `src/self_heal_infra/mcp_servers/linux.py` — 첫 MCP 서버 스켈레톤
4. `src/self_heal_infra/reasoner/dspy_modules/` — DSPy 모듈 초안
5. Langfuse 계정 세팅 + trace 붙이기
6. n8n 워크플로우 초안 (Alertmanager webhook → normalize → HTTP call)
7. `Dockerfile` + `docker-compose.yaml` 초안
