---
title: "[CS230] 4. 링킹과 예외 제어 흐름 - Linking & ECF"
date: 2022-12-15
draft: false
tags: ["CS230", "Systems Programming", "KAIST", "Linking", "ECF", "Signals", "Process"]
categories: ["Computer Science"]
summary: "링커의 역할(심볼 해석, 재배치), 예외 제어 흐름(Exceptions, Process, Signals), 시스템 콜과 프로세스 관리를 다룹니다."
ShowToc: true
TocOpen: true
---

> KAIST CS230 시스템프로그래밍 (2022 Fall) 수업 내용을 정리한 시리즈입니다.
> 교재: *Computer Systems: A Programmer's Perspective* (CS:APP 3rd Edition)

---

## 1. 링킹 (Linking)

### 컴파일 과정

```
main.c ──→ [cpp, cc1, as] ──→ main.o ─┐
                                        ├──→ [Linker (ld)] ──→ prog (실행파일)
sum.c  ──→ [cpp, cc1, as] ──→ sum.o  ─┘
```

- **Preprocessor** (cpp): `#include`, `#define` 처리
- **Compiler** (cc1): C → Assembly
- **Assembler** (as): Assembly → Relocatable Object File (`.o`)
- **Linker** (ld): 여러 `.o` 파일을 합쳐 Executable Object File 생성

### 링커가 하는 일

**Step 1: Symbol Resolution (심볼 해석)**
- 프로그램은 심볼(함수명, 전역 변수명)을 정의하고 참조한다
- 각 `.o` 파일에는 심볼 테이블(symbol table)이 있다
- 링커는 모든 심볼 참조를 정확히 하나의 심볼 정의에 연결한다

**Step 2: Relocation (재배치)**
- 각 `.o` 파일의 코드/데이터 섹션을 하나로 합침
- 심볼들의 상대 위치를 최종 절대 주소로 변환
- 모든 참조를 새 주소로 갱신

### Object File의 종류

| 종류 | 설명 |
|------|------|
| Relocatable (`.o`) | 다른 파일과 합쳐질 수 있는 코드/데이터 |
| Executable (`a.out`) | 메모리에 직접 로드하여 실행 가능 |
| Shared (`.so`) | 동적 링킹용 — 런타임에 로드 (Windows: `.dll`) |

### ELF (Executable and Linkable Format)

Linux의 표준 object file 형식:

| 섹션 | 내용 |
|------|------|
| `.text` | 컴파일된 기계어 코드 |
| `.rodata` | 읽기 전용 데이터 (상수, 문자열 리터럴) |
| `.data` | 초기화된 전역/정적 변수 |
| `.bss` | 초기화되지 않은 전역/정적 변수 (공간만 예약) |
| `.symtab` | 심볼 테이블 |
| `.rel.text` | `.text`의 재배치 정보 |
| `.rel.data` | `.data`의 재배치 정보 |

### 심볼의 종류

- **Global symbols**: 다른 모듈에서 참조 가능 (non-static 함수, 전역 변수)
- **External symbols**: 다른 모듈에서 정의된 global symbol 참조
- **Local symbols**: 해당 모듈에서만 접근 가능 (`static` 함수/변수)

> 주의: 여기서 "local"은 지역 변수가 아니라 **파일 스코프**의 static 심볼이다.

### 라이브러리

**Static Library** (`.a`):
- 링킹 시점에 실행 파일에 복사됨
- 실행 파일 크기가 커지지만 의존성 없음

**Shared Library** (`.so`, `.dll`):
- 런타임에 동적으로 로드
- 여러 프로세스가 메모리에서 공유 가능
- 실행 파일 크기 작음, 라이브러리 업데이트 용이

---

## 2. 예외 제어 흐름 (ECF)

### Control Flow

프로세서는 명령어를 순차적으로 실행한다:
```
a₁ → a₂ → a₃ → ... → aₙ
```

이 순차적 흐름을 **control flow**라 하며, jump, call, return 등으로 변경된다.

하지만 시스템 수준에서는 더 복잡한 제어 흐름 변경이 필요하다:
- 디스크에서 데이터 도착
- 타이머 인터럽트
- 자식 프로세스 종료

