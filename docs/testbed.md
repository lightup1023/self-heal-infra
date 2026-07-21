# Testbed

`self-heal-infra`의 PoC/개발용 테스트 환경 하드웨어·토폴로지 명세. 사내 프로덕션과 격리된 별도 인프라 위에서 구성한다.

## 1. 설계 원칙

- **최소 실물 + 최대 가상화**: 물리로 확보해야 하는 건 GPU와 KVM 호스트 정도. 네트워크 장비는 초기엔 containerlab, 물리는 Phase 2 이후.
- **벤더 단일화**: 네트워크/보안 모두 Juniper 계열로 통일 (Junos CLI가 스위치·라우터·SRX에서 공통이라 어댑터 재활용 이득).
- **관리 대상 다양성**: Linux + Windows + Juniper 3종은 Phase 1부터 어댑터 스켈레톤 확보.
- **증분 확장**: Phase 1은 4대 물리로 시작, 이후 필요 시 스위치·GPU 노드 추가.

## 2. 물리 하드웨어

| # | 역할 | 최소 사양 | 수량 | 비고 |
|---|---|---|---|---|
| G1 | LLM 호스팅 (Ollama) | A6000 48GB *1 or RTX 4090 24GB *2 / RAM 128GB / NVMe 1TB | 1 | qwen2.5:14b 기본, 32b Q4 fallback |
| G2 | 진단 대상 GPU 노드 | RTX 3090/4090 *1~2 / RAM 64GB | 1~2 | chaos 주입 대상 (gpu-burn, Xid 유발) |
| H1 | KVM/Proxmox 호스트 A | 24 vCPU / 128GB / SSD 2TB | 1 | 오케스트레이터 + 관측 VM 얹음 |
| H2 | KVM/Proxmox 호스트 B | 24 vCPU / 128GB / SSD 2TB | 1 | containerlab + 테스트 대상 VM 얹음 |

**합계 Phase 1: 물리 4~5대.**

## 3. 가상 머신 배치

### 3.1 KVM 호스트 A (H1)

| VM | OS | vCPU / RAM / Disk | 역할 |
|---|---|---|---|
| L1 | Ubuntu 22.04 | 8 / 32GB / 300GB | 오케스트레이터 (LangGraph 앱) + Chroma |
| L2 | Ubuntu 22.04 | 8 / 32GB / 500GB SSD | 관측 스택 (Prometheus, Alertmanager, Loki, Grafana) |
| W1 | Windows Server 2022 Std | 4 / 8GB / 100GB | 중개서버 (기존 관리 대상, WinRM 진단) |

### 3.2 KVM 호스트 B (H2)

| VM | OS | vCPU / RAM / Disk | 역할 |
|---|---|---|---|
| L3 | Ubuntu 22.04 | 16 / 64GB / 200GB | containerlab 호스트 (vJunos, vSRX, cRPD) |
| L4 | Ubuntu 22.04 | 4 / 8GB / 50GB | Apache 웹 (테스트 대상) |
| W2 | Windows Server 2022 Std | 4 / 8GB / 100GB | IIS 웹 (테스트 대상) |
| LB1 | Ubuntu 22.04 | 2 / 4GB / 20GB | HAProxy (Active) — Phase 2 |
| LB2 | Ubuntu 22.04 | 2 / 4GB / 20GB | HAProxy (Standby) — Phase 2 |

## 4. 네트워크 토폴로지

### 4.1 Phase 1 (containerlab)

```
                       ┌───────────────┐
                       │  L1  L2  W1   │  (호스트 A)
                       └───────┬───────┘
                               │ mgmt VLAN
                               │
┌──────────────────────────────┼──────────────────────────────┐
│                              │                              │
│   containerlab (on L3)                                      │
│                                                             │
│      [vJunos-switch: spine1]                                │
│              │      │                                       │
│      ┌───────┘      └───────┐                               │
│      │                      │                               │
│  [vJunos-switch: leaf1]  [vJunos-switch: leaf2]             │
│      │                      │                               │
│      ▼                      ▼                               │
│    L4 / W2                 G2 (GPU 대상)                    │
└─────────────────────────────────────────────────────────────┘
```

- spine 1 + leaf 2 미니 CLOS 구성
- vJunos-switch는 무료 (Juniper 공식 컨테이너, 2023~), QFX 계열 CLI 그대로 사용
- 물리 서버(G2)는 호스트 B의 브리지 인터페이스를 통해 containerlab MAC-VLAN에 붙임

### 4.2 Phase 2 확장 (L4 + FW)

