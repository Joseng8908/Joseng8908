# 네이버 클라우드 엔지니어 로드맵
> 서울과기대 Bigdata & HPC Lab (김성곤 교수님) 입실 → 네이버 클라우드 엔지니어

---

## 전체 전략 구조

```
[Project 1] 연구실 입실용 포트폴리오          [혼자, ~6개월]
    쿠버네티스 커스텀 스케줄러 개발
    + 치지직 스타일 벤치마킹
         ↓
[교수님 컨택]
    "이거 만들었는데 같이 연구하고 싶습니다"
         ↓
[Project 2] 연구실 진행 프로젝트               [연구실, 1년+]
    Navix 커널 소스 분석
    + 치지직 스트리밍 최적화 (커널 레벨)
    + 논문 작성
```

---

# PROJECT 1 — 연구실 입실용 포트폴리오

> **목표:** 교수님한테 "이 학생 커널/시스템 감각 있다" 인상 남기기
> **기간:** ~6개월 (혼자)

---

## 스택 구조 (이것만 이해하면 됨)

```
[Pod Spec: requests/limits]
        ↓
[kube-scheduler + Custom Plugin]   ← 여기가 Project 1 핵심
        ↓
[kubelet → cgroup v2 트리]
        ↓
[Linux CFS]
        ↓
[Hardware]
```

---

## Phase 1 — 환경 세팅 (2~3주)

**목표:** 바로 개발 들어갈 수 있는 환경 완성

- [x] `kind` or `kubeadm` 으로 로컬 k8s 클러스터 구성
- [x] Go 개발환경 세팅 (이미 Go 하니까 빠름)
- [x] Prometheus + Grafana 스택 올리기 (벤치마크 메트릭 수집용)
- [x] FFmpeg 설치 및 트랜스코딩 테스트 워크로드 확인

```bash
# 클러스터 세팅
kind create cluster --config=kind-config.yaml

# 모니터링 스택
helm install prometheus prometheus-community/kube-prometheus-stack

# 트랜스코딩 테스트
ffmpeg -i input.mp4 -c:v libx264 -b:v 3000k output.m3u8
```

**체크포인트:** `kubectl get nodes` 정상 + Grafana 대시보드 접근 가능

---

## Phase 2 — kube-scheduler 소스 이해 (3~4주)

**목표:** 코드 레벨에서 스케줄링 루프 완전히 추적

```
kubernetes/pkg/scheduler/
├── framework/interface.go    ← Plugin 인터페이스 정의
├── schedule_one.go           ← 메인 스케줄링 루프 ★
└── internal/queue/           ← PriorityQueue
```

**스케줄링 사이클:**
```
QueueSort → PreFilter → Filter → PostFilter
          → PreScore  → Score  → Reserve
          → Permit    → PreBind → Bind
```

- [x] `scheduleOne()` 전체 추적
- [ ] 기본 플러그인 (NodeResourcesFit, InterPodAffinity) 코드 읽기
- [ ] Plugin 인터페이스 구현 방법 파악

**ctags/cscope 활용 (교수님 자료):**
```bash
cd kubernetes/pkg/scheduler
ctags -R .
# vim에서 함수 정의 이동: Ctrl+]  /  돌아오기: Ctrl+t

# Go는 gopls(LSP) 추천, C 커널 분석 때는 cscope
```

**체크포인트:** 특정 Pod가 특정 노드에 배치된 이유를 소스 레벨에서 설명 가능

---

## Phase 3 — 커스텀 스케줄러 플러그인 개발 (6~8주)

### 왜 치지직인가

```
일반 웹서비스:  stateless, 짧은 요청, 어느 노드든 무관
치지직 스트리밍: stateful + 연결 지속 + 지연 민감 + 대역폭 폭발

[스트리머] → RTMP ingest → [Transcoding Pod] → [CDN Edge] → [시청자 수백만]
                                  ↑
                        CPU/GPU 집약 + 네트워크 대역폭 집중
                        → 기본 스케줄러는 이걸 모름
```

