# AI/HPC 데이터센터 스케줄링 엔지니어 로드맵

> **Primary 트랙 (R&D)**: 서울과기대 김성곤 랩 (안전망) + KAIST 한동수 랩 (stretch) → 네이버 세종 / NHN AI 인프라
> **Hedge 트랙 (SRE/플랫폼)**: 가을 인턴 사이클 → 스마일게이트 DevOps / 카카오페이 SRE 등 → 네이버 클라우드 경력 이직
> **결정 시점**: 4학년 1학기 끝 (2027년 6월경)

---

## 전체 전략 구조

```
[Project 1] 연구실 입실용 포트폴리오                  [완료, Prometheus 연동만 남음]
    치지직 스타일 BW-aware 커스텀 스케줄러
    + 벤치마킹 (BW 80% 노드 배치 0개 달성)
         ↓
[Phase 0.5] 컨택 자격 + Prometheus 연동 마무리        [이번 방학: 6~7월]
    AURORA-Q + StellaTrain 정독 + Prometheus 자동 연동
         ↓
[교수님 컨택 — 양 트랙 동시]                          [2026년 7월 말 deadline]
    김성곤 (안전망) + 한동수 (stretch)
         ↓
[Phase 1] 운영 레이어 + 가을 사이클 진입              [2026년 8~11월]
    SRE 트랙 hedge + 인턴십·신입공채 지원
         ↓
[Phase 2] 데이터 수집·평가                             [2026년 12월~2027년 5월]
    랩 노출 결과 + 인턴 결과 + ETRI 후기
         ↓
[Decision Point] 결정·베팅                             [2027년 6월]
    대학원 트랙 (R&D) vs 인턴전환/신입공채 트랙 (SRE)
         ↓
[Project 2 / 또는 첫 회사]
```

---

# PROJECT 1 — 연구실 입실용 포트폴리오

## 구현 완료 항목

| 항목 | 상태 |
|------|------|
| kind/kubeadm 클러스터 구성 | ✅ |
| kube-scheduler 소스 분석 (schedule_one.go) | ✅ |
| BandwidthScore 플러그인 | ✅ |
| TranscoderFilter 플러그인 | ✅ |
| StreamAffinity 플러그인 | ✅ |
| PredictiveEnqueue 플러그인 | ✅ |
| 단위 테스트 (tf.NewFramework 패턴) | ✅ |
| Docker 이미지 빌드 + 배포 | ✅ |
| 벤치마킹 (기본 vs chijik-scheduler) | ✅ |
| GitHub README 벤치마크 결과 포함 | ✅ |
| **Prometheus 자동 연동** | ⏳ 컨택 전 마무리 |

## 벤치마킹 결과

| 지표 | 기본 스케줄러 | chijik-scheduler |
|------|------------|----------------|
| BW 80% 노드 배치 Pod | 5개 | **0개** |
| BW 30% 노드 배치 Pod | 5개 | **10개** |
| BW 인식 여부 | ❌ | ✅ |

## 컨택 전 마무리: Prometheus 자동 연동

- 어노테이션 기반 → 실제 Prometheus 메트릭 기반으로 전환
- ServiceMonitor 붙이고 메트릭 export
- Grafana 대시보드 1개 (before/after 시계열 그래프)
- README에 "schedulable on actual production metrics" 한 줄 추가
- **이게 컨택 메일 신뢰도의 마지막 1%**

## 남은 한계 (Project 2 시작점)

- 단일 리소스(BW)만 고려 (GPU, NUMA 미고려) → Project 2의 본 과제

---

# PHASE 0.5 — 컨택 자격 확보 (2026년 6~7월)

## Reading Agenda

**진행 중 (도메인 줄기)**:
- EcoSched (Towards Energy Efficient Co-scheduling in HPC, arxiv 2604.17640)
- Slurm vs Kubernetes (SC23 슬라이드, Tim Wickberg)
- Kueue 소스코드

