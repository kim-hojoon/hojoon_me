---
title: "[CS330] 10. FFS와 Crash Consistency"
date: 2023-12-10
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "FFS", "Crash Consistency", "Journaling", "File System"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "VSFS의 성능 문제를 해결하는 FFS(Fast File System), 그리고 시스템 크래시 시 파일 시스템 일관성을 보장하는 Journaling 기법을 다룹니다."
aliases: ["/posts/cs330-10-ffs-crash-consistency/"]
ShowToc: true
TocOpen: true
---

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

*이전 글: [[CS330] 9. 저장장치와 파일 시스템](/posts/cs330/9-storage-file-system/)*

## 1. VSFS의 성능 문제

이전 글에서 살펴본 VSFS(Very Simple File System)는 기본적인 파일 시스템의 구조를 이해하는 데 유용했지만, 실제 사용에는 심각한 성능 문제가 있었다.

### 주요 문제점

**1. Poor Locality (낮은 지역성)**
- 데이터와 메타데이터가 디스크 전체에 무작위로 흩어짐
- Inode와 데이터 블록이 멀리 떨어져 있음
- 같은 디렉토리의 파일들이 연속되지 않은 inode 슬롯에 할당됨

**2. Fragmentation (단편화)**
- Freelist 방식으로 블록을 관리하면 시간이 지날수록 파일이 조각나게 됨
- 초기에는 성능이 괜찮지만, 몇 주 사용 후 디스크 대역폭의 3%만 활용

**3. 작은 블록 크기 (512 바이트)**
- 파일 인덱스가 너무 커짐
- 간접 블록이 많이 필요
- 전송률이 낮음 (한 번에 한 블록씩만 읽음)

실제 측정 결과, 원래 Unix 파일 시스템은 디스크 대역폭의 3~5%만 활용했고, 새로운 파일 시스템에서는 17.5%였으나 몇 주 후 3%로 떨어졌다.

## 2. FFS: Fast File System

1984년 McKusick, Joy, Leffler, Fabry가 개발한 **FFS(Fast File System)**는 이러한 문제를 해결하기 위해 "disk-aware" 설계를 도입했다.

### 핵심 아이디어: Disk-Awareness

FFS의 기본 철학은 다음과 같다:

1. **최신 시스템 측정**: 당시 시스템의 상태를 분석
2. **근본 문제 파악**: 원래 FS가 디스크를 랜덤 액세스 메모리처럼 다룸
3. **더 나은 시스템 구축**: FS 구조와 할당 정책을 디스크 인식적으로 설계

**핵심 원칙**: 관련된 것들을 가까이 배치하여 seek를 줄이고 디스크 활용도와 응답 시간을 개선한다.

### Cylinder Groups (실린더 그룹)

FFS는 디스크를 여러 **cylinder group** (또는 block group)으로 나눈다. 실린더는 디스크의 여러 표면에서 같은 거리에 있는 트랙들의 집합이다.

```text
[디스크 레이아웃]

전체 디스크:
+----------------+----------------+-----+------------------+
| Block group 0  | Block group 1  | ... | Block group N-1  |
+----------------+----------------+-----+------------------+

각 Block group의 내부 구조:
+----+----+----+--------+------------------+
| S  | IB | DB | Inodes |   Data Blocks    |
+----+----+----+--------+------------------+
  ↑    ↑    ↑      ↑            ↑
  |    |    |      |            |
  |    |    |      |            데이터 블록들
  |    |    |      아이노드 테이블
  |    |    데이터 비트맵
  |    아이노드 비트맵
  슈퍼블록 복사본 (신뢰성을 위해)
```

각 실린더 그룹은 자체적으로 superblock의 복사본, inode bitmap, data bitmap, inode table, data blocks를 가진다.

## 3. FFS의 배치 정책 (Allocation Policies)

FFS는 "관련된 것들을 가까이 배치"한다는 원칙을 다음과 같이 구현한다:

