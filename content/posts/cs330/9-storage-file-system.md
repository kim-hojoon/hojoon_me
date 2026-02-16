---
title: "[CS330] 9. 저장장치와 파일 시스템"
date: 2023-12-09
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "Storage", "File System", "Inode", "HDD", "SSD"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "HDD와 SSD의 동작 원리, I/O 스케줄링, 그리고 파일 시스템의 구조(Inode, 디렉토리, VSFS)를 정리합니다."
aliases: ["/posts/cs330-9-storage-file-system/"]
ShowToc: true
TocOpen: true
---

*이전 글: [[CS330] 8. 동시성 버그와 교착 상태](/posts/cs330/8-concurrency-bugs-deadlock/)*

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

## 1. Persistence의 필요성

이전 장들에서는 CPU 가상화(virtualization)와 메모리 가상화를 다루었다. 이번 장에서는 운영체제의 세 번째 핵심 기능인 **영구 저장(persistence)**에 대해 다룬다.

### 왜 영구 저장이 필요한가?

메모리(DRAM)는 휘발성(volatile)이므로 전원이 꺼지면 데이터가 사라진다. 프로그램 코드, 문서, 사진, 동영상 등 장기간 보존해야 하는 데이터를 위해서는 비휘발성(non-volatile) 저장장치가 필요하다.

### 메모리 계층 구조에서의 디스크

```text
[CPU Registers]     ← 가장 빠름, 가장 비쌈
[Cache (L1/L2/L3)]
[Main Memory (DRAM)]
[Secondary Storage] ← 가장 느림, 가장 저렴
  - HDD
  - SSD
```

저장장치는 메모리 계층의 최하단에 위치하며, 다음과 같은 특성을 가진다.

- **용량이 크다**: 수 TB까지 가능
- **가격이 저렴하다**: GB당 가격이 DRAM보다 훨씬 저렴
- **영구적이다**: 전원이 꺼져도 데이터 보존
- **느리다**: 접근 시간이 밀리초(ms) 단위로 DRAM보다 10만 배 이상 느림

## 2. HDD (Hard Disk Drive)

### HDD 구조

HDD는 기계적(mechanical) 부품과 전자(electronic) 부품으로 구성된다.

```text
                Spindle (회전축)
                    |
        +-----------+-----------+
        |   Platter (원판)     |
        |   - Surface (표면)   |
        |   - Track (트랙)     |
        |   - Sector (섹터)    |
        +---------------------+
              |
         Arm Assembly (암 어셈블리)
              |
         Read-Write Head
```

**주요 구성 요소**:

- **Platter (플래터)**: 데이터를 저장하는 원형 디스크, 여러 개가 스핀들에 쌓임
- **Surface (표면)**: 각 플래터는 양면을 사용
- **Track (트랙)**: 표면에 동심원으로 그려진 원형 경로
- **Sector (섹터)**: 트랙을 나눈 단위, 전통적으로 512B 또는 4KB
- **Cylinder (실린더)**: 모든 플래터의 같은 위치 트랙들의 집합
- **Arm Assembly**: 읽기/쓰기 헤드를 이동시키는 기계 장치
- **Read-Write Head**: 실제로 데이터를 읽고 쓰는 헤드

### 디스크 주소 지정

**CHS (Cylinder-Head-Sector) 방식**:
- 각 블록을 `<Cylinder #, Head #, Sector #>`로 주소 지정
- OS가 디스크의 물리적 구조를 알아야 함

**LBA (Logical Block Addressing) 방식**:
- 디스크를 논리적인 블록 배열 `[0, 1, 2, ..., N-1]`로 추상화
- 디스크 내부에서 LBA를 물리적 위치로 매핑
- 현대 디스크(SATA, SAS)는 LBA 사용

```text
LBA:  [0][1][2][3][4][5][6][7][8][9][10]...
        ↓
물리적 위치: Disk가 내부적으로 매핑
```

### HDD 성능 요소