**추가 (랩 컨택 자격용)**:
- AURORA-Q (김성곤 교수, IEEE ICDCS 2026) — 김성곤 컨택용
- StellaTrain (한동수 교수, 2024) — 한동수 컨택용
- (옵션) ES-MoE (ICML 2024) / SpecEdge (2025)

## 양 랩 컨택 메일 narrative

```
EcoSched (score-based GPU co-scheduling)
   ↓ 일반화
chijik-scheduler (score-based BW-aware K8s plugin)
   ↓ 분기
김성곤: AURORA-Q의 양자 시뮬레이션 HPC 스케줄링과 연결
한동수: StellaTrain의 bandwidth-limited 분산 학습과 연결
```

## 자체 deadline

```
6월 말: Prometheus 연동 완료 + repo·README 정리
7월 중순: AURORA-Q + StellaTrain 정독 완료
7월 말: 컨택 메일 2통 발송   ← 이 deadline이 핵심
```

**컨택 메일이 미뤄지면 그 이후 모든 액션은 supporting role임. 빌드는 보조, 컨택이 본진.**

---

# PROJECT 2 — GPU Topology-aware 스케줄러 [연구실]

> **목표:** 교수님 GPU 클러스터에서 실제 성능 문제 해결
> **기간:** 1년+ (연구실, 교수님과 함께)

## 왜 GPU Topology인가

```
교수님 연구:
  대용량 GPU 클러스터 + 양자 시뮬레이션 (김성곤) 또는 분산 학습 (한동수)
  → GPU 간 통신 대역폭이 성능을 좌우

현재 k8s 스케줄러의 문제:
  "GPU 2개 줘" → 어떤 GPU 2개든 상관없이 배정
  → NUMA 노드가 쪼개지면 성능 반토막
  → NVLink로 연결된 GPU 쌍을 우선 배정해야 함
```

## 해결할 문제

### 문제 1: NUMA Split

```
GPU 0, 1 → NUMA 0 (NVLink 연결, 600 GB/s)
GPU 2, 3 → NUMA 1 (NVLink 연결, 600 GB/s)

잘못된 배정: GPU 0 (NUMA 0) + GPU 2 (NUMA 1)
  → NUMA 간 통신: PCIe 경유 (~32 GB/s)
  → 성능 반토막

올바른 배정: GPU 0 + GPU 1 (같은 NUMA, NVLink)
  → 600 GB/s 풀 대역폭 활용
```

### 문제 2: Thundering Herd (Prometheus Time Lag)

```
Prometheus 수집 주기: 10~30초
스케줄러 배치 속도: 초당 수십 개

시차 동안 같은 노드에 Pod 몰림 현상 발생
→ Optimistic Reservation으로 해결
   (스케줄링 결정 즉시 내부 캐시 업데이트)
```

## Phase A — 학습 목록

### k8s 생태계

| 기술 | 설명 | 우선순위 |
|------|------|---------|
| kueue | Job 큐잉 + Gang Scheduling | 높음 |
| volcano | HPC/AI 워크로드 스케줄러 | 높음 |
| karmada | 멀티 클러스터 관리 | 중간 |
| NVIDIA NFD | 노드 GPU 토폴로지 라벨링 | 높음 |
| Backend.AI | AI 워크로드 서빙 플랫폼 | 중간 |

### 하드웨어 개념

| 기술 | 설명 |
|------|------|
| NUMA | Non-Uniform Memory Access. CPU별 메모리 접근 속도 차이 |
| NVLink | GPU 간 고속 연결 (600 GB/s). PCIe보다 ~18배 빠름 |
| NVSwitch | 여러 GPU를 NVLink로 연결하는 스위치 |
| GPU Topology | GPU 간 연결 구조 (NVLink, PCIe, NVSwitch) |

### Kubernetes 내부

