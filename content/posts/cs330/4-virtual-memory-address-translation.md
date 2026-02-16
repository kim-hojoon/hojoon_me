---
title: "[CS330] 4. 가상 메모리와 주소 변환"
date: 2023-12-04
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "Virtual Memory", "Address Translation", "Segmentation"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "가상 메모리의 개념과 목표, 정적/동적 재배치, Base and Bounds, 세그멘테이션까지 주소 변환의 발전 과정을 살펴봅니다."
aliases: ["/posts/cs330-4-virtual-memory-address-translation/"]
ShowToc: true
TocOpen: true
---

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

*이전 글: [[CS330] 3. 스레드와 CPU 스케줄링](/posts/cs330/3-thread-scheduling/)*

## 1. 왜 가상 메모리가 필요한가?

초기 컴퓨터 시스템에서는 프로그램이 물리 메모리(Physical Memory)에 직접 접근했습니다. 하지만 여러 프로세스를 동시에 실행하려고 하면 심각한 문제가 발생합니다.

```text
Physical Memory
┌─────────────────────┐ 0KB
│ Operating System    │
├─────────────────────┤ 64KB
│ Process A           │
│ (code, data, etc.)  │
├─────────────────────┤ 128KB
│ Free                │
├─────────────────────┤ 192KB
│ Process C           │
│ (code, data, etc.)  │
├─────────────────────┤ 256KB
│ Process B           │
│ (code, data, etc.)  │
├─────────────────────┤ 320KB
│ Free                │
└─────────────────────┘ 512KB
```

**보호(Protection) 문제**: Process A가 실수로 또는 악의적으로 Process B나 OS의 메모리 영역에 접근하면 시스템 전체가 망가질 수 있습니다. 프로세스 간 격리(isolation)가 보장되지 않습니다.

이러한 문제를 해결하기 위해 **가상 메모리(Virtual Memory)**라는 추상화가 등장했습니다. 가상 메모리는 각 프로세스에게 독립된 주소 공간의 환상(illusion)을 제공합니다.

## 2. 가상 메모리의 목표

가상 메모리 시스템은 세 가지 주요 목표를 달성해야 합니다:

### 2.1 투명성 (Transparency)

프로세스는 메모리가 공유되고 있다는 사실을 인식하지 못해야 합니다. 각 프로세스는 마치 자신만의 크고 연속적인(contiguous) 메모리 공간을 소유한 것처럼 느껴야 합니다. 이는 프로그래밍을 편리하게 만드는 추상화입니다.

### 2.2 효율성 (Efficiency)

- **공간 효율성**: 가변 크기 요청으로 인한 단편화(fragmentation)를 최소화해야 합니다.
- **시간 효율성**: 주소 변환 과정이 빠르게 이루어져야 하며, 하드웨어 지원을 활용해야 합니다.

### 2.3 보호 (Protection)

- 프로세스와 OS를 다른 프로세스로부터 보호해야 합니다.
- 격리(isolation): 한 프로세스가 실패해도 다른 프로세스에 영향을 주지 않아야 합니다.
- 협력하는 프로세스들은 메모리의 일부를 공유할 수 있어야 합니다.

## 3. 주소 공간 (Address Space)

**주소 공간(Address Space)**은 프로세스의 메모리에 대한 추상적 뷰입니다. 각 프로세스는 자신만의 가상 주소 공간을 갖습니다.

```text
Virtual Address Space
┌─────────────────────┐ 0
│     unused          │
├─────────────────────┤
│ read-only segment   │
│ (.init, .text,      │
│  .rodata)           │
├─────────────────────┤
│ read/write segment  │
│ (.data, .bss)       │
├─────────────────────┤
│   run-time heap     │
│ (managed by malloc) │ ← brk
│         ↓           │
│                     │
│      (free)         │
│                     │
│         ↑           │
│    user stack       │ ← stack pointer
│ (created at runtime)│
├─────────────────────┤
│ kernel virtual      │ } memory
│ memory (code, data, │ } invisible to
│  heap, stack)       │ } user code
└─────────────────────┘ N-1
```

### 주소 공간의 구성 요소

- **정적 영역 (Static Area)**: `exec()` 호출 시 할당
  - **Code & Data**: 프로그램 코드와 전역 변수

- **동적 영역 (Dynamic Area)**: 실행 시간에 할당되고 크기가 변할 수 있음
  - **Heap**: 아래 방향으로 성장 (↓), `malloc()`으로 관리
  - **Stack**: 위 방향으로 성장 (↑), 함수 호출과 지역 변수

## 4. 주소 변환 (Address Translation)

가상 메모리의 핵심은 **주소 변환(Address Translation)**입니다. 프로그램이 사용하는 가상 주소(Virtual Address)를 실제 물리 메모리의 물리 주소(Physical Address)로 변환하는 과정입니다.