### 구현할 플러그인

| 플러그인 | Extension Point | 핵심 역할 |
|---------|----------------|----------|
| `BandwidthScore` | Score | 노드 네트워크 BW 잔량 기반 점수 |
| `TranscoderFilter` | Filter | BW 임계치 초과 노드 제거 |
| `StreamAffinityScore` | Score | Ingest↔Transcoder 같은 노드 친화 |
| `PredictiveEnqueue` | PreEnqueue | 방송 예약 기반 미리 리소스 예약 |

### BandwidthScore 플러그인 구조 (Go)

```go
const Name = "BandwidthScore"

type BandwidthPlugin struct {
    handle         framework.Handle
    prometheusAddr string
    bwThreshold    float64  // 노드 BW 사용 임계치 (0.0~1.0)
}

// Score: 노드별 네트워크 BW 잔량 기반 점수 계산
func (pl *BandwidthPlugin) Score(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodeName string,
) (int64, *framework.Status) {

    usage, err := pl.queryNodeBandwidth(nodeName)  // Prometheus query
    if err != nil {
        return 0, framework.NewStatus(framework.Error, err.Error())
    }
    if usage > pl.bwThreshold {
        return 0, nil  // 이 노드 피함
    }
    score := int64((1.0 - usage) * float64(framework.MaxNodeScore))
    return score, nil
}

func (pl *BandwidthPlugin) Name() string { return Name }
```

### 커스텀 스케줄러 등록

```go
// main.go
command := app.NewSchedulerCommand(
    app.WithPlugin(bandwidthscore.Name, bandwidthscore.New),
    app.WithPlugin(transcoderfilter.Name, transcoderfilter.New),
)
```

```yaml
# scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: chijik-scheduler
    plugins:
      score:
        enabled:
          - name: BandwidthScore
            weight: 10
      filter:
        enabled:
          - name: TranscoderFilter
```

- [ ] BandwidthScore 플러그인 구현
- [ ] TranscoderFilter 플러그인 구현
- [ ] StreamAffinity 플러그인 구현
- [ ] PredictiveEnqueue 플러그인 구현

**체크포인트:** 커스텀 스케줄러가 실제로 Pod를 다른 노드에 배치하는 것 확인

---

## Phase 4 — 벤치마킹 (3~4주)

> 교수님이 강조: "성능 향상 수치만 보여주지 말고 왜 좋아졌는지 설명"

### 워크로드 설계

```bash
# 치지직 유사 트랜스코딩 워크로드 (동시 세션 시뮬레이션)
for i in {1..10}; do
    kubectl run transcoder-$i \
        --image=jrottenberg/ffmpeg \
        --requests='cpu=2,memory=4Gi' \
        -- -i rtmp://input-$i -c:v libx264 -b:v 6000k /out/stream-$i.m3u8 &
done
```

### FIO로 노드 스토리지 성능 측정 (교수님 자료)

```bash
fio --group_reporting --name=chijik_io \
    --directory=/mnt/data/ \
    --rw=randwrite \
    --numjobs=16 \
    --size=2GB \
    --bs=4KB \
    --direct=1 \        # direct=1: SSD 직접 쓰기(동기) / 0: 버퍼(비동기)
    --ioengine=posixaio \
    --iodepth=128
```

### calclock 패턴으로 플러그인 내부 타이밍 측정 (교수님 자료)

```go
// 교수님의 calclock.h 패턴을 Go로 적용
// 각 Extension Point 실행시간 측정

var (
    scoreTimeNs  int64  // 누적 Score() 실행시간 (ns)
    scoreCount   int64  // Score() 호출 횟수
)

func (pl *BandwidthPlugin) Score(...) (int64, *framework.Status) {
    start := time.Now()
    defer func() {
        atomic.AddInt64(&scoreTimeNs, time.Since(start).Nanoseconds())
        atomic.AddInt64(&scoreCount, 1)
    }()
    // ... 실제 Score 로직
}

// 결과 출력 (교수님 printk 패턴의 Go 버전)
log.Printf("BandwidthScore: total=%dns, count=%d, avg=%dns",
    scoreTimeNs, scoreCount, scoreTimeNs/scoreCount)
```

