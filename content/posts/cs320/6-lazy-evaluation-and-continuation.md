---
title: "[CS320] 6. 지연 평가와 Continuation"
date: 2023-06-11
draft: false
tags: ["CS320", "Programming Languages", "KAIST", "LFAE", "Lazy Evaluation", "Continuation", "CPS"]
categories: ["Computer Science"]
series: ["CS320 프로그래밍언어"]
summary: "지연 평가(Lazy Evaluation)를 지원하는 LFAE와 Continuation Passing Style(CPS)의 개념을 통해 프로그래밍 언어의 제어 흐름을 이해합니다."
aliases: ["/posts/cs320-6-lazy-evaluation-and-continuation/"]
ShowToc: true
TocOpen: true
---

> KAIST CS320 프로그래밍언어 (Spring 2023)
> 교재: 수업 자료 기반

---

## 1. LFAE: Lazy FAE

LFAE(Lazy FAE)는 FAE에 지연 평가(Lazy Evaluation)를 도입한 언어입니다. 기존 FAE는 함수 호출 시 인자를 즉시 평가하는 Call-by-Value 방식을 사용했지만, LFAE는 인자를 필요한 시점까지 평가를 미루는 방식을 사용합니다.

### 1.1. 구문 정의

**구체적 구문 (Concrete Syntax):**

```text
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
       | id
       | "{" id "=>" expr "}"
       | expr "(" expr ")"
```

구문 자체는 FAE와 동일하지만, 의미론(semantics)이 다릅니다.

**추상적 구문 (Abstract Syntax):**

```scala
trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
case class Id(x: String) extends Expr
case class Fun(x: String, b: Expr) extends Expr
case class App(f: Expr, a: Expr) extends Expr
```

### 1.2. 값의 표현

LFAE의 핵심은 값을 표현하는 방식에 있습니다. 기존 FAE에서는 `NumV`와 `CloV`만 있었지만, LFAE는 **평가되지 않은 표현식**을 나타내는 `ExprV`를 추가합니다.

```scala
type Env = Map[String, Value]

trait Value
case class NumV(n: Int) extends Value
case class CloV(x: String, b: Expr, fenv: Env) extends Value
case class ExprV(e: Expr, env: Env, var v: Option[Value]) extends Value
```

`ExprV`의 구성 요소:
- `e: Expr`: 평가되지 않은 표현식
- `env: Env`: 표현식을 평가할 때 사용할 환경
- `v: Option[Value]`: 평가 결과를 저장하는 캐시 (mutable)

### 1.3. 인터프리터 구현

```scala
def interp(e: Expr, env: Env): Value = e match {
  case Num(n) => NumV(n)
  case Add(l, r) => numVAdd(interp(l, env), interp(r, env))
  case Sub(l, r) => numVSub(interp(l, env), interp(r, env))
  case Id(x) => lookup(x, env)
  case Fun(x, b) => CloV(x, b, env)  // 클로저 생성 시 환경을 캡처
  case App(f, a) =>
    val fv = strict(interp(f, env))
    val av = ExprV(a, env, None)  // 인자를 평가하지 않고 ExprV로 감싼다!
    fv match {
      case CloV(x, b, fenv) =>
        interp(b, fenv + (x -> av))
      case v => error(s"not a closure: $v")
    }
}
```

핵심 변화는 `App` 케이스입니다:
- 함수 `f`는 `strict`를 통해 강제로 평가됩니다
- **인자 `a`는 즉시 평가하지 않고**, `ExprV(a, env, None)`으로 감싸서 환경에 바인딩합니다
- 이는 "나중에 필요하면 평가하겠다"는 의미입니다

### 1.4. strict 함수: 지연된 평가 강제하기

```scala
def strict(v: Value): Value = v match {
  case ev@ExprV(e, env, r) => r match {
    case Some(cache) => cache  // 이미 평가된 값이 있으면 캐시 반환
    case _ =>
      val cache = strict(interp(e, env))
      ev.v = Some(cache)       // 평가 결과를 캐시에 저장
      cache
  }
  case _ => v
}
```

`strict` 함수의 역할:
1. `ExprV`를 만나면 실제로 평가를 수행합니다
2. 평가 결과를 캐시에 저장합니다 (Call-by-Need)
3. 다음에 같은 값이 필요하면 재평가하지 않고 캐시를 반환합니다

## 2. 평가 전략 비교

프로그래밍 언어는 함수 인자를 평가하는 방식에 따라 다양한 전략을 사용합니다.

### 2.1. Call-by-Value (값에 의한 호출)

