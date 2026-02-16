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

무어의 법칙(Moore's Law)에 따라 프로세서 성능이 메모리나 스토리지보다 빠르게 발전하면서, 성능 병목이 CPU에서 I/O로 이동했다. 이제는 **CPU가 I/O를 기다리는 상황**이다.

이 문제를 해결하기 위해 운영체제는 **동시성(Concurrency)**을 지원해야 한다. 한 작업이 I/O를 수행하는 동안 CPU가 다른 작업을 처리할 수 있도록 하는 것이 **병렬 컴퓨팅(Parallel Computing)**의 핵심이다.

### 프로세스의 한계

프로세스는 훌륭한 추상화지만 몇 가지 한계가 있다:

- 단일 프로세스는 멀티코어를 활용할 수 없음
- 여러 프로세스를 사용하는 프로그램 작성이 번거로움
- 프로세스 생성, 통신, 컨텍스트 스위칭 비용이 큼

이러한 문제를 해결하기 위해 등장한 개념이 **스레드(Thread)**다.

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
    ↓                 ↓                 ↓                 ↓
[ kernel ]        [ kernel ]        [ kernel ]        [ kernel ]
[ thread ]        [ thread ]        [ thread ]        [ thread ]
```

**장점**: 멀티코어 활용 가능, 한 스레드가 블록되어도 다른 스레드 실행 가능

**단점**: 스레드 생성 및 관리에 시스템 콜 필요, 커널이 모든 스레드를 스케줄링

**구현 예**: Windows, Linux, Solaris 9+

### N:1 모델 (Many-to-One, User-level Threading)

여러 사용자 수준 스레드가 하나의 커널 스레드로 매핑된다.

```text
user thread    user thread    user thread    user thread
    \              |              |             /
     \             |              |            /
      \____________|______________|___________/
                   |
                   ↓
              [ kernel thread ]
```

**장점**: 스레드 생성 및 전환이 매우 빠름 (10~100배), 이식성이 높음, 애플리케이션에 맞게 튜닝 가능

**단점**: 멀티코어 활용 불가, 한 스레드가 블록되면 전체 프로세스 블록, 대부분 비선점형 스케줄링 사용, OS가 사용자 스레드를 모르기 때문에 잘못된 스케줄링 결정 가능

**구현 예**: Solaris Green Threads, GNU Portable Threads

### M:N 모델 (Many-to-Many, Hybrid Threading)

여러 사용자 수준 스레드를 여러 커널 스레드로 매핑한다.

```text
user thread    user thread    user thread    user thread
    \              |              |             /
     \             |              |            /
      \____________|______________|___________/
                   |              |
                   ↓              ↓
               [ kernel ]    [ kernel ]    [ kernel ]
               [ thread ]    [ thread ]    [ thread ]
```

**장점**: 충분한 수의 커널 스레드 생성 가능, 사용자 수준 스레드의 빠른 생성 및 전환, 멀티코어 활용 가능

**단점**: 구현이 복잡함, 커널과 스레드 관리자 간의 조율 필요

**구현 예**: Solaris (v9 이전), Go 언어의 goroutines

### Linux의 스레드 구현

Linux는 **task**를 기본 단위로 사용하며, 1:1 모델을 채택한다. `clone()` 시스템 콜을 통해 각 애플리케이션 스레드마다 task를 생성한다.

`clone()`에서 자원을 선택적으로 공유할 수 있다: `CLONE_VM`(가상 메모리), `CLONE_FS`(파일 시스템), `CLONE_FILES`(파일 디스크립터), `CLONE_SIGHAND`(시그널 핸들러) 등.

**POSIX threads**는 CPU 레지스터, 스택, 시그널 마스크는 스레드별로 고유하며, 나머지 자원은 프로세스 전역으로 공유한다.

## 4. CPU 스케줄링 기초

### Policy vs Mechanism

- **Policy (정책)**: 어떤 프로세스를 실행할 것인가? 언제 전환할 것인가?
- **Mechanism (메커니즘)**: 어떻게 구현할 것인가?

정책과 메커니즘을 분리하는 것은 운영체제 설계의 핵심 원칙이다.

### 스케줄링 메트릭

- **Turnaround time (반환 시간)**: T_turnaround = T_finish - T_arrival
- **Response time (응답 시간)**: T_response = T_firstrun - T_arrival

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

- **비선점형**: 프로세스가 자발적으로 CPU를 양보할 때까지 기다림
- **선점형**: 스케줄러가 프로세스를 강제로 인터럽트하고 컨텍스트 스위치 수행. 대부분의 현대 OS가 사용

## 5. 기본 스케줄링 알고리즘

초기 가정 (나중에 하나씩 완화):
1. 각 작업은 동일한 시간 동안 실행됨
2. 모든 작업이 동시에 도착함
3. 한번 시작되면, 각 작업은 완료될 때까지 실행됨
4. 모든 작업은 CPU만 사용 (I/O 없음)
5. 각 작업의 실행 시간을 미리 알고 있음

### FIFO (First In, First Out)

가장 먼저 도착한 작업을 먼저 처리한다. FCFS (First Come, First Served)라고도 한다.

**예제**: A, B, C 작업이 순서대로 도착, 각각 10초 실행 → 평균 반환 시간 = 20초

**문제점: Convoy Effect**

A가 100초, B와 C가 각각 10초 실행된다면:

```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
       |---|----|----|----|----|----|----|----|----|----|----|----|
       [AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA][BBB][CCC]
```

평균 반환 시간 = (100 + 110 + 120) / 3 = 110초. 짧은 작업이 긴 작업 뒤에서 기다리면 평균 반환 시간이 크게 증가한다.

### SJF (Shortest Job First)

가장 짧은 작업을 먼저 실행한다. **비선점형** 스케줄러다.

**예제**: A(100초), B(10초), C(10초) → 평균 반환 시간 = 50초 (FIFO보다 훨씬 개선!)

**문제점: 늦게 도착하는 작업**

A가 t=0에 도착하고, B와 C가 t=10에 도착한다면, 비선점형 SJF는 A가 완료될 때까지 기다려야 한다. 평균 반환 시간 = 103.33초

### STCF (Shortest Time-to-Completion First)

SJF에 **선점(preemption)**을 추가한 알고리즘이다. 새 작업이 시스템에 들어오면 남은 시간이 가장 적은 작업을 스케줄링한다.

**예제**: A가 t=0에 도착(100초), B와 C가 t=10에 도착(각 10초)

```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
       |---|----|----|----|----|----|----|----|----|----|----|----|
       [AAA][BBB][CCC][AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA]
            ↑
          B,C 도착, A 선점
```

평균 반환 시간 = 50초. STCF는 **반환 시간 최적화**에 매우 효과적이다!

## 6. Round Robin (RR)

STCF는 응답 시간(response time)이 좋지 않다. Round Robin은 **응답 시간 최적화**를 목표로 한다.

### RR의 동작 방식

- Run queue를 원형 FIFO 큐로 취급
- 각 작업에 **타임 슬라이스(time slice)** 부여 (보통 10~100ms)
- 타임 슬라이스만큼 실행 후 다음 작업으로 전환
- **기아(starvation) 문제가 없음**

### SJF vs RR 비교

A, B, C 작업이 t=0에 도착, 각각 30초 실행:

**SJF**: T_turnaround = 60초, T_response = 30초

**RR (타임 슬라이스 = 10초)**: T_turnaround = 80초, T_response = 10초

```text
Time:  0   10   20   30   40   50   60   70   80   90
       |---|----|----|----|----|----|----|----|----|----|
       [AAA][BBB][CCC][AAA][BBB][CCC][AAA][BBB][CCC]
```

RR은 반환 시간은 더 길지만, **응답 시간이 훨씬 짧다**!

### 타임 슬라이스 길이의 중요성

- **짧은 타임 슬라이스**: 더 나은 응답 시간, 하지만 컨텍스트 스위칭 비용 증가
- **긴 타임 슬라이스**: 스위칭 비용 감소, 하지만 응답 시간 증가

타임 슬라이스 길이는 시스템 설계자에게 **트레이드오프**를 제시한다.

## 7. 우선순위 스케줄링과 MLFQ

### 정적 우선순위 스케줄링

각 작업에 정적 우선순위를 부여하고, 가장 높은 우선순위 작업을 실행한다.

**문제점**: 높은 우선순위 작업이 계속 공급되면 낮은 우선순위 작업이 기아(Starvation) 상태에 빠짐

### MLFQ (Multi-Level Feedback Queue)

MLFQ는 **관찰된 동작에 따라 우선순위를 동적으로 변경**하는 현대적 스케줄링 알고리즘이다.

**핵심 규칙**:
- **Rule 1**: Priority(A) > Priority(B)이면, A를 실행
- **Rule 2**: Priority(A) = Priority(B)이면, A와 B를 RR로 실행
- **Rule 3**: 작업이 시스템에 들어올 때, 최상위 우선순위에 배치
- **Rule 4**: 작업이 타임 할당량을 모두 사용하면, 우선순위 낮아짐 (한 큐 아래로 이동)
- **Rule 5**: 일정 시간 S가 지나면, 모든 작업을 최상위 큐로 이동 (우선순위 부스트)

```text
Queue                    Runable processes
headers                 ____________________

Priority 4         |____|____|____|

Priority 3         |____|____|____|____|

Priority 2         |____|

Priority 1         |
```

**MLFQ의 장점**:
- 대화형 작업은 짧게 실행되어 높은 우선순위 유지 → 빠른 응답
- CPU 집약적 작업은 낮은 우선순위로 이동 → 배경 처리
- 작업의 CPU 사용량에 대한 사전 지식 불필요
- 과거 동작을 통해 미래를 예측

## 8. I/O와 스케줄링

CPU burst와 I/O burst를 독립적인 작업으로 취급하면 CPU와 I/O 연산을 중첩시킬 수 있다.

**예제**: A (대화형) + B (CPU 집약적)

**I/O-aware STCF**:
```text
Time:  0   10   20   30   40   50   60   70   80   90  100  110  120
CPU:   [AAA][BBB][AAA][BBB][AAA][BBB][AAA][BBB]
Disk:      [AAA]   [AAA]   [AAA]
```

A가 I/O를 수행하는 동안 B가 CPU를 사용하여 전체 시스템 효율성을 높인다.

## 정리

- **스레드**는 프로세스보다 가볍고 빠른 실행 단위로, I/O 병목 문제를 해결하기 위한 동시성을 제공한다
- **스레딩 모델**은 1:1(커널 스레드), N:1(유저 스레드), M:N(하이브리드)로 나뉘며, 각각 장단점이 있다
- **CPU 스케줄링**은 정책(Policy)과 메커니즘(Mechanism)을 분리하여 설계한다
- **FIFO**는 구현이 간단하지만 Convoy Effect로 인해 평균 반환 시간이 나쁠 수 있다
- **SJF/STCF**는 반환 시간을 최적화하지만, 작업의 실행 시간을 미리 알아야 한다
- **Round Robin**은 응답 시간을 최적화하며, 타임 슬라이스 길이가 성능에 큰 영향을 미친다
- **MLFQ**는 작업의 동작을 관찰하여 우선순위를 동적으로 조정하는 실용적인 알고리즘이다

*다음 글: [[CS330] 4. 가상 메모리와 주소 변환](/posts/cs330/4-virtual-memory-address-translation/)*
