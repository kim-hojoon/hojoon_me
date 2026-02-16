---
title: "[CS330] 3. 스레드와 CPU 스케줄링"
date: 2023-12-03
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "Thread", "Scheduling", "FIFO", "SJF", "Round Robin"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "스레드의 개념과 다양한 스레딩 모델(1:1, N:1, M:N), 그리고 CPU 스케줄링 알고리즘(FIFO, SJF, STCF, Round Robin)을 비교합니다."
aliases: ["/posts/cs330-3-thread-scheduling/"]
ShowToc: true
TocOpen: true
---

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

*이전 글: [[CS330] 2. 인터럽트, 시스템 콜과 프로세스](/posts/cs330/2-interrupt-syscall-process/)*

## 1. 스레드(Thread)의 필요성

### I/O 병목 현상과 동시성(Concurrency)

과거에는 메모리와 스토리지가 프로세서보다 빠른 시대가 있었다. 당시에는 CPU가 병목(bottleneck)이었고, RAM과 Storage는 CPU가 연산을 마치기를 기다려야 했다.

하지만 무어의 법칙(Moore's Law)에 따라 프로세서의 성능이 메모리나 스토리지보다 훨씬 빠르게 발전하면서, 이제는 **CPU가 I/O를 기다리는 상황**이 되었다. 즉, 성능 병목이 CPU에서 I/O로 이동한 것이다.

이 문제를 해결하기 위해 운영체제는 **동시성(Concurrency)**을 지원해야 한다. 한 작업이 I/O를 수행하는 동안 CPU가 놀지 않고 다른 작업을 처리할 수 있도록 하는 것이다. 이것이 바로 **병렬 컴퓨팅(Parallel Computing)**의 핵심 아이디어다.

### 프로세스의 한계

프로세스는 프로그램을 실행하기 위한 훌륭한 추상화 개념이다. 운영체제는 프로세스 간 보호(protection)와 격리(isolation)를 제공한다. 하지만 프로세스에는 몇 가지 한계가 있다:

- **단일 프로세스는 멀티코어를 활용할 수 없다**
- **여러 프로세스를 사용하는 프로그램을 작성하기 매우 번거롭다**
- **새 프로세스를 생성하는 비용이 크다**
- **프로세스 간 통신 오버헤드가 크다**
- **프로세스 간 컨텍스트 스위칭 비용이 크다**

이러한 문제를 해결하기 위해 등장한 개념이 바로 **스레드(Thread)**다.

## 2. 스레드(Thread)란?

### 스레드의 정의

스레드는 **독립적으로 스케줄링 가능한 단일 실행 흐름(execution sequence)**이다.

- **프로세스**: 보호 단위(Protection Unit), 기계의 추상화(Abstraction of a machine)
- **스레드**: 실행 단위(Execution Unit), CPU 코어의 추상화(Abstraction of a CPU core)

스레드는 친숙한 프로그래밍 모델(single execution sequence)을 제공하면서도, 운영체제가 언제든지 실행하거나 중단(suspend)할 수 있다.

### 스레드의 주소 공간

한 프로세스 내의 스레드들은 메모리를 공유한다:

```text
Process 1's address space

PC (T2) ──┐
PC (T1) ──┤     Code         ← 스레드 간 공유
PC (T3) ──┘

              Data          ← 스레드 간 공유

              Heap          ← 스레드 간 공유

SP (T2) ──┐
           │     Stack (T2)  ← 스레드마다 독립
SP (T3) ──┤
           │     Stack (T3)  ← 스레드마다 독립
SP (T1) ──┘
              Stack (T1)  ← 스레드마다 독립
```

- **공유 자원**: Code, Data, Heap 영역
- **독립 자원**: Stack, Program Counter(PC), Stack Pointer(SP), CPU 레지스터

각 스레드는 자신만의 스택을 가지므로, 함수 호출 시 사용하는 지역 변수는 스레드마다 독립적이다.

### 프로세스 vs 스레드 비교

|  | 프로세스 | 스레드 |
|---|---|---|
| **컨텍스트 스위치 오버헤드** | 높음 (CPU state + Memory/IO state) | 낮음 (CPU state만) |
| **생성 비용** | 높음 | 낮음 |
| **보호** | CPU: Yes, Memory/IO: Yes | CPU: Yes, Memory/IO: No |
| **공유 오버헤드** | 높음 (최소 1회 컨텍스트 스위치 필요) | 낮음 (스레드 스위치 오버헤드만) |

스레드는 프로세스에 비해 **컨텍스트 스위칭이 빠르고 생성 비용이 적지만**, 메모리와 I/O 자원에 대한 보호가 없다는 단점이 있다.

## 3. 스레드 구현 모델

운영체제는 다양한 방식으로 스레드를 구현할 수 있다.

### 1:1 모델 (One-to-One, Kernel-level Threading)

각 사용자 수준 스레드가 하나의 커널 스레드로 매핑된다.

```text
user thread       user thread       user thread       user thread
    |                 |                 |                 |
    |                 |                 |                 |
    ↓                 ↓                 ↓                 ↓
[ kernel ]        [ kernel ]        [ kernel ]        [ kernel ]
[ thread ]        [ thread ]        [ thread ]        [ thread ]
```

**장점**:
- 가장 널리 사용되는 모델
- 멀티코어 활용 가능
- 한 스레드가 블록되어도 다른 스레드 실행 가능

**단점**:
- 스레드 생성 및 관리에 시스템 콜 필요
- 커널이 모든 스레드를 스케줄링해야 함
- 여전히 프로세스보다는 저렴하지만 비용이 있음

**구현 예**: Windows XP/7/10, OS/2, Linux, Solaris 9+

### N:1 모델 (Many-to-One, User-level Threading)

여러 사용자 수준 스레드가 하나의 커널 스레드로 매핑된다.

```text
user thread    user thread    user thread    user thread
    \              |              |             /
     \             |              |            /
      \            |              |           /
       \           |              |          /
        \          |              |         /
         \         |              |        /
          \        |              |       /
           \       |              |      /
            \      |              |     /
             \     |              |    /
              \    |              |   /
               \   |              |  /
                \  |              | /
                 \ |              |/
                  [ kernel thread ]
```

**장점**:
- 스레드 생성 및 전환이 매우 빠름 (10~100배 빠름)
- 이식성이 높음
- 애플리케이션에 맞게 튜닝 가능

**단점**:
- 멀티코어 CPU를 활용할 수 없음
- 한 스레드가 블록되면 전체 프로세스가 블록됨
- 대부분의 경우 비협조적 스케줄링(non-preemptive scheduling)에 의존
  - 선점형 스케줄링은 Unix 시그널로 에뮬레이션 가능하지만 복잡함
- OS가 사용자 수준 스레드를 모르기 때문에 잘못된 스케줄링 결정을 내릴 수 있음
  - 예: 유휴 스레드만 있는 프로세스를 스케줄링
  - 예: I/O를 시작한 스레드 때문에 전체 프로세스 블록
  - 예: 락을 보유한 스레드를 언스케줄링

**구현 예**: Solaris Green Threads, GNU Portable Threads

### M:N 모델 (Many-to-Many, Hybrid Threading)

여러 사용자 수준 스레드를 여러 커널 스레드로 매핑한다.

```text
user thread    user thread    user thread    user thread
    \              |              |             /
     \             |              |            /
      \            |              |           /
       \           |              |          /
        \          |              |         /
         \         |              |        /
          \________|______________|_______/
                   |              |
                   |              |
                   ↓              ↓
               [ kernel ]    [ kernel ]    [ kernel ]
               [ thread ]    [ thread ]    [ thread ]
```

**장점**:
- OS가 충분한 수의 커널 스레드를 생성할 수 있음
- 사용자 수준 스레드의 빠른 생성 및 전환
- 멀티코어 활용 가능

**단점**:
- 구현이 복잡함
- 커널과 스레드 관리자 간의 조율 필요

**구현 예**: Solaris (v9 이전), IRIX, HP-UX, Tru64, Go 언어의 goroutines

### Linux의 스레드 구현

Linux는 **task**라는 기본 단위를 사용한다. `fork()` 및 `exec()`만 사용하는 프로그램에서 task는 프로세스와 동일하다.

Linux는 1:1 모델을 사용하며, `clone()` 시스템 콜을 통해 각 애플리케이션 스레드마다 task를 생성한다.

**Linux threads**는 하나 이상의 자원을 공유할 수 있는 별도의 task들이다. `clone()`에서 자원을 선택적으로 공유할 수 있다:
- `CLONE_VM`: 가상 메모리
- `CLONE_FS`: 파일 시스템
- `CLONE_FILES`: 파일 디스크립터
- `CLONE_SIGHAND`: 시그널 핸들러 등

**POSIX threads**는 단일 프로세스 내에서 하나 이상의 스레드를 포함한다. CPU 레지스터, 사용자 스택, 블록된 시그널 마스크는 스레드별로 고유하며, 다른 모든 자원은 프로세스 전역이다.

## 4. CPU 스케줄링 기초

### Policy vs Mechanism

CPU 스케줄링에는 두 가지 핵심 개념이 있다:

- **Policy (정책)**: *무엇을* 해야 하는가?
  - 어떤 프로세스를 다음에 실행할 것인가?
  - 언제 전환할 것인가? 누구에게 전환할 것인가?

- **Mechanism (메커니즘)**: *어떻게* 할 것인가?
  - 여러 프로세스를 동시에 실행하는 방법은?
  - 구현의 도구

정책과 메커니즘을 분리하는 것은 운영체제 설계의 핵심 원칙이다. 이를 통해 모듈성 있고 확장 가능한 OS를 구축할 수 있다.

### 스케줄링 메트릭

스케줄링 알고리즘의 성능을 측정하는 지표:

- **Turnaround time (반환 시간)**: 작업이 완료되는 데 걸리는 시간
  - T_turnaround = T_finish - T_arrival

- **Response time (응답 시간)**: 요청부터 첫 응답까지의 시간
  - T_response = T_firstrun - T_arrival

### 프로세스 상태 전이

```text
                    scheduler
        admitted     dispatch       exit
  new ───────────> ready ──────────────> terminated
                     ↑  ↑
                     |   \_________
                     |     interrupt \
                     |               \
             I/O or event            running
             completion               /
                     |               /
                     |    I/O or   /
                     |  event wait/
                     |           /
                     ↓          ↓
                   waiting
```

- **New**: 프로세스가 생성됨
- **Ready**: 프로세스가 CPU를 할당받기를 기다림
- **Running**: 프로세스가 CPU에서 실행 중
- **Waiting (Blocked)**: 프로세스가 I/O나 이벤트 완료를 기다림
- **Terminated**: 프로세스 실행 종료

### 선점형(Preemptive) vs 비선점형(Non-preemptive) 스케줄링

- **비선점형(Non-preemptive) 스케줄링**:
  - 스케줄러가 실행 중인 프로세스가 자발적으로 CPU를 양보하기를 기다림
  - 프로세스가 협조적이어야 함

- **선점형(Preemptive) 스케줄링**:
  - 스케줄러가 프로세스를 인터럽트하고 강제로 컨텍스트 스위치 수행
  - 더 나은 반환 시간을 제공하는 경우가 많음
  - 대부분의 현대 OS는 선점형 스케줄링 사용

## 5. 기본 스케줄링 알고리즘

초기 가정 (나중에 하나씩 완화):
1. 각 작업은 동일한 시간 동안 실행됨
2. 모든 작업이 동시에 도착함
3. 한번 시작되면, 각 작업은 완료될 때까지 실행됨
4. 모든 작업은 CPU만 사용 (I/O 없음)
5. 각 작업의 실행 시간을 미리 알고 있음

### FIFO (First In, First Out)

가장 먼저 도착한 작업을 먼저 처리한다. FCFS (First Come, First Served)라고도 한다.

**예제**: A, B, C 작업이 순서대로 도착, 각각 10초 실행

```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
       |---|----|----|----|----|----|----|----|----|----|----|----|
       [AAAAAAAAAAA][BBBBBBBBBBBB][CCCCCCCCCCCC]
```

평균 반환 시간 = (10 + 20 + 30) / 3 = 20초

**문제점: Convoy Effect**

가정 1을 완화하면 문제가 발생한다. A가 100초, B와 C가 각각 10초 실행된다면:

```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
       |---|----|----|----|----|----|----|----|----|----|----|----|
       [AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA][BBB][CCC]
```

평균 반환 시간 = (100 + 110 + 120) / 3 = 110초

짧은 작업들이 긴 작업 뒤에서 기다리면 평균 반환 시간이 크게 증가한다.

### SJF (Shortest Job First)

가장 짧은 작업을 먼저 실행한다. **비선점형** 스케줄러다.

**예제**: A(100초), B(10초), C(10초)를 SJF로 스케줄링

```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
       |---|----|----|----|----|----|----|----|----|----|----|----|
       [BBB][CCC][AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA]
```

평균 반환 시간 = (10 + 20 + 120) / 3 = 50초

FIFO보다 훨씬 개선되었다!

**문제점: 늦게 도착하는 작업**

가정 2를 완화하면 문제가 발생한다. A가 t=0에 도착하고, B와 C가 t=10에 도착한다면:

```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
       |---|----|----|----|----|----|----|----|----|----|----|----|
       [AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA][BBB][CCC]
       ↑
     B,C 도착
```

평균 반환 시간 = (100 + (110-10) + (120-10)) / 3 = 103.33초

A가 실행 중일 때 B와 C가 도착했지만, 비선점형 SJF는 A가 완료될 때까지 기다려야 한다.

### STCF (Shortest Time-to-Completion First)

SJF에 **선점(preemption)**을 추가한 알고리즘이다. PSJF(Preemptive Shortest Job First)라고도 한다.

새 작업이 시스템에 들어오면:
- 남은 작업들과 새 작업의 남은 시간을 비교
- 남은 시간이 가장 적은 작업을 스케줄링

**예제**: A가 t=0에 도착(100초), B와 C가 t=10에 도착(각 10초)

```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
       |---|----|----|----|----|----|----|----|----|----|----|----|
       [AAA][BBB][CCC][AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA]
            ↑
          B,C 도착, A 선점
```

평균 반환 시간 = (120-0) + (20-10) + (30-10)) / 3 = 50초

