# AI/HPC 데이터센터 스케줄링 엔지니어 로드맵
> 서울과기대 Bigdata & HPC Lab (김성곤 교수님) 입실 → 네이버 세종 데이터센터 / NHN AI 인프라

---

## 전체 전략 구조

```
[Project 1] 연구실 입실용 포트폴리오          [완료]
    치지직 스타일 BW-aware 커스텀 스케줄러
    + 벤치마킹 (BW 80% 노드 배치 0개 달성)
         ↓
[교수님 컨택]
    "이거 만들었는데 GPU Topology 스케줄링으로 발전시키고 싶습니다"
         ↓
[Project 2] 연구실 진행 프로젝트               [연구실, 1년+]
    GPU Topology-aware 스케줄러
    NUMA + NVLink 기반 최적 배치
    + 논문 작성
         ↓
[취업]
    네이버 세종 데이터센터 / NHN Cloud AI 인프라
```

---

# PROJECT 1 — 연구실 입실용 포트폴리오 [완료 ✅]

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

## 벤치마킹 결과 요약

| 지표 | 기본 스케줄러 | chijik-scheduler |
|------|------------|----------------|
| BW 80% 노드 배치 Pod | 5개 | **0개** |
| BW 30% 노드 배치 Pod | 5개 | **10개** |
| BW 인식 여부 | ❌ | ✅ |

## 현재 한계 (솔직하게)

```
어노테이션 기반 프로토타입:
  실제 BW는 수동 설정 (Prometheus 자동 연동 미구현)
  단일 리소스(BW)만 고려 (GPU, NUMA 미고려)
  → 이게 Project 2의 시작점
```

---

# PROJECT 2 — GPU Topology-aware 스케줄러 [연구실]

> **목표:** 교수님 GPU 클러스터(양자 컴퓨팅 시뮬레이션)에서 실제 성능 문제 해결
> **기간:** 1년+ (연구실, 교수님과 함께)

## 왜 GPU Topology인가

```
교수님 연구:
  대용량 GPU 클러스터 + 양자 컴퓨팅 시뮬레이션
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

### 핵심 아이디어

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

// Reserve 단계: 즉시 내부 캐시 업데이트
func (pl *GPUTopologyPlugin) Reserve(...) *framework.Status {
    pl.mu.Lock()
    pl.reserved[nodeName] += getGPUUsage(pod)  // 즉시 반영
    pl.mu.Unlock()
    return nil
}

// Unreserve 단계: 롤백
func (pl *GPUTopologyPlugin) Unreserve(...) {
    pl.mu.Lock()
    pl.reserved[nodeName] -= getGPUUsage(pod)  // 롤백
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
  3. Evaluation: 양자 시뮬레이션 워크로드 기반 실측 비교

타겟 컨퍼런스:
  OSDI / SOSP (OS 최상위)
  EuroSys
  SC (Supercomputing)
```

---

# 취업 타겟

## 포지션

```
AI/HPC 데이터센터 스케줄링 엔지니어
= "GPU 클러스터 위에서 돌아가는 k8s를 만드는 사람"
```

## 회사별 연결

| 회사 | 연결 포인트 |
|------|-----------|
| 네이버 세종 데이터센터 | HyperCLOVA 학습 인프라, GPU 클러스터 운영 |
| NHN Cloud | AI 인프라 플랫폼 |
| ETRI | AI 학습/추론 스케줄링 연구 (이미 지원 중) |
| 카카오 | 카카오브레인 GPU 클러스터 |

## 희소성

```
일반 백엔드 엔지니어:
  "k8s 위에서 서비스 올리는 사람"
  → 지원자 수백 명

이 포지션:
  "GPU Topology를 이해하고 스케줄러를 만드는 사람"
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

━━━━━━━━━━ 교수님 컨택 (지금) ━━━━━━━━━━
벤치마킹 결과 + GitHub 레포 + 메일

━━━━━━━━━━ PROJECT 2 (연구실) ━━━━━━━━━━
Month 1~2   : GPU 토폴로지 개념 학습 (NUMA, NVLink, NFD)
              kueue, volcano 소스 분석
Month 3~4   : GPU Topology-aware Score 플러그인 구현
              Optimistic Reservation 캐시 구현
Month 5~6   : 교수님 GPU 클러스터 적용 + 실측
Month 7+    : 논문 작성

━━━━━━━━━━ 취업 ━━━━━━━━━━
석사 졸업 → 네이버 세종 / NHN Cloud AI 인프라
```

---

# 교수님 컨택 메일 구조

```
제목: GPU 클러스터 스케줄링 최적화 관련 연구 참여 문의

1. 자기소개
   Go 시스템 프로그래머, 서울과기대 CS 3학년

2. 만든 것
   치지직 스타일 BW-aware 커스텀 k8s 스케줄러
   kube-scheduler 소스 분석 + 플러그인 4개 구현 + 벤치마킹
   GitHub: github.com/Joseng8908/chijik-scheduler

3. 결과
   BW 80% 노드 배치 0개 달성 (기본 스케줄러: 5개)

4. 연결
   교수님 GPU 클러스터 연구에서
   NUMA Split, Thundering Herd 문제를
   NFD + Optimistic Reservation으로 해결하고 싶음

5. 미팅 요청
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

**논문**
- Gandiva: Introspective Cluster Scheduling for Deep Learning (OSDI 2018)
- Tiresias: A GPU Cluster Manager for Distributed Deep Learning (NSDI 2019)
- AntMan: Dynamic Scaling on GPU Clusters for Deep Learning (OSDI 2020)
