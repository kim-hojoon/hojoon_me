---
title: "[CS230] 3. 어셈블리 심화 - Procedures, Data, Advanced"
date: 2022-12-15
draft: false
tags: ["CS230", "Systems Programming", "KAIST", "x86-64", "Stack", "Procedures"]
categories: ["Computer Science"]
summary: "함수 호출의 어셈블리 구현(스택 프레임, caller/callee-saved), 배열과 구조체의 메모리 배치, 버퍼 오버플로 공격과 방어를 다룹니다."
ShowToc: true
TocOpen: true
---

> KAIST CS230 시스템프로그래밍 (2022 Fall) 수업 내용을 정리한 시리즈입니다.
> 교재: *Computer Systems: A Programmer's Perspective* (CS:APP 3rd Edition)

---

## 1. Procedures (함수 호출)

Procedure = function = method = subroutine = handler — 모두 같은 개념이다.

함수 호출에는 3가지 메커니즘이 필요하다:

1. **Passing control**: 호출 시 `%rip`를 변경하여 control을 넘김
2. **Passing data**: register라는 공유자원에 값을 담아서 전달 (caller-saved / callee-saved)
3. **Memory management**: callee를 위한 임시적 memory를 Stack에 할당, 끝나면 반환

### Stack 구조

```text
높은 주소
┌─────────────┐
│    ...      │
│  이전 프레임  │
├─────────────┤
│ Return Addr │  ← call 명령어가 push
├─────────────┤
│ Saved regs  │  ← callee-saved 레지스터 백업
│ Local vars  │  ← 지역 변수
│             │
│ Arg build   │  ← 7번째 이후 인자
├─────────────┤ ← %rsp (stack top)
낮은 주소
```

- Stack은 **아래로 자란다** (높은 주소 → 낮은 주소)
- `%rsp`는 항상 스택의 최상단(가장 낮은 주소)을 가리킨다
- `push`: `%rsp` 감소 후 값 저장
- `pop`: 값 읽은 후 `%rsp` 증가

### call과 ret

```asm
call Label     # push return address, then jump to Label
ret            # pop return address, then jump to it
```

- `call`은 다음 명령어의 주소(return address)를 스택에 push하고 함수로 점프
- `ret`은 스택에서 return address를 pop하고 그 주소로 점프

### 인자 전달

x86-64에서 처음 6개 인자는 **레지스터**로 전달:

| 순서 | 레지스터 |
|------|----------|
| 1번째 | `%rdi` |
| 2번째 | `%rsi` |
| 3번째 | `%rdx` |
| 4번째 | `%rcx` |
| 5번째 | `%r8` |
| 6번째 | `%r9` |

7번째 이후 인자는 **스택**을 통해 전달된다.

반환값은 `%rax`에 저장.

### Caller-saved vs Callee-saved

**Caller-saved** (호출자가 보존 책임):
- `%rax`, `%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9`, `%r10`, `%r11`
- 함수 호출 전에 필요하면 직접 백업해야 함

**Callee-saved** (피호출자가 보존 책임):
- `%rbx`, `%rbp`, `%r12`, `%r13`, `%r14`, `%r15`
- 함수 진입 시 저장하고, 복귀 전에 복원해야 함

---

## 2. 배열 (Array)

### 메모리 배치

```c
T A[L];   // datatype이 T이고 길이가 L인 array
```

- 연속적으로 `L × sizeof(T)` bytes가 메모리에 할당된다
- `A` 자체는 시작 주소의 **포인터**로 사용 가능 (Type: `T*`)

```c
int val[5];      // 5 × 4 = 20 bytes 할당
// val + 1 = &val[1]  (포인터 산술: 4 bytes 이동)
```

### 배열 접근의 어셈블리

```c
int arr[5];
int x = arr[3];
```

```asm
# arr의 시작 주소가 %rdx에 있다고 가정
movl 12(%rdx), %eax     # *(arr + 3) = arr[3], 3×4=12 offset
```

### 다차원 배열

```c
int A[R][C];    // R행 C열
```

- **Row-major order**로 메모리에 저장 (행 우선)
- `A[i][j]`의 주소: `A + (i × C + j) × sizeof(int)`

```asm
# A[i][j] 접근 (A: %rdi, i: %rsi, j: %rdx, C가 상수)
leaq (%rsi, %rsi, C-1), %rax    # i * C
addq %rdx, %rax                  # i * C + j
movl (%rdi, %rax, 4), %eax      # A[i*C + j]
```