```scala
// FAE의 방식
case App(f, a) =>
  val fv = interp(f, env)
  val av = interp(a, env)  // 인자를 즉시 평가
  // ...
```

- 함수 호출 시 인자를 **즉시 평가**합니다
- 대부분의 프로그래밍 언어(C, Java, Python 등)가 사용하는 방식
- 장점: 예측 가능하고 구현이 단순함
- 단점: 사용되지 않는 인자도 평가하므로 비효율적일 수 있음

### 2.2. Call-by-Name (이름에 의한 호출)

```scala
case App(f, a) =>
  val fv = strict(interp(f, env))
  val av = ExprV(a, env, None)  // 평가를 미룸
  // 사용할 때마다 매번 평가
```

- 인자를 평가하지 않고 **표현식 자체**를 전달합니다
- 사용할 때마다 평가를 수행합니다
- 장점: 사용되지 않는 인자는 평가하지 않음
- 단점: 같은 인자를 여러 번 사용하면 중복 평가가 발생

### 2.3. Call-by-Need (필요에 의한 호출)

```scala
def strict(v: Value): Value = v match {
  case ev@ExprV(e, env, r) => r match {
    case Some(cache) => cache  // 캐시된 값 사용
    case _ =>
      val cache = strict(interp(e, env))
      ev.v = Some(cache)       // 첫 평가 결과를 캐시
      cache
  }
  case _ => v
}
```

- Call-by-Name + **메모이제이션(캐싱)**
- 처음 사용할 때 평가하고, 결과를 저장합니다
- 같은 인자를 다시 사용하면 캐시된 값을 반환합니다
- Haskell이 사용하는 기본 평가 전략
- 장점: 필요 없는 계산을 피하면서도 중복 평가를 방지
- LFAE가 구현하는 방식입니다

### 2.4. 예제로 비교하기

다음 프로그램을 생각해봅시다:

```scala
{x => x + x}(expensive_computation())
```

- **Call-by-Value**: `expensive_computation()`을 한 번 평가하고, 결과를 `x`에 바인딩
- **Call-by-Name**: `x + x`를 평가할 때 `expensive_computation()`을 두 번 호출
- **Call-by-Need**: 첫 번째 `x`를 평가할 때 `expensive_computation()`을 실행하고 캐시, 두 번째 `x`는 캐시 사용

## 3. Continuation: 함수형 언어의 제어 흐름

### 3.1. 명령형 언어 vs 함수형 언어

명령형 언어(Imperative Language)는 다양한 제어 구조를 제공합니다:
- `return`: 함수를 즉시 종료하고 값을 반환
- `break`: 반복문을 종료
- `continue`: 반복문의 다음 반복으로 이동
- `goto`, 예외 처리 등

함수형 언어(Functional Language)는 이러한 명령문이 없습니다. 대신 **Continuation**을 사용하여 제어 흐름을 표현합니다.

### 3.2. Continuation의 개념

**Continuation**은 "프로그램의 나머지 계산"을 의미합니다. 즉, "지금부터 무엇을 해야 하는가"를 함수로 표현한 것입니다.

일반적인 프로그래밍에서는:
1. 함수를 호출하면
2. Caller가 Callee의 결과를 **기다리고**
3. 반환된 값을 받아서
4. 나머지 계산을 수행합니다

Continuation을 사용하면:
1. 함수를 호출할 때 "결과를 받으면 무엇을 할지"를 함께 전달합니다
2. Callee는 계산을 마치면 그 continuation을 호출합니다
3. Caller는 기다릴 필요가 없습니다

## 4. Continuation Passing Style (CPS)

### 4.1. 일반적인 방식 vs CPS

덧셈을 해석하는 두 가지 방식을 비교해봅시다.

**방법 1: 일반적인 방식**

```scala
case Add(l, r) =>
  val lv = interp(l, env);
  val rv = interp(r, env);
  numVAdd(lv, rv)
```

동작 과정:
1. `interp(l, env)`을 호출하고 결과를 기다림
2. 반환된 값 `lv`를 받음
3. `interp(r, env)`을 호출하고 결과를 기다림
4. 반환된 값 `rv`를 받음
5. `lv`와 `rv`를 더해서 결과를 반환

**방법 2: Continuation Passing Style**

```scala
case Add(l, r) =>
  interp(l, env, lv =>
    interp(r, env, rv =>
      k(numVAdd(lv, rv))))
```

동작 과정:
1. `interp(l, env, ...)`을 호출하면서 "l의 값 `lv`를 얻으면 할 일"을 continuation으로 전달
2. 그 continuation 안에서 `interp(r, env, ...)`을 호출하면서 "r의 값 `rv`를 얻으면 할 일"을 전달
3. 최종 continuation `k`에 덧셈 결과를 전달