### 디렉토리 배치

새 디렉토리를 만들 때:
- 할당된 디렉토리 수가 적고 free inode가 많은 실린더 그룹을 선택
- 디렉토리를 여러 그룹에 분산시켜 밸런스를 유지

```text
[디렉토리 분산]

    Block Group 0
  ┌─────────────────┐
  │                 │
  │  Block Group 1  │
  │ ┌──────────────┐│
  │ │              ││
  │ │Block Group 2 ││
  │ │   ┌─────┐    ││
  │ │   │     │    ││
  │ └───┴─────┴────┘│
  └─────────────────┘

  디렉토리 배치: 각 그룹에 골고루 분산
```

### 파일 배치

같은 디렉토리의 파일들:
- **디렉토리의 inode와 같은 그룹**에 배치
- 파일의 **inode와 data block을 같은 그룹**에 배치
- 순차 접근을 위해 **연속된 블록을 인접한 섹터**에 배치

### 대용량 파일 처리

파일이 너무 크면 한 그룹을 가득 채워 다른 파일을 위한 공간이 없어진다. FFS는 이를 해결하기 위해:

- 파일을 **청크 단위로 분할** (예: 48KB 청크)
- 각 청크를 **다른 실린더 그룹에 분산**
- 첫 번째 청크: 파일의 inode와 같은 그룹
- 이후 청크들: 다른 그룹으로 분산

이렇게 하면 대용량 파일은 그룹 간 seek가 발생하지만, 각 청크 내에서는 순차 읽기가 가능하므로 전체적으로 좋은 성능을 유지한다.

## 4. FFS의 기타 개선사항

### Sub-block Allocation (서브블록)

큰 블록 크기(4KB 또는 8KB)는 대용량 파일에 좋지만 작은 파일에는 공간 낭비가 심하다. FFS는 **fragment** (서브블록)를 도입하여 블록을 더 작은 조각(예: 1KB)으로 나누고, 작은 파일이나 파일 끝부분에 사용한다. 파일이 커지면 fragment를 full block으로 변환한다.

### 기타 개선사항

- **긴 파일 이름**: 가변 길이 디렉토리 엔트리 (원래 14자 제한)
- **Symbolic Link**: Hard link 제약 극복, 다른 파일 시스템 참조 가능
- **Parameterized Placement**: 디스크 물리적 특성 반영한 배치 최적화

FFS는 디스크 대역폭의 **14% ~ 47%**를 활용했다 (원래 Unix FS는 3~5%). 현대 파일 시스템(Linux ext2/3/4)도 FFS의 아이디어를 계승하고 있다.

## 5. Crash Consistency 문제

파일 시스템은 디스크에 여러 번 쓰기를 수행해야 하는데, 중간에 전원이 차단되거나 시스템이 크래시하면 어떻게 될까?

### 문제 상황

