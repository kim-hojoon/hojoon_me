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

### 1.1 The Classic Example: 은행 계좌

가장 고전적인 예제로 시작해봅시다. 당신과 친구가 공동 계좌를 사용하고 있고, 잔액이 1,000,000원이 있습니다. 두 사람이 서로 다른 ATM에서 동시에 100,000원씩을 인출하면 어떻게 될까요?

```c
int withdraw(account, amount)
{
    balance = get_balance(account);
    balance = balance - amount;
    put_balance(account, balance);
    return balance;
}
```

위 코드는 단순해 보이지만, 멀티스레드 환경에서는 심각한 문제가 발생합니다. 두 스레드가 **인터리빙(interleaving)**되면서 실행될 수 있기 때문입니다.

```text
Thread 1                          Thread 2
--------                          --------
balance = get_balance(account);   (balance = 1,000,000)
balance = balance - amount;       (balance = 900,000)
                                  balance = get_balance(account);  (balance = 1,000,000!)
                                  balance = balance - amount;      (balance = 900,000)
                                  put_balance(account, balance);   (계좌 = 900,000)
put_balance(account, balance);    (계좌 = 900,000)
```

두 번의 인출이 있었는데 잔액은 900,000원만 줄어들었습니다. 이것이 바로 **race condition**입니다.

### 1.2 실제 예제: 전역 변수 증가

더 단순한 예제를 봅시다. 전역 변수 `g`를 1씩 증가시키는 함수가 있습니다.

```c
extern long g;

void inc() {
    g++;
}
```

이 C 코드는 어셈블리로 변환되면 여러 명령어가 됩니다.

```text
ld    a0, 0(s1)    # 메모리에서 g를 레지스터 a0로 로드
addi  a0, a0, 1    # a0에 1을 더함
sd    a0, 0(s1)    # a0의 값을 메모리에 저장
ret
```

두 스레드가 동시에 `inc()`를 실행하면 인터리빙이 발생합니다.

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

최종 결과는 2가 아니라 1입니다. 두 스레드의 작업 중 하나가 손실되었습니다.

### 1.3 Shared Resources

스레드 간에 공유되는 자원을 정리하면:

- **지역 변수(Local variables)**: 공유되지 않음
  - 스택에 저장되며, 각 스레드는 독립된 스택을 가짐
- **전역 변수(Global variables)**: 공유됨
  - 정적 데이터 세그먼트에 저장되어 모든 스레드가 접근 가능
- **동적 객체(Dynamic objects)**: 공유됨
  - 힙(heap)에 저장되며, 포인터를 통해 공유됨
- **공유 메모리(Shared memory)**: 프로세스 간에도 공유 가능

### 1.4 Synchronization Problem

**동기화 문제(Synchronization Problem)**는 두 개 이상의 동시 실행 스레드가 공유 자원에 접근할 때 **race condition**을 만드는 것을 말합니다.

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
- 다른 스레드들은 진입 시 대기
- 한 스레드가 나가면 다른 스레드가 진입 가능

### 2.2 Lock의 필요성

Lock은 상호 배제를 제공하는 메모리 내 객체(object)입니다.

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

```text
Thread T1:  A ──S1──S2──S3──R───────────────────────
                 │                            │
Thread T2:  ────A (waiting)────────────────S1─S2─S3─R──

A: acquire(), R: release(), S1-S3: Critical Section
```

### 2.3 동기화 요구사항

올바른 동기화를 위한 세 가지 정확성 기준:

1. **Mutual Exclusion (상호 배제)**: 한 번에 하나의 스레드만 Critical Section에 진입
2. **Progress (진행, Deadlock-free)**: 여러 스레드가 진입하려 할 때 반드시 하나는 진입 가능해야 함
3. **Bounded Waiting (제한된 대기, Starvation-free)**: 대기 중인 각 스레드는 언젠가는 반드시 진입할 수 있어야 함

추가 고려사항:

- **Fairness (공정성)**: 각 스레드가 Lock을 획득할 공정한 기회를 가져야 함
- **Performance (성능)**: Lock의 오버헤드가 작아야 하며, 멀티프로세서 환경에서도 효율적이어야 함

## 3. Lock 구현 방법

