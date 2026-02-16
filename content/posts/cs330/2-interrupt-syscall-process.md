---
title: "[CS330] 2. 인터럽트, 시스템 콜과 프로세스"
date: 2023-12-02
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "Interrupt", "System Call", "Process", "fork"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "하드웨어 인터럽트와 소프트웨어 트랩의 차이, 시스템 콜의 동작 원리, 그리고 프로세스의 개념과 생명주기를 다룹니다."
aliases: ["/posts/cs330-2-interrupt-syscall-process/"]
ShowToc: true
TocOpen: true
---

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

*이전 글: [[CS330] 1. 운영체제 소개와 커널 보호 메커니즘](/posts/cs330/1-os-intro-kernel-protection/)*

---

## 1. 인터럽트(Interrupt)

운영체제가 하드웨어와 상호작용하고 CPU 제어권을 다시 회복하기 위해서는 **인터럽트(Interrupt)** 메커니즘이 필요합니다. 인터럽트는 크게 하드웨어 인터럽트와 소프트웨어 인터럽트(Trap)로 나뉩니다.

### 1.1 인터럽트의 종류

**하드웨어 인터럽트(Hardware Interrupt)**
- 외부 하드웨어 장치에 의해 발생 (타이머, 디스크, 네트워크 카드 등)
- 비동기적(asynchronous)으로 발생
- CPU가 실행 중인 명령과 관계없이 발생 가능
- 예: 타이머 인터럽트, I/O 완료 인터럽트

**소프트웨어 인터럽트(Trap/Exception)**
- CPU가 실행 중인 명령에 의해 발생
- 동기적(synchronous)으로 발생
- 종류:
  - **시스템 콜(System Call)**: 프로세스가 커널 서비스를 요청
  - **예외(Exception)**: 프로그램 실행 중 예상치 못한 동작 발생
    - Page Fault, Segmentation Fault, Divide by Zero 등

### 1.2 인터럽트 처리 과정

인터럽트가 발생하면 다음과 같은 절차를 거칩니다:

```text
1. 인터럽트 발생
   ↓
2. 현재 실행 중인 프로세스의 상태 저장
   - PC (Program Counter), 레지스터 등을 인터럽트 스택에 저장
   ↓
3. 인터럽트 벡터 테이블(IVT) 또는 인터럽트 디스크립터 테이블(IDT) 참조
   - 인터럽트 번호에 해당하는 핸들러 주소 찾기
   ↓
4. 인터럽트 핸들러(Interrupt Handler/ISR) 실행
   - 커널 모드로 전환하여 실행
   - 인터럽트 처리 중에는 다른 인터럽트 비활성화
   ↓
5. 저장된 상태 복원 및 원래 실행으로 복귀
```

### 1.3 인터럽트 스택(Interrupt Stack)

인터럽트가 발생하면 현재 프로세스의 상태를 저장해야 합니다. 이를 위해 **인터럽트 스택**을 사용합니다:

- 프로세스별로 커널 메모리에 위치
- 유저 스택과 별도로 존재
- 저장 내용: PC, 레지스터, 플래그 등

```text
User Stack (유저 메모리)        Kernel Stack (커널 메모리)
┌─────────────┐                ┌─────────────────┐
│   Proc2     │                │                 │
│   Proc1     │                │  Interrupt      │
│   Main      │                │  Stack          │
└─────────────┘                │  ┌───────────┐  │
                               │  │ User CPU  │  │
                               │  │ State     │  │
                               │  └───────────┘  │
                               └─────────────────┘
```

### 1.4 인터럽트 마스킹(Interrupt Masking)

인터럽트 핸들러는 **인터럽트를 비활성화한 상태에서 실행**됩니다. 왜 그럴까요?

- 인터럽트 핸들러 실행 중 또 다른 인터럽트가 발생하면 복잡도가 증가
- 일관성(consistency) 문제 발생 가능

x86 아키텍처에서는:
- `CLI` (Clear Interrupt): 인터럽트 비활성화
- `STI` (Set Interrupt): 인터럽트 활성화

이러한 명령어들은 **특권 명령(privileged instruction)**으로 커널 모드에서만 실행 가능합니다.

---

## 2. 시스템 콜(System Call)

사용자 프로그램이 커널의 서비스를 이용하려면 **시스템 콜**을 통해 요청해야 합니다. 시스템 콜은 유저 모드에서 커널 모드로 전환하는 안전한 메커니즘입니다.

### 2.1 시스템 콜이 필요한 이유