데이터 블록 Db를 파일에 추가하는 경우를 생각해보자. 업데이트해야 할 3가지:
1. **Inode (I')**: 새 블록 포인터 추가
2. **Data Bitmap (B')**: 블록 5 할당됨으로 표시
3. **Data Block (Db)**: 실제 데이터

하지만 디스크는 **한 번에 하나의 섹터**만 원자적으로 쓸 수 있다. 따라서 3개 업데이트 중 일부만 완료된 상태에서 크래시가 발생할 수 있다.

### 6가지 크래시 시나리오

```text
[크래시 시나리오 매트릭스]

업데이트 항목: I (Inode), B (Bitmap), D (Data)

┌─────────────────┬──────────────┬───────────────────────────┐
│ 시나리오        │ 완료된 쓰기  │ 결과                      │
├─────────────────┼──────────────┼───────────────────────────┤
│ 1. 모두 완료    │ I, B, D      │ ✓ 문제 없음               │
│ 2. 아무것도 X   │ (없음)       │ ✓ 문제 없음 (캐시에만)    │
│ 3. D만 완료     │ D            │ ✓ 문제 없음 (고아 블록)   │
│ 4. I만 완료     │ I            │ ✗ 가비지 데이터 읽음      │
│ 5. B만 완료     │ B            │ ✗ 공간 누수 (space leak)  │
│ 6. I + B        │ I, B         │ ✗ 가비지 데이터 읽음      │
│ 7. I + D        │ I, D         │ ✗ Bitmap 불일치           │
│ 8. B + D        │ B, D         │ ✗ 공간 누수               │
└─────────────────┴──────────────┴───────────────────────────┘
```

**시나리오 4 (I만 완료)**: Inode는 블록 5를 가리키지만 bitmap은 free라고 표시. 파일을 읽으면 블록 5의 이전 내용(가비지)을 읽게 됨.

**시나리오 5 (B만 완료)**: Bitmap은 블록 5가 사용 중이라고 하지만 어떤 inode도 가리키지 않음. 해당 블록은 영원히 사용 불가능 (space leak).

**시나리오 7 (B + D만 완료)**: Bitmap은 할당됨인데 inode는 가리키지 않음. 나중에 블록 5를 다른 파일에 할당하면 두 inode가 같은 블록을 공유하게 됨!

## 6. 해결 방법 1: fsck (File System Checker)

**fsck**는 부팅 시 파일 시스템 전체를 스캔하여 불일치를 찾아 수정하는 Unix 도구다 (Windows의 Scandisk와 유사).

### fsck의 동작

Superblock, free block, inode 상태, link count, duplicate, bad block, 디렉토리 등을 전체적으로 스캔하여 불일치를 수정한다.

### fsck의 문제점

- **매우 느림**: 전체 디스크를 스캔해야 함 (600GB 디스크에 약 70분 소요)
- **불완전함**: 파일 시스템 메타데이터의 일관성만 보장, 파일 데이터 손실은 방지하지 못함
- **확장성 부족**: 디스크 용량이 커질수록 시간이 더 오래 걸림

## 7. 해결 방법 2: Journaling (Write-Ahead Logging)

**Journaling**은 데이터베이스 트랜잭션에서 사용하는 write-ahead logging 기법을 파일 시스템에 적용한 것이다.

### 기본 아이디어

1. 변경 사항을 **최종 위치에 쓰기 전에** journal(log)에 먼저 기록
2. Journal에 안전하게 기록된 후에만 실제 위치에 **checkpoint**
3. 크래시 발생 시 journal을 **replay**하여 복구

### Data Journaling 과정

```text
[Journaling 과정]

Step 1: Journal Write
┌─────────────────────────────────────┐
│ Journal Area                        │
│ +-----+-----+-----+-----+-----+     │
│ | TxB | IN' | DB' | Db  | TxE |     │
│ +-----+-----+-----+-----+-----+     │
│   ↑                          ↑      │
│   |                          |      │
│   Transaction Begin     Transaction End
└─────────────────────────────────────┘

Step 2: Journal Commit
- TxE (Transaction End) 블록 쓰기 완료
- 이제 크래시 시 복구 가능

Step 3: Checkpoint
┌─────────────────────────────────────┐
│ File System Main Area               │
│ +------+------+---------+           │
│ | DB'  | IN'  |   Db    |           │
│ +------+------+---------+           │
│   ↑      ↑        ↑                 │
│   |      |        |                 │
│   Bitmap Inode   Data               │
└─────────────────────────────────────┘
Journal 데이터를 실제 위치로 복사

Step 4: Free
- Journal 영역 해제
- 다음 트랜잭션을 위해 재사용 가능
```

### Journaling 단계별 상세

**1. Journal Write**: TxB, 메타데이터(IN', DB'), 데이터(Db), TxE를 journal에 기록
**2. Journal Commit**: TxE를 디스크에 쓰면 트랜잭션 committed (복구 가능)
**3. Checkpoint**: Journal 내용을 최종 위치에 복사 (IN' → Inode, DB' → Bitmap, Db → Data)
**4. Free**: Journal 영역 해제

크래시 후 부팅 시 journal을 스캔하여 committed 트랜잭션(TxE 있음)을 찾아 replay하고 checkpointing을 다시 수행한다.

**복구 시나리오**:
- Journal Write 중 크래시: TxE 없음 → 트랜잭션 무시, 변경 전 상태 유지
- Checkpoint 전 크래시: TxE 있음 → Replay 후 checkpointing 재수행

### Metadata Journaling (Ordered Journaling)

Data journaling은 데이터를 두 번 쓰므로 오버헤드가 크다. **Metadata journaling**은 메타데이터만 journal에 기록한다: 데이터를 먼저 최종 위치에 쓰고, 메타데이터(TxB, IN', DB', TxE)만 journal에 기록한 후 checkpoint한다. 이 방식은 **ext3의 기본 모드**(ordered mode)이며 훨씬 빠르다.

fsck보다 빠르고 메타데이터 일관성을 보장하지만, 쓰기 오버헤드와 journal 영역이 필요하다.

## 8. 해결 방법 3: Copy-on-Write (COW)

**Copy-on-Write**는 ZFS, Btrfs 등에서 사용하는 방식으로, 기존 데이터를 절대 덮어쓰지 않는다.

### COW 동작 방식

기존 데이터를 덮어쓰지 않고 새 위치에 쓴 후, 경로상의 메타데이터(inode, 루트)를 새로 생성한다. 마지막으로 슈퍼블록의 루트 포인터를 업데이트하는 한 번의 원자적 쓰기로 모든 변경사항이 반영된다.

원자적 업데이트와 스냅샷 지원이 장점이지만, 쓰기 증폭과 fragmentation이 단점이다.

## 9. 해결 방법 4: Log-Structured File System (LFS)

**LFS**는 모든 쓰기를 순차적으로 로그처럼 기록한다.

디스크를 순차 로그로 취급하여 모든 업데이트를 로그 끝에 추가한다. In-place 업데이트가 없어 원자성을 보장한다.

순차 쓰기로 최대 성능을 내고 크래시 일관성을 자연스럽게 보장하지만, 파일이 흩어져 읽기 성능이 떨어지고 garbage collection이 필요하다.

## 10. 정리: Crash Consistency 해결 방법 비교

| 방법          | 복구 시간 | 성능 오버헤드 | 일관성 보장 | 사용 예              |
|---------------|-----------|---------------|-------------|----------------------|
| fsck          | 매우 느림 | 없음          | 메타데이터  | 초기 Unix FS         |
| Journaling    | 빠름      | 중간          | 메타데이터  | ext3/4, NTFS         |
| COW           | 즉시      | 쓰기 증폭     | 완전        | ZFS, Btrfs           |
| LFS           | 즉시      | GC 필요       | 완전        | F2FS (Flash 최적화)  |

현대 파일 시스템 대부분은 **journaling**을 사용한다. COW와 LFS는 특정 상황(스냅샷, SSD)에 유리하다.

## 시리즈 마무리: CS330 Operating Systems

이것으로 CS330 운영체제 시리즈를 마무리한다. 우리는 **가상화** (프로세스, 메모리), **동시성** (스레드, 락, 세마포어), **영속성** (디스크, 파일 시스템, 크래시 일관성)을 다루었다.

운영체제는 **추상화와 보호의 예술**이다. 복잡한 하드웨어를 단순한 인터페이스로 제공하고, 프로세스를 격리하며, 제한된 자원을 효율적으로 관리한다. FFS의 지역성 최적화와 Journaling의 안정성-성능 균형이 그 좋은 예다. 이러한 지식은 시스템 프로그래밍과 성능 최적화의 토대가 될 것이다.

---

*이 시리즈의 모든 글은 KAIST CS330 강의 내용을 바탕으로 작성되었습니다.*