STCF는 **반환 시간(turnaround time) 최적화**에 매우 효과적이다!

## 6. Round Robin (RR)

STCF의 문제점은 응답 시간(response time)이다. 가정 4를 완화하여 대화형 작업을 고려하면, 사용자는 첫 응답까지의 시간을 중요하게 생각한다.

Round Robin은 **응답 시간(response time) 최적화**를 목표로 한다.

### RR의 동작 방식

- Run queue를 원형 FIFO 큐로 취급
- 각 작업에 **타임 슬라이스(time slice)** 또는 **스케줄링 퀀텀(scheduling quantum)** 부여
  - 보통 타이머 인터럽트 주기의 배수
  - 일반적으로 10~100ms
- 한 작업을 타임 슬라이스만큼 실행한 후 run queue의 다음 작업으로 전환
- **선점형** 또는 **비선점형** 모두 가능
- **기아(starvation) 문제가 없음**

### SJF vs RR 비교

**예제**: A, B, C 작업이 t=0에 도착, 각각 30초 실행

**SJF (비선점형)**:
```text
Time:  0   10   20   30   40   50   60   70   80   90
       |---|----|----|----|----|----|----|----|----|----|
       [AAAAAAAAAAAA][BBBBBBBBBBBB][CCCCCCCCCCCC]
```
- T_turnaround = (30 + 60 + 90) / 3 = 60초
- T_response = (0 + 30 + 60) / 3 = 30초