### 3.1 방법 1: 인터럽트 비활성화

가장 단순한 방법은 Critical Section 동안 인터럽트를 끄는 것입니다.

```c
void acquire(struct lock *l) {
    cli();      // disable interrupts
}

void release(struct lock *l) {
    sti();      // enable interrupts
}
```

**문제점:**

- 커널에서만 사용 가능 (사용자 프로그램에 제공하면 악용 가능)
- 멀티프로세서 시스템에서 작동하지 않음 (다른 CPU는 영향 없음)
- Critical Section이 길면 중요한 인터럽트를 놓칠 수 있음
- 현대 CPU의 원자 명령어보다 느림

### 3.2 방법 2: 단순 플래그 변수

Lock을 단순히 플래그 변수로 구현하면 어떨까요?

```c
struct lock { int held = 0; }

void acquire(struct lock *l) {
    while (l->held);     // 다른 스레드가 Lock을 가지고 있으면 대기
    l->held = 1;         // Lock 획득
}

void release(struct lock *l) {
    l->held = 0;         // Lock 해제
}
```

**문제점:** 이 구현은 작동하지 않습니다! `while`과 `l->held = 1` 사이에서 race condition이 발생할 수 있습니다.

```text
Thread 1: while (l->held);     (held == 0, 통과)
Thread 2: while (l->held);     (held == 0, 통과!)
Thread 1: l->held = 1;
Thread 2: l->held = 1;         (둘 다 Lock 획득!)
```

**결론:** Lock 구현 자체가 race condition에 취약합니다. 하드웨어의 도움이 필요합니다.

### 3.3 방법 3: Test-And-Set (Atomic Exchange)

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

### 3.4 방법 4: Compare-And-Swap

**Compare-And-Swap (CAS)**는 더 강력한 원자 명령어입니다. 메모리 값이 예상한 값과 같을 때만 새 값으로 업데이트합니다.

```c
int CompareAndSwap(int *v, int expected, int new) {
    int old = *v;
    if (old == expected)
        *v = new;
    return old;
}
```

x86에서는 `cmpxchg` 명령어가 이에 해당합니다.

CAS를 사용한 Lock:

```c
void acquire(struct lock *l) {
    while (CompareAndSwap(&l->held, 0, 1));
}
```

- `held`가 0이면 (Lock이 자유로우면) 1로 바꾸고 0을 반환 → 루프 종료
- `held`가 1이면 (Lock이 잡혀있으면) 그대로 두고 1을 반환 → 계속 대기

### 3.5 방법 5: Load-Linked / Store-Conditional (LL/SC)

일부 아키텍처(MIPS, ARM, RISC-V)는 **LL/SC** 명령어 쌍을 제공합니다.

- **Load-Linked (LL)**: 메모리에서 값을 읽어옴
- **Store-Conditional (SC)**: LL 이후 해당 메모리가 변경되지 않았으면 새 값을 저장, 성공하면 1 반환

```c
int LoadLinked(int *ptr) {
    return *ptr;
}

int StoreConditional(int *ptr, int value) {
    if (/* *ptr가 LoadLinked 이후 변경되지 않았으면 */) {
        *ptr = value;
        return 1;  // 성공
    }
    return 0;      // 실패
}
```

LL/SC를 사용한 Lock:

```c
void acquire(struct lock *l) {
    while (1) {
        while (LoadLinked(&l->held));
        if (StoreConditional(&l->held, 1))
            return;  // 성공!
    }
}
```

### 3.6 Fetch-And-Add

**Fetch-And-Add**는 값을 원자적으로 증가시키면서 이전 값을 반환합니다.

```c
int FetchAndAdd(int *v, int a) {
    int old = *v;
    *v = old + a;
    return old;
}
```

x86에서는 `xadd` 명령어가 이에 해당합니다.

이를 사용해 **Ticket Lock**을 구현할 수 있습니다 (bounded waiting 보장).

```c
struct lock {
    int ticket = 0;
    int turn = 0;
};

void acquire(struct lock *l) {
    int myturn = FetchAndAdd(&l->ticket, 1);
    while (l->turn != myturn);  // 내 차례가 올 때까지 대기
}

void release(struct lock *l) {
    l->turn = l->turn + 1;      // 다음 차례에게 양보
}
```

