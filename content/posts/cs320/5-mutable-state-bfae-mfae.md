---
title: "[CS320] 5. 가변 상태 — BFAE와 MFAE"
date: 2023-06-09
draft: false
tags: ["CS320", "Programming Languages", "KAIST", "BFAE", "MFAE", "Store", "Mutable State"]
categories: ["Computer Science"]
series: ["CS320 프로그래밍언어"]
summary: "Box를 통한 힙 메모리 모델(BFAE)과 가변 변수(MFAE)를 도입하며, Store를 이용한 상태 관리 인터프리터를 구현합니다."
aliases: ["/posts/cs320-5-mutable-state-bfae-mfae/"]
ShowToc: true
TocOpen: true
---

> KAIST CS320 프로그래밍언어 (Spring 2023)
> 교재: 수업 자료 기반

---

## 들어가며

지금까지 우리가 다룬 FAE(First-class Functions with Arithmetic Expressions)는 **불변(immutable)** 환경만을 사용했습니다. 변수에 값을 한 번 바인딩하면 그 값은 절대 변하지 않았죠. 하지만 실제 프로그래밍 언어에서는 **가변 상태(mutable state)**가 필수적입니다. 변수의 값을 바꾸거나, 힙(heap) 메모리에 데이터를 저장하고 수정하는 기능이 필요합니다.

이번 글에서는 가변 상태를 지원하는 두 가지 언어를 다룹니다:

1. **BFAE** (FAE + Boxes): 힙 메모리에 값을 저장할 수 있는 **Box**를 명시적으로 제공
2. **MFAE** (BFAE + Mutable Variables): 모든 변수가 암묵적으로 가변적인 언어

핵심은 **Store(저장소)**라는 새로운 개념을 도입하여 메모리 상태를 관리하고, 인터프리터가 계산 과정에서 이 Store를 **threading(전달)**하는 방식입니다.

---

## BFAE: FAE에 Box 추가하기

### 왜 Environment만으로는 부족한가?

FAE에서 Environment는 `Map[String, Value]` 타입으로, 변수 이름을 값에 직접 매핑했습니다. 이는 간단하지만 두 가지 문제가 있습니다:

1. **불변성**: 한 번 바인딩된 값은 절대 변경할 수 없습니다.
2. **공유 불가**: 여러 변수가 같은 메모리 위치를 가리킬 수 없습니다.

실제 프로그램에서는 다음과 같은 상황이 필요합니다:

```scala
val box = new Box(10)  // 힙에 10을 저장
box.set(20)            // 같은 위치의 값을 20으로 변경
val value = box.get    // 20을 읽어옴
```

이를 위해 **Store(저장소)**라는 개념을 도입합니다. Store는 힙 메모리를 모델링하며, **주소(address)**를 값에 매핑합니다.

### BFAE의 문법

BFAE는 FAE에 세 가지 box 연산과 순차 실행(sequencing)을 추가합니다:

**구체 문법(Concrete Syntax):**

```text
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
       | "(" expr "*" expr ")"
       | id
       | "{" id "=>" expr "}"
       | expr "(" expr ")"
       | "Box" "(" expr ")"        // 새 박스 생성
       | expr "." "set" "(" expr ")"  // 박스에 값 저장
       | expr "." "get"            // 박스에서 값 읽기
       | "{" expr ";" expr "}"     // 순차 실행
```

**추상 문법(Abstract Syntax):**

```scala
trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
case class Id(x: String) extends Expr
case class Fun(x: String, b: Expr) extends Expr
case class App(f: Expr, a: Expr) extends Expr
case class NewBox(e: Expr) extends Expr          // Box(e): 새 박스 생성
case class SetBox(b: Expr, e: Expr) extends Expr // b.set(e): 박스 업데이트
case class OpenBox(b: Expr) extends Expr         // b.get: 박스 읽기
case class Seqn(l: Expr, r: Expr) extends Expr   // {l; r}: 순차 실행
```

### Store의 도입

BFAE에서는 Environment와 Store를 분리합니다:

```scala
type Env = Map[String, Value]   // 변수 → 값
type Addr = Int                 // 주소 타입
type Sto = Map[Addr, Value]     // 주소 → 값 (힙 메모리 모델)

trait Value
case class NumV(n: Int) extends Value
case class CloV(x: String, b: Expr, fenv: Env) extends Value
case class BoxV(addr: Addr) extends Value    // Box는 주소만 저장!
```

**중요한 점:**
- `BoxV`는 값 자체가 아니라 **주소(address)**만 가지고 있습니다.
- 실제 값은 Store(`Sto`)에 저장됩니다.
- 이는 C/C++의 포인터, Java의 참조(reference)와 유사한 개념입니다.