### 측정 항목

| 메트릭 | 기본 스케줄러 | 커스텀 스케줄러 | 목표 |
|--------|-------------|----------------|------|
| 노드 네트워크 BW 편차 | - | - | 낮을수록 좋음 |
| 트랜스코딩 완료 레이턴시 | - | - | 낮을수록 좋음 |
| 노드 간 CPU 사용률 편차 | - | - | 낮을수록 좋음 |
| Score() 평균 실행시간 | - | - | <1ms 유지 |

### 결과 문서화 (교수님 논문 구조 적용)

```
Abstract:
  "기본 k8s 스케줄러 대비 노드 BW 편차 X% 감소,
   트랜스코딩 레이턴시 Y ms 단축"

Contribution (최소 3개):
  1. Analysis:    스트리밍 워크로드의 BW 집중 패턴 분석
  2. Design:      BandwidthScore 플러그인 설계 및 구현
  3. Evaluation:  FFmpeg 트랜스코딩 기반 실측 비교

성능 향상 이유 (함수 레벨):
  - Score() 평균 실행시간: Xns (스케줄링 오버헤드 무시 가능)
  - BW 인식 배치로 네트워크 핫스팟 노드 회피
```

**체크포인트:** GitHub README에 벤치마크 결과 그래프 포함, 재현 가능한 실험 스크립트 제공

---

## 교수님 컨택 전략

**타이밍:** Phase 4 완료 후

**준비물:**
- GitHub 레포 (커스텀 스케줄러 + 벤치마크 스크립트 + README)
- 벤치마크 결과 요약 (그래프 포함, 1~2페이지)

**메일 구조:**
```
제목: 쿠버네티스 스트리밍 최적화 스케줄러 개발 관련 연구 참여 문의

1. 자기소개 (Go 시스템 프로그래머, OS/커널 배경)
2. 만든 것: 치지직 스타일 BW-aware 커스텀 스케줄러
3. 결과: 기본 대비 BW 편차 X% 감소 (GitHub 링크)
4. 연구실에서 이걸 Navix 커널 레벨로 발전시키고 싶다
5. 미팅 요청
```

---
---

# PROJECT 2 — 연구실 프로젝트

> **목표:** Navix 커널 레벨에서 치지직 스트리밍 워크로드 최적화 → 논문
> **기간:** 1년+ (연구실, 교수님과 함께)
> **전제:** Project 1 완료 + 교수님 합류 승인

---

## 큰 그림

```
Project 1에서 만든 커스텀 스케줄러
    → "스케줄러가 올바른 노드를 골랐는데도 왜 레이턴시가 있지?"
    → 커널 레벨 문제 발견
         ↓
Navix 커널 CFS 분석
    → 스트리밍 워크로드에서 CFS의 한계 규명
         ↓
커널 최적화 (패치)
    → 치지직 트랜스코딩 워크로드 특화
         ↓
논문 (FAST / OSDI / SOSP / EuroSys 타겟)
```

---

## Phase A — Navix 환경 구축 및 커널 분석 툴 세팅

- [ ] Navix 설치 (커널 5.14, RHEL-compatible, OpenELA 기반)
- [ ] 커널 소스 빌드 환경 구성
- [ ] 분석 툴 세팅: `perf`, `ftrace`, `bpftrace`, `dmesg`, `dstat`

```bash
# Navix 커널 소스 태그 생성 (교수님 ctags/cscope 방식)
cd /usr/src/linux-5.14
ctags -R .
find . \( -name '*.c' -o -name '*.h' \) -print > cscope.files
cscope -i cscope.files   # 커널 전체 인덱싱

# ftrace로 CFS 스케줄링 이벤트 트레이싱
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
cat /sys/kernel/debug/tracing/trace

# perf로 컨텍스트 스위치 분석
perf stat -e context-switches,cpu-migrations ./transcoder_workload

# bpftrace로 스케줄러 레이턴시 분포
bpftrace -e 'tracepoint:sched:sched_switch { @[comm] = hist(nsecs); }'

# dstat로 실시간 I/O 모니터링 (교수님 자료)
dstat -D sda,nvme0n1
```

