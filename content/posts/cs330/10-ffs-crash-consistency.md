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

큰 블록 크기(4KB 또는 8KB)는 대용량 파일에 좋지만 작은 파일에는 공간 낭비가 심하다.

FFS는 이를 해결하기 위해 **fragment** (또는 sub-block)를 도입했다:
- 블록을 더 작은 조각(예: 1KB)으로 나눔
- 작은 파일이나 파일 끝부분에 fragment 사용
- 파일이 커지면 fragment를 full block으로 변환

```text
[Fragment 예시: Block size 4KB, Fragment size 1KB]

파일 5KB 추가 전:
+------+---+---+---+---+--------+--------+
| AAAA | B | A | B |   |        |        |
+------+---+---+---+---+--------+--------+
   ↑     ↑       ↑
   |     |       |
   |     |       file B (2KB)
   |     file B fragment
   file A (5KB, 1 full block + 1 fragment)

파일 5KB에 1KB 추가 후:
+------+------+---+---+--------+--------+
| AAAA | AAAA | B | A |        |        |
+------+------+---+---+--------+--------+
          ↑
          |
    새 블록에 재배치 (6KB = 1.5 blocks)
```

### 긴 파일 이름 지원

- 원래 Unix FS: 파일 이름 최대 14자
- FFS: 가변 길이 디렉토리 엔트리로 긴 이름 지원

### Symbolic Link

- Hard link의 제약을 극복
- 다른 파일 시스템이나 디렉토리를 가리킬 수 있음

### Parameterized Placement

FFS는 디스크의 물리적 특성(회전 속도, 헤드 이동 시간 등)을 파라미터로 받아 배치를 최적화했다.

### 성능 개선 결과

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

1. **Superblock 검사**: 파일 시스템 크기가 할당된 블록 수보다 큰지 확인
2. **Free block 검사**: inode, indirect block, 디렉토리를 스캔하여 할당된 블록 확인
3. **Inode 상태 검사**: 손상된 inode는 제거
4. **Inode link count 검사**: 실제 디렉토리 엔트리 개수와 일치하는지 확인
5. **Duplicate 검사**: 두 inode가 같은 블록을 가리키는지 확인
6. **Bad block 검사**: 범위를 벗어난 포인터 확인
7. **디렉토리 검사**: "."과 ".."가 올바른지 확인

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

**1. Journal Write (저널 쓰기)**
- TxB (Transaction Begin): 트랜잭션 ID 포함
- 메타데이터: IN', DB' (업데이트될 inode와 bitmap)
- 데이터: Db (실제 데이터)
- TxE (Transaction End): 트랜잭션 ID 포함

**2. Journal Commit (저널 커밋)**
- TxE 블록을 디스크에 쓰기
- TxE가 완료되면 트랜잭션이 **committed** 상태
- 크래시 후 TxE가 있으면 복구 가능, 없으면 무시

**3. Checkpoint (체크포인트)**
- Journal의 내용을 파일 시스템의 최종 위치에 쓰기
- IN' → Inode 영역
- DB' → Bitmap 영역
- Db → Data 영역

**4. Free (해제)**
- Journal 영역을 free로 표시
- Superblock에 journal 시작/끝 포인터 업데이트

### 복구 과정

크래시 후 부팅 시:
1. Journal을 스캔
2. Committed 트랜잭션(TxE 있음)을 찾아 replay
3. Checkpointing을 다시 수행
4. 파일 시스템 일관성 복원

```text
[복구 시나리오]

시나리오 1: Journal Write 중 크래시
┌─────────────────────────┐
│ Journal                 │
│ +-----+-----+-----+     │
│ | TxB | IN' | DB' | (X) │
│ +-----+-----+-----+     │
│   ↑                     │
│   |                     │
│   TxE 없음 → 트랜잭션 무시
└─────────────────────────┘
→ 파일 시스템은 변경 전 상태 유지

시나리오 2: Journal Commit 후, Checkpoint 전 크래시
┌─────────────────────────────────┐
│ Journal                         │
│ +-----+-----+-----+-----+-----+ │
│ | TxB | IN' | DB' | Db  | TxE | │
│ +-----+-----+-----+-----+-----+ │
│                            ↑    │
│                            |    │
│                     TxE 있음 → Replay
└─────────────────────────────────┘
→ Checkpointing 다시 수행
→ 파일 시스템 일관성 복원
```

### Metadata Journaling (Ordered Journaling)

Data journaling은 모든 데이터를 journal에 쓰므로 오버헤드가 크다 (데이터를 두 번 씀).

**Metadata journaling**은 메타데이터만 journal에 기록한다:

1. **Data write**: Db를 최종 위치에 먼저 쓰기
2. **Journal metadata write**: TxB, IN', DB'만 journal에 쓰기
3. **Journal commit**: TxE 쓰기
4. **Checkpoint metadata**: IN', DB'를 최종 위치에 쓰기
5. **Free**: Journal 해제

이 방식은 **ext3의 기본 모드**(ordered mode)이며, data journaling보다 훨씬 빠르다.

### Journaling의 장단점

**장점**:
- fsck보다 훨씬 빠름 (journal 영역만 스캔)
- 메타데이터 일관성 보장

