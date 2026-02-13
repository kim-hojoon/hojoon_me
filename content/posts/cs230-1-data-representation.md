---
title: "[CS230] 1. 데이터 표현 - Bits, Integers, Floating Point"
date: 2022-12-15
draft: false
tags: ["CS230", "Systems Programming", "KAIST", "Data Representation"]
categories: ["Computer Science"]
summary: "컴퓨터가 데이터를 표현하는 방법을 다룹니다. ASCII/Unicode 문자 인코딩부터 정수의 비트 표현(2의 보수), 부동소수점(IEEE 754)까지 정리합니다."
ShowToc: true
TocOpen: true
---

> KAIST CS230 시스템프로그래밍 (2022 Fall) 수업 내용을 정리한 시리즈입니다.
> 교재: *Computer Systems: A Programmer's Perspective* (CS:APP 3rd Edition)

---

## 1. 문자 인코딩

### ASCII

**ASCII**(American Standard Code for Information Interchange)는 영문 알파벳을 사용하는 대표적인 문자 인코딩이다.

- 7 bits encoding → 총 2^7 = **128**개의 경우의 수
- 33개의 제어 문자(출력 불가) + 95개의 출력 가능 문자
- 대소문자 영문 알파벳 52개 + 숫자 10개 + 특수문자 32개 + 공백문자 1개

C에서 `char`은 1 Byte 크기의 데이터 타입이다. `char`에 담긴 숫자를 ASCII에 따라 대응된 문자 1개로 해석하는 것이다.

```c
char s[] = "18213";
// 문자열 "18213"은 5개의 문자이지만, 배열 크기는 6이다
// 모든 string의 마지막은 NULL(\0)이어야 하기 때문
// → String should be null-terminated
```

### Unicode

ASCII code가 영어·숫자·일부 특수기호에 대한 대응표라면, 전 세계의 모든 문자에 대한 국제 표준 대응표가 **Unicode**이다.

- ASCII code와 달리 7 bit가 아닌 더 많은 bit를 사용
- 한글을 표시할 때 Unicode를 사용
- Unicode는 1~4 byte로 표현된다 (ASCII code는 항상 1 byte)

### 이스케이프 시퀀스

