---
title: "[CS330] 8. 동시성 버그와 교착 상태"
date: 2023-12-08
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "Deadlock", "Concurrency Bugs", "Synchronization"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "Atomicity violation, Order violation 등 동시성 버그의 유형과 교착 상태(Deadlock)의 4가지 조건, 예방·회피·탐지 전략을 정리합니다."
aliases: ["/posts/cs330-8-concurrency-bugs-deadlock/"]
ShowToc: true
TocOpen: true
---

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

*이전 글: [[CS330] 7. 동기화 - Lock과 Condition Variable](/posts/cs330/7-locks-condition-variables/)*

---

## 들어가며

동기화 메커니즘을 잘못 사용하면 시스템 전체를 멈추게 할 수 있습니다. 이번 글에서는 실제 프로그램에서 발생하는 동시성 버그(Concurrency Bugs)의 유형과 교착 상태(Deadlock) 문제를 다룹니다.

## 동시성 버그의 실제 연구 결과

2008년 Lu et al.의 연구 "Learning from Mistakes — A Comprehensive Study on Real World Concurrency Bug Characteristics"에서는 MySQL, Apache, Mozilla, OpenOffice 등 4개의 대규모 오픈소스 프로젝트에서 발생한 실제 동시성 버그 105개를 분석했습니다.

연구 결과, 동시성 버그는 크게 두 가지로 나뉩니다.

- **Non-Deadlock Bugs (비교착 버그)**: 약 97개 (전체의 약 92%)
  - Atomicity Violation (원자성 위반)
  - Order Violation (순서 위반)
- **Deadlock Bugs (교착 버그)**: 약 8개 (전체의 약 8%)

의외로 Deadlock 버그의 비율은 낮지만, 발생하면 시스템 전체를 멈추게 할 수 있어 매우 치명적입니다.

## Non-Deadlock Bugs

### 1. Atomicity Violation (원자성 위반)

Atomicity violation은 원자적으로(atomically) 실행되어야 할 코드 영역이 제대로 보호되지 않아 발생하는 버그입니다.

**MySQL의 실제 예제:**

```c
// Thread 1
if (thd->proc_info) {
    fputs(thd->proc_info, ...);
}
```

```c
// Thread 2
thd->proc_info = NULL;
```

만약 Thread 1이 `if` 조건을 확인한 직후, `fputs`를 실행하기 전에 Thread 2가 `proc_info`를 NULL로 설정하면 어떻게 될까요? Thread 1은 NULL 포인터를 역참조하게 되어 버그가 발생합니다.

```text
Thread 1                    Thread 2
--------                    --------
if (thd->proc_info) ✓
                            thd->proc_info = NULL;
    fputs(NULL, ...)  ← Buggy!
```

**해결 방법:** Lock을 사용해 S1과 S2를 원자적으로 실행되도록 보호합니다.

```c
// Thread 1
pthread_mutex_lock(&info_lock);
if (thd->proc_info) {
    fputs(thd->proc_info, ...);
}
pthread_mutex_unlock(&info_lock);

// Thread 2
pthread_mutex_lock(&info_lock);
thd->proc_info = NULL;
pthread_mutex_unlock(&info_lock);
```

### 2. Order Violation (순서 위반)

Order violation은 특정 코드가 반드시 어떤 순서로 실행되어야 하는데, 그 순서가 보장되지 않아 발생하는 버그입니다.

**Mozilla의 실제 예제:**

```c
// Thread 1 (init)
void init() {
    ...
    mThread = PR_CreateThread(mMain, ...);
    ...
}

// Thread 2 (mMain)
void mMain(...) {
    ...
    mState = mThread->State;
    ...
}
```

Thread 1이 `mThread`를 초기화하기 전에 Thread 2가 `mThread->State`에 접근하면 버그가 발생합니다.

```text
Thread 1                           Thread 2
--------                           --------
init() {
    ...                            mMain() {
                                       mState = mThread->State;  ← Bug!
    mThread = PR_CreateThread(...);    (mThread not initialized yet)
}                                  }
```

**해결 방법:** Condition Variable을 사용해 순서를 보장합니다.

```c
pthread_mutex_t mtLock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t mtCond = PTHREAD_COND_INITIALIZER;
int mtInit = 0;

// Thread 1
void init() {
    ...
    mThread = PR_CreateThread(mMain, ...);

    pthread_mutex_lock(&mtLock);
    mtInit = 1;
    pthread_cond_signal(&mtCond);
    pthread_mutex_unlock(&mtLock);
}

// Thread 2
void mMain(...) {
    pthread_mutex_lock(&mtLock);
    while (mtInit == 0)
        pthread_cond_wait(&mtCond, &mtLock);
    pthread_mutex_unlock(&mtLock);

    mState = mThread->State;  // Safe!
}
```

## 교착 상태 (Deadlock)

### Deadlock이란?

**Deadlock**은 두 개 이상의 스레드가 서로가 가진 자원을 기다리면서 영원히 진행하지 못하는 상태입니다.