```
Topology Manager: CPU/Memory/GPU 할당 위치 정렬
Pod Topology Spread Constraints: 노드/존 간 균등 배포
Device Plugin: GPU 리소스를 k8s에 노출 (nvidia.com/gpu)
NVIDIA NFD: 노드에 GPU 토폴로지 라벨 자동 등록
```

## Phase B — GPU Topology-aware 플러그인 설계

```go
// NFD가 등록한 노드 라벨 예시
labels:
  nvidia.com/gpu.count: "4"
  nvidia.com/gpu.memory: "80Gi"
  nvidia.com/gpu.product: "A100-SXM4-80GB"
  nvidia.com/gpu.topology.nvlink: "0-1,2-3"  // NVLink 쌍
  numa.topology.kubernetes.io/numa-nodes: "0,1"

// 요청된 GPU 개수가 단일 NUMA + NVLink로 충족 가능한지 검사
func (pl *GPUTopologyPlugin) Score(...) (int64, *framework.Status) {
    requestedGPU := getRequestedGPU(pod)
    nvlinkPairs := parseNVLinkTopology(node)

    // 단일 NUMA 내 NVLink 쌍으로 충족 가능 → 100점
    if canSatisfyWithNVLink(requestedGPU, nvlinkPairs) {
        return framework.MaxNodeScore, nil
    }
    // NUMA 쪼개짐 발생 → 0점
    return 0, nil
}
```

### Optimistic Reservation 캐시

```go
type GPUTopologyPlugin struct {
    handle    framework.Handle
    promClient prometheus.Client

    mu       sync.RWMutex
    reserved map[string]float64  // nodeName → 가상 GPU 사용률
}

func (pl *GPUTopologyPlugin) Reserve(...) *framework.Status {
    pl.mu.Lock()
    pl.reserved[nodeName] += getGPUUsage(pod)
    pl.mu.Unlock()
    return nil
}

func (pl *GPUTopologyPlugin) Unreserve(...) {
    pl.mu.Lock()
    pl.reserved[nodeName] -= getGPUUsage(pod)
    pl.mu.Unlock()
}
```

## Phase C — 논문 작성

```
Abstract:
  GPU 클러스터에서 NUMA Split과 Thundering Herd 문제
  → NFD 라벨 기반 Topology-aware 스케줄러로 해결
  → GPU 간 통신 대역폭 X% 향상, 학습 시간 Y% 단축

Contribution:
  1. Analysis: NUMA Split이 GPU 클러스터 성능에 미치는 영향 분석
  2. Design: NFD + Optimistic Reservation 기반 스케줄러 설계
  3. Evaluation: 양자 시뮬레이션 / 분산 학습 워크로드 기반 실측 비교

타겟 컨퍼런스:
  OSDI / SOSP (OS 최상위)
  EuroSys
  SC (Supercomputing)
  NSDI (네트워크 시스템 — 한동수 랩일 경우)
```

---

# 운영 레이어 — SRE 트랙 Hedge (병행)

> **목적**: R&D 트랙 결과 미지근할 시 인턴/신입공채 SRE 트랙으로 전환 가능하도록 무기 확보
> **시기**: 2026년 8~10월 (Phase 0.5 끝난 직후부터 가을 사이클까지)
> **원칙**: 메인 트랙 잠식 금지. 빌드는 최소, 부산물(블로그) 최대.

## 액션

| # | 액션 | 시기 | 우선순위 |
|---|------|------|---------|
| 1 | Navix 홈서버 + Prometheus + Grafana + Loki + Alertmanager 풀스택 (chijik Prometheus 연동과 합쳐서 일타이피) | 7~8월 | 높음 |
| 2 | K8s 클러스터 장애 시뮬레이션 (노드 죽이기, etcd 백업/복구) | 8월 | 중간 |
| 3 | Terraform NCP 한 번 굴려보기 | 8월 | 낮음 |
| 4 | ArgoCD GitOps 한 사이클 | 8월 | 낮음 |
| 5 | Chaos Mesh로 의도적 장애 주입 + 블로그 1편 | 9월 | 높음 |
| 6 | Volcano good-first-issue PR 1건 머지 | 병렬 | 중간 |

