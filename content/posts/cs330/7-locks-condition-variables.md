---
title: "[CS330] 7. 동기화 - Lock과 Condition Variable"
date: 2023-12-07
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "Lock", "Synchronization", "Condition Variable", "Semaphore"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "Race condition과 Critical section 문제부터 Lock 구현(Test-And-Set, Compare-And-Swap), Condition Variable, Semaphore까지 동기화 메커니즘을 정리합니다."
aliases: ["/posts/cs330-7-locks-condition-variables/"]
ShowToc: true
TocOpen: true
---

*이전 글: [[CS330] 6. 스와핑과 페이지 교체 정책](/posts/cs330/6-swapping-page-replacement/)*

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

멀티스레드 환경에서 여러 스레드가 공유 자원(shared resource)에 동시에 접근하면 어떤 일이 발생할까요? 이번 글에서는 동시성(concurrency) 문제의 핵심인 race condition과 이를 해결하기 위한 동기화 메커니즘인 Lock, Condition Variable, Semaphore를 살펴봅니다.

## 1. Race Condition과 동기화 문제

### 1.1 Race Condition 예제

전역 변수 `g`를 1씩 증가시키는 단순한 함수를 봅시다.

```c
extern long g;

void inc() {
    g++;
}
```

이 C 코드 한 줄은 어셈블리로 변환되면 여러 명령어가 됩니다.

```text
ld    a0, 0(s1)    # 메모리에서 g를 레지스터 a0로 로드
addi  a0, a0, 1    # a0에 1을 더함
sd    a0, 0(s1)    # a0의 값을 메모리에 저장
ret
```

두 스레드가 동시에 `inc()`를 실행하면 **인터리빙(interleaving)**이 발생합니다.

```text
Thread 1                    Thread 2
--------                    --------
ld    a0, 0(s1)            (a0 = 0)
addi  a0, a0, 1            (a0 = 1)
                           ld    a0, 0(s1)    (a0 = 0!)
                           addi  a0, a0, 1    (a0 = 1)
                           sd    a0, 0(s1)    (g = 1)
sd    a0, 0(s1)            (g = 1)
```

최종 결과는 2가 아니라 1입니다. 두 스레드의 작업 중 하나가 손실되었습니다. 이것이 **race condition**입니다.

### 1.2 Shared Resources

스레드 간에 공유되는 자원:

- **전역 변수(Global variables)**: 정적 데이터 세그먼트에 저장, 모든 스레드가 접근 가능
- **동적 객체(Heap objects)**: 힙에 저장, 포인터를 통해 공유
- **공유 메모리(Shared memory)**: 프로세스 간에도 공유 가능
- **지역 변수(Local variables)**: 공유되지 않음 (각 스레드의 스택에 독립적으로 존재)

### 1.3 Synchronization Problem

**동기화 문제(Synchronization Problem)**는 여러 스레드가 공유 자원에 동시 접근할 때 발생합니다.

- 프로그램의 출력이 비결정적(non-deterministic)으로 변함
- 같은 입력으로도 타이밍에 따라 결과가 달라짐
- 디버깅이 매우 어려움 (하이젠버그 버그, Heisenbugs)

이를 해결하기 위해 **동기화 메커니즘(synchronization mechanisms)**이 필요합니다.

## 2. Critical Section과 상호 배제

### 2.1 Critical Section

**Critical Section(임계 영역)**은 공유 자원에 접근하는 코드의 일부분입니다.

```c
ld    a0, 0(s1)    ┐
addi  a0, a0, 1    ├─ Critical Section
sd    a0, 0(s1)    ┘
```

Critical Section에 대한 요구사항:

- **원자성(Atomicity)**: Critical Section은 all-or-nothing으로 실행되어야 함
- **상호 배제(Mutual Exclusion)**: 한 번에 하나의 스레드만 Critical Section에 진입

### 2.2 Lock의 필요성

Lock은 상호 배제를 제공하는 메모리 내 객체입니다.

```c
int withdraw(account, amount)
{
    acquire(lock);           // Lock 획득
    balance = get_balance(account);
    balance = balance - amount;
    put_balance(account, balance);
    release(lock);           // Lock 해제
    return balance;
}
```

Lock을 사용하면 두 스레드가 동시에 Critical Section에 진입하지 못합니다.

### 2.3 동기화 요구사항

올바른 동기화를 위한 세 가지 정확성 기준:

1. **Mutual Exclusion (상호 배제)**: 한 번에 하나의 스레드만 Critical Section에 진입
2. **Progress (진행, Deadlock-free)**: 여러 스레드가 진입하려 할 때 반드시 하나는 진입 가능해야 함
3. **Bounded Waiting (제한된 대기, Starvation-free)**: 대기 중인 각 스레드는 언젠가는 반드시 진입할 수 있어야 함

추가 고려사항:

- **Fairness (공정성)**: 각 스레드가 Lock을 획득할 공정한 기회를 가져야 함
- **Performance (성능)**: Lock의 오버헤드가 작아야 하며, 멀티프로세서 환경에서도 효율적이어야 함

## 3. Lock 구현 방법

Lock을 올바르게 구현하려면 하드웨어의 도움이 필요합니다.

**나이브한 접근법의 문제:**
- **인터럽트 비활성화**: 멀티프로세서에서 작동하지 않음 (다른 CPU는 영향 없음)
- **단순 플래그**: `while (l->held)`와 `l->held = 1` 사이에 race condition 발생

따라서 하드웨어가 제공하는 **원자 명령어(atomic instructions)**가 필요합니다.

### 3.1 Test-And-Set (Atomic Exchange)

**원자 명령어(Atomic instructions)**는 read-modify-write 연산을 원자적으로 수행하도록 하드웨어가 보장합니다.

**Test-And-Set** 명령어는 메모리 위치의 이전 값을 반환하면서 동시에 새 값으로 업데이트합니다.

```c
int TestAndSet(int *v, int new) {
    int old = *v;
    *v = new;
    return old;
}
```

이 전체 동작이 **원자적(atomically)**으로 수행됩니다. x86에서는 `xchg` 명령어, RISC-V에서는 `amoswap` 명령어가 이에 해당합니다.

```text
┌─────────────────────────────────┐
│  old = *v; *v = new; return old;│  <- 이 전체가 원자적!
└─────────────────────────────────┘
     Thread 1: 시작 ───────── 끝
                   Thread 2: 대기 중...
```

Test-And-Set을 사용한 Spinlock 구현:

```c
struct lock { int held = 0; }

void acquire(struct lock *l) {
    while (l->held);
    l->held = 1;
}

void release(struct lock *l) {
    l->held = 0;
}
```

이것을:

```c
void acquire(struct lock *l) {
    while (TestAndSet(&l->held, 1));
}

void release(struct lock *l) {
    l->held = 0;
}
```

이렇게 바꾸면 됩니다. `TestAndSet`이 원자적이므로 race condition이 발생하지 않습니다.

### 3.2 Compare-And-Swap

**Compare-And-Swap (CAS)**는 메모리 값이 예상한 값과 같을 때만 새 값으로 업데이트합니다.

```c
int CompareAndSwap(int *v, int expected, int new) {
    int old = *v;
    if (old == expected)
        *v = new;
    return old;
}
```

x86에서는 `cmpxchg` 명령어가 이에 해당합니다.

```c
void acquire(struct lock *l) {
    while (CompareAndSwap(&l->held, 0, 1));
}
```

- `held`가 0이면 1로 바꾸고 0을 반환 → 루프 종료
- `held`가 1이면 그대로 두고 1을 반환 → 계속 대기

### 3.3 Ticket Lock (Fetch-And-Add)

**Fetch-And-Add**는 값을 원자적으로 증가시키면서 이전 값을 반환합니다 (x86의 `xadd`). 이를 사용해 **Ticket Lock**을 구현하면 bounded waiting과 공정성을 보장합니다.

```c
struct lock { int ticket = 0; int turn = 0; };

void acquire(struct lock *l) {
    int myturn = FetchAndAdd(&l->ticket, 1);
    while (l->turn != myturn);  // 내 차례가 올 때까지 대기
}

void release(struct lock *l) {
    l->turn = l->turn + 1;      // 다음 차례에게 양보
}
```

## 4. Spinlock의 문제점과 개선

### 4.1 Spinlock의 문제와 개선

위 Lock들은 모두 **Spinlock**으로, Lock을 얻을 때까지 busy-waiting하여 CPU 사이클을 낭비합니다. 개선 방법:

- **Yield**: CPU를 자발적으로 양보하지만 context switch 오버헤드 존재
- **Sleep-based Lock**: Lock을 얻을 수 없을 때 잠들고, release 시 대기 스레드를 깨움 (운영체제 지원 필요)
- **Two-Phase Lock**: 짧은 시간 spin 후, Lock을 얻지 못하면 sleep (실전에서 많이 사용)

### 4.2 Two-Phase Lock

실제로는 **Two-Phase Lock**을 많이 사용합니다.