디스크 접근 시간은 세 가지 요소의 합으로 결정된다.

**T_I/O = T_seek + T_rotation + T_transfer**

#### 1) Seek Time (탐색 시간)

디스크 암을 목표 실린더로 이동시키는 시간.

- 거리에 비례하지만 순수 선형 관계는 아님 (가속/감속 필요)
- 평균 탐색 시간은 전체 탐색 시간의 약 1/3
- 예: Cheetah 15K.5 HDD → 평균 4ms

#### 2) Rotational Delay (회전 지연)

헤드가 목표 섹터가 회전해 오기를 기다리는 시간.

- RPM (Revolutions Per Minute)에 의존
- 일반 HDD: 5400, 7200 RPM
- 서버용 HDD: 10K, 15K RPM

평균 회전 지연 = (60초 / RPM) / 2

예시:
- 7200 RPM: (60 / 7200) / 2 = 4.2ms
- 15000 RPM: (60 / 15000) / 2 = 2ms

#### 3) Transfer Time (전송 시간)

디스크 표면에서 데이터를 읽어 디스크 컨트롤러로 전송하고 다시 호스트로 보내는 시간.

- 전송 속도에 의존 (예: 125 MB/s, 105 MB/s)
- 일반적으로 seek + rotation보다 훨씬 작음

### HDD 성능 비교 예제

| 특성 | Cheetah 15K.5 | Barracuda |
|------|---------------|-----------|
| 용량 | 300 GB | 1 TB |
| RPM | 15,000 | 7,200 |
| 평균 Seek | 4 ms | 9 ms |
| 최대 전송 속도 | 125 MB/s | 105 MB/s |

**Random Read (4 KB)**:
- Cheetah: 4ms + 2ms + 0.032ms ≈ 6ms (0.66 MB/s)
- Barracuda: 9ms + 4.2ms + 0.037ms ≈ 13.2ms (0.31 MB/s)

**Sequential Read (100 MB)**:
- Cheetah: 100MB / 125MB = 0.8s (125 MB/s)
- Barracuda: 100MB / 105MB = 0.95s (105 MB/s)

**핵심**: Sequential 접근은 Random 접근보다 수백 배 빠르다!

### I/O 스케줄링

여러 I/O 요청이 큐에 대기 중일 때, 어떤 순서로 처리할 것인가?

#### FCFS (First-Come-First-Served)

- 도착 순서대로 처리
- 공평하지만 성능이 떨어질 수 있음

#### SSTF (Shortest Seek Time First)

- 현재 헤드 위치에서 가장 가까운 요청을 먼저 처리
- Seek time을 최소화하지만 starvation 발생 가능

#### SCAN (Elevator Algorithm)

- 헤드가 한 방향으로 이동하며 경로상의 모든 요청 처리
- 끝에 도달하면 방향 전환
- 공평성과 성능의 균형

#### C-SCAN (Circular SCAN)

- 한 방향으로만 요청을 처리
- 끝에 도달하면 처음으로 빠르게 이동 (서비스 없이)
- SCAN보다 더 균등한 대기 시간

## 3. SSD (Solid State Drive)

### Flash Memory 기본

SSD는 NAND Flash 메모리를 사용한다. 기계 부품 없이 전자적으로만 동작한다.

```text
Block (Erase 단위)
  ├─ Page 0 (Read/Write 단위, 보통 4KB)
  ├─ Page 1
  ├─ Page 2
  └─ ...
```

**계층 구조**:
- **Page**: Read/Write의 기본 단위 (4KB ~ 16KB)
- **Block**: Erase의 기본 단위 (128KB ~ 256KB, 수십~수백 pages)

### Flash Memory 특성

#### Erase-Before-Write

```text
초기 상태:  [1][1][1][1][1][1][1][1]
             ↓ write (program)
쓰기 후:    [1][1][0][1][1][0][1][0]
             ↓ erase
지운 후:    [1][1][1][1][1][1][1][1]
```