Ticket Lock은 FIFO 순서를 보장하여 공정성을 제공합니다.

## 4. Spinlock의 문제점과 개선

### 4.1 Spinlock의 문제

위에서 구현한 Lock들은 모두 **Spinlock**입니다. Lock을 얻을 때까지 계속 루프를 돌면서 **busy-waiting**합니다.

```c
void acquire(struct lock *l) {
    while (TestAndSet(&l->held, 1));  // CPU 사이클 낭비!
}
```

**문제점:**

- CPU 사이클을 낭비함
- Critical Section이 길수록 spin 시간도 길어짐
- Lock을 가진 스레드가 비자발적 context switch로 중단되면 다른 스레드들은 계속 spin

### 4.2 개선 1: Yield

Lock을 얻지 못하면 CPU를 자발적으로 양보합니다.

```c
void acquire(struct lock *l) {
    while (TestAndSet(&l->held, 1))
        yield();  // 스케줄러에게 CPU 양보
}
```

이렇게 하면 다른 스레드가 실행될 수 있어 CPU를 낭비하지 않습니다. 하지만 여전히 context switch 오버헤드가 발생하고, starvation 문제가 남아있습니다.

### 4.3 개선 2: Sleep-based Lock (Blocking Lock)

더 나은 방법은 Lock을 얻을 수 없을 때 **잠자기(sleep)**하는 것입니다.

```c
void acquire(struct lock *l) {
    while (TestAndSet(&l->held, 1))
        sleep();  // 잠들기
}

void release(struct lock *l) {
    l->held = 0;
    wakeup(waiting_threads);  // 대기 중인 스레드 깨우기
}
```

하지만 이 구현도 문제가 있습니다. `sleep()`과 `wakeup()`을 어떻게 구현할까요? 이를 위해 운영체제의 지원이 필요하며, 이것이 바로 **Condition Variable**입니다.

### 4.4 Two-Phase Lock

실제로는 **Two-Phase Lock**을 많이 사용합니다.

1. **Phase 1**: 짧은 시간 동안 spin (Lock이 곧 풀릴 것으로 예상)
2. **Phase 2**: spin 후에도 Lock을 얻지 못하면 sleep

이렇게 하면 짧은 Critical Section에서는 context switch 오버헤드를 피하고, 긴 Critical Section에서는 CPU를 낭비하지 않습니다.

## 5. Condition Variable

### 5.1 Lock만으로는 부족하다

Lock은 상호 배제는 제공하지만, **순서 동기화(ordering/event synchronization)**는 제공하지 않습니다.

예를 들어, 한 스레드가 특정 조건이 만족될 때까지 기다려야 한다면 어떻게 할까요?

```c
// 잘못된 방법: Busy polling
void consumer() {
    lock.acquire();
    while (count == 0) {
        lock.release();
        lock.acquire();  // 계속 확인
    }
    // 아이템 소비
    lock.release();
}
```

이것은 비효율적입니다. 조건이 만족될 때까지 효율적으로 기다리는 방법이 필요합니다.

### 5.2 Condition Variable이란?

**Condition Variable (CV)**은 이벤트를 기다리는 메커니즘을 제공합니다.

- Producer/Consumer 문제에서 사용
- Pipeline 동기화
- 백그라운드 스레드와의 작업 조정

Condition Variable은 항상 **mutex(Lock)**와 함께 사용됩니다.

```text
        ┌──────────┐       ┌────────┐
Producer│────────→ │Buffer│ ────→  │Consumer
        └──────────┘       └────────┘
              ↑                ↑
           CV: full        CV: empty
```

### 5.3 Condition Variable 연산

Condition Variable은 세 가지 연산을 제공합니다.

**1. cond_wait(cond_t *cv, mutex_t *mutex)**

- mutex를 가지고 있는 상태에서 호출
- 호출자를 재우고 mutex를 **원자적으로** 해제
- 깨어나면 다시 mutex를 획득한 후 반환

```c
void wait_example() {
    pthread_mutex_lock(&m);
    pthread_cond_wait(&c, &m);  // 잠들기 + mutex 해제 (원자적)
    pthread_mutex_unlock(&m);    // 깨어난 후
}
```

