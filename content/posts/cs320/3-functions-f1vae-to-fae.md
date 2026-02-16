---
title: "[CS320] 3. 함수의 세계 — F1VAE에서 FAE까지"
date: 2023-06-05
draft: false
tags: ["CS320", "Programming Languages", "KAIST", "F1VAE", "FVAE", "FAE", "Closure", "Scope"]
categories: ["Computer Science"]
series: ["CS320 프로그래밍언어"]
summary: "일급이 아닌 함수(F1VAE)에서 일급 함수와 클로저(FVAE, FAE)까지 발전하는 과정과 Static/Dynamic Scope의 차이를 살펴봅니다."
aliases: ["/posts/cs320-3-functions-f1vae-to-fae/"]
ShowToc: true
TocOpen: true
---

> KAIST CS320 프로그래밍언어 (Spring 2023)
> 교재: 수업 자료 기반

---

## 들어가며

프로그래밍 언어의 핵심 추상화 도구는 바로 **함수(function)**입니다. 하지만 함수를 어떻게 다루는지에 따라 언어의 표현력이 크게 달라집니다. 이번 글에서는 함수가 **일급 값(first-class value)**이 아닌 상태에서 시작하여, 점차 일급 함수와 **클로저(closure)**를 갖춘 언어로 발전하는 과정을 살펴봅니다.

우리는 다음 세 가지 언어를 순서대로 살펴볼 것입니다:

1. **F1VAE (First-order VAE)**: 함수가 일급이 아닌 언어
2. **FVAE (First-class VAE)**: 일급 함수를 도입한 언어
3. **FAE (Function AE)**: 간결화된 일급 함수 언어

그리고 마지막으로 **Static Scope**와 **Dynamic Scope**의 차이를 이해하고, 왜 대부분의 현대 언어가 Static Scope를 선택했는지 알아봅니다.

---

## F1VAE — 일급이 아닌 함수

F1VAE는 **일급이 아닌 함수(first-order function)**를 지원하는 언어입니다. 여기서 핵심은 **함수 정의(`fdef`)가 값(value)이 아니라는 점**입니다. 함수를 평가(evaluate)하지 않고, 함수로부터 값을 얻지도 않습니다. 함수는 단지 이름으로 호출할 수 있는 코드 조각일 뿐입니다.

### 문법

**구체적 문법 (Concrete Syntax, BNF):**

```text
fdef ::= "def" id "(" id ")" "=" expr
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
       | "{" "val" id "=" expr ";" expr "}"
       | id
       | id "(" expr ")"
```

여기서 `fdef`는 함수 정의(function definition)이고, `expr`은 표현식(expression)입니다. 중요한 점은 `fdef`가 `expr`에 포함되지 않는다는 것입니다. 즉, 함수 정의를 표현식처럼 평가할 수 없습니다.

**추상적 문법 (Abstract Syntax, Scala):**

```scala
case class Fdef(f: String, x: String, b: Expr)

trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
case class Val(x: String, i: Expr, b: Expr) extends Expr
case class Id(x: String) extends Expr
case class App(f: String, a: Expr) extends Expr

type Env = Map[String, Int]
type FDs = List[Fdef]
```

여기서 `Fdef`는 함수 정의를 나타내며, `Expr` 트레이트(trait)를 확장하지 않습니다. `App`은 함수 호출(application)을 나타내는데, 첫 번째 인자가 `String`(함수 이름)이라는 점에 주목하세요. 즉, 함수를 식으로 계산할 수 없고, 오직 이름으로만 호출할 수 있습니다.

### 인터프리터 의미론

```scala
def interp(e: Expr, env: Env, fs: FDs): Int = e match {
  case Num(n) => n
  case Add(l, r) => interp(l, env, fs) + interp(r, env, fs)
  case Sub(l, r) => interp(l, env, fs) - interp(r, env, fs)
  case Val(x, i, b) =>
    interp(b, env + (x -> interp(i, env, fs)), fs)
  case Id(x) => lookup(x, env)
  case App(f, a) => lookupFD(f, fs) match {
    case Fdef(fname, pname, body) =>
      val aval = interp(a, env, fs)
      interp(body, Map() + (pname -> aval), fs)
  }
}
```

### 핵심 특징

1. **함수 정의는 변하지 않는다**: `fs: FDs`는 인터프리터 실행 중에 변하지 않습니다. 처음 주어진 함수 정의 리스트가 그대로 유지됩니다.

