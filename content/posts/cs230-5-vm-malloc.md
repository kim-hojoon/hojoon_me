---
title: "[CS230] 5. 가상 메모리와 동적 할당 - Virtual Memory & Malloc"
date: 2022-12-15
draft: false
tags: ["CS230", "Systems Programming", "KAIST", "Virtual Memory", "Malloc", "Page Table"]
categories: ["Computer Science"]
summary: "가상 메모리의 원리(페이지 테이블, 주소 변환, TLB), 동적 메모리 할당(malloc/free, implicit free list, fragmentation)을 다룹니다."
ShowToc: true
TocOpen: true
---

> KAIST CS230 시스템프로그래밍 (2022 Fall) 수업 내용을 정리한 시리즈입니다.
> 교재: *Computer Systems: A Programmer's Perspective* (CS:APP 3rd Edition)

---

## 1. 가상 메모리 (Virtual Memory)

### 왜 Virtual Memory인가?

- **Physical Memory**는 실제 DRAM에 존재하는 메모리
- **Virtual Memory**는 프로세스에게 **독립적인 거대한 주소 공간**을 제공하는 추상화

Virtual Memory가 제공하는 것:
- 각 프로세스가 **전체 주소 공간을 독점**하는 것처럼 보임
- 실제로는 DRAM과 디스크를 조합하여 관리
- 프로세스 간 메모리 보호 (isolation)
- 효율적인 메모리 공유 (shared libraries)

### 핵심 관계

```text
VPN (Virtual Page Number) → PPN (Physical Page Number)
```

- 각 프로세스는 자기만의 **Virtual Address Space**를 가짐
- OS와 하드웨어가 협력하여 virtual address를 physical address로 변환

---

## 2. 페이지 (Page)

메모리를 고정 크기의 블록으로 나눈 단위:

- **Virtual Page (VP)**: 가상 주소 공간을 나눈 블록
- **Physical Page (PP)** = Page Frame: 물리 메모리를 나눈 블록
- 일반적인 page 크기: **4 KB** (= 2^12 bytes)

각 virtual page는 3가지 상태 중 하나:

| 상태 | 설명 |
|------|------|
| **Unallocated** | 아직 할당되지 않은 페이지 |
| **Cached** | 물리 메모리(DRAM)에 올라와 있는 페이지 |
| **Uncached** | 할당되었지만 디스크에만 있는 페이지 |

### Page Number와 Offset

가상 주소 구조 (예: 14-bit address, 4 KB pages):

```text
┌─────────────┬──────────┐
│ VPN (상위)   │ Offset   │
│ (page 번호)  │ (12 bit) │
└─────────────┴──────────┘
```

- **VPN**: 어떤 page인지 결정
- **Offset**: page 내에서의 위치 (page 크기가 4 KB이므로 12 bit)
- Physical Address도 동일 구조: PPN + Offset

---

## 3. 페이지 테이블 (Page Table)

각 프로세스마다 하나의 page table을 가지며, VPN → PPN 매핑을 저장한다.

Page table은 **DRAM**에 존재하고, OS가 관리한다.

### PTE (Page Table Entry)

Page table의 각 항목:

```text
┌───────┬─────────────┐
│ Valid  │ PPN / Disk  │
│  bit   │  address    │
└───────┴─────────────┘
```

| 필드 | 설명 |
|------|------|
| **Valid bit** | 1: DRAM에 캐시됨, 0: 디스크에 있거나 미할당 |
| **Permission bits** | 읽기/쓰기/실행 권한 |
| **PPN** | 물리 페이지 번호 (valid일 때) |
| **Disk address** | 디스크 위치 (invalid일 때) |

- Valid bit = 1 → PPN을 읽어서 물리 주소 계산 → **Page Hit**
- Valid bit = 0, allocated → **Page Fault** 발생
- NULL이면 → 미할당 페이지

---

## 4. Page Fault

요청한 virtual page가 DRAM에 없을 때 발생하는 **fault exception**:

1. CPU가 PTE를 확인 → valid bit = 0
2. **Page fault exception** 발생 → OS 커널로 제어 이동
3. 커널이 디스크에서 해당 page를 DRAM으로 로드
4. 필요하면 다른 page를 디스크로 내보냄 (**eviction**)
5. PTE 갱신 후 원래 명령어 **재실행**

이 메커니즘을 **Demand Paging**이라 한다 — 실제로 접근할 때만 메모리에 올린다.