→ 이를 처리하기 위한 메커니즘이 **ECF** (Exceptional Control Flow)

### ECF의 계층

| 수준 | 메커니즘 |
|------|----------|
| Hardware | Interrupt, Exception → Exception handler 실행 |
| OS Kernel | Context switch (프로세스 전환) |
| OS | Signal → Signal handler |
| Application | `setjmp`/`longjmp` (nonlocal jump) |

---

## 3. Exceptions

Exception은 **프로세서 상태 변화에 대한 응답**으로 OS 커널에 control을 넘기는 것이다.

```
User code 실행 중
    ↓ (event 발생)
Exception handler 실행 (kernel mode)
    ↓ (처리 후)
1) 원래 명령어로 복귀 / 2) 다음 명령어로 복귀 / 3) 프로그램 종료
```

각 exception에는 고유 번호가 있고, **exception table**에서 해당 handler의 주소를 찾는다.

### Exception의 4가지 종류

#### 1. Interrupt (비동기)

외부 I/O 장치에 의해 발생하는 **asynchronous** exception:

- 타이머 칩, 키보드, 네트워크 등 외부 장치가 CPU interrupt pin에 신호
- 현재 instruction 완료 후 handler로 이동
- 처리 후 **다음 명령어**로 복귀
- 예: 타이머 인터럽트, I/O 인터럽트

#### 2. Trap (동기)

**의도적으로** 발생시키는 synchronous exception:

- `syscall` 등의 instruction으로 발생
- 시스템 콜(system call)이 대표적
- 처리 후 **다음 명령어**로 복귀

#### 3. Fault (동기, 복구 가능)

잠재적으로 **복구 가능**한 에러:

- exception handler가 문제를 고치면 → **현재 명령어 재실행**
- 복구 불가능하면 → 프로그램 종료
- 예: **Page fault** (가장 흔함), Floating point exception

#### 4. Abort (동기, 복구 불가)

**복구 불가능**한 치명적 에러:

- 현재 프로그램을 종료
- 예: Illegal instruction, Machine check (하드웨어 오류)

---

## 4. 프로세스 (Process)

Process = **실행 중인 프로그램의 인스턴스**

### 두 가지 핵심 추상화

1. **Logical Control Flow**: 각 프로세스가 CPU를 독점하는 것처럼 보임 → **context switch**로 구현
2. **Private Address Space**: 각 프로세스가 메모리를 독점하는 것처럼 보임 → **virtual memory**로 구현

### Concurrent Processes

여러 프로세스의 control flow가 시간적으로 겹치면 **concurrent**하다고 한다.

### System Call

프로세스가 커널 서비스를 요청하는 인터페이스:

```c
// C에서는 wrapper function 사용 (실제로는 syscall instruction 실행)
pid_t getpid(void);       // 현재 프로세스 ID
pid_t fork(void);          // 새 프로세스 생성
void exit(int status);     // 프로세스 종료
pid_t waitpid(pid_t pid, int *statusp, int options);  // 자식 대기
```

### fork()

```c
pid_t pid = fork();
```

- 호출한 프로세스(parent)를 복제하여 새 프로세스(child) 생성
- **한 번 호출, 두 번 반환**:
  - Parent에게: child의 PID 반환
  - Child에게: **0** 반환
- Child는 parent의 메모리 복사본을 가짐 (별도 address space)
- 실행 순서는 OS 스케줄러가 결정 (비결정적)

```c
int main() {
    pid_t pid = fork();
    if (pid == 0) {
        printf("child\n");   // child process
    } else {
        printf("parent\n");  // parent process
    }
}
```

### waitpid / wait

```c
pid_t waitpid(pid_t pid, int *statusp, int options);
```

- 특정 자식 프로세스의 종료를 기다린다
- `pid = -1`: 아무 자식이나 대기
- 자식의 종료 상태를 `statusp`로 받을 수 있다
- 옵션: `WNOHANG` (non-blocking), `WUNTRACED` (stopped도 감지)

**Zombie 프로세스**: 종료했지만 parent가 아직 `wait()`하지 않은 프로세스. 커널이 exit status를 유지한다.

### exec

```c
int execve(const char *filename, char *const argv[], char *const envp[]);
```