**단점**:
- 쓰기 성능 오버헤드 (특히 data journaling)
- Journal 영역 필요
- 파일 데이터 손실은 여전히 가능 (metadata journaling의 경우)

## 8. 해결 방법 3: Copy-on-Write (COW)

**Copy-on-Write**는 ZFS, Btrfs 등에서 사용하는 방식으로, 기존 데이터를 절대 덮어쓰지 않는다.

### COW 동작 방식

```text
[COW 파일 시스템 업데이트 과정]

초기 상태:
         Root
          │
    ┌─────┴─────┐
    A           B
    │           │
  [Da]        [Db]

파일 A에 새 데이터 추가:
Step 1: 새 위치에 데이터 쓰기
         Root
          │
    ┌─────┴─────┐
    A           B         [Da']  (새 위치)
    │           │
  [Da]        [Db]

Step 2: 새 inode 생성
         Root
          │
    ┌─────┴─────┐
    A           B    A'  (새 inode)
    │           │     │
  [Da]        [Db]  [Da']

Step 3: 새 루트 블록 생성
         Root'  (새 루트)
          │
    ┌─────┴─────┐
    A'          B
    │           │
  [Da']       [Db]

Step 4: 슈퍼블록 업데이트 (원자적)
슈퍼블록의 루트 포인터를 Root → Root'로 변경
→ 이 한 번의 쓰기로 모든 변경사항이 원자적으로 반영됨!
```

### COW의 장점

- **원자적 업데이트**: 슈퍼블록 업데이트 하나로 모든 변경사항 반영
- **Snapshot 지원**: 이전 버전의 루트를 유지하면 스냅샷
- **Journal 불필요**: 모든 쓰기가 원자적

### COW의 단점

- **쓰기 증폭**: 데이터뿐 아니라 경로상의 모든 메타데이터 업데이트 필요
- **Fragmentation**: 파일이 디스크 곳곳에 흩어짐
- **성능**: 랜덤 쓰기 성능이 떨어질 수 있음

## 9. 해결 방법 4: Log-Structured File System (LFS)

**LFS**는 모든 쓰기를 순차적으로 로그처럼 기록한다.

### 기본 아이디어

- 디스크를 거대한 순차 로그로 취급
- 모든 업데이트(데이터, 메타데이터)를 로그 끝에 추가
- In-place 업데이트 없음 → 원자성 보장
- 쓰기 성능 최적화 (순차 쓰기만 수행)

### LFS의 장단점

**장점**:
- 순차 쓰기로 최대 성능
- 크래시 일관성 자연스럽게 보장

**단점**:
- 읽기 성능 (파일이 흩어져 있음)
- Garbage collection 필요 (오래된 버전 정리)

## 10. 정리: Crash Consistency 해결 방법 비교

| 방법          | 복구 시간 | 성능 오버헤드 | 일관성 보장 | 사용 예              |
|---------------|-----------|---------------|-------------|----------------------|
| fsck          | 매우 느림 | 없음          | 메타데이터  | 초기 Unix FS         |
| Journaling    | 빠름      | 중간          | 메타데이터  | ext3/4, NTFS         |
| COW           | 즉시      | 쓰기 증폭     | 완전        | ZFS, Btrfs           |
| LFS           | 즉시      | GC 필요       | 완전        | F2FS (Flash 최적화)  |

현대 파일 시스템 대부분은 **journaling**을 사용한다. COW와 LFS는 특정 상황(스냅샷, SSD)에 유리하다.

## 시리즈 마무리: CS330 Operating Systems

이것으로 CS330 운영체제 시리즈의 모든 글을 마무리한다. 우리는 다음 주제들을 다루었다:

**Part 1: Virtualization (가상화)**
- CPU 가상화: 프로세스, 스케줄링
- 메모리 가상화: 주소 공간, 페이징, 세그멘테이션, TLB

**Part 2: Concurrency (동시성)**
- 스레드와 락
- 조건 변수와 세마포어
- 동시성 버그와 해결 방법

**Part 3: Persistence (영속성)**
- I/O 장치와 디스크
- 파일 시스템: VSFS, FFS
- 크래시 일관성: Journaling, COW

### 핵심 메시지

운영체제는 **추상화와 보호의 예술**이다:
- **추상화**: 복잡한 하드웨어를 단순한 인터페이스로 제공 (프로세스, 가상 메모리, 파일)
- **보호**: 프로세스 간 격리, 권한 관리, 일관성 보장
- **성능**: 제한된 자원을 효율적으로 관리 (스케줄링, 캐싱, 버퍼링)

운영체제는 하드웨어와 소프트웨어 사이에서 끊임없이 트레이드오프를 조율한다. FFS의 지역성 최적화, Journaling의 안정성과 성능 균형이 그 좋은 예다.

이 시리즈를 통해 우리는 컴퓨터 시스템의 가장 근본적인 소프트웨어인 운영체제가 어떻게 동작하는지, 그리고 왜 그렇게 설계되었는지를 이해할 수 있었다. 이러한 지식은 시스템 프로그래밍, 성능 최적화, 분산 시스템 등 더 깊은 학습의 토대가 될 것이다.

---

*이 시리즈의 모든 글은 KAIST CS330 강의 내용을 바탕으로 작성되었습니다.*