## 블로그 — 부산물 원칙

위 액션과 reading의 자연스러운 부산물로 글이 떨어지게:

**R&D 트랙 강함**:
- chijik 스케줄러 회고 (디자인·trade-off)
- EcoSched 논문 리뷰 + chijik 연결
- AURORA-Q 리뷰
- StellaTrain 리뷰
- Slurm vs K8s 자원 모델 비교

**SRE 트랙 강함**:
- Navix 홈서버 SLO 만들기
- Chaos Mesh 장애 주입 후기
- Volcano PR 머지 후기

**양 트랙 fit**:
- 컨테이너 직접 구현기 회고

- **플랫폼**: Velog 또는 GitHub Pages + Hugo
- **빈도**: 분기 1~2편 (깊이 있는 1편 > 얕은 10편)
- **금지**: 컨택 메일 미루는 도구로 쓰지 말 것

---

# 가을 사이클 — 인턴십 + 신입공채 (2026년 9~11월)

> **시점**: 3학년 2학기. 2026 동계 인턴십 + 2027 신입공채 동시 모집

## 지원 타겟

| 트랙 | 지원처 | 모집 시점 |
|------|--------|---------|
| 동계 인턴십 (2026.12~2027.2) | 네이버랩스·카카오·토스·카카오페이·스마일게이트 DevOps·라인플러스·쿠팡 | 9~11월 |
| 신입공채 (2027 입사) | 카카오 신입크루 Tech 공통·네이버·라인 신입 개발자 | 9~11월 |
| 학부생 연구참여 | 김성곤 랩 (Phase 0.5 컨택 성공시) | 9월~ |

## 자소서 양면 narrative

같은 chijik 프로젝트를 두 각도로:
- **R&D 각도**: "score-based scheduling 알고리즘 연구·구현"
- **SRE 각도**: "프로덕션 K8s 클러스터에 스케줄러 plugin 적용·운영"

각 회사·직무에 맞춰 angle 조정.

---

# Decision Point — 2027년 6월 (4학년 1학기 끝)

지금까지 수집한 데이터를 종합해서 한 트랙에 베팅.

## 시나리오 결정 매트릭스

| ETRI + 랩 노출 결과 | 인턴/신입공채 결과 | → 결정 |
|---|---|---|
| 연구 재미·적성 확인 | 어디든 | **대학원 트랙** → Project 2로 직진 |
| 연구 매력 약함 | 인턴 전환 성공 | **인턴 전환 트랙** |
| 연구 매력 약함 | 신입공채 합격 | **신입공채 트랙** (Tech 공통 → 사내 인프라 이동) |
| 둘 다 미지근함 | 둘 다 미지근함 | 휴학·재정비 / 미국 석사 자비 검토 |

## 대학원 원서 시점

- 2027년 봄(3월) 입학: 이미 늦음 (3학년 종료 시점에서)
- 2027년 가을(9월) 입학: 2027년 5~7월 원서 ← Decision Point와 맞음
- 2028년 봄(3월) 입학: 2027년 9~11월 원서

**3학년 → 4학년 → 졸업 (2028년 2월) 가정 시, 2027년 5~7월 원서 (가을 입학)이 자연스러움.**

---

# 컨택 메일 템플릿

## 김성곤 교수 (서울과기대 Bigdata & HPC Lab)