백슬래시(`\`) 뒤에 문자 1개 혹은 숫자 조합이 오는 문자열로, 컴퓨터와 주변기기의 상태를 바꾸는 데 쓰이는 일련의 문자열이다.

| 시퀀스 | 의미 |
|--------|------|
| `\n` | 줄바꿈 (newline) |
| `\t` | 탭 (tab) |
| `\0` | NULL |
| `\\` | 백슬래시 |
| `\'` | 작은따옴표 |
| `\"` | 큰따옴표 |

---

## 2. C 언어 기초 정리

### 데이터 타입과 크기

C의 데이터 타입에서 중요한 점:

- **고정 너비 타입**(`stdint.h`): `int32_t`, `uint32_t` 등 — 플랫폼에 관계없이 크기가 보장된다
- `size_t`: 해당 시스템에서 어떤 오브젝트의 크기도 담을 수 있는 **unsigned** 타입
- `t`와 `type`의 차이: `typedef`의 약자인지, 직접 크기를 갖는 타입인지 구분

### 포인터 (Pointer)

포인터는 **변수의 주소를 저장**하는 변수이다.

```c
int *p;        // int형 포인터 선언
int num = 10;
p = &num;      // num의 주소를 p에 저장
*p = 20;       // 역참조로 값 변경 → num은 이제 20
```

주요 특징:
- `*p`는 주소가 가리키는 곳의 값 (역참조)
- `&num`은 num 변수의 메모리 주소
- 포인터의 크기는 시스템에 따라 다름 (32bit: 4 bytes, 64bit: 8 bytes)

### void 포인터

`void *`는 어떤 타입이든 가리킬 수 있는 포인터이다.

- **type casting** 없이 그냥 어떤 포인터든 받을 수 있다
- 역참조하려면 반드시 type casting이 필요
- Explicit Type Casting: 직접 `(int *)` 등을 써주는 것
- Implicit Type Casting: 컴파일러가 자동 변환

### malloc과 free

동적 메모리 할당 방법:

```c
int *numPtr;
numPtr = malloc(sizeof(int));   // heap에 int 크기만큼 메모리 할당
*numPtr = 10;                   // 역참조로 값 저장
free(numPtr);                   // 사용 후 반드시 해제
```

- 변수 선언으로 할당된 메모리는 **Stack**에 위치
- `malloc`으로 할당된 메모리는 **Heap**에 위치
- `malloc` → 사용 → `free` 순서를 지켜야 한다

### 구조체 (struct)

```c
struct 구조체이름 {
    자료형 멤버변수;
    // ...
};

struct Person p1;
p1.name = 222;     // 점(.) 연산자로 멤버 접근
```

- 구조체 멤버 접근: `.` (직접), `->` (포인터를 통해)
- `typedef`를 사용하면 `struct` 키워드 없이 사용 가능
- 구조체 포인터: `malloc(sizeof(struct 구조체이름))`으로 동적 할당 가능

### static과 스코프

```c
static int count = 0;  // 파일 내에서만 접근 가능
```

- `static` 지역 변수: 함수가 끝나도 값이 유지됨
- `static` 전역 변수/함수: 해당 파일 내에서만 접근 가능 (scope 제한)
- `extern`: 다른 파일의 전역 변수를 참조

**Declaration vs Definition:**
- **Declaration**: 컴파일러에게 identifier의 이름을 알리는 것 (메모리 할당 없음)
- **Definition**: 실제로 code를 부여하거나 메모리를 할당하는 것

---

## 3. Everything is Bits

컴퓨터에서 **모든 것은 비트(bits)** 이다.

- Each bit is 0 or 1 (`true` / `false`)
- **Byte** = 8 bits → 2^8 = 256가지 경우의 수
- 모든 데이터는 결국 0과 1로 표현된다

### 데이터 타입별 크기

| 타입 | 32-bit | 64-bit |
|------|--------|--------|
| `char` | 1 | 1 |
| `short` | 2 | 2 |
| `int` | 4 | 4 |
| `long` | 4 | **8** |
| `float` | 4 | 4 |
| `double` | 8 | 8 |
| `pointer` | 4 | **8** |

> `long`과 `pointer`는 32-bit과 64-bit 시스템에서 크기가 다르다!

### Boolean Algebra

비트 단위 논리 연산:

| 연산  | 기호   | 설명         |
| --- | ---- | ---------- |
| AND | `&`  | 둘 다 1이면 1  |
| OR  | `\|` | 하나라도 1이면 1 |
| NOT | `~`  | 반전         |
| XOR | `^`  | 서로 다르면 1   |

특수 성질:
- `A ^ A = 0` (자기 자신과 XOR하면 0)
- `A ^ 0 = A`
- XOR은 교환법칙, 결합법칙 성립

### Shift 연산

- **Left Shift** (`x << k`): 왼쪽으로 k비트 이동, 오른쪽은 0으로 채움 → **×2^k** 효과
- **Logical Right Shift** (`>>`, unsigned): 오른쪽으로 이동, 왼쪽은 0으로 채움
- **Arithmetic Right Shift** (`>>`, signed): 오른쪽으로 이동, 왼쪽은 **부호 비트**로 채움

---

## 4. 정수 (Integers)

### Unsigned 정수

- B2U (Binary to Unsigned): 각 비트의 가중치를 합산
- w비트일 때 범위: **0 ~ 2^w - 1**

### Signed 정수: 2의 보수 (Two's Complement)

- B2T (Binary to Two's complement)
- **최상위 비트(MSB)** 가 부호를 나타냄 (sign bit)
  - MSB = 0 → 양수
  - MSB = 1 → 음수
- w비트일 때 범위: **-2^(w-1) ~ 2^(w-1) - 1**

예시 (4비트):
- `0111` = 7 (최대 양수)
- `1000` = -8 (최소 음수)
- `1111` = -1

### Unsigned와 Signed의 관계

- **nonnegative** 범위에서는 unsigned와 signed의 비트 패턴이 동일
- 같은 비트 패턴도 해석 방식에 따라 값이 달라진다
- C에서 signed와 unsigned를 섞어 쓰면 **implicit casting**이 발생 → 주의!

### Expanding과 Truncating

**Expanding** (작은 타입 → 큰 타입):
- Unsigned: 앞에 **0**으로 채움 (zero extension)
- Signed: 앞에 **부호 비트**로 채움 (sign extension)
- 값이 보존된다

**Truncating** (큰 타입 → 작은 타입):
- 상위 비트를 잘라냄
- 값이 변할 수 있다! → `yields unexpected results`

---

## 5. 정수 연산

### 덧셈과 오버플로

**Unsigned Addition:**
- `u + v`가 2^w를 넘으면 → **overflow** 발생 (상위 비트 버림)

**Signed Addition:**
- 양수 + 양수 → 음수가 되면: **positive overflow**
- 음수 + 음수 → 양수가 되면: **negative overflow**

### 곱셈과 나눗셈

**Power of 2 곱셈:**
- `u << k` = u × 2^k (unsigned, signed 모두)

**Power of 2 나눗셈 (with round down):**
- Unsigned: `u >> k` = u / 2^k (logical right shift)
- Signed: `(x + (1 << k) - 1) >> k` — 음수일 때 round toward zero를 위해 bias 추가

---

## 6. 메모리와 바이트 순서

### Word Size

- **Word size**는 CPU 레지스터의 크기를 결정하며, 주소 공간의 크기를 결정한다
- 32-bit 시스템: 주소 공간 = 2^32 = **4 GB**
- 64-bit 시스템: 주소 공간 = 2^64 = **16 EB**

### Byte Addressing

대부분의 컴퓨터는 **byte addressable** — 메모리의 가장 작은 접근 단위가 1 byte이다.

- 포인터의 값 = 해당 데이터가 저장된 **첫 번째 바이트의 주소**
- `int`(4 bytes)를 가리키는 포인터: 4바이트 중 **시작 주소**만 저장

### Byte Ordering: Big Endian vs Little Endian

4 byte 데이터 `0x01234567`의 메모리 저장 방식:

**Big Endian** (MSB가 낮은 주소에):
```
주소:   0x100  0x101  0x102  0x103
데이터:  01     23     45     67
```

**Little Endian** (LSB가 낮은 주소에):
```
주소:   0x100  0x101  0x102  0x103
데이터:  67     45     23     01
```

- **Big Endian**: MIPS, Sun, PPC, Mac, Internet 바이트 순서
- **Little Endian**: x86, ARM (보통), Android, iOS, Windows

> 네트워크 통신 시 Big Endian(Network Byte Order)으로 통일한다.

---

## 7. 부동소수점 (Floating Point)

### 고정 소수점의 한계

Binary point를 고정하면 표현 가능한 수의 범위가 제한된다.
→ **Floating point**: binary point 위치를 유동적으로 조정하여 넓은 범위를 표현

### IEEE 754 표준

현대 거의 모든 CPU가 지원하는 부동소수점 산술 표준.

**구조:**
```
(-1)^s × M × 2^E
```

| 필드 | 설명 |
|------|------|
| **s** (sign bit) | 부호 (0: 양수, 1: 음수) |
| **exp** (exponent) | 지수부 |
| **frac** (fraction) | 가수부 (mantissa) |

### 정밀도별 크기

| 정밀도 | 총 비트 | exp | frac |
|--------|---------|-----|------|
| Single (float) | 32 | 8 | 23 |
| Double (double) | 64 | 11 | 52 |

### 인코딩 규칙

**Normalized Values** (exp ≠ 000...0 이고 exp ≠ 111...1):
- E = exp - Bias (Bias = 2^(k-1) - 1)
- M = 1.frac (implicit leading 1)
- Single precision: Bias = 127, Double: Bias = 1023

**Denormalized Values** (exp = 000...0):
- E = 1 - Bias
- M = 0.frac (no implicit leading 1)
- 0에 가까운 매우 작은 수를 표현

**Special Values** (exp = 111...1):
- frac = 0 → ±∞ (Infinity)
- frac ≠ 0 → **NaN** (Not a Number)

### Rounding (반올림)

IEEE 754는 기본적으로 **Round-to-Even** (Banker's Rounding)을 사용한다:
- 정확히 중간 값일 때, 가장 가까운 **짝수**로 반올림
- 통계적 bias를 줄이기 위한 방법

다른 반올림 모드:
- Round toward zero
- Round down (toward -∞)
- Round up (toward +∞)

### C에서의 Float

```c
float f = 1.0f;    // 4 byte, single precision
double d = 1.0;    // 8 byte, double precision
```

**Casting 시 주의:**
- `int` → `float`: 큰 값에서 정밀도 손실 가능 (rounding)
- `int` → `double`: 정밀도 보존 (52-bit mantissa > 32-bit int)
- `float` → `int`: 소수점 이하 버림, 범위 초과 시 **undefined behavior**
- `double` → `float`: 정밀도 손실 + 범위 초과 가능

---

## 정리

| 주제 | 핵심 |
|------|------|
| ASCII/Unicode | 문자를 숫자로 대응하는 인코딩 표준 |
| Boolean Algebra | AND, OR, NOT, XOR — 비트 연산의 기초 |
| Unsigned | 0 ~ 2^w - 1, 항상 양수 |
| Two's Complement | -2^(w-1) ~ 2^(w-1) - 1, MSB가 부호 |
| Byte Ordering | Big/Little Endian — 멀티바이트 데이터 저장 순서 |
| IEEE 754 | (-1)^s × M × 2^E, 정규화/비정규화/특수값 |

---

*다음 글: [[CS230] 2. 어셈블리와 기계어 기초](/posts/cs230-2-machine-basics/)*
