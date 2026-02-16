---
title: "[CS330] 6. 스와핑과 페이지 교체 정책"
date: 2023-12-06
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "Swapping", "Page Replacement", "LRU", "mmap"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "물리 메모리보다 큰 가상 메모리를 지원하는 스와핑, 다양한 페이지 교체 알고리즘(OPT, FIFO, LRU, Clock), 그리고 메모리 매핑(mmap)을 다룹니다."
aliases: ["/posts/cs330-6-swapping-page-replacement/"]
ShowToc: true
TocOpen: true
---

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

*이전 글: [[CS330] 5. 페이징과 TLB](/posts/cs330/5-paging-tlb/)*

---

## 1. 메모리 계층 구조와 스와핑

### 1.1 메모리 계층 구조 (Memory Hierarchy)

현대 컴퓨터 시스템은 성능과 용량의 트레이드오프를 관리하기 위해 계층적인 메모리 구조를 사용한다.

```text
               ▲
              /│\
             / │ \        Registers (가장 빠름, 가장 비쌈)
            /  │  \
           /   │   \
          /    │    \
         /     │     \      Cache
        /      │      \
       /       │       \
      /        │        \   Main Memory (RAM)
     /         │         \
    /          │          \
   /___________│___________\ Disk Storage (가장 느림, 가장 저렴)
              │
         Size │
              ↓
```

각 계층은 위 계층의 "백킹 스토어(backing store)" 역할을 수행한다:
- 레지스터의 백킹 스토어는 캐시
- 캐시의 백킹 스토어는 메인 메모리
- **메인 메모리의 백킹 스토어는 디스크**

### 1.2 스와핑이란 무엇인가?

**스와핑(Swapping)**은 물리 메모리가 부족할 때 디스크를 메모리 확장 공간으로 활용하는 메커니즘이다. 가상 주소(Virtual Address)와 물리 주소(Physical Address)의 위치가 독립적이라는 특성을 활용한다.

#### 왜 스와핑이 필요한가?

1. **물리 메모리의 한계**: 프로세스가 필요로 하는 메모리가 물리 메모리보다 클 수 있다
2. **메모리 독립성**: 사용자 프로그램은 실제 물리 메모리 크기와 무관하게 동작해야 한다
3. **참조의 지역성(Locality of Reference)**: 프로세스는 한 순간에 작은 양의 메모리만 활발히 사용한다
4. **효율적 자원 활용**: 현재 사용하지 않는 페이지를 디스크로 내보내고, 필요한 페이지만 메모리에 유지한다

### 1.3 스와핑의 종류

**Process-level Swapping** (프로세스 단위 스와핑)
- 초기 시스템에서 사용하던 방식
- 프로세스 전체를 임시로 디스크로 내보냈다가 나중에 다시 메모리로 가져온다
- 스왑 단위가 커서 효율성이 떨어진다

**Page-level Swapping** (페이지 단위 스와핑)
- 현대 운영체제에서 사용하는 방식
- 페이지 단위로 swap-out (page-out) 및 swap-in (page-in)을 수행한다
- 더 세밀한 제어가 가능하다

### 1.4 스왑 공간 (Swap Space)

스왑 공간은 디스크에 할당된 영역으로, 메모리 페이지를 저장하는 용도로 사용된다.

**특징:**
- 페이지를 왔다 갔다 옮기기 위한 디스크 공간
- 스왑 공간의 크기가 사용 가능한 최대 메모리 페이지 수를 결정한다
- 블록 크기는 페이지 크기와 동일하다 (일반적으로 4KB)
- 전용 파티션(dedicated partition) 또는 파일 시스템 내 파일로 구현된다

**예시:**

```text
Physical Memory:  [PFN 0]  [PFN 1]  [PFN 2]  [PFN 3]
                   PID 0    PID 1    PID 1    PID 2
                  (VPN 0)  (VPN 1)  (VPN 2)  (VPN 0)

Swap Space:       [Blk 0]  [Blk 1]  [Blk 2]  [Blk 3]  [Blk 4]  [Blk 5]  [Blk 6]  [Blk 7]
                   PID 0    PID 0   (Free)    PID 1    PID 1    PID 3    PID 2    PID 3
                  (VPN 1)  (VPN 2)            (VPN 0)  (VPN 1)  (VPN 0)  (VPN 1)  (VPN 1)
```