**한 줄로 모든 작업이 표현됩니다!**

### 4.2. CPS의 특징

```scala
// 일반적인 interp
def interp(e: Expr, env: Env): Value = ...

// CPS 스타일의 interp
def interp(e: Expr, env: Env, k: Value => Result): Result = ...
```

CPS에서는:
- 함수가 값을 직접 반환하지 않습니다
- 대신 continuation `k`를 받아서, 결과를 `k`에 전달합니다
- 모든 함수 호출이 "tail position"에 위치합니다 (꼬리 호출 최적화 가능)

### 4.3. 비유적인 이해

**일반적인 방식:**

> Caller: "Callee야, 이 값 계산해줘."
>
> (기다림...)
>
> Callee: "여기 결과야."
>
> Caller: "고마워! 이제 이 값 계산해줘."
>
> (기다림...)
>
> Callee: "여기 결과야."
>
> Caller: "좋아! 이제 이 둘을 더해서 반환하면 되겠군."

**Continuation Passing Style:**

> Caller: "Callee야, 이 값 계산하고, 그 결과로 저 값 계산하고, 그 둘을 더해서 내 continuation에 전달해줘! 내가 해야 할 일을 모두 설명했으니 잘 처리해줘."

### 4.4. CPS의 강력함

Continuation을 명시적으로 다루면:
- **제어 흐름을 값처럼 조작**할 수 있습니다
- `return`, `break`, `continue` 같은 제어 구조를 함수로 표현할 수 있습니다
- 예외 처리, 코루틴, 비동기 프로그래밍 등을 통합적으로 표현할 수 있습니다

예를 들어, continuation을 중간에 바꾸면:
```scala
// 정상 continuation
interp(e, env, v => doSomething(v))

// 예외 처리를 위한 continuation
interp(e, env, v => if (isError(v)) handleError(v) else doSomething(v))
```

Continuation을 저장했다가 나중에 호출하면 "그 시점으로 돌아가는" 효과를 낼 수 있습니다.

## 5. LFAE에서의 strict 배치 전략

LFAE에서 `strict` 함수를 호출하는 위치에 따라 다양한 의미론을 구현할 수 있습니다.

### 5.1. 함수 적용 시점에 strict

```scala
case App(f, a) =>
  val fv = strict(interp(f, env))  // 함수는 반드시 평가
  val av = ExprV(a, env, None)     // 인자는 lazy
  // ...
```

인자는 lazy하게, 함수 자체는 즉시 평가합니다.

### 5.2. 함수 본체 평가 시점에 strict

```scala
case App(f, a) =>
  val fv = strict(interp(f, env))
  val av = ExprV(a, env, None)
  fv match {
    case CloV(x, b, fenv) =>
      strict(interp(b, fenv + (x -> av)))  // 결과를 strict하게
    // ...
  }
```

함수 본체의 평가 결과도 강제로 값으로 만듭니다.

### 5.3. 식별자 참조 시점에 strict

```scala
case Id(x) => strict(lookup(x, env))  // 변수 참조 시 평가
```

변수를 사용하는 시점에 평가를 강제합니다.

### 5.4. 산술 연산 시점에 strict

```scala
case Add(l, r) =>
  numVAdd(strict(interp(l, env)), strict(interp(r, env)))
```

덧셈을 수행하려면 양쪽 모두 숫자여야 하므로 strict하게 평가합니다.

이처럼 `strict`를 어디에 배치하느냐에 따라 언어의 평가 전략을 세밀하게 조정할 수 있습니다.

## 6. 정리

### LFAE의 핵심

1. **ExprV**: 평가되지 않은 표현식을 값처럼 다룹니다
2. **strict**: 필요한 시점에 평가를 강제하고 결과를 캐싱합니다
3. **Call-by-Need**: 지연 평가 + 메모이제이션으로 효율성과 유연성을 모두 확보합니다

### Continuation의 핵심

1. **제어 흐름의 명시화**: "다음에 할 일"을 함수로 표현합니다
2. **CPS 변환**: 모든 함수 호출이 tail position에 오도록 변환합니다
3. **강력한 추상화**: 제어 구조를 1급 값처럼 다룰 수 있습니다

지연 평가와 Continuation은 함수형 프로그래밍 언어의 핵심 개념으로, 프로그램의 평가 순서와 제어 흐름을 유연하게 다룰 수 있게 해줍니다. LFAE는 이러한 개념을 단순한 형태로 구현하여, 실제 언어(Haskell, Scheme 등)가 어떻게 동작하는지 이해하는 데 도움을 줍니다.