- 이미 데이터가 있는 페이지에는 직접 쓸 수 없음
- 먼저 전체 블록을 erase해야 함 (블록 전체를 1로 설정)
- 따라서 **in-place update 불가능**

#### 읽기/쓰기/지우기 비대칭

- **Read**: 빠름 (~25μs)
- **Write**: 중간 (~200μs)
- **Erase**: 매우 느림 (~1.5ms)
- Erase 단위(Block) > Read/Write 단위(Page)

#### NAND Flash 종류

| 종류 | Cell 당 비트 | Program/Erase 사이클 | 특징 |
|------|-------------|---------------------|------|
| SLC | 1 bit | ~100,000 | 빠르고 내구성 좋음, 비쌈 |
| MLC | 2 bits | ~3,000 | 중간 성능, 중간 가격 |
| TLC | 3 bits | ~1,000 | 느리고 내구성 낮음, 저렴 |

#### 제약 사항

- **Limited P/E cycles**: 셀의 수명이 제한적 (Wear out)
- **Bit errors**: ECC (Error Correction Codes) 필요
- **Bad blocks**: 공장 출하 시 또는 런타임에 발생

### FTL (Flash Translation Layer)

FTL은 Flash의 특성을 숨기고 일반 블록 디바이스처럼 보이게 하는 소프트웨어 계층이다.

```text
File System
     ↓ (Logical Block Address)
    FTL
     ↓ (Physical Page Address)
 Flash Memory
```

**주요 기능**:

1. **Address Mapping**: Logical → Physical 주소 변환
2. **Wear Leveling**: 모든 블록이 균등하게 소모되도록 관리
3. **Garbage Collection**: 무효화된 페이지를 포함한 블록을 재활용

#### Address Mapping 예제

Flash는 in-place update가 불가능하므로 수정 시 새 위치에 쓰고 mapping table을 업데이트한다.

```text
Page Map Table        Physical Flash
 LPN | PPN           PBN 0: [0][1][2][8]
  0  |  0            PBN 1: [4][5][9][3]
  1  |  1            PBN 2: [X][X][X][X]
  2  |  2
  3  |  7            X: Invalid (old data)
  4  |  4
  5  |  8            ← 새로 쓴 페이지 5
```

논리 페이지 5를 수정하려면:
1. 새 물리 페이지(예: PBN 1의 페이지 2)에 쓰기
2. Page Map Table 업데이트: LPN 5 → PPN 8
3. 이전 물리 페이지(PBN 1의 페이지 1)는 무효화

### Garbage Collection

무효화된 페이지가 쌓이면 새로운 쓰기 공간이 부족해진다. GC는 유효한 페이지만 복사하여 블록을 재활용한다.

```text
Before GC:
Block 0: [Valid][Valid][Invalid][Valid]
Block 1: [Invalid][Valid][Invalid][Invalid]

After GC:
Block 0: [Valid][Valid][Valid][Valid]  ← 유효 페이지 모음
Block 1: [Erased] ← 재사용 가능
```

### Wear Leveling

모든 블록이 균등하게 사용되도록 FTL이 자주 쓰지 않는 블록으로도 데이터를 분산시킨다.

### HDD vs SSD 비교

| 특성 | SSD (Samsung 850 Evo) | HDD (Seagate M9T) |
|------|----------------------|-------------------|
| 용량 | 2TB | 2TB |
| Form Factor | 2.5", 66g | 2.5", 130g |
| DRAM | 2 GB | 32 MB |
| 인터페이스 | SATA-3 (6.0 Gbps) | SATA-3 (6.0 Gbps) |
| 소비 전력 | 3.7 / 4.7 / 0.05 W | 2.3 / 0.7 / 0.18 W |
| Sequential Read | 544 MB/s | 124 MB/s |
| Sequential Write | 520 MB/s | 124 MB/s |
| Random Read (4KB) | 97,687 IOPS | 56 IOPS |
| Random Write (4KB) | 89,049 IOPS | 98 IOPS |
| 평균 Seek | - | 12/14 ms |
| 평균 Latency | - | 5.6 ms |
| 가격 | 940,910원 (470원/GB) | 175,900원 (88원/GB) |