2. **함수 환경은 항상 비어있다**: `Fdef`는 `Expr`이 아니므로, 함수 환경(fenv)은 항상 비어있습니다. `App` 케이스를 보면 `interp(body, Map() + (pname -> aval), fs)`처럼 새로운 빈 환경(`Map()`)에서 시작합니다.

3. **Static Scope**: 함수 본체(body)를 평가할 때 호출 지점의 환경(`env`)을 버리고 빈 환경에서 시작합니다. 이는 Static Scope의 초기 형태로 볼 수 있습니다.

### 예제

```text
def square(x) = x * x

{
  val y = 3;
  square(y + 2)
}
```

이 코드는 다음과 같이 동작합니다:
- `square` 함수가 `fs`에 저장됩니다.
- `y = 3`이 환경에 추가됩니다.
- `square(5)`를 호출할 때, 새로운 빈 환경에 `x = 5`만 추가하여 `x * x`를 평가합니다.

### 한계

F1VAE의 가장 큰 한계는 **함수를 값으로 다룰 수 없다**는 점입니다. 함수를 다른 함수의 인자로 전달하거나, 함수가 함수를 반환하거나, 변수에 함수를 저장하는 것이 불가능합니다. 이는 고차 함수(higher-order function)와 함수형 프로그래밍의 핵심 패턴을 사용할 수 없다는 의미입니다.

---

## FVAE — 일급 함수의 도입

FVAE는 **일급 함수(first-class function)**를 지원하는 언어입니다. 여기서 함수는 다른 값들과 동등하게 취급됩니다. 함수를 생성하고, 전달하고, 반환할 수 있습니다.

### 문법

**구체적 문법 (BNF):**

```text
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
       | "{" "val" id "=" expr ";" expr "}"
       | id
       | "{" id "=>" expr "}"
       | expr "(" expr ")"
```

주목할 변화:
- `{id => expr}`: **람다 표현식(lambda expression)**이 추가되었습니다.
- 함수 호출에서 `id(expr)` 대신 `expr(expr)`을 사용합니다. 즉, 함수 위치에 임의의 표현식이 올 수 있습니다.
- 함수 정의(`fdef`)가 사라지고, 함수가 표현식의 일부가 되었습니다.

**추상적 문법 (Scala):**

```scala
trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
case class Val(x: String, i: Expr, b: Expr) extends Expr
case class Id(x: String) extends Expr
case class Fun(x: String, b: Expr) extends Expr
case class App(f: Expr, a: Expr) extends Expr

type Env = Map[String, Value]

trait Value
case class NumV(n: Int) extends Value
case class CloV(x: String, b: Expr, fenv: Env) extends Value
```

### 클로저(Closure)의 등장

가장 중요한 변화는 **`CloV` (Closure Value)**의 등장입니다. 클로저는 다음 세 가지 요소로 구성됩니다:

1. **매개변수 이름 (`x: String`)**: 함수가 받을 인자의 이름
2. **함수 본체 (`b: Expr`)**: 함수가 실행할 표현식
3. **포획된 환경 (`fenv: Env`)**: **함수가 정의될 당시의 환경**

이 세 번째 요소가 핵심입니다. 클로저는 자신이 정의된 시점의 환경을 "기억"합니다.

### 인터프리터 의미론

```scala
def interp(e: Expr, env: Env): Value = e match {
  case Num(n) => NumV(n)
  case Add(l, r) => numVAdd(interp(l, env), interp(r, env))
  case Sub(l, r) => numVSub(interp(l, env), interp(r, env))
  case Val(x, i, b) =>
    val ival = interp(i, env)
    interp(b, env + (x -> ival))
  case Id(x) => lookup(x, env)
  case Fun(x, b) => CloV(x, b, env)  // 핵심: 현재 환경을 포획!
  case App(f, a) =>
    interp(f, env) match {
      case CloV(x, b, fenv) =>
        val aval = interp(a, env)
        interp(b, fenv + (x -> aval))  // fenv 사용 (Static Scope)
    }
}
```

### 핵심 포인트

1. **`Fun(x, b)`의 평가**: 함수를 만날 때 **본체를 평가하지 않습니다**. 대신 현재 환경(`env`)을 포획하여 `CloV(x, b, env)`를 생성합니다.