```text
Virtual Address Space (P1)          Physical Memory
┌──────────┐ 0                     ┌──────────┐
│   Code   │                       │   Code   │
├──────────┤    Address            ├──────────┤
│   Data   │ ─── Translation ────→ │   Data   │
├──────────┤    Mechanism          ├──────────┤
│   Heap   │                       │   Heap   │
├──────────┤                       ├──────────┤
│   Stack  │                       │   Stack  │
└──────────┘                       ├──────────┤
                                   │   Code   │
Virtual Address Space (P2)         ├──────────┤
┌──────────┐ 0                     │   Data   │
│   Code   │                       ├──────────┤
├──────────┤    Address            │   Heap   │
│   Data   │ ─── Translation ────→ ├──────────┤
├──────────┤    Mechanism          │   Stack  │
│   Heap   │                       ├──────────┤
├──────────┤                       │          │
│   Stack  │                       └──────────┘
└──────────┘
```

주소 변환은 하드웨어(MMU: Memory Management Unit)와 OS의 협력으로 이루어집니다:

- **하드웨어(MMU)**: 모든 메모리 참조 명령마다 주소 변환을 수행합니다. 가상 주소가 유효하지 않으면 예외(exception)를 발생시킵니다.
- **운영체제**: 현재 프로세스의 유효한 주소 공간 정보를 MMU에 전달합니다.

간단한 예시를 살펴보겠습니다:

```text
Address Space              Physical Memory
┌───────────────┐ 0KB     ┌───────────────┐ 0KB
│ Program Code  │   128:  │               │
│               │   movl 0x0(%ebx), %eax  │
│               │   132:  ├───────────────┤ 1KB
│               │   addl $0x03, %eax      │
│     Heap      │   135:  │ Program Code  │ ← ebx points to 15KB
│       ↓       │   movl %eax, 0x0(%ebx)  │
│               │         ├───────────────┤ 2KB
│    (free)     │         │     Heap      │
│               │         │       ↓       │
│     stack     │         │               │
│       ↑       │         │    (free)     │
├───────────────┤ 14KB    │               │
│     Stack     │ 3000    │     stack     │
└───────────────┘ 16KB    │       ↑       │
                          ├───────────────┤ 15KB
                          │     Stack     │ 3000
                          └───────────────┘ 16KB
```

이 예시에서 프로세스는 주소 128에서 시작하는 명령을 실행합니다. 하지만 물리 메모리에서는 1KB부터 시작합니다. 주소 변환 메커니즘이 이 차이를 해결해줍니다.

## 5. 정적 재배치 (Static Relocation)

초기 해결책은 **정적 재배치(Static Relocation)**였습니다. 로더(loader)가 프로그램을 메모리에 로드할 때 주소를 다시 쓰는(rewrite) 방식입니다. OS가 프로그램을 특정 메모리 위치(예: 0x1000)에 로드하기로 결정하면, 로더가 모든 정적 데이터와 함수의 주소를 재작성합니다.

**장점:** 하드웨어 지원이 필요 없음

**단점:**
- **보호 불가능**: 프로세스가 OS나 다른 프로세스의 메모리 영역을 파괴할 수 있음
- **재배치 불가능**: 주소 공간이 배치된 후에는 이동시킬 수 없어 외부 단편화 문제 발생

## 6. 동적 재배치 (Dynamic Relocation) - Base and Bounds

**동적 재배치(Dynamic Relocation)**는 하드웨어 기반 재배치 방식으로, MMU가 모든 메모리 참조 명령마다 주소 변환을 수행합니다.

### 6.1 Base and Bounds 메커니즘

가장 단순한 동적 재배치 방식은 **Base and Bounds** (또는 Base and Limit)입니다. IBM OS/MVT에서 사용되었습니다.

```text
               Base register      Base register
               0x2000             0x3200
Virtual        ┌───┐              ┌───┐
address        │   │              │   │
0x362   ────→ <+>  ────→ 0x2362  <+> ←─── Yes ←─── < ? ←─── 0x362
                                   │                          Bound register
                                   ↓                          0x2200
Physical Memory           protection fault ──← No
┌─────────────┐ 0
│     OS      │
├─────────────┤ 0x1000
│ Partition 0 │
├─────────────┤ 0x3200        Virtual address    Physical Memory
│             │                    0x362          ┌─────────────┐ 0
│ Partition 1 │ ←── 0x3562        ─────→         │     OS      │
│             │                                   ├─────────────┤ 0x1000
├─────────────┤ 0x7000                            │ Partition 0 │
│ Partition 2 │                                   ├─────────────┤ 0x3200
├─────────────┤ 0x8000                            │             │
│ Partition 3 │                                   │ Partition 1 │
└─────────────┘                                   │             │
                                                  ├─────────────┤ 0x7000
                                                  │ Partition 2 │
                                                  ├─────────────┤ 0x8000
                                                  │ Partition 3 │
                                                  └─────────────┘
```