### 관련 용어

| 용어 | 설명 |
|------|------|
| **Demand Paging** | page fault 발생 시 디스크에서 page를 가져오는 방식 |
| **Working Set** | 프로세스가 활발히 사용하는 page들의 집합 |
| **Thrashing** | working set > main memory → 끊임없이 page fault 발생하며 성능 급락 |

---

## 5. 주소 변환 (Address Translation)

### 기본 과정

```text
Virtual Address (VA)
    ↓
CPU → MMU (Memory Management Unit) → Page Table 참조
    ↓
Physical Address (PA)
    → DRAM 접근
```

1. CPU가 virtual address 생성
2. MMU가 VPN 추출 → page table에서 PTE 조회
3. PTE의 PPN + offset = **physical address**
4. 해당 물리 주소로 DRAM 접근

### PTBR (Page Table Base Register)

- CPU에 있는 레지스터로, 현재 프로세스의 page table 시작 주소를 저장
- Context switch 시 PTBR이 갱신됨

---

## 6. TLB (Translation Lookaside Buffer)

Page table이 DRAM에 있으므로, 매번 참조하면 메모리 접근이 **2번** 필요하다:
1. PTE 조회를 위한 DRAM 접근
2. 실제 데이터를 위한 DRAM 접근

이를 해결하기 위한 **하드웨어 캐시**가 TLB:

- MMU 내부에 있는 소형 캐시
- 최근 사용된 PTE를 저장
- **TLB Hit**: PTE를 TLB에서 바로 가져옴 (매우 빠름)
- **TLB Miss**: DRAM의 page table에서 PTE를 가져와 TLB에 저장

---

## 7. Multi-level Page Table

문제: 32-bit 시스템에서 4 KB 페이지 → page table entry 수 = 2^20 = **1M개**
→ page table 자체가 4 MB의 메모리를 차지!

해결: **다단계 page table**

```text
Level 1 Page Table (1024 entries)
    ↓ (VPN 상위 10 bit)
Level 2 Page Table (1024 entries 각각)
    ↓ (VPN 하위 10 bit)
PPN
```

장점:
- Level 1에서 참조하지 않는 영역의 Level 2 table은 **생성하지 않음**
- 실제 사용하는 메모리만큼만 page table이 존재 → 대폭 절약
- Level 2의 PTE가 NULL이면 해당 virtual page는 미할당

---

## 8. DRAM Cache와 메모리 계층

### 메모리 계층 구조

```text
CPU Registers  ← 가장 빠름, 가장 작음
    ↕
L1 Cache (SRAM)      ← 수 ns
    ↕
L2 Cache (SRAM)
    ↕
L3 Cache (SRAM)
    ↕
Main Memory (DRAM)   ← 수십 ns
    ↕
Local Disk (SSD/HDD) ← 수 ms (DRAM 대비 10만배 느림)
    ↕
Remote Storage
```

DRAM은 디스크의 **캐시** 역할을 한다:
- Page 단위로 캐싱 (4 KB)
- Miss penalty가 매우 크므로 (디스크 접근) → fully associative + 정교한 교체 정책 사용

### Cache의 핵심 개념

- **Cache Hit**: 원하는 데이터가 상위 계층에 존재
- **Cache Miss**: 하위 계층에서 가져와야 함
- **Locality**: 캐시가 효과적인 이유
  - **Temporal locality**: 최근 접근한 데이터를 다시 접근할 가능성이 높음
  - **Spatial locality**: 인접한 주소의 데이터를 접근할 가능성이 높음

---

## 9. 동적 메모리 할당 (Dynamic Memory Allocation)

### 개요

```text
User Stack       ← 높은 주소
    ↓
   (빈 공간)
    ↑
Heap (via malloc) ← brk pointer가 상단을 가리킴
BSS (.bss)
Data (.data)
Text (.text)      ← 낮은 주소
```

- **Heap**: 동적으로 크기가 변하는 메모리 영역
- 프로그래머가 `malloc`으로 할당, `free`로 해제

### malloc 패키지

```c
#include <stdlib.h>

void *malloc(size_t size);
// 최소 size bytes의 메모리 블록 반환
// 8-byte (x86) 또는 16-byte (x86-64) 정렬 보장
// 실패 시 NULL 반환

void free(void *ptr);
// malloc으로 할당받은 블록을 반환
// ptr은 반드시 malloc/realloc의 반환값이어야 함

void *calloc(size_t n, size_t size);
// n × size bytes 할당, 0으로 초기화

void *realloc(void *ptr, size_t size);
// 기존 블록의 크기 변경
```