- 현재 프로세스의 코드/데이터를 **새 프로그램**으로 교체
- `fork()` + `exec()`로 새 프로그램 실행하는 것이 일반적 패턴

---

## 5. 시그널 (Signals)

### 개요

Signal은 OS 커널이 프로세스에게 **이벤트 발생을 알리는** 메커니즘이다.

| Signal | 번호 | 기본 동작 | 원인 |
|--------|------|----------|------|
| SIGINT | 2 | 종료 | Ctrl+C |
| SIGKILL | 9 | 종료 (변경 불가) | `kill -9` |
| SIGSEGV | 11 | 종료+코어덤프 | Segmentation fault |
| SIGALRM | 14 | 종료 | 타이머 |
| SIGCHLD | 17 | 무시 | 자식 프로세스 종료 |
| SIGTSTP | 20 | 정지 | Ctrl+Z |
| SIGCONT | 18 | 계속 | 정지된 프로세스 재개 |

### Signal 전달 과정

1. **Send**: 커널이 대상 프로세스의 context에 signal bit를 설정
2. **Receive**: 대상 프로세스가 커널에서 user mode로 전환될 때 signal 확인

Signal의 상태:
- **Pending**: 보내졌지만 아직 받지 않은 상태
- **Blocked**: 받을 수 있지만 일시적으로 차단된 상태

> 같은 종류의 pending signal은 **최대 1개**만 유지된다 (큐잉되지 않음!)

### Signal Handler

기본 동작 대신 **사용자 정의 handler**를 설치할 수 있다:

```c
#include <signal.h>

void handler(int sig) {
    // signal 처리 코드
    printf("Caught signal %d\n", sig);
}

int main() {
    signal(SIGINT, handler);   // SIGINT에 대한 handler 설치
    // ...
}
```

### Signal Block

특정 signal을 일시적으로 차단:

```c
sigset_t mask, prev_mask;
sigemptyset(&mask);
sigaddset(&mask, SIGCHLD);

// SIGCHLD를 block
sigprocmask(SIG_BLOCK, &mask, &prev_mask);

// 여기서는 SIGCHLD가 전달되지 않음 (critical section)

// 이전 상태로 복원
sigprocmask(SIG_SETMASK, &prev_mask, NULL);
```

### Signal 처리 시 주의사항

1. **Pending signal은 큐잉되지 않는다**: 같은 signal이 여러 번 와도 1개만 유지
2. **Signal handler는 가능한 간단하게**: async-signal-safe 함수만 사용
3. **errno 보존**: handler에서 errno를 변경할 수 있으므로 진입 시 저장, 복귀 시 복원
4. **전역 데이터 접근**: `volatile sig_atomic_t` 타입 사용
5. **Slow system call**: signal에 의해 중단될 수 있음 → 재시도 로직 필요

### kill과 alarm

```c
kill(pid, SIGKILL);     // 특정 프로세스에 signal 전송
alarm(5);               // 5초 후 자신에게 SIGALRM 전달
```

---

## 6. Foreground / Background 프로세스

Shell에서의 프로세스 관리:

```bash
./program &      # background에서 실행
```

- **Foreground**: 터미널 입력을 받는 프로세스 (한 번에 최대 1개)
- **Background**: 터미널 입력 없이 실행되는 프로세스

관련 signal:
- `Ctrl+C` → foreground 프로세스 그룹에 `SIGINT` 전송
- `Ctrl+Z` → foreground 프로세스 그룹에 `SIGTSTP` 전송 (정지)

---

## 정리

| 주제 | 핵심 |
|------|------|
| Linker | Symbol resolution + Relocation, Static/Shared library |
| Exception | Interrupt(비동기), Trap(의도적), Fault(복구가능), Abort(치명적) |
| Process | fork()로 생성, exec()로 프로그램 교체, wait()로 회수 |
| Signal | 커널 → 프로세스 이벤트 알림, handler 설치 가능, pending은 큐잉 안 됨 |

---

*이전 글: [[CS230] 3. 어셈블리 심화](/posts/cs230-3-machine-advanced/)*
*다음 글: [[CS230] 5. 가상 메모리와 동적 할당](/posts/cs230-5-vm-malloc/)*
