# Testbed

`self-heal-infra`의 PoC/개발용 테스트 환경 하드웨어·토폴로지 명세. 사내 프로덕션과 격리된 별도 인프라 위에서 구성한다.

## 1. 설계 원칙

- **최소 실물 + 최대 가상화**: 물리로 확보해야 하는 건 GPU와 호스트 물리 서버 정도. 네트워크 장비는 초기엔 containerlab, 물리는 Phase 2 이후.
- **네트워크 벤더 단일화 (보안은 별도)**: 네트워크(스위치·라우터)는 Juniper로 통일. 보안장비는 사내 표준인 **Fortinet(FortiGate)·SECUI(엑스게이트)** 계열이므로 별도 어댑터 트랙.
- **관리 대상 다양성**: Linux + Windows + Juniper + 보안(Fortinet/SECUI) 4종 어댑터 스켈레톤 확보.
- **증분 확장**: Phase 1은 4대 물리로 시작, 이후 필요 시 스위치·GPU 노드 추가.

## 2. 물리 하드웨어

| # | 역할 | 최소 사양 | 수량 | 비고 |
|---|---|---|---|---|
| G1 | LLM 호스팅 (Ollama) | A6000 48GB *1 or RTX 4090 24GB *2 / RAM 128GB / NVMe 1TB | 1 | qwen2.5:14b 기본, 32b Q4 fallback |
| G2 | 진단 대상 GPU 노드 | RTX 3090/4090 *1~2 / RAM 64GB | 1~2 | chaos 주입 대상 (gpu-burn, Xid 유발) |
| H1 | 호스트 물리 서버 A | 24 vCPU(코어) / RAM 96GB / SSD 2TB | 1 | 오케스트레이터 + 관측 VM 얹음 |
| H2 | 호스트 물리 서버 B | 32~36 vCPU(코어) / RAM 112GB / SSD 2TB | 1 | containerlab + 테스트 대상 VM 얹음 |

**합계 Phase 1: 물리 4~5대.**

> **2026-09-15 업데이트 (물리 통합):** H1/H2는 신규 물리 확보 없이 기존 `OpenStack-IDC` 서버(Rocky Linux 10.2,
> 32 vCPU / 93GB RAM, GPU 미장착, kolla-ansible OpenStack 운영 중)로 통합. 신규 물리 확보 대상은 **G1/G2(GPU
> 노드) 2대로 축소**. 상세 근거는 §2.1 참고.

### 2.1 H1/H2 최소 스펙 산출 근거

§3의 VM 배치표(vCPU/RAM/Disk 합) + 하이퍼바이저(Proxmox) 오버헤드(+2 vCPU/+8GB/+32GB) + 여유분(~15%, 스냅샷·버스트 대비)로 역산.

| 호스트 | VM 합계 | + 오버헤드/여유분 | 최소 권장 |
|---|---|---|---|
| H1 (LX1+LX2+W1) | 20 vCPU / 72GB / 900GB | +4 vCPU / +20GB / +182GB | **24 vCPU / 96GB / 1.1TB** |
| H2 (LX3+LX4+W2+LB1+LB2) | 28 vCPU / 88GB / 390GB | +6 vCPU / +22GB / +97GB | **34 vCPU / 110GB / ~500GB** |

H2는 VM vCPU 합(28)이 물리 24코어보다 많아 오버서브스크립션이 빡빡했음 — containerlab의 vJunos들은 대부분
유휴 상태라 어느 정도 오버서브는 괜찮지만, chaos 시나리오 도중 CPU 경합으로 진단이 왜곡될 위험이 있어 물리
코어를 32~36개로 올림. RAM/Disk는 여유가 있어서 기존 128GB/2TB 유지해도 무방 (표에는 최소값만 반영).

G1/G2(GPU 노드)는 H1/H2와 별개 물리라 이 산출에는 포함하지 않음.