### Deadlock 시나리오 예제

가장 전형적인 교착 상태는 두 개의 락(lock)을 서로 다른 순서로 획득하려 할 때 발생합니다.

```c
// Thread A
pthread_mutex_lock(&lock1);
pthread_mutex_lock(&lock2);
// Critical section
pthread_mutex_unlock(&lock2);
pthread_mutex_unlock(&lock1);

// Thread B
pthread_mutex_lock(&lock2);
pthread_mutex_lock(&lock1);
// Critical section
pthread_mutex_unlock(&lock1);
pthread_mutex_unlock(&lock2);
```

**Deadlock 발생 시나리오:**

```text
Time    Thread A                 Thread B
----    --------                 --------
t1      lock1.acquire() ✓
t2                               lock2.acquire() ✓
t3      lock2.acquire()          (waiting...)
        (waiting...)
t4                               lock1.acquire()
                                 (waiting...)

→ Thread A는 lock2를 기다리고, Thread B는 lock1을 기다림
→ 영원히 진행 불가 (Deadlock!)
```


### Deadlock의 필요 조건 (Coffman Conditions)

Deadlock이 발생하려면 다음 **4가지 조건이 동시에** 만족되어야 합니다. 이를 Coffman conditions라고 합니다.

1. **Mutual Exclusion (상호 배제)**: 자원은 한 번에 하나의 스레드만 사용 가능
2. **Hold and Wait (보유 및 대기)**: 자원을 보유한 채로 다른 자원을 대기
3. **No Preemption (비선점)**: 자원을 강제로 빼앗을 수 없음
4. **Circular Wait (순환 대기)**: 스레드들이 순환 형태로 자원을 대기 (T1→T2→T3→T1)

이 4가지 조건이 **모두** 만족되어야 deadlock이 발생하므로, 하나라도 깨뜨리면 예방할 수 있습니다.

## Deadlock 예방 (Prevention)

Deadlock prevention은 4가지 조건 중 **최소 하나를 원천적으로 제거**하는 방법입니다.

### 1. Circular Wait 깨기: Lock Ordering

모든 락에 전체 순서(total ordering)를 부여하고, 항상 정해진 순서대로만 락을 획득하도록 합니다.

```c
// 올바른 방법: 모든 스레드가 같은 순서로 획득
// Thread A
pthread_mutex_lock(&lock1);  // 순서: 1번 먼저
pthread_mutex_lock(&lock2);  // 순서: 2번 나중에
// ...
pthread_mutex_unlock(&lock2);
pthread_mutex_unlock(&lock1);

// Thread B
pthread_mutex_lock(&lock1);  // 순서: 1번 먼저 (동일!)
pthread_mutex_lock(&lock2);  // 순서: 2번 나중에 (동일!)
// ...
pthread_mutex_unlock(&lock2);
pthread_mutex_unlock(&lock1);
```

**장점:** 단순하고 효과적 | **단점:** 복잡한 시스템에서는 전역 순서 관리가 어려움

### 2. Hold and Wait 깨기: All-at-Once Locking

필요한 모든 락을 한 번에 획득하거나, 하나라도 실패하면 모두 포기합니다.

```c
pthread_mutex_lock(&master_lock);
pthread_mutex_lock(&lock1);
pthread_mutex_lock(&lock2);
pthread_mutex_unlock(&master_lock);
// Critical section
pthread_mutex_unlock(&lock2);
pthread_mutex_unlock(&lock1);
```

**장점:** 확실한 방지 | **단점:** 동시성 감소, 불필요한 대기 발생

### 3. No Preemption 깨기: trylock() 사용

락 획득 실패 시 이미 획득한 락을 반환하고 재시도합니다.

```c
while (1) {
    pthread_mutex_lock(&lock1);
    if (pthread_mutex_trylock(&lock2) == 0)
        break;  // 성공
    pthread_mutex_unlock(&lock1);  // 실패: 재시도
    usleep(rand() % 100);
}
// Critical section
pthread_mutex_unlock(&lock2);
pthread_mutex_unlock(&lock1);
```

**장점:** Deadlock 완전 방지 | **단점:** Livelock 가능성, 성능 오버헤드

### 4. Mutual Exclusion 깨기: Lock-Free 자료구조

Lock 없이 Atomic operation (Compare-And-Swap 등)을 사용합니다.

```c
// Lock-free stack using CAS
void push(int value) {
    node_t* new_node = malloc(sizeof(node_t));
    new_node->value = value;
    do {
        new_node->next = top;
    } while (!compare_and_swap(&top, new_node->next, new_node));
}
```

**장점:** Deadlock 원천 차단, 높은 동시성 | **단점:** 구현/디버깅 매우 어려움

## Deadlock 회피 (Avoidance)

시스템이 **unsafe state로 진입하지 않도록** 자원 할당을 제어합니다.

### Banker's Algorithm (Dijkstra)

각 프로세스의 최대 자원 요구량을 미리 알고 있을 때, safe state를 유지하도록 할당합니다.