---

## 2. 페이지 폴트와 페이지 교체

### 2.1 Present Bit

페이지 테이블 엔트리(PTE)에는 **present bit**가 있어서 페이지가 현재 물리 메모리에 있는지 디스크에 있는지를 나타낸다.

| Present Bit 값 | 의미 |
|:---:|:---|
| 1 | 페이지가 물리 메모리에 존재함 |
| 0 | 페이지가 메모리에 없고 디스크에 있음 |

### 2.2 페이지 폴트 (Page Fault)

**페이지 폴트**는 프로세스가 현재 물리 메모리에 없는 페이지에 접근할 때 발생하는 이벤트다.

페이지 폴트가 발생하면:
1. 하드웨어가 PTE를 검사하고 present bit가 0임을 발견한다
2. 페이지 폴트 예외(exception)가 발생하여 운영체제로 제어가 넘어간다
3. 운영체제는 페이지를 디스크에서 메모리로 가져와야 한다 (swap-in)
4. 필요하다면 다른 페이지를 디스크로 내보낸다 (swap-out)

### 2.3 페이지 폴트 처리 과정

```text
    [1] Load M 명령 실행
          |
          v
    [2] Page Fault 발생 (Trap to OS)
          |
          v
    [3] OS: 페이지가 디스크에 있는지 확인
          |
          v
    [4] OS: 디스크에서 페이지 읽기
          |
          |     +-----------------+
          |     | Secondary       |
          +---->| Storage (Disk)  |
                |   [Page Frame]  |----+
                +-----------------+    |
                                       |
    [5] Page Table 업데이트 <----------+
          |
          v
    [6] 명령 재실행 (Reinstruction)
```

### 2.4 페이지 교체 (Page Replacement)

물리 메모리가 가득 찬 상태에서 새로운 페이지를 가져와야 할 때, 운영체제는 기존 페이지 중 하나를 선택하여 내보내야 한다. 이 과정을 **페이지 교체(page replacement)**라고 하며, 어떤 페이지를 내보낼지 결정하는 정책을 **페이지 교체 정책(page replacement policy)**이라고 한다.

### 2.5 페이지 축출 (Page Eviction)

페이지를 메모리에서 제거하는 방법은 페이지 타입에 따라 다르다:

**코드 페이지 (Code Page)**
- 읽기 전용이므로 디스크의 실행 파일에서 다시 읽어올 수 있다
- 스왑 공간에 쓸 필요 없이 그냥 제거한다

**데이터 페이지 (Data Page)**
- Modified (dirty): 스왑 공간에 기록해야 한다
- Unmodified (clean): 스왑 공간에서 버린다 (discard)

**익명 페이지 (Anonymous Page)**
- 파일 백킹이 없는 페이지 (스택, 힙 등)
- Clean anonymous page:
  - 이전에 스왑 아웃된 적이 있으면: 메모리에서 제거만 (backing store가 디스크에 이미 있음)
  - 한 번도 스왑 아웃된 적이 없으면: 새 스왑 공간 할당하고 기록
  - 예외: Zero page (모두 0인 페이지)는 스왑 아웃 불필요
- Dirty anonymous page:
  - 이전에 스왑 아웃된 적이 있으면: 스왑 공간에 덮어쓰기
  - 한 번도 스왑 아웃된 적이 없으면: 새 스왑 공간 할당하고 기록

### 2.6 스와핑 시점 (When to Perform Swap)

**Lazy Approach** (게으른 접근)
- 메모리가 완전히 찬 후에만 페이지를 교체한다
- 비현실적이다