```
   [Alertmanager] ──▶ [Orchestrator]
                            │
                            ▼
              ┌── vSRX (Firewall, Junos) ──┐
              │                            │
         [HAProxy Active] ─── [HAProxy Standby]
              │
     ┌────────┴────────┐
     ▼                 ▼
  [Apache / L4]     [IIS / W2]
```

- vSRX는 60일 eval, 이후 사내 SRX 재고나 라이선스 확보 여부에 따라 결정
- HAProxy는 keepalived 없이 시작 (풀 상태 조작만 시나리오화)

### 4.3 Phase 3 (물리 편입)

- 사내 EOL 재고 **EX3400 / EX4300** 급 스위치 1~2대 도입
- 물리 케이블·CRC·광 감쇠 등 하드웨어 유발 이슈 시나리오 검증
- G2 GPU 노드를 물리 스위치로 이관, containerlab는 상위 시뮬레이션 유지

## 5. 진단 어댑터 우선순위

| 어댑터 | 대상 | 라이브러리 | Phase |
|---|---|---|---|
| Linux SSH | L1~L4, G1, G2 | `asyncssh` | 1 |
| Juniper CLI | vJunos, 물리 EX | `netmiko` (device_type=`juniper_junos`) or `scrapli` | 1 |
| WinRM | W1, W2 | `pywinrm` + async wrapper | 1 |
| Prometheus API | 관측 | `httpx` | 1 |
| DCGM Exporter | G2 | Prometheus 경유 | 1 |
| Juniper SRX (vSRX) | Firewall | 위 Juniper 어댑터 재사용 | 2 |
| HAProxy stats/socket | LB1, LB2 | `httpx` + unix socket | 2 |
| F5 iControl REST | (옵션) | `httpx` | 3 |
| FortiGate/Palo REST | (옵션) | `httpx` | 3 |

## 6. Chaos 시나리오 (Phase 1 최소 세트)

| ID | 대상 | 유발 방법 | 기대 진단 결과 |
|---|---|---|---|
| CX-01 | G2 | `nvidia-smi -r` 실패 유도, gpu-burn 도중 킬 | GPU hang / Xid 감지 |
| CX-02 | L4/W2 | Apache/IIS 프로세스 kill | HTTP 5xx → 서비스 다운 판정 |
| CX-03 | vJunos leaf1 | `set interfaces ge-0/0/x disable` | 링크 다운 → 상단 경로 원인 판정 |
| CX-04 | vJunos spine1 | LLDP·MAC 테이블 비우기 (재기동) | 토폴로지 loss 감지 |
| CX-05 | W1 | WinRM 서비스 stop | 중개서버 접근 불가 감지 |
| CX-06 | LB1 (Phase 2) | 풀 멤버 강제 disable | LB 헬스체크 실패 판정 |

각 시나리오는 chaos 스크립트(`chaos/*.py`)로 재현 가능해야 한다.

## 7. 오픈 이슈

- **중개서버 W1의 역할 정의**: 단순 관리 대상인지, 에이전트가 최종 장비 접근 시 경유하는 jump host인지? 후자면 asyncssh `ProxyCommand` 로직 필요.
- **vJunos 라이선스**: vJunos-switch는 무료지만 **vMX / vSRX는 eval 60일**. Phase 2 이후 사내 라이선스 확보 여부 확인.
- **호스트 A/B 이중화**: 현재 단일 호스트 장애 시 오케스트레이터 자체가 다운. Phase 3~4에서 이중화 정책 결정.
- **네트워크 세그먼트**: mgmt / data / chaos 트래픽을 별도 VLAN으로 분리할지, 초기엔 단일 VLAN에서 시작할지.
- **Windows 진단 깊이**: WinRM PS Remoting 범위. AD 도메인 미참여 stand-alone 서버 가정으로 시작하는 게 안전.

## 8. 초기 세팅 체크리스트

- [ ] 물리 4대 확보 (G1, G2, H1, H2)
- [ ] H1/H2에 Proxmox VE 8.x 설치
- [ ] 인벤토리 YAML 초안 작성 (`inventory/testbed.yaml`)
- [ ] Ollama on G1 + qwen2.5:14b pull
- [ ] containerlab on L3 + vJunos-switch spine/leaf 토폴로지
- [ ] Prometheus + Alertmanager on L2, 최소 스크레이프 타겟 등록
- [ ] Linux/Windows/Juniper 어댑터 스켈레톤 3종
- [ ] Chaos 시나리오 CX-01 ~ CX-05 스크립트