### Store-Passing Style 인터프리터

가변 상태를 다루기 위해 인터프리터의 시그니처가 변경됩니다:

```scala
// Before (FAE):
def interp(e: Expr, env: Env): Value

// After (BFAE):
def interp(e: Expr, env: Env, sto: Sto): (Value, Sto)
```

이제 `interp`는 **Store를 입력으로 받고, (결과값, 새로운 Store)의 튜플을 반환**합니다. 이를 **store-passing style**이라고 합니다.

### BFAE 인터프리터 구현

```scala
def interp(e: Expr, env: Env, sto: Sto): (Value, Sto) = e match {
  // 기본 연산: Store는 변경 없이 그대로 전달
  case Num(n) => (NumV(n), sto)

  // 산술 연산: Store를 왼쪽에서 오른쪽으로 threading
  case Add(l, r) =>
    val (lv, ls) = interp(l, env, sto)  // 왼쪽 먼저 계산, 새 Store ls
    val (rv, rs) = interp(r, env, ls)   // 오른쪽 계산할 때 ls 사용, 새 Store rs
    (numVAdd(lv, rv), rs)               // 최종 Store rs 반환

  case Sub(l, r) =>
    val (lv, ls) = interp(l, env, sto)
    val (rv, rs) = interp(r, env, ls)
    (numVSub(lv, rv), rs)

  // 변수 참조와 함수 생성: Store 변경 없음
  case Id(x) => (lookup(x, env), sto)
  case Fun(x, b) => (CloV(x, b, env), sto)

  // 함수 적용: Store threading 필수
  case App(f, a) =>
    val (fv, fs) = interp(f, env, sto)  // 함수 계산
    fv match {
      case CloV(x, b, fenv) =>
        val (av, as) = interp(a, env, fs)  // 인자 계산, fs 사용
        interp(b, fenv + (x -> av), as)    // 본체 계산, as 사용
      case v => error(s"not a closure: $v")
    }

  // NewBox: 새 주소 할당 후 Store에 추가
  case NewBox(e) =>
    val (v, s) = interp(e, env, sto)   // 저장할 값 계산
    val addr = malloc(s)               // 새 주소 생성 (예: maxKey + 1)
    (BoxV(addr), s + (addr -> v))      // BoxV(주소) 반환, Store 업데이트

  // OpenBox: Store에서 값 읽기
  case OpenBox(b) =>
    val (bv, bs) = interp(b, env, sto)
    bv match {
      case BoxV(addr) =>
        (storeLookup(addr, bs), bs)    // Store에서 값 조회, Store 변경 없음
      case _ => error(s"not a box: $bv")
    }

  // SetBox: Store의 값 업데이트
  case SetBox(b, e) =>
    val (bv, bs) = interp(b, env, sto) // box 계산
    bv match {
      case BoxV(addr) =>
        val (v, s) = interp(e, env, bs)  // 새 값 계산, bs 사용
        (v, s + (addr -> v))             // Store 업데이트 (기존 값 덮어쓰기)
      case _ => error(s"not a box: $bv")
    }

  // Seqn: 왼쪽 계산 후 오른쪽 계산 (왼쪽 결과값은 버림)
  case Seqn(l, r) =>
    val (lv, ls) = interp(l, env, sto)
    interp(r, env, ls)  // ls를 사용하여 r 계산
}
```

### Store Threading의 중요성

위 코드에서 Store가 **왼쪽에서 오른쪽으로** 정확하게 전달되는 것을 주목하세요:

```scala
case Add(l, r) =>
  val (lv, ls) = interp(l, env, sto)  // 1. 왼쪽 먼저 계산
  val (rv, rs) = interp(r, env, ls)   // 2. 왼쪽의 결과 Store(ls)를 사용
  (numVAdd(lv, rv), rs)               // 3. 최종 Store(rs) 반환
```

만약 `interp(r, env, sto)`처럼 원래 Store를 사용하면, `l`의 side effect가 `r`에 반영되지 않습니다! 이는 프로그램의 평가 순서(evaluation order)와 직결됩니다.

### BFAE 예제

```scala
// { val b = Box(0); { b.set(5); b.get } }
val program = App(
  Fun("b",
    Seqn(
      SetBox(Id("b"), Num(5)),
      OpenBox(Id("b"))
    )
  ),
  NewBox(Num(0))
)

interp(program, Map.empty, Map.empty)
// => (NumV(5), Map(0 -> NumV(5)))
```