**Proactive Approach** (적극적 접근)
- 운영체제는 항상 일정량의 여유 메모리를 유지하려고 한다
- 두 개의 임계값 사용: **HW (High Watermark)**와 **LW (Low Watermark)**
- **Swap Daemon** (또는 Page Daemon, Linux에서는 `kswapd`)이라는 백그라운드 스레드가 메모리를 관리한다

```text
Free Pages < LW  → swap daemon이 페이지를 축출하기 시작
Free Pages > HW  → swap daemon이 잠들기 (sleep)
```

---

## 3. 페이지 교체 알고리즘

### 3.1 목표: 페이지 폴트율 최소화

페이지 교체 정책의 목표는 **페이지 폴트율(page fault rate)** 또는 **미스율(miss rate)**을 최소화하는 것이다.

디스크 접근 비용은 메모리 접근 비용보다 훨씬 크므로(100,000배 이상), 작은 미스율 차이도 전체 성능에 큰 영향을 미친다.

**평균 메모리 접근 시간 (AMAT: Average Memory Access Time)**

```text
AMAT = (P_Hit × T_M) + (P_Miss × T_D)
```

- `T_M`: 메모리 접근 비용
- `T_D`: 디스크 접근 비용
- `P_Hit`: 히트 확률
- `P_Miss`: 미스 확률

### 3.2 OPT (Optimal) - 최적 알고리즘

**Belady의 최적 교체 정책 (1966)**은 "미래에 가장 오랫동안 사용되지 않을 페이지"를 교체한다.

**특징:**
- 어떤 페이지 참조 스트림(page reference string)에 대해서도 가장 낮은 폴트율을 보인다
- 다른 알고리즘의 성능 비교 기준(benchmark)으로 사용된다
- **실용적이지 않다**: 미래를 예측해야 하므로 실제로는 구현 불가능하다

**시뮬레이션 예시 (캐시 크기 = 3):**

Reference String: `0 1 2 0 1 3 0 3 1 2 1`

| Access | Hit/Miss | Evict | Cache State |
|:------:|:--------:|:-----:|:-----------:|
| 0      | Miss     | -     | 0           |
| 1      | Miss     | -     | 0, 1        |
| 2      | Miss     | -     | 0, 1, 2     |
| 0      | Hit      | -     | 0, 1, 2     |
| 1      | Hit      | -     | 0, 1, 2     |
| 3      | Miss     | **2** | 0, 1, 3     |
| 0      | Hit      | -     | 0, 1, 3     |
| 3      | Hit      | -     | 0, 1, 3     |
| 1      | Hit      | -     | 0, 1, 3     |
| 2      | Miss     | **3** | 0, 1, 2     |
| 1      | Hit      | -     | 0, 1, 2     |

**Hit Rate = 6 / 11 = 54.6%**

OPT는 페이지 2를 축출할 때 "가장 나중에 다시 참조될" 페이지를 선택한다.

### 3.3 FIFO (First-In First-Out)

**FIFO**는 가장 오래 전에 메모리에 로드된 페이지를 교체한다.

**특징:**
- 구현이 매우 간단하다 (큐 자료구조 사용)
- 가장 오래된 페이지가 사용되지 않을 것이라는 가정
- **문제점**: 오래된 페이지가 계속 사용될 수도 있다 (temporal locality 무시)

**시뮬레이션 예시 (캐시 크기 = 3):**

Reference String: `0 1 2 0 1 3 0 3 1 2 1`

| Access | Hit/Miss | Evict | Cache State |
|:------:|:--------:|:-----:|:-----------:|
| 0      | Miss     | -     | 0           |
| 1      | Miss     | -     | 0, 1        |
| 2      | Miss     | -     | 0, 1, 2     |
| 0      | Hit      | -     | 0, 1, 2     |
| 1      | Hit      | -     | 0, 1, 2     |
| 3      | Miss     | **0** | 1, 2, 3     |
| 0      | Miss     | **1** | 2, 3, 0     |
| 3      | Hit      | -     | 2, 3, 0     |
| 1      | Miss     | **2** | 3, 0, 1     |
| 2      | Miss     | **3** | 0, 1, 2     |
| 1      | Hit      | -     | 0, 1, 2     |