1. **Phase 1**: 짧은 시간 동안 spin (Lock이 곧 풀릴 것으로 예상)
2. **Phase 2**: spin 후에도 Lock을 얻지 못하면 sleep

이렇게 하면 짧은 Critical Section에서는 context switch 오버헤드를 피하고, 긴 Critical Section에서는 CPU를 낭비하지 않습니다.

## 5. Condition Variable

### 5.1 Condition Variable이란?

Lock은 상호 배제는 제공하지만 **순서 동기화(ordering/event synchronization)**는 제공하지 않습니다. **Condition Variable (CV)**은 이벤트를 효율적으로 기다리는 메커니즘을 제공하며, 항상 **mutex(Lock)**와 함께 사용됩니다.

### 5.2 Condition Variable 연산

Condition Variable은 세 가지 연산을 제공합니다.

**1. cond_wait(cond_t *cv, mutex_t *mutex)**

- mutex를 가지고 있는 상태에서 호출
- 호출자를 재우고 mutex를 **원자적으로** 해제
- 깨어나면 다시 mutex를 획득한 후 반환

**2. cond_signal(cond_t *cv)**

- cv에서 대기 중인 스레드 중 하나를 깨움
- 대기 중인 스레드가 없으면 아무 일도 하지 않음

**3. cond_broadcast(cond_t *cv)**

- cv에서 대기 중인 모든 스레드를 깨움

**중요:** Condition Variable은 **상태를 저장하지 않습니다 (memoryless)**. `signal`이나 `broadcast`를 호출했을 때 대기 중인 스레드가 없으면 그 신호는 사라집니다.

### 5.3 Producer-Consumer 문제

Bounded Buffer를 사용하는 고전적인 Producer-Consumer 문제를 Condition Variable로 해결해봅시다.

```c
mutex_t m;
cond_t full, empty;  // 두 개의 CV 사용

void put(data) {
    mutex_lock(&m);
    while (tail - front == MAX)
        cond_wait(&full, &m);   // 버퍼가 가득 찼으면 대기

    tail = (tail+1) % MAX;
    buf[tail] = data;

    cond_signal(&empty);         // consumer에게 신호
    mutex_unlock(&m);
}

void get() {
    mutex_lock(&m);
    while (front == tail)
        cond_wait(&empty, &m);  // 버퍼가 비었으면 대기

    front = (front+1) % MAX;
    item = buf[front];

    cond_signal(&full);          // producer에게 신호
    mutex_unlock(&m);
    return item;
}
```

**핵심 원칙:**

1. **항상 `while` 루프 안에서 `wait()` 호출** (if가 아니라!)
   - Lock만으로는 버퍼가 비었을 때 효율적으로 대기할 수 없음
   - `if`로 조건 확인 시 spurious wakeup에 취약
2. 두 개의 CV 사용: `full` (producer 대기), `empty` (consumer 대기)
3. 신호를 보내기 전에 mutex를 획득한 상태여야 함

### 5.4 Mesa vs. Hoare Semantics

Condition Variable에는 두 가지 의미론이 있습니다.

**Hoare Semantics:**
- `signal()` 호출 시 대기 중인 스레드가 **즉시** 실행됨
- 신호를 보낸 스레드는 대기

**Mesa Semantics (대부분의 시스템에서 사용):**
- `signal()` 호출 시 대기 중인 스레드를 **ready 큐**에 넣음
- 신호를 보낸 스레드가 계속 실행
- 깨어난 스레드는 나중에 스케줄됨

Mesa semantics에서는 깨어난 스레드가 실행될 때 조건이 다시 false가 될 수 있습니다. 따라서 **반드시 while 루프**를 사용해야 합니다!

## 6. Semaphore

### 6.1 Semaphore란?

**Semaphore**는 Lock보다 더 일반화된 동기화 도구로, 정수 값과 대기 큐를 가지며 두 개의 원자적 연산으로만 조작됩니다 (1968년 Dijkstra 발명).

### 6.2 Semaphore 연산

**sem_wait() (또는 P(), down())**

- 값을 감소시키고, 값이 0 미만이면 대기

```c
void wait(semaphore *S) {
    S->value--;
    if (S->value < 0) {
        add this process to S->Q;
        block();
    }
}
```

**sem_post() (또는 V(), up())**

- 값을 증가시키고, 대기 중인 스레드가 있으면 하나를 깨움

```c
void signal(semaphore *S) {
    S->value++;
    if (S->value <= 0) {
        remove a process P from S->Q;
        wakeup(P);
    }
}
```

**중요:** `wait()`와 `signal()`은 서로에 대해 원자적으로 실행되어야 합니다.