### 할당기의 제약

- 임의 순서의 malloc/free 요청을 처리해야 함
- malloc 요청에 즉시 응답해야 함 (버퍼링/재정렬 불가)
- free 메모리에서만 할당 가능
- 이미 할당된 블록은 이동 불가 (compaction 불가)
- 정렬 요구사항 충족 필요

### Fragmentation (단편화)

**Internal Fragmentation:**
- 할당된 블록 내부에 사용하지 않는 공간
- 정렬, 헤더, 최소 크기 요구 등으로 발생

**External Fragmentation:**
- free 메모리의 **총량**은 충분하지만, 연속된 블록이 없어서 할당 불가
- 더 심각한 문제 — 예측이 어려움

### Implicit Free List

가장 기본적인 할당기 구현 방법:

각 블록의 구조:
```text
┌──────────────────────────┐
│ Header (size | allocated) │  ← 4 bytes
├──────────────────────────┤
│ Payload                   │
│ (사용자 데이터)            │
├──────────────────────────┤
│ Padding (optional)        │
└──────────────────────────┘
```

- **Header**: 블록 크기 + 할당 여부 (하위 1 bit)
  - 크기가 8의 배수이므로 하위 3 bit가 항상 0 → 1 bit를 할당 플래그로 사용
- 블록들이 연속적으로 배치 → header의 size로 다음 블록을 찾아감

### Free 블록 관리

**Placement (배치) 정책:**

| 정책 | 설명 | 장단점 |
|------|------|--------|
| First fit | 처음 발견한 충분한 블록 사용 | 빠름, 앞쪽에 작은 조각 생김 |
| Next fit | 마지막 검색 위치부터 시작 | first fit보다 약간 빠름 |
| Best fit | 가장 적합한 크기의 블록 선택 | 외부 단편화 최소, 느림 |

**Splitting (분할):**
- 할당 요청보다 큰 free 블록을 찾으면, 필요한 만큼만 할당하고 나머지는 새 free 블록으로

**Coalescing (병합):**
- `free()` 시 인접한 free 블록들을 합쳐서 큰 블록으로 만듦
- 외부 단편화를 줄이는 핵심 기법
- **Immediate coalescing**: free 즉시 병합
- **Deferred coalescing**: 나중에 한꺼번에 병합

### Boundary Tag (Footer)

블록 끝에도 header와 동일한 정보를 저장:

```text
┌─────────┐
│ Header  │
├─────────┤
│ Payload │
├─────────┤
│ Footer  │  ← header와 동일한 정보
└─────────┘
```

- **이전 블록**의 크기/할당 상태를 O(1)에 확인 가능
- 양방향 coalescing이 가능해짐 (앞뒤 블록 모두 병합)
- 단점: 각 블록마다 4 bytes 추가 오버헤드

---

## 10. 일반적인 메모리 관련 버그

| 버그 | 설명 |
|------|------|
| Dangling pointer | `free()` 후에도 포인터를 계속 사용 |
| Memory leak | `malloc()` 후 `free()`를 하지 않음 |
| Double free | 같은 포인터를 두 번 `free()` |
| Buffer overflow | 할당된 크기를 넘어서 쓰기 |
| Use after free | 해제된 메모리에 접근 |

---

## 정리

| 주제 | 핵심 |
|------|------|
| Virtual Memory | 프로세스마다 독립적 주소 공간, DRAM+디스크 조합 |
| Page Table | VPN → PPN 매핑, PTE에 valid bit + 권한 + 주소 |
| Page Fault | DRAM miss → 디스크에서 로드 (demand paging) |
| TLB | PTE 캐시, 주소 변환 속도 향상 |
| Multi-level PT | 사용 영역만 page table 생성 → 메모리 절약 |
| malloc/free | heap 영역 동적 관리, fragmentation 주의 |
| Implicit Free List | header에 size\|alloc 저장, coalescing으로 단편화 완화 |

---

*이전 글: [[CS230] 4. 링킹과 예외 제어 흐름](/posts/cs230-4-linking-ecf/)*
*다음 글: [[CS230] 6. I/O, 네트워크, 동시성](/posts/cs230-6-io-network-concurrency/)*