- 유저 프로그램은 직접 하드웨어에 접근하거나 특권 명령을 실행할 수 없음
- 파일 I/O, 네트워크 통신, 프로세스 생성 등 모두 커널의 도움 필요
- 보호(protection)와 추상화(abstraction)를 제공

### 2.2 시스템 콜의 동작 원리

시스템 콜은 특수한 **trap 명령어**를 통해 실행됩니다:

```text
User Application
     │
     │ 1. 시스템 콜 함수 호출 (예: read())
     ↓
C Library (libc)
     │
     │ 2. 시스템 콜 번호를 레지스터에 설정
     │ 3. trap 명령어 실행 (IA-32: INT, RISC-V: ECALL)
     ↓
───────────────────── [User Mode → Kernel Mode] ─────────────────────
     │
     ↓ 4. 모드 비트 변경 (1 → 0)
Kernel
     │
     │ 5. 시스템 콜 테이블에서 핸들러 찾기
     ↓
System Call Handler
     │
     │ 6. 요청된 작업 수행
     │ 7. 반환 값 설정
     ↓
───────────────────── [Kernel Mode → User Mode] ─────────────────────
     │
     │ 8. 유저 프로그램으로 복귀
     ↓
User Application
```

### 2.3 시스템 콜 처리 상세

시스템 콜 처리 시 다음과 같은 정보가 사용됩니다:

1. **시스템 콜 번호**: 어떤 시스템 콜인지 식별
2. **매개변수**: 시스템 콜에 필요한 인자들 (레지스터나 스택을 통해 전달)
3. **시스템 콜 테이블**: 시스템 콜 번호를 핸들러 주소로 매핑

**예시: `read()` 시스템 콜**

```c
count = read(fd, buffer, nbytes);
```

```text
Address
0xFFFFFFFF ┌──────────────────────────────┐
           │  Return to caller            │
           │  Trap to the kernel          │ ← Library
           │  Put code for read in reg    │   procedure
           ├──────────────────────────────┤   read
           │  Increment SP                │
User Space │  Call read                   │ ← User program
           │  Push fd                     │   calling read
           │  Push &buffer                │
           │  Push nbytes                 │
           ├──────────────────────────────┤
0          │                              │
═══════════╪══════════════════════════════╪═══════════════════════
Kernel     │  Dispatch                    │
Space      │  Sys call handler            │
(Operating │                              │
System)    └──────────────────────────────┘
```

### 2.4 시스템 콜 예제

주요 시스템 콜들:
- **파일 I/O**: `open()`, `read()`, `write()`, `close()`
- **프로세스 관리**: `fork()`, `exec()`, `wait()`, `exit()`
- **메모리 관리**: `mmap()`, `munmap()`, `brk()`
- **통신**: `socket()`, `send()`, `recv()`

---

## 3. I/O와 DMA

### 3.1 I/O 처리 방식

CPU가 I/O를 효율적으로 처리하려면 어떻게 해야 할까요?

**문제점**:
- I/O 작업은 CPU에 비해 매우 느림
- CPU가 I/O 완료를 대기하면 시간 낭비

**해결책: DMA (Direct Memory Access)**

```text
┌───────────┐      (1) Initiate Block Read
│ Processor │────────────────────────────────┐
│   [Reg]   │                                │
└───────────┘      (3) Read Done             │
      ↓                                       ↓
   Cache                           ┌──────────────────┐
      ↓                            │                  │
   Memory ←────(2) DMA Transfer────│  I/O Controller  │
                                   │     [buffer]     │
                                   └──────────────────┘
                                            ↓
                                          Disk
```

DMA를 사용하면:
1. CPU가 I/O 컨트롤러에게 작업 지시
2. I/O 컨트롤러가 메모리와 직접 데이터 전송 (CPU 개입 없이)
3. 작업 완료 시 인터럽트로 CPU에 알림
4. CPU는 그 사이 다른 작업 수행 가능

### 3.2 인터럽트를 통한 I/O 완료 통지

```text
                        Disk drive
                          ┌──┐
                          └──┘
CPU ←─③ Interrupt ───┐     │
 ↑                    │  ② Queue command & ack
 │                    │     │
 │ ① send read        │  ┌──▼─────────┐
 │   command          │  │   Disk     │
 └────────────────────┴──│ controller │
                         └────────────┘
                      ④ perform disk read
```

---

## 4. 프로세스(Process)

### 4.1 프로세스란?

**프로세스**는 실행 중인 프로그램의 인스턴스입니다. 프로세스는 다음과 같이 정의할 수 있습니다:

```text
Process = Address Space + Thread(s) + Resources
```

- **Address Space**: 프로세스가 접근할 수 있는 메모리 영역
- **Thread**: CPU 코어의 추상화 (실행 단위)
- **Resources**: 열린 파일, 소켓, 세마포어 등

### 4.2 프로세스 주소 공간

프로세스의 가상 주소 공간은 다음과 같이 구성됩니다:

```text
High Address
0xFFFFFFFF ┌─────────────────────┐
           │  Kernel Space       │ ← 모든 프로세스가 공유
           │  (공통)             │
           ├─────────────────────┤
           │  Stack              │ ← 함수 호출, 지역 변수
           │        ↓            │
           │                     │
           │                     │
           │        ↑            │
           │  Heap               │ ← 동적 메모리 할당
           ├─────────────────────┤
           │  BSS                │ ← 초기화되지 않은 전역 변수
           ├─────────────────────┤
           │  Data               │ ← 초기화된 전역 변수
           ├─────────────────────┤
           │  Text (Code)        │ ← 프로그램 코드
0x00400000 └─────────────────────┘
Low Address
```

**각 영역의 역할**:
- **Text**: 실행 가능한 기계어 코드
- **Data**: 초기화된 전역/정적 변수
- **BSS**: 초기화되지 않은 전역/정적 변수 (Block Started by Symbol)
- **Heap**: `malloc()` 등으로 동적 할당된 메모리
- **Stack**: 함수 호출 스택, 지역 변수

### 4.3 프로그램에서 프로세스로

디스크에 저장된 프로그램이 프로세스가 되는 과정:

```text
    Disk                        Memory
┌──────────┐                ┌──────────┐
│  ┌────┐  │                │          │
│  │code│  │───────┐        │  Code    │ ← PC
│  └────┘  │       │        ├──────────┤
│  ┌────┐  │       └────→   │  Data    │
│  │data│  │                ├──────────┤
│  └────┘  │                │  Heap    │
│ program  │                │    ↓     │
└──────────┘                │          │
                            │    ↑     │
                            │  Stack   │ ← SP
                            └──────────┘
```

**로딩(Loading)**:
1. 디스크에서 프로그램의 code와 data를 읽어옴
2. 메모리에 프로세스 주소 공간 생성
3. Stack과 Heap 영역 초기화
4. PC를 첫 번째 명령어로 설정

### 4.4 PCB (Process Control Block)

운영체제는 각 프로세스의 정보를 **PCB**에 저장합니다:

```c
struct task_struct {  // Linux에서 약 6016 bytes (4.15.0-91)
    // 프로세스 식별
    pid_t pid;
    pid_t ppid;

    // 상태 정보
    int state;              // Running, Ready, Blocked

    // CPU 레지스터
    struct pt_regs *regs;   // PC, SP, 범용 레지스터 등

    // 스케줄링 정보
    int priority;
    unsigned long time_slice;

    // 메모리 관리
    struct mm_struct *mm;   // 페이지 테이블 포인터

    // 파일 시스템
    struct files_struct *files;  // 열린 파일 목록

    // I/O 상태
    // 자격 증명(credentials)
    // 등등...
};
```

### 4.5 프로세스 상태 전이

프로세스는 생명주기 동안 다양한 상태를 거칩니다:

```text
                  Created
                     │
                     ↓
        ┌─────── Ready ────────┐
        │          ↑           │
        │          │           │ Scheduled
   I/O or         │ Time       │
   event      exhausted         ↓
 completion       │         Running
        │         │            │
        │         │            │ I/O or
        ↓         │            │ event wait
      Blocked ────┘            │
                               ↓ exit
                            Terminated
```

- **Ready**: 실행 준비 완료, CPU 할당 대기
- **Running**: CPU에서 실행 중
- **Blocked**: I/O 작업 등을 기다리는 중

### 4.6 컨텍스트 스위치(Context Switch)

CPU가 한 프로세스에서 다른 프로세스로 전환하는 과정:

```text
Process A                 OS Kernel               Process B
  │                           │                      │
  │ ──(1) Running──→          │                      │
  │                           │                      │
  │ ◄─(2) Timer Interrupt─    │                      │
  │                           │                      │
  │                        (3) Save                  │
  │                        context of A              │
  │                        to PCB_A                  │
  │                           │                      │
  │                        (4) Load                  │
  │                        context of B              │
  │                        from PCB_B                │
  │                           │                      │
  │                           │   ─(5) Resume──→     │
  │                           │                      │
  │                           │         ◄──(6) Running──
```