**2026-09-15 업데이트:** 위 표의 vCPU/RAM/Disk 값은 이제 "신규 구매 사양"이 아니라 기존 `OpenStack-IDC`
서버(32 vCPU / 93GB RAM) 위에서 H1+H2 VM 전체를 함께 태울 때의 **합산 리소스 예산 체크**로 재해석한다.
H1+H2 합산 필요치(vCPU 20+28=48, RAM 72+88=160GB)가 실제 호스트 물리 스펙(32 vCPU/93GB)을 초과하므로,
Phase 1은 우선순위가 낮은 VM(LB1/LB2, 관측 스택 일부)을 축소 배치하거나 순차 기동으로 시작하고, 실사용량을
보며 조정한다.

## 2.1 하이퍼바이저 선정 (1차 결정: Proxmox VE → 2차 결정: OpenStack-IDC 재활용)

H1/H2에 올릴 하이퍼바이저로 OpenStack(Kolla-ansible) vs libvirt 직접 vs Proxmox VE를 검토함.
별도 물리 서버(OpenStack-IDC)에 Kolla-ansible로 All-in-One OpenStack을 직접 구축·운영해본 뒤 **OpenStack은 제외**하기로 결정.

**OpenStack을 뺀 이유 (오버스펙 판단):**
- 멀티테넌시(Keystone), self-service API, HA 컨트롤플레인(haproxy+keepalived+갈레라 클러스터) 등
  OpenStack의 핵심 가치를 이 testbed에서는 전혀 쓰지 않음 (VM 9대 고정 세트, 혼자 운영, 이중화 불필요)
- Neutron이 네트워킹(VXLAN 오버레이·보안그룹·플로팅IP)을 OVS로 직접 소유하려고 해서, **4.1의 "순수 브리지로
  containerlab MAC-VLAN에 붙인다"는 설계와 상충** — Neutron 추상화를 우회하거나 억지로 맞춰야 함
- 컨트롤플레인 자체가 컨테이너 38개, 유휴 상태에서도 RAM 14GB+ 소모 → "진단 대상 VM"보다 "VM을 관리하는
  시스템"이 더 무거워지는 역전이 생김
- VIP 충돌, 컨테이너 config diff 버그, Nova cell DB 동기화 등 이 testbed 목적과 무관한 트러블슈팅 비용이 계속 발생

**Proxmox VE를 선택한 이유:**
- 설치~운영이 웹 UI 하나로 완결 (Horizon+Keystone+Neutron+Cinder처럼 여러 서비스를 조합해서 이해할 필요 없음)
- 기본 네트워킹이 순수 리눅스 브리지(`vmbr0`) — containerlab에 붙이는 구조(4.1)와 바로 맞음
- 스냅샷/클론 내장 — chaos 시나리오(6번) 실행 후 VM 상태 원복이 UI 클릭 몇 번으로 가능
- GPU 패스스루(G2)는 IOMMU/VFIO 설정이 필요하지만, 홈랩 커뮤니티 자료가 많고 OpenStack Nova의 PCI
  whitelist 설정보다 단순함
- H1/H2 각각 단일 노드로 시작. 클러스터링/라이브마이그레이션은 필요해지면(§7 "호스트 A/B 이중화") 이후 검토

## 2.2 하이퍼바이저 결정 번복 (2026-09-15: OpenStack-IDC 재활용)

**1차 결정(위 §2.1, 2026-08-10)을 번복하고 H1/H2를 신규 Proxmox 물리 대신 기존 `OpenStack-IDC` 서버로
통합한다.** 이 서버는 개인 OpenStack 학습용으로 이미 구축·운영 중이던 kolla-ansible All-in-One 배포다
([[project-openstack-idc-server]]).

**번복 이유: 물리 서버 통합/재활용.** H1/H2용 물리 2대를 새로 마련하는 대신, 이미 32 vCPU / 93GB RAM
(여유 대부분)이 확보된 기존 서버 하나로 self-heal-infra 테스트베드까지 같이 굴린다. `terraform-openstack`
(provision) + Ansible(configure) 파이프라인이 이미 검증되어 있어 VM 생성/재현이 바로 재사용 가능.

**1차 결정 당시 기각 사유가 여전히 유효한 부분과, 이번에 감수하기로 한 트레이드오프:**