---

## Phase B — CFS 동작 분석 (스트리밍 워크로드 관점)

> Navix는 커널 5.14 기반 → CFS (EEVDF 이전)

### CFS 핵심 개념

```
vruntime: 각 태스크의 가상 실행시간
  → vruntime 가장 작은 태스크를 다음에 실행
  → Red-Black Tree로 관리 (O(log n))
  → cpu.weight(cgroup)로 가중치 보정

Go GMP 모델과 비교:
  CFS per-CPU runqueue  ↔  Go local run queue (P)
  CFS load balancing    ↔  Go work stealing
  CFS 선점형            ↔  Go cooperative + 비동기 선점
  vruntime 공정성       ↔  Go round-robin (단순)
```

### 분석할 핵심 파일

```
kernel/sched/
├── fair.c     ← CFS 메인 ★ (enqueue_task_fair, pick_next_task_fair)
├── core.c     ← 스케줄러 코어 (__schedule())
├── sched.h    ← 자료구조 정의
└── pelt.c     ← Per-Entity Load Tracking
```

### 스트리밍 워크로드에서 CFS 문제점 가설

```
가설 1: I/O 대기 후 재진입 시 vruntime 보너스 과다
  FFmpeg 트랜스코딩: I/O 대기(sleep) + CPU 버스트 반복
  → CFS는 sleep 후 깨어난 태스크에 vruntime 보너스 부여
  → 갑자기 높은 우선순위 획득 → 다른 세션 레이턴시 스파이크

가설 2: 멀티 세션 간 load balancing 부족
  → 여러 트랜스코딩 세션이 같은 CPU에 몰림
  → BW 집중 + CPU 경쟁 동시 발생
```

### calclock.h 패턴으로 커널 함수 타이밍 측정 (교수님 자료)

```c
// fair.c에 삽입 (교수님 방식 그대로)
#include "calclock.h"

unsigned long long pick_next_time, pick_next_count;

static struct task_struct *
pick_next_task_fair_measured(struct rq *rq, ...)
{
    struct timespec local_time[2];
    getrawmonotonic(&local_time[0]);

    struct task_struct *result = pick_next_task_fair_internal(rq, ...);

    getrawmonotonic(&local_time[1]);
    calclock(local_time, &pick_next_time, &pick_next_count);
    return result;
}

// dmesg로 확인
// printk: pick_next_task_fair: time=Xns, count=Y
```

---

## Phase C — 커널 최적화 설계 및 구현

> **연구 방향은 Phase B 분석 결과에 따라 확정** — 아래는 가능한 방향

### 방향 1: 스트리밍 워크로드 인식 스케줄링

```
현재: CFS가 트랜스코딩을 일반 태스크로 처리
개선: 실시간 스트리밍 특성 인식 (주기적 I/O + 소프트 데드라인)

검토할 것:
  - SCHED_DEADLINE (EDF) 적용 가능성
  - sched_ext (BPF 기반 커스텀 스케줄러, 6.11+) Navix 백포팅 가능성
  - vruntime 보너스 상한 제한 패치
```

### 방향 2: NUMA-aware 스트리밍 배치

```
현재: CFS load balancing이 NUMA 토폴로지를 충분히 활용 못함
개선: 트랜스코딩 세션을 같은 NUMA 노드 CPU에 고정
      → 메모리 접근 레이턴시 감소 + BW 집중 완화

k8s 연계:
  Project 1 커스텀 스케줄러 (topologySpreadConstraints)
  + Project 2 커널 패치
  → 두 레이어의 시너지 효과
```

### 방향 3: cgroup BW Throttling 튜닝