**하드웨어 요구사항:**
- **Base Register**: 주소 공간의 시작 위치를 가리킵니다.
- **Bounds Register**: 주소 공간의 크기를 나타냅니다.

**주소 변환 공식:**
```text
Physical Address = Virtual Address + Base
if (Virtual Address >= Bounds) then
    protection fault
```

Base 레지스터는 컨텍스트 스위치 시 OS가 로드합니다.

### 6.2 메모리 배치 예시

```text
Address Space               Physical Memory
┌─────────────┐ 0KB         ┌─────────────┐ 0KB
│Program Code │             │Operating Sys│
├─────────────┤             ├─────────────┤ 16KB
│    Heap     │             │ (not in use)│
│      ↓      │             ├─────────────┤ 32KB
│             │  Relocated  │    Code     │ ← base register: 32KB
│   (free)    │  Process    │    Heap     │
│             │             │      ↓      │
│    stack    │             │ (allocated  │
│      ↑      │             │but not used)│
├─────────────┤             ├─────────────┤ 48KB
│    Stack    │             │    Stack    │
└─────────────┘ 16KB        ├─────────────┤ 64KB
                            │ (not in use)│
                            └─────────────┘

                    bounds register: 16KB
```

### 6.3 OS의 역할

OS는 Base-and-Bounds 방식을 구현하기 위해 세 가지 시점에서 개입합니다:

**1. 프로세스가 실행을 시작할 때:** Free list를 조회하여 새로운 주소 공간을 위한 공간을 할당합니다.

**2. 프로세스가 종료될 때:** 메모리를 free list에 반환합니다.

**3. 컨텍스트 스위치가 발생할 때:** PCB(Process Control Block)에 Base-and-Bounds 쌍을 저장하고 복원합니다. 프로세스 전환 시 하드웨어 레지스터에 새로운 Base와 Bounds 값을 로드합니다.

### 6.4 Base and Bounds의 장단점

**장점:**
- 구현이 간단하고 저렴함
- 빠른 컨텍스트 스위치 (레지스터 값만 변경)

**단점:**
- **내부 단편화(Internal Fragmentation)**: 주소 공간 내의 큰 free 공간이 물리 메모리를 낭비합니다.

```text
Address Space
┌─────────────┐ 0KB
│Program Code │
├─────────────┤ 1KB
│             │
├─────────────┤ 2KB
│             │
├─────────────┤ 3KB     Internal fragmentation
│             │         • Big chunk of "free" space
├─────────────┤ 4KB     • "free" space takes up
│             │           physical memory
├─────────────┤ 5KB
│    Heap     │
├─────────────┤ 6KB
│             │
│   (free)    │
│             │
├─────────────┤ 14KB
├─────────────┤ 15KB
│    Stack    │
└─────────────┘ 16KB
```

- **외부 단편화(External Fragmentation)**: 물리 메모리 전체에 구멍(holes)이 흩어져 있어, 컴팩션(compaction)이 필요할 수 있습니다.
- **부분 공유 불가**: 주소 공간의 일부를 공유할 수 없습니다.

## 7. 세그멘테이션 (Segmentation)

Base and Bounds의 내부 단편화 문제를 해결하기 위해 **세그멘테이션(Segmentation)**이 등장했습니다.

### 7.1 세그멘테이션의 개념

주소 공간을 논리적 세그먼트(logical segments)로 나눕니다:
- 각 세그먼트는 주소 공간의 논리적 엔터티에 대응합니다 (Code, Data, Stack, Heap 등)
- 사용자는 메모리를 Base-and-Bound 영역들의 집합으로 봅니다
- 각 세그먼트는 독립적으로 관리됩니다:
  - 물리 메모리에 개별적으로 배치 가능
  - 크기 조정 가능
  - 보호 가능 (별도의 read/write/execute 보호 비트)

세그멘테이션은 가변 파티션(variable partitions)의 자연스러운 확장입니다:
- **Base and Bounds**: 프로세스당 1개의 세그먼트
- **세그멘테이션**: 프로세스당 여러 개의 세그먼트

### 7.2 세그먼트 주소 지정

**명시적 접근 방식 (Explicit Approach):**

가상 주소의 일부를 세그먼트 번호로 사용하고, 나머지 비트는 세그먼트 내의 오프셋을 나타냅니다.

```text
31                                                    0
┌──┬─────────────────────────────────────────────────┐
│  │                                                  │
└──┴─────────────────────────────────────────────────┘
 ↑
 └─ Segment bits

00: P0 (User code, data, heap)
01: P1 (User stack)
1X: S  (System)
```