**Hit Rate = 4 / 11 = 36.4%**

#### Belady's Anomaly (벨레이디의 역설)

FIFO는 특이한 현상을 보인다: **메모리를 늘렸는데 오히려 페이지 폴트가 증가할 수 있다.**

**예시: Reference String `1 2 3 4 1 2 5 1 2 3 4 5`**

**3개 프레임일 때:**
```text
참조:  1   2   3   4   1   2   5   1   2   3   4   5
      [1] [1] [1] [4] [4] [4] [5] [5] [5] [5] [5] [5]
          [2] [2] [2] [1] [1] [1] [1] [1] [3] [3] [3]
              [3] [3] [3] [2] [2] [2] [2] [2] [2] [4]
      M   M   M   M   M   M   M   H   H   M   M   H
```
**9 misses / 12 accesses = 75% miss rate**

**4개 프레임일 때:**
```text
참조:  1   2   3   4   1   2   5   1   2   3   4   5
      [1] [1] [1] [1] [1] [1] [5] [5] [5] [5] [4] [4]
          [2] [2] [2] [2] [2] [2] [1] [1] [1] [1] [5]
              [3] [3] [3] [3] [3] [3] [2] [2] [2] [2]
                  [4] [4] [4] [4] [4] [4] [3] [3] [3]
      M   M   M   M   H   H   M   M   M   M   M   M
```
**10 misses / 12 accesses = 83% miss rate**

이는 FIFO가 페이지의 실제 사용 패턴을 고려하지 않기 때문에 발생한다.

### 3.4 Random

**Random**은 무작위로 페이지를 선택하여 교체한다.

**특징:**
- 교체할 페이지를 선택하는 데 지능적인 판단을 하지 않는다
- 구현이 간단하다
- 성능은 운에 따라 달라진다

### 3.5 LRU (Least Recently Used)

**LRU**는 "가장 오래 전에 사용된 페이지"를 교체한다. 과거를 이용해 미래를 예측하는 접근법이다.

**원리:**
- **Recency (최근성)**: 최근에 접근된 페이지는 곧 다시 접근될 가능성이 높다
- **과거의 사용 패턴이 미래를 근사**한다는 가정

**시뮬레이션 예시 (캐시 크기 = 3):**

Reference String: `0 1 2 0 1 3 0 3 1 2 1`

| Access | Hit/Miss | Evict | Cache State |
|:------:|:--------:|:-----:|:-----------:|
| 0      | Miss     | -     | 0           |
| 1      | Miss     | -     | 0, 1        |
| 2      | Miss     | -     | 0, 1, 2     |
| 0      | Hit      | -     | 1, 2, 0     |
| 1      | Hit      | -     | 2, 0, 1     |
| 3      | Miss     | **2** | 0, 1, 3     |
| 0      | Hit      | -     | 1, 3, 0     |
| 3      | Hit      | -     | 1, 0, 3     |
| 1      | Hit      | -     | 0, 3, 1     |
| 2      | Miss     | **0** | 3, 1, 2     |
| 1      | Hit      | -     | 3, 2, 1     |

**Hit Rate = 7 / 11 = 63.6%**

LRU는 OPT에 가까운 성능을 보이며, FIFO보다 훨씬 좋다.

#### LRU의 문제점: 구현 비용

완벽한 LRU를 구현하려면 모든 페이지 접근마다 접근 시간을 기록해야 한다.
- 하드웨어 지원이 필요하다
- 매 메모리 접근마다 타임스탬프 업데이트는 오버헤드가 크다

따라서 실제로는 LRU를 근사하는 알고리즘을 사용한다.

### 3.6 Clock Algorithm - LRU의 근사

**Clock Algorithm**은 LRU를 근사하는 실용적인 알고리즘이다.

**하드웨어 지원: Use Bit (Reference Bit)**
- 페이지가 참조될 때마다 하드웨어가 use bit를 1로 설정한다
- 하드웨어는 이 비트를 지우지 않는다 (OS의 책임)