| 1차 결정의 기각 사유 | 이번 재검토 결과 |
|---|---|
| 멀티테넌시/self-service API/HA 컨트롤플레인 불필요 | 여전히 안 씀 — 하지만 오버스펙이어도 "이미 떠 있는" 비용은 0이므로 감수 |
| 컨트롤플레인 자체가 무겁다 (컨테이너 38개, RAM 14GB+) | 그대로 유효한 단점. 다만 호스트 RAM 93GB 중 여유가 크므로 실질 영향은 적음 |
| Neutron이 네트워킹을 OVS로 직접 소유 → containerlab MAC-VLAN 설계와 상충 | **완전히 해소되진 않음.** 아래 새 네트워크 설계로 우회 (§4.1 갱신) |
| VIP 충돌, Nova cell DB 동기화 등 트러블슈팅 비용 | 그대로 유효한 리스크. `install-guide.md`에 이미 문서화된 기지 이슈 다수라 대응 비용은 낮아짐 |

**Nova/Neutron API 자체가 진단/원격조치 어댑터의 실습 대상이 되는 부수 이득**도 있음 — Proxmox API보다
공식 SDK(OpenStack SDK/python-novaclient)가 성숙해 있어 Autonomous Action 계층 학습에 오히려 유리.

**GPU(G1) 관련 제약:** 이 서버엔 GPU가 물리적으로 없음(`lspci`/sysfs 확인, 온보드 mgag200 콘솔카드뿐).
Proxmox든 OpenStack이든 GPU 미장착이면 동일하게 막히는 문제라 하이퍼바이저 선택과 무관. 당장은 G1(Ollama)을
원격 LLM API 또는 CPU 추론으로 대체하고, GPU 확보 시 Nova PCI passthrough(whitelist 설정) 별도 검토.

**네트워크 재설계 (containerlab ↔ Neutron 경계, §4.1 diagram 갱신 근거):**
Neutron의 ml2/OVS 설정은 건드리지 않는다. 대신:
- Neutron은 순수 서버 세그먼트(`shared-net`, 192.168.50.0/24, 외부 게이트웨이 없음)만 담당.
- containerlab(vJunos spine/leaf)은 Neutron과 무관한 별도 Docker 네임스페이스로 독립 실행.
- 두 계층은 **L3 경계로만 연결**: `shared-net`의 Neutron 라우터가 갖는 `qrouter-<id>` netns에 veth pair를
  꽂아 containerlab 엣지(leaf) 스위치와 잇는다. (`ip netns exec qrouter-<id> ...` 접근 패턴은
  [[project-terraform-openstack]]에서 이미 SSH/Ansible 용도로 검증된 방식과 동일한 원리.)
- **트레이드오프:** ARP/브로드캐스트 스톰 같은 L2 인접성이 필요한 장애 주입 시나리오는 이 구조로 재현
  불가능하다. 필요해지면 그때 Neutron mechanism driver를 linuxbridge로 바꾸는 옵션(위험도 높음 — 운영 중인
  OVS 기반 네트워크 전체에 영향)을 재검토한다.

**§3(VM 배치)의 H1/H2 물리 서버 구분은 이제 물리적 구분이 아니라 논리적 그룹(Nova 인스턴스 그룹)으로
읽는다** — 모든 VM이 같은 물리 호스트 위 Nova 인스턴스로 올라간다.

## 3. 가상 머신 배치

### 3.1 호스트 물리 서버 A (H1)

| VM | OS | vCPU / RAM / Disk | 역할 |
|---|---|---|---|
| LX1 | Ubuntu 22.04 | 8 / 32GB / 300GB | 오케스트레이터 (LangGraph 앱) + Chroma |
| LX2 | Ubuntu 22.04 | 8 / 32GB / 500GB SSD | 관측 스택 (Prometheus, Alertmanager, Loki, Grafana) |
| W1 | Windows Server 2022 Std | 4 / 8GB / 100GB | 중개서버 (기존 관리 대상, WinRM 진단) |

### 3.2 호스트 물리 서버 B (H2)