- **Safe State**: 모든 프로세스가 완료 가능한 순서(safe sequence)가 존재
- **Unsafe State**: Safe sequence 없음 (Deadlock 가능)

**예제: 3개의 프로세스, 12개의 자원**

```text
Process    Maximum    Allocation    Need    Available
-------    -------    ----------    ----    ---------
  P0         10           5          5
  P1          4           2          2         3
  P2          9           2          7
```

현재 상태에서 safe sequence를 찾을 수 있을까요?

1. P1 실행 가능 (Need=2, Available=3) → P1 완료 후 Available=5
2. P0 실행 가능 (Need=5, Available=5) → P0 완료 후 Available=10
3. P2 실행 가능 (Need=7, Available=10) → P2 완료

따라서 **Safe sequence: P1 → P0 → P2** 가 존재하므로 현재 상태는 safe합니다.

만약 P2에게 자원 1개를 추가로 할당하면?

```text
Process    Maximum    Allocation    Need    Available
-------    -------    ----------    ----    ---------
  P0         10           5          5
  P1          4           2          2         2  ← Changed!
  P2          9           3          6         ↑
```

- P1 실행 가능 (Need=2, Available=2) → P1 완료 후 Available=4
- P0 실행 불가 (Need=5, Available=4)
- P2 실행 불가 (Need=6, Available=4)

**Unsafe state!** 따라서 P2의 추가 요청을 거부합니다.

**장점:** Deadlock 완전 방지 | **단점:** 최대 자원량 사전 지식 필요, 실시간 계산 오버헤드, **실무에서 거의 미사용**

## Deadlock 탐지 (Detection)와 복구 (Recovery)

Deadlock 발생 후 **탐지하고 복구**하는 방법입니다.

### Resource Allocation Graph

자원 할당 그래프에서 순환(cycle)을 탐지합니다. 순환이 있으면 deadlock입니다.

### Recovery 방법

#### 1. 프로세스 종료
- **전체 종료**: 확실하지만 비용 큼
- **점진적 종료**: 순환이 깨질 때까지 하나씩 종료 (우선순위/실행시간/자원 사용량 기준)

#### 2. 자원 선점
자원을 강제 회수하고 프로세스를 rollback합니다. Starvation 방지 필요.

**장점:** 높은 동시성 | **단점:** 탐지/복구 오버헤드, 복잡한 구현

## 실무에서의 접근: Ostrich Algorithm

대부분의 실제 OS(Linux, Windows)는 **Ostrich Algorithm**(타조 알고리즘)을 사용합니다: "머리를 모래에 묻고 무시하기"

### 왜 무시하는가?

1. **Deadlock은 드물다**: 잘 설계된 프로그램에서는 거의 발생하지 않음
2. **예방/회피/탐지 비용이 크다**: 성능 오버헤드, 구현 복잡도
3. **실용적 해결책 존재**: 프로세스 종료(Ctrl+C), 시스템 재시작, Watchdog 타이머

### 실무 권장사항

1. **Lock ordering**: 일관된 순서로 락 획득, 중첩 락 최소화
2. **감시 메커니즘**: Timeout, Health check
3. **테스트와 코드 리뷰**: Static analysis 도구 활용

## 정리

### 동시성 버그 요약

| 버그 유형 | 설명 | 해결 방법 |
|---------|------|----------|
| Atomicity Violation | 원자적이어야 할 연산이 깨짐 | Lock으로 critical section 보호 |
| Order Violation | 실행 순서 보장이 깨짐 | Condition Variable로 순서 강제 |
| Deadlock | 순환 대기로 진행 불가 | Prevention / Avoidance / Detection |

### Deadlock 대응 전략 비교

| 전략 | 방법 | 장점 | 단점 | 실무 활용 |
|-----|------|------|------|----------|
| **Prevention** | 4가지 조건 중 하나 제거 | 확실한 방지 | 동시성 감소 | ★★★ 많이 사용 |
| **Avoidance** | Safe state 유지 | 높은 자원 활용 | 구현 복잡, 사전 정보 필요 | ☆ 거의 미사용 |
| **Detection** | 주기적 탐지 후 복구 | 높은 동시성 | 탐지/복구 비용 | ★ 특수 상황만 |
| **Ostrich** | 무시 | 오버헤드 없음 | Deadlock 방치 | ★★★ 대부분의 OS |

### 핵심 요점

1. **Deadlock의 4가지 조건**을 모두 만족해야 발생
2. **Lock ordering**이 가장 실용적인 예방 기법
3. **Banker's Algorithm**은 이론적으로만 사용, 실무에서는 거의 미사용
4. **Ostrich Algorithm**: 대부분의 OS는 deadlock을 무시
5. **Prevention > Detection > Avoidance** 순으로 실용성 높음

동시성 프로그래밍의 핵심은 **간단하게 유지하기(Keep It Simple)**입니다. 명확한 락 순서와 최소한의 중첩이 최선입니다.

---

*다음 글: [[CS330] 9. 저장장치와 파일 시스템](/posts/cs330/9-storage-file-system/)*