```
제목: GPU 클러스터 스케줄링 최적화 관련 연구 참여 문의

1. 자기소개
   Go 시스템 프로그래머, 서울과기대 CS 3학년 OOO입니다.

2. 만든 것
   치지직 스타일 BW-aware 커스텀 K8s 스케줄러 (chijik-scheduler)
   kube-scheduler 소스 분석 + 플러그인 4개 구현 + Prometheus 연동 + 벤치마킹
   GitHub: github.com/Joseng8908/chijik-scheduler

3. 결과
   BW 80% 노드 배치 0개 달성 (기본 스케줄러: 5개)
   Grafana 대시보드로 before/after 시계열 검증

4. 교수님 연구와의 연결
   AURORA-Q에서 다룬 양자 시뮬레이션의 HPC 자원 스케줄링 문제와
   제가 구현한 K8s 스케줄러 plugin 구조가 같은 줄기라고 생각합니다.
   특히 NUMA Split, Thundering Herd 문제를
   NFD + Optimistic Reservation으로 해결하는 방향에 관심있습니다.

5. 미팅 요청
   교수님 일정 가능하시면 30분 미팅 부탁드립니다.
```

## 한동수 교수 (KAIST EE, INA Lab)

```
제목: GPU 클러스터 스케줄링 관련 연구 참여 문의

1. 자기소개
   Go 시스템 프로그래머, 서울과기대 CS 3학년 OOO입니다.
   분산 시스템 자원 배치·스케줄링에 관심 있어 메일드립니다.

2. 만든 것
   치지직 스타일 BW-aware 커스텀 K8s 스케줄러 (chijik-scheduler)
   - kube-scheduler 소스 분석 + score-based 플러그인 구현
   - Prometheus 자동 연동 + 벤치마킹
   GitHub: github.com/Joseng8908/chijik-scheduler

3. 결과
   네트워크 대역폭이 자원 제약일 때, BW 80% 노드 배치 0개 달성

4. 교수님 연구와의 연결
   교수님 StellaTrain 논문에서 다룬 "bandwidth-limited 환경에서의
   분산 학습 가속" 문제 의식이 제가 transcoding 워크로드에 적용한
   score-based scheduling 접근과 같은 줄기라고 생각합니다.
   이를 GPU 분산 학습 워크로드로 일반화하는 게 다음 관심사입니다.

5. 미팅 요청
   교수님 일정 가능하시면 온라인 미팅 부탁드립니다.
```

---

# 취업 타겟

```
AI/HPC 데이터센터 스케줄링 엔지니어
= "GPU 클러스터 위에서 돌아가는 K8s를 만드는 사람"
```

## 회사별 연결

| 회사 | 연결 포인트 | 진입 경로 |
|------|-----------|---------|
| 네이버 세종 데이터센터 | HyperCLOVA 학습 인프라, GPU 클러스터 운영 | 대학원 (R&D 직진) |
| NHN Cloud | AI 인프라 플랫폼 | 대학원 또는 신입공채 |
| ETRI | AI 학습/추론 스케줄링 연구 (지원 중) | 학부 인턴 (현재) |
| 네이버 1784 R&D | 네이버랩스, 네이버 클라우드 R&D | 대학원 (KAIST 한동수 랩 경유 시 강함) |
| 카카오 / 카카오페이 / 토스 | SRE / 플랫폼 인프라 | SRE 트랙 경력 점프 |
| 스마일게이트 홀딩스 | DevOps, SE 신입 채용 트랙 | SRE 트랙 진입 |

## 희소성

```
일반 백엔드 엔지니어:
  지원자 수백 명

이 포지션:
  GPU Topology를 이해하고 스케줄러를 만드는 사람
  → 국내에 손에 꼽힘
  → 경쟁률 낮음, 연봉 높음
```

---

# 전체 타임라인