2. **`App(f, a)`의 평가**:
   - 먼저 `f`를 평가하여 클로저를 얻습니다.
   - 인자 `a`를 현재 환경(`env`)에서 평가합니다.
   - 함수 본체 `b`를 **포획된 환경 `fenv`**에 인자 바인딩을 추가하여 평가합니다.

### 예제: 클로저의 위력

```scala
{
  val x = 10;
  val f = { y => x + y };
  {
    val x = 20;
    f(5)
  }
}
```

이 코드의 실행 과정:
1. `x = 10`을 환경에 추가: `env1 = {x -> 10}`
2. 함수 `{ y => x + y }`를 평가하여 `CloV("y", Add(Id("x"), Id("y")), env1)` 생성
3. 내부 스코프에서 `x = 20`을 추가: `env2 = {x -> 20, f -> CloV(...)}`
4. `f(5)` 호출 시, 클로저의 `fenv = {x -> 10}`에 `y -> 5`를 추가하여 평가
5. 결과: `10 + 5 = 15`

내부 스코프의 `x = 20`이 아니라 **함수 정의 시점의 `x = 10`**을 사용합니다. 이것이 바로 클로저와 Static Scope의 핵심입니다.

---

## FAE — 간결한 일급 함수

FAE는 FVAE를 간결하게 만든 버전입니다. `Val` 표현식을 제거하고 순수하게 함수와 산술 연산만 남겼습니다.

### 문법

**구체적 문법 (BNF):**

```text
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
       | id
       | "{" id "=>" expr "}"
       | expr "(" expr ")"
```

**추상적 문법 (Scala):**

```scala
trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
case class Id(x: String) extends Expr
case class Fun(x: String, b: Expr) extends Expr
case class App(f: Expr, a: Expr) extends Expr

type Env = Map[String, Value]

trait Value
case class NumV(n: Int) extends Value
case class CloV(x: String, b: Expr, fenv: Env) extends Value
```

### 인터프리터

```scala
def interp(e: Expr, env: Env): Value = e match {
  case Num(n) => NumV(n)
  case Add(l, r) => numVAdd(interp(l, env), interp(r, env))
  case Sub(l, r) => numVSub(interp(l, env), interp(r, env))
  case Id(x) => lookup(x, env)
  case Fun(x, b) => CloV(x, b, env)
  case App(f, a) => interp(f, env) match {
    case CloV(x, b, fenv) =>
      val aval = interp(a, env)
      interp(b, fenv + (x -> aval))
    case v => error(s"not a closure: $v")
  }
}
```

FVAE와 거의 동일하지만, `Val` 케이스가 없어 더 간결합니다. 하지만 `Val`이 없어도 람다 표현식의 즉시 호출(immediately invoked function expression)로 동일한 효과를 낼 수 있습니다:

```scala
// Val(x, 10, Add(Id(x), 5))와 동일
App(Fun("x", Add(Id("x"), Num(5))), Num(10))
```

### 수학적 표기법

FAE는 수학적으로 다음과 같이 표기할 수 있습니다:

```text
e ::= n
    | e + e
    | e - e
    | x
    | λx. e
    | e e

v ::= n
    | < λ x. e, σ >
```

여기서:
- `λx. e`는 람다 표현식 (함수)
- `e e`는 함수 적용 (application)
- `< λ x. e, σ >`는 클로저 (람다 표현식과 환경 σ의 쌍)

이 표기법은 람다 대수(lambda calculus)의 표준 표기법과 유사하며, 프로그래밍 언어 이론에서 널리 사용됩니다.

---

## Scope — Static vs Dynamic

이제 프로그래밍 언어 설계에서 가장 중요한 선택 중 하나인 **Scope** 문제를 살펴봅시다.

### 핵심 질문

**함수를 호출할 때, 어떤 환경을 사용하여 함수 본체를 평가할 것인가?**

두 가지 선택지가 있습니다:

1. **함수가 정의될 당시의 환경** (Static Scope)
2. **함수가 호출될 당시의 환경** (Dynamic Scope)

### Static Scope (Lexical Scope)

**정의**: 함수가 정의된 시점의 환경을 사용합니다.

**구현** (FVAE/FAE):
```scala
case Fun(x, b) => CloV(x, b, env)  // 정의 시점 환경 포획

case App(f, a) => interp(f, env) match {
  case CloV(x, b, fenv) =>
    val aval = interp(a, env)
    interp(b, fenv + (x -> aval))  // fenv 사용!
}
```