**Round Robin (타임 슬라이스 = 10초)**:
```text
Time:  0   10   20   30   40   50   60   70   80   90
       |---|----|----|----|----|----|----|----|----|----|
       [AAA][BBB][CCC][AAA][BBB][CCC][AAA][BBB][CCC]
```
- T_turnaround = (70 + 80 + 90) / 3 = 80초
- T_response = (0 + 10 + 20) / 3 = 10초

RR은 반환 시간은 더 길지만, **응답 시간이 훨씬 짧다**!

### 타임 슬라이스 길이의 중요성

- **짧은 타임 슬라이스**:
  - 더 나은 응답 시간
  - 컨텍스트 스위칭 비용이 전체 성능을 지배할 수 있음

- **긴 타임 슬라이스**:
  - 스위칭 비용을 분산(amortize)
  - 더 나쁜 응답 시간

타임 슬라이스 길이 결정은 시스템 설계자에게 **트레이드오프(trade-off)**를 제시한다.

## 7. 우선순위 스케줄링과 MLFQ

### 정적 우선순위 스케줄링

각 작업에 (정적) 우선순위를 부여하고, 가장 높은 우선순위의 작업을 다음에 실행한다. 선점형 또는 비선점형 모두 가능하다.

**문제점: 기아(Starvation)**
- 높은 우선순위 작업이 계속 공급되면 낮은 우선순위 작업은 영원히 실행되지 못할 수 있음