**컨텍스트 스위치 오버헤드**:
- CPU 레지스터 저장/복원
- TLB (Translation Lookaside Buffer) 플러시
- 캐시 효율성 감소
- 일반적으로 수 마이크로초 소요

---

## 5. 프로세스 API

UNIX/Linux는 프로세스 생성 및 관리를 위한 강력한 API를 제공합니다.

### 5.1 fork(): 프로세스 복제

`fork()`는 현재 프로세스의 복사본을 생성합니다:

```c
int child_pid = fork();

if (child_pid == 0) {
    // 자식 프로세스
    printf("I am process #%d\n", getpid());
    return 0;
} else {
    // 부모 프로세스
    printf("I am parent of process #%d\n", child_pid);
    return 0;
}
```

**fork()의 동작**:
1. 새로운 PCB 생성
2. 새로운 주소 공간 생성
3. 부모의 주소 공간을 자식에게 복사
4. 커널 리소스(열린 파일 등)를 자식이 공유하도록 설정
5. PCB를 Ready 큐에 추가
6. **부모에게는 자식 PID 반환, 자식에게는 0 반환**

```text
    fork
   ┌────┐
   │    │              After fork
   │ P  │         ┌──────┐    ┌──────┐
   │    │         │ P    │    │ C    │
   └────┘         │ DATA │    │ DATA │
                  │ stack│    │ stack│
                  │ Heap │    │ Heap │
                  └──────┘    └──────┘
                  PID = 12870  PID = 14891
```

### 5.2 exec(): 새 프로그램 실행

`exec()` 계열 함수는 현재 프로세스의 주소 공간을 새로운 프로그램으로 덮어씁니다:

```c
if (child_pid == 0) {
    // 자식 프로세스
    if (execv(argv[0], argv) < 0) {
        printf("%s: command not found\n", argv[0]);
        exit(0);
    }
}
```

**exec() 실행 후**:
- Text, Data, BSS, Heap을 새 프로그램으로 교체
- Stack 초기화
- PID는 변경되지 않음
- 성공 시 리턴하지 않음 (새 프로그램이 실행되므로)

### 5.3 wait(): 자식 프로세스 대기

부모 프로세스는 `wait()`로 자식의 종료를 기다립니다:

```c
waitpid(pid, &status, 0);
```

### 5.4 Shell 구현 예제

간단한 셸은 fork-exec-wait 패턴을 사용합니다:

```c
int main(void) {
    char cmdline[MAXLINE];
    char *argv[MAXARGS];
    pid_t pid;
    int status;

    while (getcmd(cmdline, MAXLINE) >= 0) {
        parsecmd(cmdline, argv);

        if (!builtin_command(argv)) {
            if ((pid = fork()) == 0) {
                // 자식 프로세스: 명령어 실행
                if (execv(argv[0], argv) < 0) {
                    printf("%s: command not found\n", argv[0]);
                    exit(0);
                }
            }
            // 부모 프로세스: 자식 대기
            waitpid(pid, &status, 0);
        }
    }
}
```

---

## 6. 정리

이번 글에서는 운영체제의 핵심 메커니즘들을 다뤘습니다:

1. **인터럽트**: 하드웨어와 커널이 소통하는 방법
   - 하드웨어 인터럽트 vs 소프트웨어 인터럽트(Trap)
   - 인터럽트 핸들러와 인터럽트 스택
   - 인터럽트 마스킹

2. **시스템 콜**: 유저 프로그램이 커널 서비스를 이용하는 방법
   - Trap 명령어를 통한 모드 전환
   - 시스템 콜 테이블과 핸들러
   - 안전한 매개변수 전달

3. **I/O와 DMA**: 효율적인 I/O 처리
   - DMA를 통한 CPU 부담 감소
   - 인터럽트를 통한 I/O 완료 통지

4. **프로세스**: 실행 중인 프로그램의 추상화
   - 프로세스 주소 공간 구조
   - PCB를 통한 프로세스 관리
   - 프로세스 상태 전이와 컨텍스트 스위치

5. **프로세스 API**: 프로세스 생성과 관리
   - `fork()`: 프로세스 복제
   - `exec()`: 새 프로그램 로드
   - `wait()`: 자식 프로세스 대기

다음 글에서는 프로세스보다 가벼운 실행 단위인 **스레드(Thread)**와 CPU **스케줄링** 알고리즘에 대해 알아보겠습니다.

---

*다음 글: [[CS330] 3. 스레드와 CPU 스케줄링](/posts/cs330/3-thread-scheduling/)*