실행 과정:
1. `NewBox(Num(0))`: 주소 0에 0 저장 → `(BoxV(0), Map(0 -> NumV(0)))`
2. 함수 적용: `b`를 `BoxV(0)`에 바인딩
3. `SetBox(Id("b"), Num(5))`: 주소 0의 값을 5로 변경 → `Map(0 -> NumV(5))`
4. `OpenBox(Id("b"))`: 주소 0에서 5를 읽음 → `NumV(5)`

---

## MFAE: 가변 변수로 확장하기

BFAE에서는 **명시적인 Box**를 통해서만 가변 상태를 만들 수 있었습니다. 하지만 대부분의 프로그래밍 언어(Python, JavaScript, Java 등)에서는 **변수 자체가 가변적**입니다:

```python
x = 10
x = 20  # 같은 변수 x의 값을 변경
```

MFAE는 이러한 **가변 변수(mutable variables)**를 지원합니다.

### MFAE의 문법

BFAE에 변수 할당(assignment) 연산을 추가합니다:

**구체 문법:**

```text
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
       | "(" expr "*" expr ")"
       | id
       | "{" id "=>" expr "}"
       | expr "(" expr ")"
       | "Box" "(" expr ")"
       | expr "." "set" "(" expr ")"
       | expr "." "get"
       | "{" expr ";" expr "}"
       | "{" id "=" expr "}"        // 변수 할당 추가!
```

**추상 문법:**

```scala
case class Set(x: String, e: Expr) extends Expr  // { x = e }
```

### 핵심 아이디어: Environment의 타입 변경

MFAE의 가장 중요한 변화는 **Environment의 타입**입니다:

```scala
// BFAE:
type Env = Map[String, Value]   // 변수 → 값 직접 매핑

// MFAE:
type Env = Map[String, Addr]    // 변수 → 주소 매핑!
type Sto = Map[Addr, Value]     // 주소 → 값

trait Value
case class NumV(n: Int) extends Value
case class CloV(x: String, b: Expr, var fenv: Env) extends Value
case class BoxV(addr: Addr) extends Value
```

**변수가 값 대신 주소를 가리키도록 변경**했습니다! 이제:
- 변수 참조 `x`: `env(x)`로 주소를 얻고 → `sto(addr)`로 값을 조회
- 변수 할당 `x = e`: `env(x)`로 주소를 얻고 → `sto(addr) = v`로 값을 변경

모든 변수가 암묵적으로 "박스"처럼 동작하게 됩니다.

### MFAE 인터프리터 (주요 변경 사항)

```scala
def interp(e: Expr, env: Env, sto: Sto): (Value, Sto) = e match {
  case Num(n) => (NumV(n), sto)

  // Id: 이제 간접 참조(indirection) 필요
  case Id(x) =>
    val addr = lookup(x, env)        // 1. 환경에서 주소 조회
    (storeLookup(addr, sto), sto)    // 2. Store에서 값 조회

  case Fun(x, b) => (CloV(x, b, env), sto)

  // App: 두 가지 경우로 나뉨
  case App(f, a) =>
    val (fv, fs) = interp(f, env, sto)
    fv match {
      case CloV(x, b, fenv) =>
        f match {
          // Case 1: f가 변수 이름인 경우 (call-by-reference 유사)
          case Id(name) =>
            val addr = lookup(name, env)     // 함수의 주소 재사용
            interp(b, fenv + (x -> addr), fs)

          // Case 2: f가 다른 표현식인 경우
          case _ =>
            val (av, as) = interp(a, env, fs)
            val addr = malloc(as)            // 새 주소 할당
            interp(b, fenv + (x -> addr), as + (addr -> av))
        }
      case v => error(s"not a closure: $v")
    }

  // Set: 변수에 새 값 할당
  case Set(x, e) =>
    val (v, s) = interp(e, env, sto)   // 새 값 계산
    val addr = lookup(x, env)          // 변수의 주소 조회
    (v, s + (addr -> v))               // Store 업데이트 (덮어쓰기)
}
```

### 왜 App에서 두 가지 경우를 구분하나?

```scala
// Case 1: f가 Id(name)인 경우
val inc = { x => { x = x + 1; x } }
inc(y)  // y의 주소를 그대로 전달 → y가 수정됨 (call-by-reference)

// Case 2: f가 복잡한 표현식인 경우
({ x => x + 1 })(y)  // y의 값을 복사하여 새 주소에 할당 (call-by-value)
```

- `Id(name)`인 경우: 변수의 주소를 **재사용**하여 call-by-reference처럼 동작
- 그 외의 경우: 새 주소를 **할당**하여 call-by-value처럼 동작

### MFAE 예제

```scala
// { val x = 10; { x = 20; x } }
val program = App(
  Fun("x",
    Seqn(
      Set("x", Num(20)),
      Id("x")
    )
  ),
  Num(10)
)

interp(program, Map.empty, Map.empty)
// => (NumV(20), Map(0 -> NumV(20)))
```

