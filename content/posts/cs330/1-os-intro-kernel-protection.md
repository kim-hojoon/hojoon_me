---
title: "[CS330] 1. 운영체제 소개와 커널 보호 메커니즘"
date: 2023-12-01
draft: false
tags: ["CS330", "Operating Systems", "KAIST", "Kernel", "Dual Mode", "Protection"]
categories: ["Computer Science"]
series: ["CS330 운영체제"]
summary: "운영체제가 무엇인지, 왜 커널 보호가 필요한지, 그리고 이중 모드(Dual Mode)를 통한 보호 메커니즘까지 정리합니다."
aliases: ["/posts/cs330-1-os-intro-kernel-protection/"]
ShowToc: true
TocOpen: true
---

> KAIST CS330 운영체제 및 실험 (Fall 2023, Prof. Youngjin Kwon)
> 교재: *Operating Systems: Three Easy Pieces* (OSTEP), *Operating Systems: Principles and Practice* (OSPP)

---

## 운영체제란 무엇인가?

운영체제(Operating System, OS)는 하드웨어와 응용 프로그램 사이에 위치한 시스템 소프트웨어 계층이다. 운영체제의 정의를 다시 살펴보면:

**운영체제는 하드웨어에 직접 접근하는 특권을 가지고, 하드웨어의 복잡성을 숨기며, 여러 응용 프로그램 간에 하드웨어 자원을 관리하고 공유하며, 응용 프로그램들을 서로 격리(isolation)하고 보호하는 시스템 소프트웨어이다.**

```text
┌─────────────────────────────────────────┐
│        Applications (Programs)          │
├─────────────────────────────────────────┤
│          Middleware & SDK               │
├─────────────────────────────────────────┤
│  Software Development Environment       │
│  (compilers, loaders, editors, etc.)    │
├─────────────────────────────────────────┤
│      Operating System (Kernel)          │
├─────────────────────────────────────────┤
│  Computer System Architecture (HW)      │
└─────────────────────────────────────────┘
```

운영체제는 응용 프로그램에게 API를 제공하여 하드웨어를 쉽게 사용할 수 있도록 하며, 하드웨어의 세부 사항(details)을 숨긴다.

### 운영체제의 역할

운영체제는 세 가지 주요 역할을 수행한다:

1. **Resource Manager (자원 관리자)**: CPU, 메모리, 디스크 등의 하드웨어 자원을 여러 프로그램에 효율적으로 분배한다.
2. **Illusionist (환상 제공자)**: 각 응용 프로그램이 마치 독립된 컴퓨터를 사용하는 것처럼 느끼게 한다.
3. **Glue (접착제)**: 하드웨어와 소프트웨어를 연결하는 인터페이스 역할을 한다.

### 왜 운영체제가 필요한가?

운영체제 없이 프로그램이 직접 하드웨어를 제어했던 옛날 방식을 상상해보자. 모든 프로그램이 같은 메모리 공간에서 실행되고, 커널도 같은 주소 공간에 위치한다. 이 경우 응용 프로그램이 커널의 명령어와 데이터를 자유롭게 읽고 수정할 수 있다.

이런 환경에서는 다음과 같은 문제가 발생한다:
- 악의적이거나 버그가 있는 프로그램이 커널을 손상시킬 수 있다
- 프로그램이 시스템을 강제 종료시킬 수 있다
- 다른 프로그램의 메모리를 침범할 수 있다
- 무한 루프에 빠져 CPU를 독점할 수 있다

**따라서 운영체제는 커널을 보호하고, 프로그램 간의 간섭을 방지하며, 시스템 자원을 공정하게 분배하기 위해 필요하다.**

---

## OS의 세 가지 핵심 축

운영체제는 세 가지 핵심 개념을 중심으로 설계된다:

```text
                 OS Three Pillars

    ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
    │                │   │                │   │                │
    │ Virtualization │   │  Concurrency   │   │  Persistence   │
    │                │   │                │   │                │
    └────────────────┘   └────────────────┘   └────────────────┘
         Process            Threads            Storage
      CPU Scheduling    Synchronization     File Systems
      Virtual Memory
```