**핵심**:
- SSD는 Random access에서 압도적 (IOPS 수천 배 차이)
- HDD는 가격 대비 용량에서 여전히 우위
- SSD는 Sequential/Random 성능 차이가 HDD보다 작음

## 4. 파일 시스템 (File System)

### 파일 시스템의 역할

디스크는 블록(sector) 배열로 추상화되지만, 사용자는 파일(file)과 디렉토리(directory) 단위로 데이터를 다룬다. 파일 시스템은 이 두 추상화를 연결한다.

```text
사용자 관점:  파일/디렉토리 계층 구조
              ↕ (파일 시스템)
디스크 관점:  블록 배열 [0, 1, 2, ..., N-1]
```

### 파일 (File)

**파일의 정의**:
- Named collection of related information
- 영구 저장장치에 기록된 연관 정보의 집합

**파일은 두 부분으로 구성**:
1. **Data (내용)**: 파일의 실제 데이터 (바이트 시퀀스)
2. **Metadata (속성)**: 파일에 대한 정보
   - 파일 타입 (일반 파일, 디렉토리, ...)
   - 소유자 (UID)
   - 권한 (RWX)
   - 크기 (bytes)
   - 타임스탬프 (access, create, modify)
   - 데이터 블록 위치

### Inode (Index Node)

파일의 메타데이터는 **inode**라는 자료구조에 저장된다. 각 파일은 고유한 inode 번호를 가진다.

```text
┌─────────────────────┐
│     Inode #32       │
├─────────────────────┤
│ type: regular file  │
│ uid: 1000          │
│ rwx: rw-r--r--     │
│ size: 8192 bytes   │
│ blocks: 2          │
│ atime: ...         │
│ ctime: ...         │
│ mtime: ...         │
│ links_count: 1     │
│ addrs[N]: [5,12,...]│
└─────────────────────┘
```

**주의**: Inode는 파일 이름을 포함하지 않는다! 이름은 디렉토리에 저장된다.

### 디렉토리 (Directory)

디렉토리는 **파일 이름 → inode 번호** 매핑을 저장하는 특수한 파일이다.

```text
디렉토리 구조:
┌───────────────────────┐
│ File Name → Inode #   │
├───────────────────────┤
│  .        →    2      │
│  ..       →    2      │
│  foo      →   12      │
│  bar      →   24      │
│  foobar   →   24      │
└───────────────────────┘
```

- `.`: 현재 디렉토리
- `..`: 부모 디렉토리
- 여러 이름이 같은 inode를 가리킬 수 있음 (hard link)

### 경로 탐색 (Path Traversal)

파일 경로 `/foo/bar`를 열려면:

1. 루트 디렉토리 `/`를 연다 (inode #2, 미리 알려진 번호)
2. `/`의 디렉토리 엔트리에서 `foo` 검색 → inode #X 획득
3. inode #X를 읽어 `foo`가 디렉토리인지 확인
4. `foo`의 디렉토리 엔트리에서 `bar` 검색 → inode #Y 획득
5. inode #Y를 읽어 파일 메타데이터와 데이터 블록 위치 획득

이처럼 파일 시스템은 경로를 따라 내려가며(walking down) 디렉토리를 탐색한다.

### Hard Link vs Symbolic Link

#### Hard Link

```bash
$ ln old.txt new.txt
```

- 새로운 디렉토리 엔트리를 생성하되 같은 inode를 가리킴
- 두 이름은 동등함 (원본/사본 구분 없음)
- Inode의 `links_count` 증가
- 파일 삭제는 `links_count`를 감소, 0이 되면 실제 삭제
- 파일 시스템 경계를 넘을 수 없음

```text
ls -i old.txt new.txt
67158084 old.txt
67158084 new.txt  ← 같은 inode
```

#### Symbolic Link (Soft Link)

```bash
$ ln -s old.txt new.txt
```

- 새로운 파일을 생성하고 원본 파일 경로를 저장
- 새로운 inode 번호를 가짐
- 원본 파일이 삭제되면 dangling reference 발생
- 파일 시스템 경계를 넘을 수 있음

```text
drwxr-x--- 2 remzi remzi 29 May 3 19:10 ./
drwxr-x--- 27 remzi remzi 4096 May 3 15:14 ../
rw-r----- 1 remzi remzi 6 May 3 19:10 file
lrwxrwxrwx 1 remzi remzi 4 May 3 19:10 file2 -> file
```

### 파일 시스템 마운트 (Mount)

파일 시스템을 사용하려면 먼저 마운트해야 한다.

- **Windows**: 드라이브 문자 할당 (C:, D:, ...)
- **Unix**: 기존 디렉토리에 마운트 (마운트 포인트)

```text
/
├─ users/
│   ├─ sue/       ← 파일 시스템 A
│   └─ jane/      ← 파일 시스템 B
└─ ...
```

여러 파일 시스템을 하나의 통합된 디렉토리 트리로 구성할 수 있다.

## 5. 파일 시스템 디스크 레이아웃

### VSFS: Very Simple File System

VSFS는 Unix 파일 시스템의 단순화된 버전이다. 디스크를 다음과 같이 구성한다.

```text
┌─┬─┬─┬─────┬─────┬───────────────────────────┐
│S│i│d│  I  │  I  │     D   D   D   D   D    │
└─┴─┴─┴─────┴─────┴───────────────────────────┘
 0 1 2  3-7   8-12        13-31

S: Superblock
i: Inode bitmap
d: Data bitmap
I: Inode blocks
D: Data blocks
```

#### Superblock

파일 시스템의 전역 메타데이터를 저장한다.

- 파일 시스템 타입 (예: VSFS)
- 블록 크기 (예: 4KB)
- 전체 블록 수
- Inode 개수
- Data/Inode bitmap 블록 위치
- ...

#### Bitmaps

각 블록/inode의 사용 여부를 비트로 표시한다.

- **Inode bitmap**: 각 비트는 inode가 사용 중(1) 또는 free(0)
- **Data bitmap**: 각 비트는 data block이 사용 중(1) 또는 free(0)

한 블록(4KB = 32768 bits)의 bitmap으로 최대 4096개 블록/inode를 추적 가능.

#### Inode Table

모든 inode를 저장하는 영역.

- 각 inode는 고정 크기 (보통 128B 또는 256B)
- 4KB 블록에 256B inode라면 16개 inode 저장 가능
- 총 80개 inode라면 5개 블록 필요

#### Data Blocks

실제 파일/디렉토리 내용을 저장하는 영역. 디스크의 대부분을 차지한다.

## 6. Inode 구조와 Multi-level Indexing

### Inode의 포인터 배열

Inode는 파일의 데이터 블록 위치를 포인터 배열로 저장한다. 하지만 Inode 크기가 고정되어 있어 큰 파일을 표현하기 어렵다.

해결책: **Multi-level Indexing** (다단계 인덱싱)

```text
┌────────────────┐
│  Inode         │
├────────────────┤
│  metadata      │
│  size          │
│  ...           │
├────────────────┤
│  direct[0]  ──┼──→ [Data Block 0]
│  direct[1]  ──┼──→ [Data Block 1]
│  ...          │
│  direct[11] ──┼──→ [Data Block 11]
├────────────────┤
│  single    ──┼──→ [Pointer Block]
│  indirect     │      ├─→ [Data]
│               │      ├─→ [Data]
│               │      └─→ [Data]
├────────────────┤
│  double    ──┼──→ [Pointer Block]
│  indirect     │      ├─→ [Pointer Block]
│               │      │      ├─→ [Data]
│               │      │      └─→ [Data]
│               │      └─→ ...
├────────────────┤
│  triple    ──┼──→ (더 깊은 계층)
│  indirect     │
└────────────────┘
```

### VSFS Inode 예제

**설정**:
- Inode에 12개 direct pointer + 1개 single indirect pointer
- 4-byte disk address (포인터)
- 4KB 블록 크기
- 따라서 한 블록에 1024개 포인터 저장 가능

**최대 파일 크기**:
- Direct: 12 × 4KB = 48KB
- Single indirect: 1024 × 4KB = 4MB
- **Total**: (12 + 1024) × 4KB = 4144KB ≈ 4MB

더 큰 파일을 지원하려면 double/triple indirect pointer를 추가한다.

### 파일 읽기 예제

VSFS에서 `/foo/bar` 파일의 처음 3개 블록을 읽는 과정:

#### 1) open("/foo/bar")

| 작업 | 읽는 블록 |
|------|----------|
| 루트 `/` 디렉토리의 inode 읽기 | Inode bitmap, Inode[2] |
| 루트 디렉토리 데이터에서 `foo` 검색 | Data block |
| `foo` inode 읽기 | Inode[X] |
| `foo` 디렉토리 데이터에서 `bar` 검색 | Data block |
| `bar` inode 읽기 | Inode[Y] |
| `bar` inode 내용을 메모리에 캐싱 | - |

#### 2) read()

| 작업 | 읽는 블록 |
|------|----------|
| `bar` inode에서 data block 주소 획득 (캐싱됨) | - |
| Data block 0 읽기 | Data[A] |
| Data block 1 읽기 | Data[B] |
| Data block 2 읽기 | Data[C] |

경로 탐색은 open() 시 한 번만 수행되고, read()는 데이터만 읽으면 된다.

### 파일 쓰기

새 파일 `/foo/bar`를 생성하고 3개 블록을 쓰는 과정:

#### 1) create("/foo/bar")

| 작업 | 읽는 블록 | 쓰는 블록 |
|------|----------|----------|
| 루트 inode 읽기 | Inode[2] | - |
| 루트 디렉토리 데이터 읽기 | Data block | - |
| `foo` inode 읽기 | Inode[X] | - |
| `foo` 디렉토리 데이터 읽기 | Data block | - |
| Inode bitmap 읽기/수정 | Inode bitmap | Inode bitmap |
| 새 inode 할당 및 초기화 | - | Inode[Y] |
| `foo` 디렉토리에 `bar` 엔트리 추가 | - | Data block |

#### 2) write()

각 write() 호출마다:

| 작업 | 읽는 블록 | 쓰는 블록 |
|------|----------|----------|
| Data bitmap 읽기/수정 | Data bitmap | Data bitmap |
| Data block 할당 및 쓰기 | - | Data[A] |
| `bar` inode 업데이트 (size, 포인터) | - | Inode[Y] |

3번의 write()이면 위 과정을 3번 반복.

### 캐싱 (Page Cache)

파일 시스템은 자주 사용하는 블록을 메모리에 캐싱한다.

- **Write buffering**: write()는 메모리에만 쓰고 나중에 디스크에 flush (최대 30초)
- **Read caching**: 읽은 블록을 메모리에 보관

**fsync() 시스템 콜**:
- 파일의 모든 dirty data와 metadata를 디스크에 강제로 flush
- 데이터 무결성이 중요한 경우 사용 (예: 데이터베이스)

## 7. 파일 할당 방식

파일의 데이터 블록을 디스크에 어떻게 배치할 것인가?

### Contiguous Allocation (연속 할당)

파일을 연속된 블록에 할당.

```text
[ ][A][A][A][ ][B][B][B][B][C][C][C][ ][ ]
```

- **Metadata**: `<starting block #, length>`
- **장점**: Sequential access 성능 우수, 간단한 random access
- **단점**: 심각한 외부 단편화(external fragmentation), 파일 크기 증가 어려움
- **예**: CD-ROM, IBM OS/360

### Linked Allocation (연결 할당)

각 블록이 다음 블록의 포인터를 포함.

```text
[ ][ ][A]→[A]→[A][ ][B]→[B]→[C]→[C]→[B]→[B][ ][C]
```

- **Metadata**: `<starting block #>`
- **장점**: 외부 단편화 없음, 파일 크기 증가 용이
- **단점**: Random access 느림, 포인터 공간 낭비, 포인터 손상 시 데이터 손실
- **예**: TOPS-10, Alto

### File Allocation Table (FAT)

Linked allocation의 변형. 모든 포인터를 별도 테이블(FAT)에 저장.

```text
[ ][ ][A][A][A][ ][B][B][C][C][B][B][ ][C]

FAT:
[0][0][19][20][-1][0][23][26][25][29][27][-1][0][-1]
 16 17 18  19  20  21 22  23  24  25  26  27 28  29
```

- **FAT은 메모리에 캐싱**되므로 Random access 성능 개선
- **예**: MS-DOS, Windows (FAT12, FAT16, FAT32)

### Indexed Allocation (인덱싱 할당)

각 파일에 포인터 배열(index block)을 할당.

```text
Index block for "jeep":
[19][-1][-1][-1]...

Disk:
[ ][ ][ ]...[19][data]...
```

- **장점**: 외부 단편화 없음, Random access 지원, 파일 증가 가능
- **단점**: 작은 파일도 index block 필요 (메타데이터 오버헤드)
- **해결책**: Multi-level indexing (Unix FFS, Ext2/3)

### Extent-based Allocation

여러 개의 연속 영역(extent)을 할당.

```text
Extent = <starting block #, length>

File X: [(18, 3), (23, 4), (29, 1)]
```

- **장점**: 메타데이터 오버헤드 감소, sequential access 성능 좋음
- **예**: Linux Ext4, XFS

## 8. 디렉토리 구현

디렉토리는 `<file name, inode number>` 엔트리의 리스트를 저장하는 특수 파일이다.

### VSFS 디렉토리

```text
┌─────────────────────────────────┐
│ inode | record | name  | name   │
│ num   | length | len   |        │
├─────────────────────────────────┤
│   5   │   4    │  2    │  .\0\0 │
│   2   │   4    │  3    │  ..\0  │
│  12   │   4    │  4    │ foo\0  │
│  24   │   8    │  7    │foobar\0│
└─────────────────────────────────┘
```

- **Variable-sized names** 지원 (Linux Ext2와 유사)
- **Record length**: 다음 엔트리까지의 바이트 수
- **Name length**: 실제 파일 이름 길이

파일을 찾으려면 디렉토리를 **선형 탐색(linear search)**해야 한다.

### 디렉토리 최적화

- **정렬**: 이진 탐색 가능 (하지만 삽입/삭제 비용 증가)
- **해시 테이블**: 빠른 검색, 확장성 문제
- **B-tree**: 대규모 디렉토리에 유용

## 9. 정리

이번 장에서는 영구 저장장치와 파일 시스템의 기초를 다루었다.

**저장장치**:
- **HDD**: 기계적 부품, sequential access에 유리, 느린 random access
- **SSD**: Flash memory 기반, 빠른 random access, 수명 제한, FTL 필요

**파일 시스템**:
- **추상화**: 블록 배열 → 파일/디렉토리
- **메타데이터**: Inode에 파일 속성 및 블록 위치 저장
- **다단계 인덱싱**: Direct/indirect pointers로 큰 파일 지원
- **디렉토리**: 이름 → inode 매핑
- **할당 방식**: Contiguous, Linked, Indexed, Extent-based
- **캐싱**: Page cache로 성능 향상, fsync()로 데이터 무결성 보장

다음 글에서는 더 발전된 파일 시스템인 FFS와 crash consistency 문제를 다룬다.

*다음 글: [[CS330] 10. FFS와 Crash Consistency](/posts/cs330/10-ffs-crash-consistency/)*