**동작 방식:**
1. 모든 페이지를 원형 리스트로 구성한다
2. 시계 바늘(clock hand)이 특정 페이지를 가리킨다
3. 페이지 교체가 필요할 때:
   - Use bit가 1이면: 0으로 바꾸고 다음 페이지로 이동
   - Use bit가 0이면: 해당 페이지를 축출

**ASCII 다이어그램:**

```text
           A
          / \
         /   \
        H     B        Clock Hand →
         \   /
          \ /
         G   C          [Use Bit 테이블]
          \ /           +---------+---------+
          F-D           | Use=0   | 축출    |
           E            | Use=1   | 0으로   |
                        +---------+---------+

교체 과정:
1. 현재 페이지의 use bit 확인
2. Use bit = 1 → 0으로 설정하고 시계 방향으로 이동
3. Use bit = 0 → 해당 페이지 축출
```

**예시:**

```text
초기 상태: A(1) - B(1) - C(0) - D(1) - E(1)
                  ↑ (시계 바늘)

Step 1: B의 use bit = 1 → 0으로 바꾸고 다음으로
        A(1) - B(0) - C(0) - D(1) - E(1)
                      ↑

Step 2: C의 use bit = 0 → C를 축출!
```

**특징:**
- LRU를 근사하지만 완벽하지는 않다
- 구현이 간단하다
- 성능은 LRU와 비슷하고, FIFO나 Random보다 훨씬 좋다
- 실제 운영체제에서 널리 사용된다

### 3.7 워크로드별 성능 비교

#### No-Locality Workload (지역성 없음)

페이지 참조가 완전히 무작위인 경우, 캐시가 충분히 크면 모든 알고리즘의 성능이 비슷하다.

```text
Hit Rate
100% ┤              ╭──────
     │            ╭─╯
 80% ┤          ╭─╯
     │        ╭─╯
 60% ┤      ╭─╯         OPT, LRU, FIFO, RAND 모두 동일
     │    ╭─╯
 40% ┤  ╭─╯
     │╭─╯
 20% ┤╯
     │
  0% └─────────────────────────
      0   20   40   60   80  100
          Cache Size (Blocks)
```

#### 80-20 Workload (지역성 있음)

참조의 80%가 전체 페이지의 20%에 집중되는 경우, LRU가 "핫 페이지(hot pages)"를 캐시에 유지하여 우수한 성능을 보인다.

```text
Hit Rate
100% ┤         ╭─────────
     │       ╭─╯      OPT (최고)
 80% ┤     ╭─╯      LRU (우수)
     │   ╭─╯       FIFO
 60% ┤ ╭─╯        RAND (최악)
     │╭╯
 40% ┤╯
     │
 20% ┤
     │
  0% └─────────────────────────
      0   20   40   60   80  100
          Cache Size (Blocks)
```

#### Looping Sequential Workload (순차 반복)

50개 페이지를 순차적으로 반복 접근하는 경우 (0→1→...→49→0→1→...), LRU가 최악의 성능을 보인다.

```text
Hit Rate
100% ┤            ╭──────
     │            │
     │            │   OPT, FIFO
 60% ┤            │
     │            │
 40% ┤            │
     │            │
 20% ┤            │
     │╭───────────╯   LRU (최악!)
  0% ┤╯
     └─────────────────────────
      0   20   40   60   80  100
          Cache Size (Blocks)
```

캐시 크기가 50보다 작으면 LRU는 0% hit rate를 기록한다. LRU는 가장 최근에 사용된 페이지를 유지하지만, 다음에 필요한 페이지는 가장 오래 전에 사용된 페이지이기 때문이다.

---

## 4. Working Set과 Thrashing

### 4.1 Working Set (작업 집합)

**Working Set**은 프로세스가 현재 활발히 사용하고 있는 페이지들의 집합이다.

**특징:**
- 시간에 따라 변한다
- 프로그램의 실행 단계(phase)에 따라 다르다
- 참조의 지역성(locality of reference)을 반영한다

### 4.2 Thrashing (스래싱)