실행 과정:
1. `Num(10)` 계산 → `NumV(10)`
2. 새 주소 0 할당, `Map(0 -> NumV(10))` 생성
3. `x`를 주소 0에 바인딩: `env = Map("x" -> 0)`
4. `Set("x", Num(20))`: `env("x") = 0` → `sto(0) = NumV(20)`
5. `Id("x")`: `env("x") = 0` → `sto(0) = NumV(20)` 반환

---

## BFAE vs MFAE 비교

| 특성 | BFAE | MFAE |
|------|------|------|
| Environment 타입 | `Map[String, Value]` | `Map[String, Addr]` |
| 가변 상태 방식 | 명시적 Box (`NewBox`, `SetBox`, `OpenBox`) | 모든 변수가 암묵적으로 가변 |
| 변수 참조 | 직접 값 조회 | 주소를 통한 간접 조회 |
| 변수 할당 | 없음 (Box만 수정 가능) | `Set(x, e)` 연산 |
| 실제 언어 유사성 | Rust의 `Box<T>`, ML의 `ref` | Python, JavaScript, Java의 변수 |

**개념적 차이:**
- BFAE: 기본적으로 불변, 필요할 때만 Box로 가변성 도입
- MFAE: 모든 변수가 가변적 (주소를 통한 간접 참조)

---

## Store-Passing Style의 의미

두 언어 모두 **store-passing style**을 사용합니다:

```scala
def interp(e: Expr, env: Env, sto: Sto): (Value, Sto)
```

이는 **명령형 언어의 상태 변화를 함수형 스타일로 표현**하는 기법입니다:
- Store를 명시적으로 전달하고 반환
- 부수 효과(side effect)를 타입으로 명확하게 표현
- 평가 순서가 Store의 threading 순서로 명확히 결정됨

이 방식의 장점:
1. **순수 함수(pure function)** 유지: 전역 변수 없이 모든 상태를 매개변수로 전달
2. **타입 안전성**: Store 변경이 타입에 명시됨
3. **명확한 평가 순서**: threading 순서가 곧 실행 순서

---

## 수학적 표기법

위 구현은 다음과 같은 수학적 의미론에 대응됩니다:

**BFAE/MFAE 구문:**

```text
e ::= n
    | e + e
    | e - e
    | x
    | λx. e
    | e e
    | ref e      // NewBox
    | e := e     // SetBox or Set
    | !e         // OpenBox
    | e; e       // Seqn

v ::= n
    | < λx. e, σ >  // 클로저 (σ는 환경)
```

**의미 규칙 (Store-passing semantics):**

```text
⟦n⟧(σ, μ) = (n, μ)
⟦x⟧(σ, μ) = (μ(σ(x)), μ)        // MFAE: 간접 참조
⟦ref e⟧(σ, μ) = (a, μ[a ↦ v])   where ⟦e⟧(σ, μ) = (v, μ'), a는 새 주소
⟦!e⟧(σ, μ) = (μ(a), μ')         where ⟦e⟧(σ, μ) = (a, μ')
⟦e₁ := e₂⟧(σ, μ) = (v, μ''[a ↦ v])
                                where ⟦e₁⟧(σ, μ) = (a, μ'),
                                      ⟦e₂⟧(σ, μ') = (v, μ'')
```

여기서:
- `σ` (sigma): Environment
- `μ` (mu): Store
- `a`: Address
- `⟦e⟧(σ, μ) = (v, μ')`: 환경 σ와 Store μ에서 e를 평가하면 값 v와 새 Store μ'

---

## 마치며

이번 글에서는 가변 상태를 지원하는 두 가지 언어를 살펴봤습니다:

1. **BFAE**: 명시적인 Box를 통해 힙 메모리를 모델링
   - `NewBox`, `SetBox`, `OpenBox` 연산
   - Environment는 여전히 `Map[String, Value]`
   - Box만 가변적, 나머지는 불변

2. **MFAE**: 모든 변수가 암묵적으로 가변적
   - Environment를 `Map[String, Addr]`로 변경
   - 모든 변수 참조가 주소를 통한 간접 참조
   - `Set` 연산으로 변수 값 변경

핵심 개념:
- **Store**: 주소를 값에 매핑하는 힙 메모리 모델
- **Store-passing style**: Store를 명시적으로 전달/반환하는 함수형 패턴
- **Indirection**: 값 대신 주소를 사용하여 가변성 구현

다음 글에서는 재귀 함수(recursion)와 게으른 평가(lazy evaluation) 등 더 고급 기능을 다룰 예정입니다.