**예제**:
```scala
{
  val x = 1;
  val f = { y => x + y };
  {
    val x = 2;
    f(10)
  }
}
```

Static Scope에서:
1. `f`는 `x = 1`인 환경에서 정의됨
2. `f(10)` 호출 시 `fenv = {x -> 1}`에 `y -> 10` 추가
3. 결과: `1 + 10 = 11`

### Dynamic Scope

**정의**: 함수가 호출된 시점의 환경을 사용합니다.

**구현** (만약 Dynamic Scope를 사용한다면):
```scala
case App(f, a) => interp(f, env) match {
  case CloV(x, b, fenv) =>
    val aval = interp(a, env)
    interp(b, env + (x -> aval))  // env 사용! (fenv 무시)
}
```

**같은 예제**:
```scala
{
  val x = 1;
  val f = { y => x + y };
  {
    val x = 2;
    f(10)
  }
}
```

Dynamic Scope에서:
1. `f`는 생성되지만 환경은 중요하지 않음
2. `f(10)` 호출 시 **현재 환경 `{x -> 2, f -> ...}`**에 `y -> 10` 추가
3. 결과: `2 + 10 = 12`

### 왜 Static Scope가 더 나은가?

#### 1. 컴파일 타임 에러 검출

**Static Scope**에서는 자유 변수(free identifier) 에러를 컴파일 타임에 발견할 수 있습니다:

```scala
val f = { y => x + y }  // 에러! x가 정의되지 않음
```

함수 정의 시점에 `x`가 없으므로 즉시 에러를 검출합니다.

**Dynamic Scope**에서는 같은 코드가 컴파일됩니다:

```scala
val f = { y => x + y }  // OK (아직 에러 아님)
// ...
val x = 10
f(5)  // OK, 15 반환
// ...
val x = 20
f(5)  // OK, 25 반환
// ...
f(5)  // 런타임 에러! x가 스코프에 없음
```

에러가 런타임까지 미뤄지고, 함수의 동작이 호출 위치에 따라 달라져 예측하기 어렵습니다.

#### 2. 모듈성과 추론 용이성

Static Scope에서는 함수의 동작을 **정의 위치만 보고** 추론할 수 있습니다:

```scala
def makeAdder(x: Int): Int => Int = {
  { y => x + y }
}

val add5 = makeAdder(5)
// add5는 항상 인자에 5를 더함 (어디서 호출하든)
```

Dynamic Scope에서는 함수의 동작을 알기 위해 **모든 호출 위치**를 추적해야 합니다.

#### 3. 최적화 가능성

Static Scope는 컴파일러가 변수 바인딩을 컴파일 타임에 결정할 수 있어 최적화가 쉽습니다. Dynamic Scope는 런타임에 환경을 탐색해야 하므로 성능 오버헤드가 있습니다.

### 현대 언어의 선택

거의 모든 현대 프로그래밍 언어(JavaScript, Python, Java, Scala, Haskell, Rust 등)는 **Static Scope**를 사용합니다. Dynamic Scope는 일부 오래된 Lisp 방언과 Emacs Lisp에서만 볼 수 있습니다.

---

## 마치며

이번 글에서는 함수가 일급 값이 아닌 F1VAE에서 시작하여, 클로저를 갖춘 일급 함수(FVAE, FAE)로 발전하는 과정을 살펴보았습니다. 핵심 내용을 정리하면:

1. **F1VAE**: 함수를 이름으로만 호출할 수 있고, 값으로 다룰 수 없음
2. **FVAE/FAE**: 함수가 일급 값이 되고, 클로저를 통해 환경을 포획
3. **Closure**: 함수 + 정의 시점 환경의 조합
4. **Static Scope**: 함수 정의 시점의 환경 사용 (대부분의 언어)
5. **Dynamic Scope**: 함수 호출 시점의 환경 사용 (레거시)

클로저와 Static Scope의 조합은 현대 프로그래밍 언어의 표현력과 안정성을 크게 향상시켰습니다. 이를 통해 고차 함수, 커링(currying), 부분 적용(partial application) 등의 강력한 함수형 프로그래밍 패턴을 안전하게 사용할 수 있게 되었습니다.

다음 글에서는 이러한 일급 함수를 활용한 더 복잡한 언어 기능들을 살펴볼 것입니다.