| VM | OS | vCPU / RAM / Disk | 역할 |
|---|---|---|---|
| LX3 | Ubuntu 22.04 | 16 / 64GB / 200GB | containerlab 호스트 (vJunos, vSRX, cRPD) |
| LX4 | Ubuntu 22.04 | 4 / 8GB / 50GB | Apache 웹 (테스트 대상) |
| W2 | Windows Server 2022 Std | 4 / 8GB / 100GB | IIS 웹 (테스트 대상) |
| LB1 | Ubuntu 22.04 | 2 / 4GB / 20GB | HAProxy (Active) — Phase 2 |
| LB2 | Ubuntu 22.04 | 2 / 4GB / 20GB | HAProxy (Standby) — Phase 2 |

## 4. 네트워크 토폴로지

### 4.1 Phase 1 (containerlab)

```
                       ┌───────────────┐
                       │  LX1  LX2  W1   │  (호스트 A)
                       └───────┬───────┘
                               │ mgmt VLAN
                               │
┌──────────────────────────────┼──────────────────────────────┐
│                              │                              │
│   containerlab (on LX3)                                      │
│                                                             │
│      [vJunos-switch: spine1]                                │
│              │      │                                       │
│      ┌───────┘      └───────┐                               │
│      │                      │                               │
│  [vJunos-switch: leaf1]  [vJunos-switch: leaf2]             │
│      │                      │                               │
│      ▼                      ▼                               │
│    LX4 / W2                 G2 (GPU 대상)                    │
└─────────────────────────────────────────────────────────────┘
```

- spine 1 + leaf 2 미니 CLOS 구성
- vJunos-switch는 무료 (Juniper 공식 컨테이너, 2023~), QFX 계열 CLI 그대로 사용
- 물리 서버(G2)는 호스트 B의 브리지 인터페이스를 통해 containerlab MAC-VLAN에 붙임
- **2026-09-15 갱신:** 위 다이어그램은 Proxmox 물리 브리지 전제. OpenStack-IDC 재활용 후에는 LX1/LX2/W1/LX4/W2/LB1/LB2가
  전부 Neutron `shared-net` 위 Nova 인스턴스이고, containerlab(vJunos spine/leaf, LX3에 해당하는 역할)만
  별도 Docker 네임스페이스로 떠서 `shared-net`의 `qrouter` netns와 veth로 L3 연결된다 (근거: §2.2).

### 4.2 Phase 2 확장 (로드밸런서 + 방화벽)

```
   [Alertmanager] ──▶ [Orchestrator]
                            │
                            ▼
              ┌── FortiGate VM (or SECUI) ──┐
              │                             │
         [HAProxy Active] ─── [HAProxy Standby]
              │
     ┌────────┴────────┐
     ▼                 ▼
  [Apache / LX4]     [IIS / W2]
```

- **FortiGate VM eval** (FortiGate-VM64, 15일 평가판)으로 방화벽 계층 시작. 이후 사내 라이선스 or EOL 재고로 전환.
- **SECUI 엑스게이트**는 벤더 특성상 VM 배포가 제한적. 사내 재고 실물 확보 시 Phase 3에서 편입.
- HAProxy는 keepalived 없이 시작 (풀 상태 조작만 시나리오화)

### 4.3 Phase 3 (물리 편입)

- 사내 EOL 재고 **EX3400 / EX4300** 급 스위치 1~2대 도입
- 물리 케이블·CRC·광 감쇠 등 하드웨어 유발 이슈 시나리오 검증
- G2 GPU 노드를 물리 스위치로 이관, containerlab는 상위 시뮬레이션 유지

## 5. 진단 어댑터 우선순위

| 어댑터 | 대상 | 라이브러리 | Phase |
|---|---|---|---|
| Linux SSH | LX1~LX4, G1, G2 | `asyncssh` | 1 |
| Juniper CLI | vJunos, 물리 EX | `netmiko` (device_type=`juniper_junos`) or `scrapli` | 1 |
| WinRM | W1, W2 | `pywinrm` + async wrapper | 1 |
| Prometheus API | 관측 | `httpx` | 1 |
| DCGM Exporter | G2 | Prometheus 경유 | 1 |
| FortiGate REST | FortiGate VM / 물리 | `fortiosapi` or `httpx` (FortiOS REST) | 2 |
| HAProxy stats/socket | LB1, LB2 | `httpx` + unix socket | 2 |
| SECUI 엑스게이트 | 사내 재고 편입 시 | SSH CLI + syslog 파싱 (공식 라이브러리 없음, 자체 어댑터) | 3 |
| F5 iControl REST | (옵션) | `httpx` | 3 |