**Thrashing**은 물리 메모리가 부족하여 시스템이 대부분의 시간을 페이지 스와핑에 소모하는 현상이다.

**발생 원인:**
- 메모리가 **과다 할당(oversubscribed)**되어 있다
- 실행 중인 프로세스들의 working set 합이 물리 메모리보다 크다
- 물리 메모리가 모든 프로세스의 working set을 담을 수 없다

**결과:**
- 대부분의 시간을 디스크 I/O에 소모한다
- CPU 활용률이 급격히 떨어진다
- 시스템 반응성이 극도로 나빠진다

**CPU Utilization vs Degree of Multiprogramming:**

```text
CPU
Utilization
    ▲
    │                     ╭─╮
    │                   ╭─╯  ╰─╮
    │                 ╭─╯       ╰──╮
    │               ╭─╯              ╰───
    │             ╭─╯
    │          ╭──╯
    │      ╭───╯         ← Thrashing 구간
    │  ╭───╯
    └──┴────────────────────────────────→
           Degree of Multiprogramming
```

**해결 방법:**
1. 실행 중인 프로세스 수를 줄인다
2. 일부 프로세스를 중단하여 남은 프로세스들의 working set이 메모리에 맞도록 한다
3. 메모리를 추가한다

---

## 5. 메모리 매핑 (Memory Mapping)

### 5.1 mmap이란?

**mmap()**은 파일이나 장치를 프로세스의 가상 주소 공간에 매핑하는 시스템 콜이다.

**장점:**
1. 파일을 마치 메모리처럼 접근할 수 있다 (read/write 시스템 콜 불필요)
2. 여러 프로세스가 동일한 파일을 공유할 수 있다
3. 대용량 파일을 효율적으로 처리할 수 있다
4. Lazy loading: 실제로 접근할 때만 페이지가 로드된다

### 5.2 메모리 매핑의 분류

메모리 매핑은 두 가지 축으로 분류된다:

**매핑 타입 (Mapping Type):**
- **File-backed Mapping**: 실제 파일과 연결
- **Anonymous Mapping**: 파일 없이 메모리만 할당 (힙, 스택 등)

**공유 방식 (Sharing Mode):**
- **Shared Mapping**: 변경사항이 다른 프로세스나 파일에 반영
- **Private Mapping**: 변경사항이 해당 프로세스에만 보임 (Copy-on-Write 사용)

**2×2 매트릭스:**

```text
                  File-backed              Anonymous
              ┌──────────────────┬──────────────────────┐
              │                  │                      │
   Shared     │ • 파일 공유      │ • IPC용 공유 메모리  │
              │ • 프로세스간     │ • System V shm       │
              │   데이터 교환    │ • POSIX shm          │
              ├──────────────────┼──────────────────────┤
              │                  │                      │
   Private    │ • 실행 파일      │ • malloc() 힙        │
              │   로딩 (.text)   │ • 스택 (Stack)       │
              │ • 공유 라이브러리│ • Anonymous 메모리   │
              │   (.so, .dll)    │   할당               │
              │                  │                      │
              └──────────────────┴──────────────────────┘
```

### 5.3 File-backed Shared Mapping

**사용 사례:**
- 여러 프로세스가 동일한 파일을 공유
- 한 프로세스의 변경사항이 파일에 반영되고, 다른 프로세스도 볼 수 있다

**예시:**
```c
int fd = open("data.txt", O_RDWR);
char *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd, 0);
addr[0] = 'X';  // 파일 내용 변경 → 다른 프로세스도 볼 수 있음
```

### 5.4 File-backed Private Mapping

**사용 사례:**
- 실행 파일의 코드 섹션 (.text) 로딩
- 공유 라이브러리 (.so, .dll) 로딩
- 한 프로세스의 변경사항은 해당 프로세스에만 보임

**Copy-on-Write (CoW):**
- 처음에는 모든 프로세스가 동일한 물리 페이지를 공유한다
- 어떤 프로세스가 페이지를 수정하려고 하면, 운영체제가 해당 페이지를 복사한다
- 이후 변경사항은 복사본에만 반영된다