**2. cond_signal(cond_t *cv)**

- cv에서 대기 중인 스레드 중 하나를 깨움
- 대기 중인 스레드가 없으면 아무 일도 하지 않음

**3. cond_broadcast(cond_t *cv)**

- cv에서 대기 중인 모든 스레드를 깨움

```c
void signal_example() {
    pthread_mutex_lock(&m);
    pthread_cond_signal(&c);     // 대기 중인 스레드 하나 깨우기
    pthread_mutex_unlock(&m);
}
```

**중요:** Condition Variable은 **상태를 저장하지 않습니다 (memoryless)**. `signal`이나 `broadcast`를 호출했을 때 대기 중인 스레드가 없으면 그 신호는 사라집니다.

### 5.4 Producer-Consumer 문제

Bounded Buffer를 사용하는 고전적인 Producer-Consumer 문제를 봅시다.

```text
         front
           ↓
    ┌─┬─┬─┬─┬─┬─┬─┐
    │ │█│█│█│ │ │ │  <- 순환 버퍼 (MAX 크기)
    └─┴─┴─┴─┴─┴─┴─┘
             ↑
            tail
```

**첫 번째 시도: Lock만 사용**

```c
mutex_t m;
cond_t cond;

void put(data) {
    mutex_lock(&m);
    if (count < MAX) {
        tail = (tail+1) % MAX;
        buf[tail] = data;
        count++;
    }
    mutex_unlock(&m);
}

void get() {
    mutex_lock(&m);
    if (count > 0) {
        front = (front+1) % MAX;
        item = buf[front];
        count--;
    }
    mutex_unlock(&m);
    return item;
}
```

문제: 버퍼가 비어있을 때 consumer는 어떻게 해야 할까요? Busy polling은 비효율적입니다.

**두 번째 시도: Condition Variable 추가 (잘못됨)**

```c
void put(data) {
    mutex_lock(&m);
    if (tail - front == MAX)
        cond_wait(&cond, &m);  // 버퍼가 가득 찼으면 대기

    tail = (tail+1) % MAX;
    buf[tail] = data;
    count++;

    cond_signal(&cond);         // consumer 깨우기
    mutex_unlock(&m);
}

void get() {
    mutex_lock(&m);
    if (front == tail)
        cond_wait(&cond, &m);   // 버퍼가 비었으면 대기

    front = (front+1) % MAX;
    item = buf[front];
    count--;

    cond_signal(&cond);          // producer 깨우기
    mutex_unlock(&m);
    return item;
}
```

문제: `if`를 사용하면 **spurious wakeup** 시 조건을 다시 확인하지 않습니다!

**세 번째 시도: while 사용 (올바름)**

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
2. 두 개의 CV 사용: `full` (producer 대기), `empty` (consumer 대기)
3. 신호를 보내기 전에 mutex를 획득한 상태여야 함

### 5.5 Mesa vs. Hoare Semantics

Condition Variable에는 두 가지 의미론이 있습니다.

**Hoare Semantics:**
- `signal()` 호출 시 대기 중인 스레드가 **즉시** 실행됨
- 신호를 보낸 스레드는 대기

**Mesa Semantics (대부분의 시스템에서 사용):**
- `signal()` 호출 시 대기 중인 스레드를 **ready 큐**에 넣음
- 신호를 보낸 스레드가 계속 실행
- 깨어난 스레드는 나중에 스케줄됨

Mesa semantics에서는 깨어난 스레드가 실행될 때 조건이 다시 false가 될 수 있습니다. 따라서 **반드시 while 루프**를 사용해야 합니다!

```c
// Mesa semantics에서 필수
while (조건이 거짓)
    cond_wait(&cv, &mutex);

// Hoare semantics에서는 가능하지만, Mesa에서는 틀림
if (조건이 거짓)
    cond_wait(&cv, &mutex);
```

## 6. Semaphore

### 6.1 Semaphore란?

**Semaphore**는 Lock보다 더 일반화된 동기화 도구입니다. 1968년 Dijkstra가 THE OS의 일부로 발명했습니다.

Semaphore의 특징:

- Busy waiting이 필요 없음
- 정수 값(state)을 가진 객체
- 사용자 프로그램이 직접 값에 접근할 수 없음
- 두 개의 원자적 연산으로만 조작

```c
typedef struct {
    int value;
    struct process *Q;  // 대기 큐
} semaphore;
```

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

**중요:** `wait()`와 `signal()`은 서로에 대해 원자적으로 실행되어야 합니다. 이를 위해 내부적으로 하드웨어 원자 명령어나 인터럽트 비활성화를 사용합니다.

### 6.3 Semaphore의 두 가지 용도

**1. Binary Semaphore (= Mutex/Lock)**

초기값 1로 설정하면 상호 배제를 구현할 수 있습니다.

```c
semaphore S = 1;

sem_wait(&S);  // Lock 획득
// Critical section goes here
sem_post(&S);  // Lock 해제
```

**2. Counting Semaphore**

초기값 N으로 설정하면 N개의 리소스를 관리할 수 있습니다.

```c
semaphore resources = N;  // N개의 단위 사용 가능

sem_wait(&resources);  // 리소스 하나 사용
// 작업 수행
sem_post(&resources);  // 리소스 반환
```

여러 스레드가 동시에 진입할 수 있지만, 최대 N개까지만 가능합니다.

### 6.4 Semaphore로 Producer-Consumer 구현

Semaphore를 사용하면 Producer-Consumer 문제를 간결하게 해결할 수 있습니다.

```c
semaphore mutex = 1;   // 상호 배제용
semaphore full = 0;    // 가득 찬 슬롯 수
semaphore empty = MAX; // 빈 슬롯 수

void producer() {
    while (1) {
        item = produce();

        sem_wait(&empty);   // 빈 슬롯 대기
        sem_wait(&mutex);   // Critical section 진입

        buffer[in] = item;
        in = (in + 1) % MAX;

        sem_post(&mutex);   // Critical section 나감
        sem_post(&full);    // 가득 찬 슬롯 증가
    }
}

void consumer() {
    while (1) {
        sem_wait(&full);    // 가득 찬 슬롯 대기
        sem_wait(&mutex);   // Critical section 진입

        item = buffer[out];
        out = (out + 1) % MAX;

        sem_post(&mutex);   // Critical section 나감
        sem_post(&empty);   // 빈 슬롯 증가

        consume(item);
    }
}
```

**주의:** `sem_wait(&empty)`와 `sem_wait(&mutex)`의 순서가 중요합니다! 순서를 바꾸면 교착 상태(deadlock)가 발생할 수 있습니다.

### 6.5 Semaphore vs. Condition Variable

|  | Semaphore | Condition Variable |
|---|---|---|
| 상태 | 정수 값 유지 | 상태 없음 (memoryless) |
| 사용 | 단독으로 사용 가능 | 반드시 mutex와 함께 사용 |
| Signal | 대기 스레드가 없어도 값 증가 | 대기 스레드가 없으면 신호 손실 |
| 용도 | 리소스 카운팅, 상호 배제 | 이벤트 대기, 순서 동기화 |

Semaphore는 더 강력하지만, Condition Variable이 더 명시적이고 의도를 명확히 표현할 수 있습니다.

## 7. 동기화 요구사항 재확인

이제 우리가 배운 동기화 메커니즘들이 어떻게 요구사항을 만족하는지 확인해봅시다.

### 7.1 Correctness (정확성)

1. **Mutual Exclusion**: Lock, Binary Semaphore 모두 보장
2. **Progress (Deadlock-free)**: 올바르게 구현하면 보장됨
3. **Bounded Waiting (Starvation-free)**: Ticket Lock, FIFO 큐를 사용하는 Semaphore가 보장

### 7.2 Fairness (공정성)

- Spinlock: 공정성 보장 어려움
- Ticket Lock: FIFO 순서로 공정성 보장
- Sleep-based Lock with FIFO queue: 공정성 보장

### 7.3 Performance (성능)

- Spinlock: Critical Section이 짧을 때 효율적 (context switch 없음)
- Sleep-based Lock: Critical Section이 길 때 효율적 (CPU 낭비 없음)
- Two-Phase Lock: 두 가지 장점을 결합

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