**암묵적 접근 방식 (Implicit Approach):**

메모리 참조 유형에 따라 세그먼트를 결정합니다:
- PC 기반 주소 지정: code 세그먼트
- SP 또는 BP 기반 주소 지정: stack 세그먼트

### 7.3 세그먼트 테이블

각 세그먼트는 Base와 Bounds를 갖습니다. 하드웨어는 **세그먼트 테이블(Segment Table)**을 사용합니다:

```text
Segment Table Entry:
┌────────┬────────┬───────────┬──────┐
│  Base  │ Bounds │ Direction │ Prot │
└────────┴────────┴───────────┴──────┘
```

- **Base**: 세그먼트의 물리 메모리 시작 주소
- **Bounds**: 세그먼트의 크기
- **Direction**: 세그먼트의 성장 방향 (예: Stack은 위로, Heap은 아래로)
- **Protection bits**: Read/Write/Execute 권한

### 7.4 세그멘테이션 메모리 배치

```text
Virtual Address Space         Physical Memory
┌─────────────┐ 0KB          ┌─────────────┐ 0KB
│Program Code │              │     OS      │
├─────────────┤              ├─────────────┤
│    Heap     │              │ (not in use)│
│      ↓      │              ├─────────────┤
│             │   Segment 0: │    Code     │
│   (free)    │   Code       ├─────────────┤
│             │              │ (not in use)│
│    stack    │              ├─────────────┤
│      ↑      │   Segment 1: │    Heap     │
├─────────────┤   Heap       │      ↓      │
│    Stack    │              ├─────────────┤
└─────────────┘              │ (not in use)│
                             ├─────────────┤
                  Segment 2: │    Stack    │
                  Stack      │      ↑      │
                             └─────────────┘
```

각 세그먼트가 독립적으로 배치되므로 내부 단편화가 크게 줄어듭니다.

### 7.5 세그멘테이션의 장단점

**장점:**
- 내부 단편화 해결
- 세그먼트별 보호 비트 설정 가능
- 코드 세그먼트 등을 프로세스 간 공유 가능

**단점:**
- **외부 단편화(External Fragmentation)**: 가변 크기 세그먼트로 인해 물리 메모리에 구멍이 생깁니다.

```text
Initial State (after A, B, C allocated, then B deallocated):
┌───────────┐ 0KB
│     A     │     Total free space: 8KB + 6KB = 14KB
│   (4KB)   │     But cannot allocate 9KB!
├───────────┤ 4KB
│   Free    │     → Largest contiguous block: 8KB
│   (8KB)   │     → External Fragmentation!
├───────────┤ 12KB
│     C     │     Solution: Compaction
│   (6KB)   │     (move segments to consolidate free space)
├───────────┤ 18KB
│   Free    │
│   (6KB)   │
└───────────┘ 24KB
```

외부 단편화를 줄이기 위해 컴팩션(compaction)을 사용할 수 있지만, 이는 비용이 많이 듭니다.

## 8. Fragmentation 정리

메모리 단편화는 두 가지 유형이 있습니다:

### Internal Fragmentation (내부 단편화)
- 할당된 메모리 영역 **내부**에서 발생
- 할당받은 공간이 실제 필요한 것보다 큼
- Base and Bounds에서 발생
- 예: 16KB를 할당받았지만 7KB만 사용 → 9KB 낭비

### External Fragmentation (외부 단편화)
- 할당된 메모리 영역 **사이**에서 발생
- 총 여유 공간은 충분하지만 연속적이지 않음
- 세그멘테이션에서 발생
- 예: 8KB + 6KB = 14KB 여유 공간이 있지만 9KB 요청을 만족하지 못함

## 9. 다음 단계: 페이징

세그멘테이션의 외부 단편화 문제를 해결하기 위해 **페이징(Paging)**이 등장합니다. 페이징은 고정 크기 단위로 메모리를 나누어 외부 단편화를 제거합니다. 이는 다음 글에서 자세히 다루겠습니다.

## 정리

- **가상 메모리**는 프로세스에게 독립적이고 연속적인 주소 공간의 환상을 제공합니다.
- **투명성, 효율성, 보호**가 가상 메모리의 세 가지 목표입니다.
- **정적 재배치**는 보호와 재배치가 불가능하여 실용적이지 않습니다.
- **동적 재배치 (Base and Bounds)**는 하드웨어 지원으로 보호를 제공하지만 내부 단편화가 발생합니다.
- **세그멘테이션**은 내부 단편화를 해결하지만 외부 단편화 문제가 있습니다.
- **페이징**은 고정 크기 단위로 외부 단편화를 해결하는 다음 진화 단계입니다.

*다음 글: [[CS330] 5. 페이징과 TLB](/posts/cs330/5-paging-tlb/)*