### MLFQ (Multi-Level Feedback Queue)

MLFQ는 범용 워크로드를 위한 현대적 스케줄링 알고리즘이다.

**핵심 아이디어**:
- 우선순위 레벨마다 별도의 큐를 유지
- 큐 간에는 우선순위 스케줄링, 같은 큐 내에서는 Round Robin
- **관찰된 동작에 따라 우선순위를 동적으로 변경**

**기본 규칙**:
- **Rule 1**: Priority(A) > Priority(B)이면, A를 실행 (B는 실행하지 않음)
- **Rule 2**: Priority(A) = Priority(B)이면, A와 B를 RR로 실행

- **Rule 3**: 작업이 시스템에 들어올 때, 최상위 우선순위(topmost queue)에 배치
- **Rule 4**: 작업이 주어진 레벨에서 타임 할당량을 모두 사용하면, 우선순위가 낮아짐 (한 큐 아래로 이동)
- **Rule 5**: 일정 시간 S가 지나면, 시스템의 모든 작업을 최상위 큐로 이동 (우선순위 부스트)

```text
Queue                    Runable processes
headers                 ____________________

Priority 4         |____|____|____|

Priority 3         |____|____|____|____|

Priority 2         |____|

Priority 1         |
```

**MLFQ의 장점**:
- 대화형 작업: 짧게 실행되고, 빠른 응답 시간 필요 → 높은 우선순위 유지
- CPU 집약적 작업: 긴 CPU 시간 필요, 응답 시간 덜 중요 → 낮은 우선순위로 이동
- 작업의 CPU 사용량에 대한 사전 지식이 필요 없음
- 과거를 통해 미래를 예측 (분기 예측이나 캐시 알고리즘과 유사)