```
━━━━━━━━━━ PROJECT 1 (완료) ━━━━━━━━━━
✅ chijik-scheduler 개발 + 벤치마킹
✅ ETRI 인턴 지원
✅ 홈서버 구축 (NAVIX + kubeadm + Cilium)

━━━━━━━━━━ PHASE 0.5 (6~7월, 이번 방학) ━━━━━━━━━━
6월 말 : Prometheus 자동 연동 완료
        EcoSched / Slurm vs K8s / Kueue 소스 정독 완료
7월 중 : AURORA-Q + StellaTrain 정독 완료
7월 말 : 김성곤 + 한동수 컨택 메일 2통 발송  ← deadline

━━━━━━━━━━ PHASE 1 (8~11월, 가을) ━━━━━━━━━━
8월    : 운영 레이어 구축 (Prometheus/Grafana/Loki/Alertmanager)
9월    : Chaos Engineering + 블로그 1편
9~11월 : 인턴십·신입공채 가을 사이클 지원
        랩 학부생 연구참여 (컨택 성공시)
병렬   : Volcano PR 머지 시도

━━━━━━━━━━ PHASE 2 (2026.12~2027.5) ━━━━━━━━━━
동계 인턴 (지원 성공시)
4학년 진입
ETRI 인턴 종료·평가
데이터 수집 종료

━━━━━━━━━━ DECISION POINT (2027.6) ━━━━━━━━━━
결정: 대학원 / 인턴전환 / 신입공채 / 재정비

━━━━━━━━━━ PROJECT 2 (석사 입학 후, 1년+) ━━━━━━━━━━
GPU Topology-aware 스케줄러 + 논문

━━━━━━━━━━ 취업 ━━━━━━━━━━
석사 졸업 (2029~2030) → 네이버 세종 / NHN Cloud / 네이버 1784 R&D
또는
신입 (2028) → 스마일게이트/카카오페이/토스 → 5~7년 후 네이버 클라우드
```

---

# 레퍼런스

**k8s 생태계**
- [kueue](https://github.com/kubernetes-sigs/kueue)
- [volcano](https://github.com/volcano-sh/volcano)
- [karmada](https://github.com/karmada-io/karmada)
- [NVIDIA NFD](https://github.com/NVIDIA/gpu-feature-discovery)
- [Topology Manager](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)

**하드웨어**
- [NVLink Architecture](https://developer.nvidia.com/nvlink)
- [NUMA on Linux](https://www.kernel.org/doc/html/latest/vm/numa.html)

**김성곤 랩 (서울과기대 Bigdata & HPC)**
- AURORA-Q: Asynchronous Unified Resource Optimizer for Quantum Simulation on HPC System (IEEE ICDCS 2026)

**한동수 랩 (KAIST EE, INA Lab)**
- Homepage: http://ina.kaist.ac.kr
- StellaTrain (2024) — bandwidth-limited 분산 학습 가속
- ES-MoE (ICML 2024) — MoE 모델 학습 GPU 메모리 효율화
- SpecEdge (2025) — 데이터센터 + 엣지 GPU 협업 LLM serving

**스케줄링 일반**
- Gandiva: Introspective Cluster Scheduling for Deep Learning (OSDI 2018)
- Tiresias: A GPU Cluster Manager for Distributed Deep Learning (NSDI 2019)
- AntMan: Dynamic Scaling on GPU Clusters for Deep Learning (OSDI 2020)
- EcoSched / Towards Energy Efficient Co-scheduling in HPC (arxiv 2604.17640)
- Slurm and/or vs Kubernetes (SC23, Tim Wickberg)

**HPC + Cloud-native**
- Slurm: https://slurm.schedmd.com/

---

# 자기 점검 룰

매주 이 로드맵 다시 볼 때 체크:

1. **컨택 메일 deadline (7월 말) 지키고 있나?**
   안 지키고 있으면 그게 0순위 알람. 다른 거 다 멈추고 컨택부터.

2. **운영 레이어 / 블로그가 컨택 메일보다 먼저 가고 있나?**
   그러면 회피 신호. 우선순위 다시 잡기.

3. **로드맵 다듬는 시간 > 로드맵 실행하는 시간 ?**
   로드맵 정교화 자체가 회피일 수 있음.

**행동이 답. 로드맵은 도구.**