### 6.3 Semaphore의 두 가지 용도

**1. Binary Semaphore (= Mutex/Lock)**

초기값 1로 설정하면 상호 배제를 구현할 수 있습니다.

```c
semaphore S = 1;

sem_wait(&S);  // Lock 획득
// Critical section
sem_post(&S);  // Lock 해제
```

**2. Counting Semaphore**

초기값 N으로 설정하면 N개의 리소스를 관리할 수 있습니다. 여러 스레드가 동시에 진입할 수 있지만, 최대 N개까지만 가능합니다.

### 6.4 Semaphore로 Producer-Consumer 구현

Semaphore를 사용하면 Producer-Consumer 문제를 간결하게 해결할 수 있습니다.

```c
semaphore mutex = 1;   // 상호 배제용
semaphore full = 0;    // 가득 찬 슬롯 수
semaphore empty = MAX; // 빈 슬롯 수

void producer() {
    while (1) {
        item = produce();
        sem_wait(&empty);   // 빈 슬롯 대기 (없으면 block)
        sem_wait(&mutex);   // Critical section 진입
        buffer[in] = item;
        in = (in + 1) % MAX;
        sem_post(&mutex);   // Critical section 나감
        sem_post(&full);    // 가득 찬 슬롯 증가 (consumer 깨움)
    }
}

void consumer() {
    while (1) {
        sem_wait(&full);    // 가득 찬 슬롯 대기 (없으면 block)
        sem_wait(&mutex);   // Critical section 진입
        item = buffer[out];
        out = (out + 1) % MAX;
        sem_post(&mutex);   // Critical section 나감
        sem_post(&empty);   // 빈 슬롯 증가 (producer 깨움)
        consume(item);
    }
}
```

**주의:** `sem_wait(&empty)`를 `sem_wait(&mutex)` 앞에 호출해야 합니다. 순서를 바꾸면 교착 상태가 발생할 수 있습니다.

### 6.5 Semaphore vs. Condition Variable

|  | Semaphore | Condition Variable |
|---|---|---|
| 상태 | 정수 값 유지 | 상태 없음 (memoryless) |
| 사용 | 단독으로 사용 가능 | 반드시 mutex와 함께 사용 |
| Signal | 대기 스레드가 없어도 값 증가 | 대기 스레드가 없으면 신호 손실 |
| 용도 | 리소스 카운팅, 상호 배제 | 이벤트 대기, 순서 동기화 |

Semaphore는 더 강력하지만, Condition Variable이 더 명시적이고 의도를 명확히 표현할 수 있습니다.

## 7. 동기화 메커니즘 비교

### 7.1 정확성과 공정성

- **Mutual Exclusion**: Lock, Binary Semaphore 모두 보장
- **Progress**: 올바르게 구현하면 보장
- **Bounded Waiting**: Ticket Lock, FIFO 큐 사용 Semaphore가 보장
- **Fairness**: Ticket Lock과 FIFO 큐 기반 Sleep-based Lock이 보장

### 7.2 성능

- **Spinlock**: Critical Section이 짧을 때 효율적 (context switch 없음)
- **Sleep-based Lock**: Critical Section이 길 때 효율적 (CPU 낭비 없음)
- **Two-Phase Lock**: 두 가지 장점 결합

## 정리

이번 글에서는 멀티스레드 환경에서 발생하는 race condition 문제와 이를 해결하기 위한 동기화 메커니즘을 살펴보았습니다.

**핵심 개념:**

1. **Race Condition**: 여러 스레드가 공유 자원에 동시 접근할 때 발생하는 버그
2. **Critical Section**: 공유 자원에 접근하는 코드 영역
3. **Lock**: 상호 배제를 제공하는 기본 도구
   - 하드웨어 원자 명령어 필요: Test-And-Set, Compare-And-Swap, LL/SC
   - Spinlock vs. Sleep-based Lock
4. **Condition Variable**: 이벤트 대기를 위한 도구
   - 반드시 mutex와 함께 사용
   - `wait()` 호출은 항상 `while` 루프 안에서
5. **Semaphore**: 더 일반화된 동기화 도구
   - Binary Semaphore = Lock
   - Counting Semaphore = 리소스 관리

**다음 글 예고:**

다음 글에서는 동시성 프로그래밍에서 발생할 수 있는 버그들(Deadlock, Race Condition, Atomicity Violation)과 교착 상태(Deadlock)의 조건 및 해결 방법을 다룰 예정입니다.

*다음 글: [[CS330] 8. 동시성 버그와 교착 상태](/posts/cs330/8-concurrency-bugs-deadlock/)*