### 1. Virtualization (가상화)

**각 응용 프로그램이 자원을 독점하고 있다고 믿게 만드는 것**

- **Process (프로세스)**: CPU를 가상화하여 각 프로그램이 CPU를 독점하는 것처럼 보이게 한다
- **CPU Scheduling (CPU 스케줄링)**: 여러 프로세스가 CPU를 공유하도록 한다
- **Virtual Memory (가상 메모리)**: 각 프로그램이 독립적인 메모리 공간을 가진 것처럼 보이게 한다

### 2. Concurrency (동시성)

**여러 작업을 동시에 올바르고 효율적으로 처리하는 방법**

- **Threads (스레드)**: 프로세스 내의 여러 실행 흐름
- **Synchronization (동기화)**: 동시에 실행되는 작업들 간의 조정

### 3. Persistence (영속성)

**전원이 꺼져도 정보가 살아남도록 하는 방법**

- **Storage (저장장치)**: HDD, SSD 등
- **File Systems (파일 시스템)**: 데이터를 구조화하고 관리

---

## 추상화(Abstraction): 컴퓨터 과학의 핵심

John Ousterhout의 말에 따르면, "컴퓨터 과학에서 가장 중요한 개념 하나를 꼽으라면 그것은 **추상화(Abstraction)**이다."

**추상화는 중요하지 않은 세부 사항을 무시함으로써 무언가를 더 쉽게 이해할 수 있게 만드는 과정이다.** 예를 들어, 클래스 사용자는 내부 구조를 몰라도 public 메서드만으로 기능을 사용할 수 있다. 추상화는 기계어에서 어셈블리, 고급 언어, 객체지향으로 진화하면서 프로그래머가 신경 써야 할 세부 사항을 줄여왔다.

### 운영체제가 추상화하는 대상

운영체제는 다음 자원들을 추상화한다:

- **CPU processor** → Process 추상화
- **DRAM** → Virtual Memory 추상화
- **Storage** → File System 추상화
- **Network card** → Socket 추상화

이러한 추상화를 통해:
- **Build Abstractions (추상화 구축)**: 하드웨어 복잡성을 숨긴다
- **Providing protections (보호 제공)**: 응용 프로그램을 격리한다
- **Sharing resources (자원 공유)**: 여러 프로그램이 하드웨어를 함께 사용한다

---

## 커널 보호의 필요성

### 문제 상황: 고대 시대의 부팅

옛날 컴퓨터 시스템에서는 부팅할 때 다음과 같은 일이 벌어졌다:

```text
       Disk                Physical Memory
┌──────────────┐          ┌──────────────┐
│              │   (1)    │     BIOS     │
│ Bootloader   │─────────→│──────────────│
│              │          │  Bootloader  │
├──────────────┤   (2)    │  instructions│
│  OS kernel   │─────────→│   and data   │
│              │          │──────────────│
├──────────────┤   (3)    │  OS kernel   │
│  Login app   │─────────→│  instructions│
│              │          │   and data   │
└──────────────┘          │──────────────│
                          │  Login app   │
                          │  instructions│
                          │   and data   │
                          └──────────────┘
```

이 시스템에서는:
- **OS 커널과 응용 프로그램이 같은 주소 공간에서 실행된다**
- 응용 프로그램이 커널의 명령어와 데이터를 자유롭게 읽고 수정할 수 있다

### Direct Execution의 문제점

운영체제가 프로그램을 실행할 때 가장 간단한 방법은 **Direct Execution**이다:

```text
OS                               Program
────────────────────────────────────────
1. Create entry for process list
2. Allocate memory for program
3. Load program into memory
4. Set up stack with argc/argv
5. Clear registers
6. Execute call main()
                                 7. Run main()
                                 8. Execute return from main()
9. Free memory of process
10. Remove from process list
```

이 방식에서는 **OS가 프로그램에 대해 아무 제어도 하지 않는다**. OS는 단지 라이브러리처럼 동작할 뿐이다.

