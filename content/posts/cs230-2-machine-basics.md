---
title: "[CS230] 2. 어셈블리와 기계어 기초 - Machine Basics & Control"
date: 2022-12-15
draft: false
tags: ["CS230", "Systems Programming", "KAIST", "x86-64", "Assembly"]
categories: ["Computer Science"]
summary: "C 코드가 어셈블리와 기계어로 변환되는 과정, x86-64 레지스터, mov/leaq 명령어, 조건 분기와 반복문의 어셈블리 구현을 다룹니다."
ShowToc: true
TocOpen: true
---

> KAIST CS230 시스템프로그래밍 (2022 Fall) 수업 내용을 정리한 시리즈입니다.
> 교재: *Computer Systems: A Programmer's Perspective* (CS:APP 3rd Edition)

---

## 1. 코드의 변환 과정

```text
C 코드 → Assembly 코드 → Machine 코드 (Object Code)
```

- **Assembly language**: 기계어와 1:1 대응되는 human readable programming language
- **Machine code**: CPU가 직접 해석하는 binary (ISA에 의해 정의된다)
- 컴파일러가 C 코드를 어셈블리로 변환하고, 어셈블러가 기계어로 변환한다

### ISA (Instruction Set Architecture)

ISA는 software와 hardware 사이의 인터페이스(약속)이다.

- ISA에 따라 어떤 instruction이 존재하고, 어떤 레지스터를 사용하는지 등이 결정된다
- 같은 CPU라도 다른 ISA를 쓸 수 있고, 같은 ISA라도 다른 CPU에서 돌릴 수 있다
- **CISC** (Complex Instruction Set Computer): 많은 수의 복잡한 명령어 — x86, x86-64
- **RISC** (Reduced Instruction Set Computer): 적은 수의 간단한 명령어 — ARM, MIPS, RISC-V

---

## 2. 어셈블리 데이터 타입

| 접미사 | 크기 | C 타입 |
|--------|------|--------|
| `b` (byte) | 1 byte | `char` |
| `w` (word) | 2 bytes | `short` |
| `l` (long word) | 4 bytes | `int` |
| `q` (quad word) | 8 bytes | `long`, pointer |

- 어셈블리에서는 Array, Structure 같은 aggregate type은 연속된 바이트의 묶음으로 취급

---

## 3. CPU 구조와 레지스터

CPU의 기본 구성:
```text
CPU: [Registers] ←→ [ALU]
         ↕
    Address Bus / Data Bus
         ↕
      Memory (Code, Data, Stack)
```

### x86-64 범용 레지스터 (64-bit)

| 레지스터 | 용도 |
|----------|------|
| `%rax` | 반환값 (return value) |
| `%rbx` | callee-saved |
| `%rcx` | 4번째 인자 |
| `%rdx` | 3번째 인자 |
| `%rsi` | 2번째 인자 |
| `%rdi` | 1번째 인자 |
| `%rbp` | callee-saved (base pointer) |
| `%rsp` | **stack pointer** (특수) |
| `%r8` | 5번째 인자 |
| `%r9` | 6번째 인자 |
| `%r10`~`%r11` | caller-saved |
| `%r12`~`%r15` | callee-saved |

- 64-bit 레지스터 (`%rax`)의 하위 32/16/8 비트도 접근 가능: `%eax`, `%ax`, `%al`
- `%rsp`와 `%rbp`는 특수 용도 (스택 관리)로 쓰이므로 일반 연산에 사용하지 않는다
- **Program Counter** (`%rip`): 다음에 실행할 명령어의 주소를 저장

---

## 4. 데이터 이동 (mov)

### mov 명령어

```asm
mov  S, D      # Source의 값을 Destination에 복사
```

- `movb`, `movw`, `movl`, `movq` — 크기별 접미사
- operand 유형: **immediate** / **register** / **memory**

| Source | Destination | 가능? |
|--------|-------------|-------|
| Imm | Reg | O |
| Imm | Mem | O |
| Reg | Reg | O |
| Reg | Mem | O |
| Mem | Reg | O |
| **Mem** | **Mem** | **X** (memory-to-memory 불가!) |

> memory → memory는 한 번에 불가능. register를 거쳐야 한다.

### movz와 movs (확장 이동)

작은 크기 → 큰 크기로 이동할 때:

- **movz** (zero-extending): 상위 비트를 **0**으로 채움 → unsigned
- **movs** (sign-extending): 상위 비트를 **부호 비트**로 채움 → signed

```asm
movzbl %al, %edx     # 1 byte → 4 bytes, zero extension
movslq %eax, %rdx    # 4 bytes → 8 bytes, sign extension
```

특이사항: `movl`로 32-bit 레지스터에 쓰면 상위 32비트가 **자동으로 0**이 된다.

---

## 5. leaq와 산술 연산

### leaq (Load Effective Address)

```asm
leaq S, D     # S의 address mode expression 결과를 D에 저장
```

- S: address mode expression
- D: **register only** (memory 불가)
- 실제로 메모리에 접근하지 않음 — 주소 계산만 수행
- 곱셈/덧셈 조합을 한 번에 할 수 있어서 산술 연산에도 자주 사용

### Addressing Mode

```text
D(Rb, Ri, S) = Rb + S·Ri + D
```

- `Rb`: base register
- `Ri`: index register
- `S`: scale factor (1, 2, 4, 8 중 하나만 가능)
- `D`: displacement (상수)

예시:
```asm
leaq (%rdi, %rdi, 2), %rax    # rax = rdi + 2*rdi = 3*rdi
leaq 7(%rdx, %rdx, 4), %rax   # rax = 5*rdx + 7
```

### 산술/논리 연산 명령어