## 6. Chaos 시나리오 (Phase 1 최소 세트)

| ID | 대상 | 유발 방법 | 기대 진단 결과 |
|---|---|---|---|
| CX-01 | G2 | `nvidia-smi -r` 실패 유도, gpu-burn 도중 킬 | GPU hang / Xid 감지 |
| CX-02 | LX4/W2 | Apache/IIS 프로세스 kill | HTTP 5xx → 서비스 다운 판정 |
| CX-03 | vJunos leaf1 | `set interfaces ge-0/0/x disable` | 링크 다운 → 상단 경로 원인 판정 |
| CX-04 | vJunos spine1 | LLDP·MAC 테이블 비우기 (재기동) | 토폴로지 loss 감지 |
| CX-05 | W1 | WinRM 서비스 stop | 중개서버 접근 불가 감지 |
| CX-06 | LB1 (Phase 2) | 풀 멤버 강제 disable | LB 헬스체크 실패 판정 |

각 시나리오는 chaos 스크립트(`chaos/*.py`)로 재현 가능해야 한다.

## 7. 오픈 이슈

- **중개서버 W1의 역할 정의**: 단순 관리 대상인지, 에이전트가 최종 장비 접근 시 경유하는 jump host인지? 후자면 asyncssh `ProxyCommand` 로직 필요.
- **vJunos 라이선스**: vJunos-switch는 무료지만 **vMX는 eval 60일**. Phase 2 이후 사내 라이선스 확보 여부 확인.
- **FortiGate VM 라이선스**: 평가판 15일. 개발 사이클이 길어지면 사내 라이선스 확보 or 물리 편입 필요.
- **SECUI 어댑터 개발 리소스**: 국내 벤더 특성상 공식 SDK/파이썬 라이브러리 부재. CLI/syslog 기반 자체 어댑터 개발 공수 산정 필요.
- **호스트 A/B 이중화 → 단일 물리 서버 SPOF**: 2026-09-15 OpenStack-IDC 통합 이후 H1/H2 구분이 사라지고
  전부 같은 물리 서버 위에 올라가므로, 이 서버 하나가 죽으면 오케스트레이터+테스트 대상 전체가 동시에 다운된다.
  기존 "호스트 A/B 이중화" 논의보다 더 근본적인 SPOF. Phase 3~4에서 별도 물리 확보 여부 재검토.
- **네트워크 세그먼트**: mgmt / data / chaos 트래픽을 별도 VLAN으로 분리할지, 초기엔 단일 VLAN에서 시작할지.
- **Windows 진단 깊이**: WinRM PS Remoting 범위. AD 도메인 미참여 stand-alone 서버 가정으로 시작하는 게 안전.

## 8. 초기 세팅 체크리스트

- [ ] 물리 4대 확보 (G1, G2, H1, H2)
- [x] H1/H2 하이퍼바이저 선정 — ~~Proxmox VE~~ → **OpenStack-IDC 서버 재활용** (근거: §2.2, 2026-09-15 번복)
- [x] H1/H2에 Proxmox VE 설치 — ~~불필요 (OpenStack-IDC 재활용으로 대체)~~
- [ ] 인벤토리 YAML 초안 작성 (`inventory/testbed.yaml`)
- [ ] Ollama on G1 + qwen2.5:14b pull
- [ ] containerlab on LX3 + vJunos-switch spine/leaf 토폴로지
- [ ] Prometheus + Alertmanager on LX2, 최소 스크레이프 타겟 등록
- [ ] Linux/Windows/Juniper 어댑터 스켈레톤 3종 (보안 어댑터는 Phase 2)
- [ ] Chaos 시나리오 CX-01 ~ CX-05 스크립트