## 8. I/O와 스케줄링

가정 4를 완화하여 I/O를 고려하면:

- CPU burst와 I/O burst를 독립적인 작업으로 취급
- CPU와 I/O의 연산을 중첩(overlap)시킬 수 있음

**예제**: A (대화형) + B (CPU 집약적)

**STCF (I/O 인식 없음)**:
```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
CPU:   [AAA][AAA][AAA][AAA][BBBB][BBBB][BBBB][BBBB]
Disk:      [AAA]   [AAA]   [AAA]
```
A의 각 CPU burst 후 I/O 수행

**I/O-aware STCF**:
```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
CPU:   [AAA][BBB][AAA][BBB][AAA][BBB][AAA][BBB]
Disk:      [AAA]   [AAA]   [AAA]
```
A가 I/O를 수행하는 동안 B가 CPU 사용

I/O를 인식하는 스케줄링은 CPU와 I/O 장치를 모두 활용하여 전체 시스템 효율성을 높인다.

## 정리

- **스레드**는 프로세스보다 가볍고 빠른 실행 단위로, I/O 병목 문제를 해결하기 위한 동시성을 제공한다
- **스레딩 모델**은 1:1(커널 스레드), N:1(유저 스레드), M:N(하이브리드)로 나뉘며, 각각 장단점이 있다
- **CPU 스케줄링**은 정책(Policy)과 메커니즘(Mechanism)을 분리하여 설계한다
- **FIFO**는 구현이 간단하지만 Convoy Effect로 인해 평균 반환 시간이 나쁠 수 있다
- **SJF/STCF**는 반환 시간을 최적화하지만, 작업의 실행 시간을 미리 알아야 한다
- **Round Robin**은 응답 시간을 최적화하며, 타임 슬라이스 길이가 성능에 큰 영향을 미친다
- **MLFQ**는 작업의 동작을 관찰하여 우선순위를 동적으로 조정하는 실용적인 알고리즘이다

*다음 글: [[CS330] 4. 가상 메모리와 주소 변환](/posts/cs330/4-virtual-memory-address-translation/)*