| 명령어 | 효과 | 설명 |
|--------|------|------|
| `INC D` | D ← D+1 | 증가 |
| `DEC D` | D ← D-1 | 감소 |
| `NEG D` | D ← -D | 부호 반전 |
| `NOT D` | D ← ~D | 비트 반전 |
| `ADD S, D` | D ← D+S | 덧셈 |
| `SUB S, D` | D ← D-S | 뺄셈 |
| `IMUL S, D` | D ← D×S | 곱셈 |
| `XOR S, D` | D ← D^S | 배타적 OR |
| `OR S, D` | D ← D\|S | OR |
| `AND S, D` | D ← D&S | AND |
| `SAL k, D` | D ← D<<k | 왼쪽 시프트 |
| `SHL k, D` | D ← D<<k | SAL과 동일 |
| `SAR k, D` | D ← D>>ₐk | 산술 오른쪽 시프트 |
| `SHR k, D` | D ← D>>ₗk | 논리 오른쪽 시프트 |

---

## 6. Condition Codes

산술/논리 연산 실행 시 CPU가 자동으로 설정하는 1-bit 플래그:

| 플래그 | 의미 |
|--------|------|
| **CF** (Carry Flag) | unsigned overflow 발생 시 1 |
| **SF** (Sign Flag) | 결과의 최상위 비트 = 1이면 1 (음수) |
| **ZF** (Zero Flag) | 결과가 0이면 1 |
| **OF** (Overflow Flag) | signed overflow 발생 시 1 |

### Implicitly Set

- 대부분의 산술/논리 연산이 condition code를 **암묵적으로** 설정
- `leaq`는 condition code를 설정하지 **않는다**

### Explicitly Set

**cmp** (compare):
```asm
cmpq S₂, S₁     # S₁ - S₂ 를 계산하고 결과는 버림, condition code만 설정
```

**test**:
```asm
testq S₂, S₁    # S₁ & S₂ 를 계산하고 결과는 버림, condition code만 설정
```

### set 명령어

condition code 조합에 따라 destination의 하위 1 byte를 0 또는 1로 설정:

| 명령어 | 조건 | 설명 |
|--------|------|------|
| `sete` | ZF | Equal (==) |
| `setne` | ~ZF | Not Equal (!=) |
| `setl` | SF^OF | Less (signed <) |
| `setge` | ~(SF^OF) | Greater or Equal (signed >=) |
| `setb` | CF | Below (unsigned <) |
| `seta` | ~CF & ~ZF | Above (unsigned >) |

---

## 7. 분기 (Jump)

### 무조건 점프

```asm
jmp Label      # 무조건 해당 Label로 이동
jmp *Operand   # indirect jump — 레지스터/메모리의 값을 주소로 사용
```

### 조건 점프

condition code에 따라 분기:

| 명령어 | 조건 | 설명 |
|--------|------|------|
| `je` | ZF | Equal |
| `jne` | ~ZF | Not Equal |
| `jl` | SF^OF | Less (signed) |
| `jge` | ~(SF^OF) | Greater or Equal (signed) |
| `jb` | CF | Below (unsigned) |
| `ja` | ~CF & ~ZF | Above (unsigned) |
| `js` | SF | Negative |

### 조건부 이동 (cmov)

분기 대신 조건에 따라 값을 이동하는 명령어:

```asm
cmovle %rdx, %rax    # if (SF^OF)|ZF then rax ← rdx
```

장점: 분기 예측(branch prediction) 실패로 인한 파이프라인 페널티를 피할 수 있다.

주의: 양쪽 operand가 **모두 계산**되므로, side effect가 있는 경우에는 사용 불가.

### C의 조건식과 어셈블리

```c
val = Test ? Then_Expr : Else_Expr;
```

어셈블리 (goto 스타일):
```c
result = Then_Expr;
eval = Else_Expr;
nt = !Test;
if (nt) result = eval;
return result;
```

---

## 8. 반복문

### do-while

```c
do {
    Body;
} while (Test);
```

어셈블리 패턴:
```text
loop:
    Body
    if (Test) goto loop
```

### while

**Jump-to-middle 변환:**
```text
    goto test
loop:
    Body
test:
    if (Test) goto loop
```

**Do-while 변환 (GCC -O1):**
```text
    if (!Test) goto done
loop:
    Body
    if (Test) goto loop
done:
```

### for

```c
for (Init; Test; Update) {
    Body;
}
```

while로 변환 후 동일한 패턴 적용:
```c
Init;
while (Test) {
    Body;
    Update;
}
```

### switch

```c
switch (x) {
    case 1: Body1; break;
    case 2: Body2; break;
    default: BodyD;
}
```

컴파일러는 **jump table**을 생성하여 O(1)로 분기:
```asm
    goto *jmptable[x];    # 간접 점프
```

- case 값의 범위가 작고 밀집되어 있으면 jump table이 효율적
- 범위 밖의 값은 `default`로 분기

---

## 정리

| 주제 | 핵심 |
|------|------|
| ISA | 소프트웨어와 하드웨어의 인터페이스 (x86-64, ARM 등) |
| mov | 데이터 이동, memory↔memory 불가 |
| leaq | 주소 계산 (실제 메모리 접근 안 함), 산술에도 활용 |
| Condition Codes | CF, SF, ZF, OF — 연산 결과에 따라 자동 설정 |
| Jump | 조건 분기로 if/else, loop 구현 |
| cmov | 분기 없는 조건부 이동 — 파이프라인 효율 향상 |
| switch | jump table로 O(1) 분기 |

---

*이전 글: [[CS230] 1. 데이터 표현](/posts/cs230-1-data-representation/)*
*다음 글: [[CS230] 3. 어셈블리 심화](/posts/cs230-3-machine-advanced/)*