```
현재: cpu.cfs_quota_us가 트랜스코딩 버스트를 과도하게 제한
개선: 스트리밍 워크로드 특성에 맞는 quota/period 비율 최적화
      또는 cpu.cfs_burst_us (burst 허용) 파라미터 튜닝
```

---

## Phase D — 논문 작성 (교수님 자료 구조 그대로 적용)

```
Abstract:
  문제 → 우리 접근 → 평가 수치 (한 문단)

Introduction:
  1. 스트리밍 서비스의 규모와 중요성
  2. 기존 k8s 스케줄러 + CFS의 한계
  3. 관련 연구 비교
  4. 우리 접근법 (In this paper, ...)
  5. Contribution (최소 3개: Analysis / Design & Impl / Evaluation)
  6. 논문 구성

Background:   k8s 스케줄링, CFS, cgroup, 스트리밍 워크로드 특성
Design:       전체 아키텍처 → 세부 설계 → 구현 디테일 (재현 가능하게)
Evaluation:   왜 성능이 좋아졌는지 함수 레벨로 설명 → State-of-Art 비교
Conclusion:   요약 + 한계 + 향후 방향
```

**교수님이 강조한 핵심:**
- 리뷰어는 Abstract → Figure/Table → Conclusion 순으로 봄
- 성능 향상 이유를 **함수 레벨 프로파일링**으로 설명 (calclock 결과 활용)
- 구현 디테일 충분히: 독자가 재현 가능해야 함

**타겟 컨퍼런스 (BK21):**
- OSDI / SOSP (OS 최상위)
- FAST (스토리지/파일시스템)
- EuroSys

---

## 전체 타임라인

```
━━━━━━━━━━━━━━━━━━━━ PROJECT 1 (혼자) ━━━━━━━━━━━━━━━━━━━━
Month 1      : 환경 세팅 (k8s, Prometheus, FFmpeg)
Month 2      : kube-scheduler 소스 이해
Month 3~4    : 커스텀 플러그인 개발 (4개)
Month 5~6    : 벤치마킹 + 문서화
Month 6 말   : 교수님 컨택

━━━━━━━━━━━━━━━━━━━━ PROJECT 2 (연구실) ━━━━━━━━━━━━━━━━━━
Month 7~8    : Navix 세팅 + 커널 분석 툴
Month 9~11   : CFS 분석 (스트리밍 워크로드 관점, 문제점 규명)
Month 12~15  : 커널 최적화 설계 + 구현
Month 16+    : 논문 작성 + 컨퍼런스 제출
```

---

## 포트폴리오 구성

| 시점 | 결과물 | 용도 |
|------|--------|------|
| Phase 4 끝 | GitHub 레포 (스케줄러 + 벤치마크) | 교수님 컨택 |
| Phase 4 끝 | 결과 요약 문서 (1~2p, 그래프 포함) | 교수님 컨택 |
| Phase B 끝 | CFS 분석 보고서 (내부) | 연구 방향 확정 |
| Phase D 끝 | 논문 초안 | 컨퍼런스 제출 |

---

## 레퍼런스

**소스코드**
- [kubernetes/pkg/scheduler](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler)
- [Navix GitHub (Bug Tracker)](https://github.com/NaverCloudPlatform/Navix)
- [Linux kernel/sched/fair.c](https://github.com/torvalds/linux/blob/master/kernel/sched/fair.c)

**문서**
- [Kubernetes Scheduler Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [cgroup v2 kernel docs](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [sched_ext: Extensible Scheduler Class](https://lwn.net/Articles/922405/)
- [LWN: EEVDF scheduler](https://lwn.net/Articles/925371/)

**논문**
- EEVDF: "Earliest Eligible Virtual Deadline First" (Stoica & Zhang, 1996)
- FAST, OSDI, SOSP, EuroSys 최신 논문 — scholar.google.com

**교수님 자료 (Bigdata & HPC Lab, SeoulTech)**
- Freshman Orientation: Tools (fdisk, dstat, FIO, calclock.h 프로파일링)
- Freshman Orientation: Writing Papers (논문 구조, contribution 작성법)
- 논문 관리: Mendeley