### 중첩 배열 vs 다중 레벨 배열

**중첩 배열** (`int A[3][5]`):
- 하나의 연속된 메모리 블록 (60 bytes)
- 접근: 한 번의 메모리 참조

**다중 레벨 배열** (`int *A[3]`):
- 포인터 배열 — 각 원소가 별도 배열을 가리킴
- 접근: 두 번의 메모리 참조 (포인터 → 실제 데이터)

---

## 3. 구조체 (Structures)

### 메모리 배치

```c
struct rec {
    int a[4];    // 16 bytes
    size_t i;    // 8 bytes
    struct rec *next;  // 8 bytes
};
```

- 멤버들이 선언 순서대로 연속 배치
- 컴파일러가 각 멤버의 offset을 계산

### 정렬 (Alignment)

CPU가 메모리를 효율적으로 접근하기 위해 데이터를 특정 배수의 주소에 배치하는 규칙:

- K-byte 데이터는 **K의 배수** 주소에 배치되어야 함
- `int` (4 bytes) → 4의 배수 주소
- `double` (8 bytes) → 8의 배수 주소

```c
struct S1 {
    char c;      // offset 0 (1 byte)
    // 3 bytes padding
    int i;       // offset 4 (4 bytes)
    char d;      // offset 8 (1 byte)
    // 7 bytes padding (구조체 전체가 8의 배수가 되도록)
};  // 총 크기: 16 bytes (실제 데이터는 6 bytes)
```

**최적화 팁**: 큰 타입을 먼저 선언하면 padding을 줄일 수 있다.

```c
struct S2 {
    int i;       // offset 0
    char c;      // offset 4
    char d;      // offset 5
    // 2 bytes padding
};  // 총 크기: 8 bytes — 같은 멤버인데 8 bytes 절약!
```

---

## 4. 버퍼 오버플로 (Buffer Overflow)

### 취약점

스택에 할당된 버퍼의 범위를 초과하여 데이터를 쓰면, **return address를 덮어쓸 수 있다**.

```c
void echo() {
    char buf[8];
    gets(buf);      // 입력 길이 제한 없음 → 위험!
}
```

공격자가 return address를 악의적인 코드의 주소로 덮어쓰면 → **임의 코드 실행**

### 방어 기법

**1. Stack Canary (Stack Protector)**
- 함수 진입 시 return address 앞에 랜덤 값(canary)을 배치
- 함수 복귀 전에 canary가 변조되었는지 검사
- GCC: `-fstack-protector`

**2. ASLR (Address Space Layout Randomization)**
- 프로그램 실행 시마다 스택, 힙, 라이브러리의 주소를 랜덤화
- 공격자가 특정 주소를 예측하기 어렵게 만듦

**3. NX bit (No-Execute)**
- 스택 영역을 실행 불가능으로 표시
- 스택에 주입된 코드의 실행을 방지

**4. 안전한 코딩**
- `gets()` 대신 `fgets()` 사용
- `strcpy()` 대신 `strncpy()` 사용
- 항상 버퍼 크기를 확인

---

## 5. Union

```c
union U {
    int i;
    double d;
    char c;
};
```

- 모든 멤버가 **같은 메모리 공간**을 공유
- 크기 = 가장 큰 멤버의 크기
- 한 번에 하나의 멤버만 유효하게 사용 가능
- 메모리 절약이나 type punning에 활용

---

## 정리

| 주제 | 핵심 |
|------|------|
| Procedure | call/ret로 제어, 레지스터 6개로 인자 전달, %rax로 반환 |
| Stack Frame | 아래로 성장, return addr → saved regs → locals → args |
| Caller/Callee-saved | 호출 규약에 따라 레지스터 보존 책임이 다름 |
| Array | 연속 메모리, 포인터 산술로 접근 |
| Struct Alignment | K-byte 데이터는 K의 배수 주소에 정렬 (padding 발생) |
| Buffer Overflow | return address 덮어쓰기 → canary, ASLR, NX로 방어 |

---

*이전 글: [[CS230] 2. 어셈블리와 기계어 기초](/posts/cs230-2-machine-basics/)*
*다음 글: [[CS230] 4. 링킹과 예외 제어 흐름](/posts/cs230-4-linking-ecf/)*