### 커널이 보호받아야 하는 이유

```text
For protection:

┌─────┐  ┌─────┐  ┌─────┐
│ APP │  │ APP │  │ APP │    Untrusted
└─────┘  └─────┘  └─────┘
─────────────────────────────
┌──────────────────────────┐
│   Operating System       │    Trusted
│        Kernel            │
└──────────────────────────┘
```

만약 응용 프로그램을 신뢰할 수 없다면:
- `int *i; i = 0; *i = 1;` → 널 포인터 역참조로 시스템 크래시
- `i = -1; while (i < 0) do something;` → 무한 루프로 CPU 독점
- `i = kernel_address; *i = 1;` → 커널 메모리 침범

**커널은 버그가 있거나 악의적인 응용 프로그램으로부터 반드시 보호되어야 한다.**

---

## 프로세스: 보호의 단위

### 보호를 위한 OS 설계

OS 설계자는 다음 두 가지를 해야 한다:

1. **보호의 단위(Unit of Protection) 정의**
   - 각 응용 프로그램이 마치 독립된 기계에서 실행되는 환상을 제공한다

2. **커널을 위한 보호 메커니즘 고안**
   - 응용 프로그램의 악의적 행동을 제한한다

### Process: 보호의 추상화

```text
         ┌────────────────────┐
         │ A(int tmp) {       │  ← Abstraction of CPU
         │   if (tmp<2)       │
         │     B();           │  ┌──────────────────┐
         │   printf(tmp);     │  │ Abstraction of   │
         │ }                  │  │     memory       │
         │ B() {              │  └──────────────────┘
         │   C();             │
         │ }                  │  ┌──────────────────┐
         │ C() {              │  │ Abstraction of   │
         │   A(2);            │  │   IO devices     │
         │ }                  │  └──────────────────┘
         │ A(1);              │
         │ ...                │
         └────────────────────┘
```

**프로세스(Process)**는 다음과 같이 정의된다:
- 실행 중인 프로그램의 인스턴스(instance)
- 제한된 접근 권한(limited access rights)을 가진 실행 단위
- CPU, 메모리, I/O 디바이스에 대한 추상화를 담는 컨테이너

**OS 설계자는 이 추상화를 "프로세스"라고 명명했다.**

### 프로세스가 있는 컴퓨터 시스템

```text
Physical Memory
┌──────────────┐
│   Process    │ ← User Space
│ Instructions │
│     Data     │
│     Heap     │
│     Stack    │
├──────────────┤
│  OS Kernel   │ ← Kernel Space
│ Instructions │
│     Data     │
│     Heap     │
│     Stack    │
└──────────────┘
```

컴파일된 실행 파일이 OS에 의해 메모리에 로드되면, 프로세스와 커널이 메모리에 공존한다. 이제 **커널을 프로세스로부터 보호**해야 한다.

---

## 보호 문제의 세분화

보호 문제를 더 구체적으로 나누면:

### 1. Privileged Instructions (특권 명령어)

**응용 프로그램이 중요한 명령어를 실행하지 못하게 막기**

- 시스템 종료 명령어
- 하드웨어 제어 명령어
- 커널 메모리 접근 명령어

### 2. Memory Protection (메모리 보호)

**응용 프로그램이 다른 응용 프로그램이나 커널의 메모리를 읽거나 쓰지 못하게 막기**

### 3. Timer Interrupt (타이머 인터럽트)

**OS가 응용 프로그램으로부터 제어권을 다시 가져오기**

---

## 이중 모드(Dual Mode): 하드웨어 기반 보호

소프트웨어만으로 보호 메커니즘을 구현하면 너무 느리다. 따라서 **하드웨어 기반 보호 메커니즘**이 필요하다.

### Dual Mode Operation

현대 CPU는 **모드 비트(mode bit)**를 하드웨어로 제공한다. 이론적으로 x86 아키텍처는 Ring 0~3의 4단계 특권 레벨을 지원하지만, 대부분의 OS는 **이중 모드(Dual Mode)**만 사용한다:

- **Kernel Mode (Ring 0)**: 모든 명령어 실행 가능
- **User Mode (Ring 3)**: 제한된 명령어만 실행 가능

### Privileged Instructions (특권 명령어)

일부 명령어는 **Kernel Mode에서만 실행 가능**하다:
- 시스템 종료 명령어
- I/O 명령어
- 메모리 관리 명령어
- 인터럽트 제어 명령어

User Mode에서 이런 명령어를 실행하려고 하면 **예외(exception)**가 발생하고, 제어권이 커널로 넘어간다.

### 모드 전환(Mode Switch)

```text
         User Mode              Kernel Mode
              │                      │
              │                      │
              │   ──trap/syscall──→  │
              │                      │
              │  (execute kernel     │
              │   code)              │
              │                      │
              │   ←──return──────    │
              │                      │
              ↓                      ↓
```

**User → Kernel 전환**:
- System Call (시스템 콜)
- Exception (예외)
- Interrupt (인터럽트)

**Kernel → User 전환**:
- Return from interrupt
- Return from exception
- Process scheduling

---

## 부팅 과정(Boot Process)

컴퓨터가 켜지면서 Kernel Mode로 시작하는 과정:

1. **Power On**: CPU는 미리 정해진 주소의 펌웨어(BIOS/UEFI)를 Kernel Mode에서 실행
2. **BIOS/UEFI**: 하드웨어 초기화 및 부트 로더 로딩 (Kernel Mode)
3. **Boot Loader**: 디스크에서 OS 커널을 메모리로 로드 (Kernel Mode)
4. **OS Init**: 커널 초기화 및 시스템 서비스 시작 (Kernel Mode)
5. **User Space**: User Mode로 전환하여 응용 프로그램 실행

부팅이 끝나면 CPU는 User Mode에 있고, OS는 Timer Interrupt를 통해 주기적으로 제어권을 되찾는다.

---

## Limited Direct Execution

### 기본 아이디어

**프로그램을 CPU에서 직접 실행하되, 보호 메커니즘으로 제한을 둔다.**

이것이 바로 **Limited Direct Execution**이다:
- **Direct**: 프로그램이 CPU에서 직접 실행된다 (오버헤드 최소화)
- **Limited**: 하드웨어 보호 메커니즘으로 제한된다

### Limited Direct Execution의 문제점과 해결책

| 문제 | 해결책 |
|------|--------|
| 프로그램이 특권 명령어를 실행하면? | **Dual Mode + Privileged Instructions** |
| 프로그램이 다른 메모리를 침범하면? | **Memory Protection (가상 메모리)** |
| 프로그램이 CPU를 독점하면? | **Timer Interrupt** |

다음 글에서는 Timer Interrupt와 System Call에 대해 자세히 다룰 것이다.

---

## 정리

이번 글에서는 다음 내용을 살펴보았다:

1. **운영체제의 정의와 역할**: 자원 관리자, 환상 제공자, 접착제
2. **OS의 세 가지 축**: Virtualization, Concurrency, Persistence
3. **추상화의 중요성**: 컴퓨터 과학의 핵심 개념
4. **커널 보호의 필요성**: 버그/악의적 프로그램으로부터 보호
5. **프로세스**: 보호의 단위이자 CPU/메모리/I/O의 추상화
6. **이중 모드**: User Mode vs Kernel Mode
7. **특권 명령어**: Kernel Mode에서만 실행 가능
8. **부팅 과정**: Kernel Mode로 시작해서 User Mode로 전환
9. **Limited Direct Execution**: 직접 실행 + 하드웨어 보호

운영체제는 **추상화를 통해 하드웨어를 숨기고, 프로세스를 통해 응용 프로그램을 격리하며, 이중 모드를 통해 커널을 보호한다.** 이 모든 것이 안전하고 효율적인 멀티태스킹 환경을 만들어낸다.

---

*다음 글: [[CS330] 2. 인터럽트, 시스템 콜과 프로세스](/posts/cs330/2-interrupt-syscall-process/)*