```text
초기 상태 (공유):
Process A → [Physical Page 100] ← Process B

Write 발생 후 (CoW):
Process A → [Physical Page 100 (original)]
Process B → [Physical Page 200 (copy)]
```

### 5.5 Anonymous Shared Mapping

**사용 사례:**
- 프로세스 간 통신 (IPC)
- 부모-자식 프로세스 간 데이터 공유
- System V shared memory, POSIX shared memory

**예시:**
```c
char *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                  MAP_SHARED | MAP_ANONYMOUS, -1, 0);
```

### 5.6 Anonymous Private Mapping

**사용 사례:**
- `malloc()`이 큰 메모리를 할당할 때 내부적으로 사용
- 프로세스의 힙, 스택
- 파일 백킹이 필요 없는 메모리

**특징:**
- 스왑 공간이 backing store 역할을 한다
- Zero page 최적화: 처음 할당 시 모든 페이지는 0으로 초기화된 공유 페이지를 가리킨다
- 실제로 쓰기가 발생할 때 새 페이지가 할당된다 (CoW)

**예시:**
```c
char *heap = mmap(NULL, 1024*1024, PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
```

### 5.7 fork()와 Copy-on-Write

`fork()` 시스템 콜은 Private Mapping과 Copy-on-Write를 활용한다.

**동작 과정:**
1. `fork()` 호출 시 자식 프로세스가 생성된다
2. 부모의 모든 페이지를 즉시 복사하지 않는다
3. 대신 부모와 자식이 동일한 물리 페이지를 공유한다 (read-only로 표시)
4. 어느 한쪽이 페이지를 수정하려고 하면 page fault 발생
5. 운영체제가 해당 페이지만 복사하여 독립적인 페이지를 할당한다

**장점:**
- `fork()` 성능이 크게 향상된다
- 실제로 수정되는 페이지만 복사하므로 메모리 낭비가 줄어든다
- `fork()` 직후 `exec()`을 호출하는 경우 복사가 거의 발생하지 않는다

---

## 6. 정리

### 스와핑 핵심 개념

1. **스와핑은 디스크를 메모리 확장으로 활용**하여 물리 메모리보다 큰 가상 메모리 공간을 지원한다
2. **Swap space**는 페이지를 저장하는 디스크 영역이다
3. **Present bit**는 페이지가 메모리에 있는지 디스크에 있는지를 나타낸다
4. **Page fault**는 메모리에 없는 페이지 접근 시 발생하며, 운영체제가 디스크에서 페이지를 가져온다

### 페이지 교체 알고리즘 비교

| 알고리즘 | 장점 | 단점 | 실용성 |
|:---:|:---|:---|:---:|
| **OPT** | 최적 성능 (이론적 기준) | 미래 예측 불가능 | ✗ |
| **FIFO** | 구현 간단 | 지역성 무시, Belady's Anomaly | △ |
| **Random** | 구현 간단 | 성능 예측 불가 | △ |
| **LRU** | 우수한 성능, 지역성 활용 | 구현 비용 높음 | ✗ |
| **Clock** | LRU 근사, 구현 간단 | LRU보다 약간 떨어짐 | ✓ |

### Working Set과 Thrashing

- **Working set**: 프로세스가 활발히 사용하는 페이지 집합
- **Thrashing**: 메모리 부족으로 과도한 스와핑이 발생하는 현상
- **해결**: 프로세스 수 감소, 메모리 추가

### 메모리 매핑

| 매핑 타입 | Shared | Private |
|:---:|:---|:---|
| **File-backed** | 프로세스 간 파일 공유 | 실행 파일 로딩, 공유 라이브러리 |
| **Anonymous** | IPC용 공유 메모리 | malloc 힙, 스택 |

**Copy-on-Write**는 `fork()`와 Private Mapping의 효율성을 크게 향상시킨다.

---

*다음 글: [[CS330] 7. 동기화 - Lock과 Condition Variable](/posts/cs330/7-locks-condition-variables/)*
