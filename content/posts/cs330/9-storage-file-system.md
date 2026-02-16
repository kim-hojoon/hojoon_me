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
            Spindle
                |
        +---------------+
        |   Platter     |
        |   - Surface   |
        |   - Track     |
        |   - Sector    |
        +---------------+
                |
          Arm Assembly
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

**CHS (Cylinder-Head-Sector)**: 물리적 위치 `<Cylinder #, Head #, Sector #>`로 직접 지정

**LBA (Logical Block Addressing)**: 디스크를 논리적 블록 배열 `[0, 1, ..., N-1]`로 추상화하여 현대 디스크에서 사용

### HDD 성능 요소

**T_I/O = T_seek + T_rotation + T_transfer**

- **Seek Time**: 암을 목표 실린더로 이동 (평균 4~9ms)
- **Rotational Delay**: 섹터가 회전해 오기를 대기 (평균 = 60/RPM/2, 예: 7200 RPM → 4.2ms)
- **Transfer Time**: 데이터 전송 (일반적으로 seek + rotation보다 훨씬 작음)

**성능 비교**: Random access는 ~6ms (seek + rotation 지배), Sequential access는 ~125 MB/s (transfer 지배) → **Sequential이 수백 배 빠름**

### I/O 스케줄링

| 알고리즘 | 설명 | 특징 |
|---------|------|------|
| **FCFS** | 도착 순서대로 처리 | 공평하지만 성능 낮음 |
| **SSTF** | 가장 가까운 요청 먼저 | Seek time 최소화, starvation 가능 |
| **SCAN** | 한 방향 이동하며 처리, 끝에서 방향 전환 | 공평성과 성능 균형 |
| **C-SCAN** | 한 방향만 서비스, 끝에서 처음으로 복귀 | 더 균등한 대기 시간 |

## 3. SSD (Solid State Drive)

### Flash Memory 특성

SSD는 NAND Flash 메모리를 사용한다. 기계 부품 없이 전자적으로만 동작한다.

**계층 구조**:
- **Page**: Read/Write 단위 (4KB ~ 16KB)
- **Block**: Erase 단위 (128KB ~ 256KB, 여러 pages 포함)

**핵심 특성**:
- **Erase-Before-Write**: 이미 쓰인 페이지는 직접 덮어쓸 수 없음, 블록 전체를 erase 후 쓰기 → **in-place update 불가능**
- **비대칭 성능**: Read (~25μs) < Write (~200μs) << Erase (~1.5ms)
- **제약**: P/E 사이클 제한 (SLC ~100K, MLC ~3K, TLC ~1K), Bit errors (ECC 필요), Bad blocks

### FTL (Flash Translation Layer)

FTL은 Flash의 특성을 숨기고 일반 블록 디바이스처럼 보이게 하는 소프트웨어 계층이다.

**주요 기능**:

1. **Address Mapping**: Logical → Physical 주소 변환. In-place update 불가하므로 수정 시 새 위치에 쓰고 mapping table 업데이트
2. **Garbage Collection**: 무효화된 페이지가 쌓인 블록에서 유효 페이지만 복사하여 블록 재활용
3. **Wear Leveling**: 모든 블록이 균등하게 소모되도록 데이터 분산

### HDD vs SSD 비교

| 특성 | SSD (Samsung 850 Evo) | HDD (Seagate M9T) |
|------|----------------------|-------------------|
| Sequential Read | 544 MB/s | 124 MB/s |
| Random Read (4KB) | 97,687 IOPS | 56 IOPS |
| Random Write (4KB) | 89,049 IOPS | 98 IOPS |
| 가격 (2TB 기준) | 470원/GB | 88원/GB |

**핵심**: SSD는 Random access에서 압도적 (IOPS 수천 배 차이), HDD는 가격 대비 용량 우위

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

각 블록/inode의 사용 여부를 비트로 표시. Inode bitmap과 Data bitmap으로 구성.

#### Inode Table

모든 inode 저장. 각 inode는 고정 크기 (128B 또는 256B).

#### Data Blocks

실제 파일/디렉토리 내용 저장. 디스크의 대부분을 차지.

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

**설정**: 12 direct + 1 single indirect pointer, 4KB 블록, 4-byte 포인터 → 한 블록에 1024 포인터

**최대 파일 크기**: Direct 48KB + Indirect 4MB = 약 4MB. 더 큰 파일은 double/triple indirect 추가.

### 파일 읽기 예제

`/foo/bar` 읽기 과정:

**open("/foo/bar")**: 루트 inode → 루트 데이터에서 `foo` 검색 → `foo` inode → `foo` 데이터에서 `bar` 검색 → `bar` inode 캐싱

**read()**: 캐싱된 `bar` inode에서 data block 주소 획득 → 데이터 블록들 읽기

경로 탐색은 open() 시 한 번만 수행.

### 파일 쓰기

**create("/foo/bar")**: 경로 탐색 → Inode bitmap 수정 → 새 inode 할당/초기화 → `foo` 디렉토리에 엔트리 추가

**write()**: 각 호출마다 Data bitmap 수정 → Data block 할당/쓰기 → Inode 업데이트 (size, 포인터)

### 캐싱 (Page Cache)

파일 시스템은 자주 사용하는 블록을 메모리에 캐싱한다.

- **Write buffering**: write()는 메모리에만 쓰고 나중에 디스크에 flush (최대 30초)
- **Read caching**: 읽은 블록을 메모리에 보관

**fsync() 시스템 콜**:
- 파일의 모든 dirty data와 metadata를 디스크에 강제로 flush
- 데이터 무결성이 중요한 경우 사용 (예: 데이터베이스)

## 7. 파일 할당 방식

| 방식 | 장점 | 단점 | 사용 예 |
|------|------|------|---------|
| **Contiguous** (연속) | Sequential 성능 우수 | 외부 단편화 심각, 크기 증가 어려움 | CD-ROM |
| **Linked** (연결) | 단편화 없음, 크기 증가 용이 | Random access 느림, 포인터 손상 위험 | TOPS-10 |
| **FAT** (연결 변형) | Random access 개선 (메모리 캐싱) | FAT 크기 제한 | MS-DOS, Windows |
| **Indexed** (인덱싱) | Random access 지원, 단편화 없음 | 작은 파일도 index block 필요 | Unix FFS, Ext2/3 (multi-level) |
| **Extent-based** | 메타데이터 오버헤드 감소, Sequential 성능 좋음 | 일부 단편화 가능 | Ext4, XFS |

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

Variable-sized names 지원. 파일 검색은 선형 탐색. 최적화: 정렬(이진 탐색), 해시 테이블, B-tree(대규모 디렉토리)

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
