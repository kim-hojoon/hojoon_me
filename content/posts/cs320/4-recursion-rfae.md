---
title: "[CS320] 4. 재귀와 조건 분기 — RFAE"
date: 2023-06-07
draft: false
tags: ["CS320", "Programming Languages", "KAIST", "RFAE", "Recursion", "mkRec"]
categories: ["Computer Science"]
series: ["CS320 프로그래밍언어"]
summary: "조건 분기(if0)와 재귀 함수(Rec)를 지원하는 RFAE 언어를 설계하고, mkRec 패턴과 var 키워드를 이용한 재귀 구현 기법을 살펴봅니다."
aliases: ["/posts/cs320-4-recursion-rfae/"]
ShowToc: true
TocOpen: true
---

> KAIST CS320 프로그래밍언어 (Spring 2023)
> 교재: 수업 자료 기반

---

지금까지 FAE(First-class functions with Arithmetic Expressions)를 통해 일급 함수(first-class functions)를 다루는 방법을 배웠습니다. 이번 글에서는 FAE를 확장하여 **조건 분기(conditional branching)**와 **재귀 함수(recursive functions)**를 지원하는 RFAE(Recursive FAE)를 살펴봅니다.

재귀 함수는 현대 프로그래밍 언어의 필수 요소입니다. 하지만 재귀를 구현하는 방법은 생각보다 까다롭습니다. 함수가 자기 자신을 참조해야 하는데, 함수를 정의하는 시점에는 아직 그 함수가 존재하지 않기 때문입니다. RFAE는 이 문제를 **순환 참조(circular reference)**를 통해 해결합니다.

## 1. RFAE의 문법

RFAE는 FAE에 두 가지 기능을 추가합니다:

1. **if0 표현식**: 조건 분기를 위한 구문
2. **재귀 함수 정의**: `def` 키워드를 이용한 재귀 함수 선언

### 구체 문법 (Concrete Syntax)

```text
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
       | "(" expr "*" expr ")"
       | id
       | "{" id "=>" expr "}"
       | expr "(" expr ")"
       | "if0" "(" expr ")" "{" expr "}" "else" "{" expr "}"
       | "{" "def" id "(" id ")" "=" expr ";" expr "}"
```

마지막 두 줄이 새로 추가된 부분입니다:
- `if0 (조건) {참} else {거짓}`: 조건이 0이면 참 블록 실행, 아니면 거짓 블록 실행
- `{ def f(x) = body; expr }`: 재귀 함수 f를 정의하고, expr에서 f를 사용

### 추상 문법 (Abstract Syntax)

Scala로 표현한 추상 문법 트리(AST) 정의입니다:

```scala
trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
case class Id(x: String) extends Expr
case class Fun(x: String, b: Expr) extends Expr
case class App(f: Expr, a: Expr) extends Expr
case class If0(c: Expr, t: Expr, f: Expr) extends Expr
case class Rec(f: String, x: String, b: Expr, e: Expr) extends Expr
```

`If0`와 `Rec` 두 케이스 클래스가 추가되었습니다.

### 값(Value)과 환경(Environment)

```scala
type Env = Map[String, Value]

trait Value
case class NumV(n: Int) extends Value
case class CloV(x: String, b: Expr, var fenv: Env) extends Value
```

여기서 **주목할 점**은 `CloV`의 `fenv` 필드가 `var`로 선언되어 있다는 것입니다. 이전 FAE에서는 `val`이었습니다. 이 `var` 키워드가 재귀 구현의 핵심입니다!

## 2. if0: 조건 분기

먼저 비교적 간단한 `if0`부터 살펴봅시다.

```scala
case If0(c, t, f) => interp(c, env) match {
  case NumV(0) => interp(t, env)
  case _ => interp(f, env)
}
```

동작 방식은 직관적입니다:
1. 조건식 `c`를 평가합니다
2. 결과가 `NumV(0)`이면 참 분기 `t`를 평가
3. 그 외의 값이면 거짓 분기 `f`를 평가

예를 들어 다음 코드는:

```text
if0 (1 - 1) {
  42
} else {
  0
}
```

`1 - 1`이 `0`이므로 `42`를 반환합니다.

## 3. 재귀 함수의 문제점

재귀 함수를 구현하려면 **순환 참조(circular reference)** 문제를 해결해야 합니다. 다음 factorial 함수를 생각해봅시다:

```text
{
  def factorial(n) =
    if0 (n) {
      1
    } else {
      n * factorial(n - 1)
    };
  factorial(5)
}
```

`factorial` 함수의 본문에서 `factorial`을 다시 호출합니다. 하지만 함수를 정의하는 시점에는 아직 `factorial`이라는 이름이 환경(environment)에 바인딩되지 않았습니다. 어떻게 해야 할까요?

## 4. 재귀 구현 방법 1: var 키워드

RFAE는 Scala의 **가변 참조(mutable reference)**를 이용하여 이 문제를 해결합니다.

```scala
case Rec(f, x, b, e) =>
  val cloV = CloV(x, b, ???)        // 1. 클로저를 먼저 만듭니다 (환경은 아직 미정)
  val nenv = env + (f -> cloV)      // 2. 새 환경을 만들고 f를 cloV에 바인딩
  cloV.fenv = nenv                  // 3. 클로저의 환경을 업데이트 (순환 참조 완성!)
  interp(e, nenv)                   // 4. 새 환경에서 본문 expr 실행
```

### 단계별 실행 과정

factorial 예제로 단계를 따라가 봅시다:

**Step 1: 클로저 생성**
```scala
val cloV = CloV("n", <if0 body>, ???)
```

일단 클로저 객체를 만들지만, `fenv`는 아직 미정입니다.

**Step 2: 새 환경 생성**
```scala
val nenv = env + ("factorial" -> cloV)
```

현재 환경에 `"factorial" -> cloV` 바인딩을 추가한 새 환경을 만듭니다.

**Step 3: 순환 참조 완성**
```scala
cloV.fenv = nenv
```

여기가 마법이 일어나는 순간입니다! 클로저의 환경을 `nenv`로 설정하면:
- `cloV`는 `nenv`를 참조하고
- `nenv`는 `"factorial" -> cloV` 바인딩을 포함하므로 다시 `cloV`를 참조합니다

이제 `factorial` 함수 본문에서 `factorial`을 조회하면 자기 자신을 찾을 수 있습니다!

**Step 4: 본문 실행**
```scala
interp(e, nenv)  // e는 factorial(5)
```

새로운 환경 `nenv`에서 `factorial(5)`를 실행합니다.

### 왜 var가 필요한가?

Scala의 `val`은 불변(immutable) 참조입니다. 한번 할당하면 변경할 수 없습니다. 하지만 순환 참조를 만들려면:
1. 먼저 객체를 만들고
2. 그 객체를 포함하는 환경을 만든 후
3. 다시 객체가 그 환경을 참조하도록 업데이트

해야 합니다. 3번 단계에서 **업데이트**가 필요하므로 `var`를 사용해야 합니다.

## 5. 재귀 구현 방법 2: mkRec 패턴

재귀를 언어 차원에서 지원하지 않더라도, 파싱과 desugaring을 통해 재귀를 구현할 수 있습니다. 이것이 **mkRec** 패턴입니다.

### FVAE에서 재귀 흉내내기

다음 RFAE 코드:

```text
{
  def id1(id2) = expr1;
  expr2
}
```

를 FVAE 코드로 변환할 수 있습니다:

```scala
{
  val id1 = mkRec({ id1 => { id2 => expr1 } });
  expr2
}
```

또는 더 직접적으로:

```scala
{ id1 => expr2 }( mkRec({ id1 => { id2 => expr1 } }) )
```

### mkRec의 구현

`mkRec`은 일종의 **불동점 조합자(fixed-point combinator)**입니다. Y 조합자(Y combinator)의 변형으로 볼 수 있습니다:

```scala
val mkRec = { bodyProc =>
  { val fX = { fY =>
      { val f = { x => fY(fY)(x) };
        bodyProc(f) }
    };
    fX(fX)
  }
};
```

이 코드가 어떻게 작동하는지 이해하기는 쉽지 않습니다. 핵심 아이디어는:
- `fY(fY)`를 통해 자기 자신을 복제하면서 호출
- `bodyProc(f)`를 통해 재귀 함수 본문에 "자기 자신"을 주입

### mkRec vs var: 어떤 방법이 더 나은가?

| 방법 | 장점 | 단점 |
|------|------|------|
| **var 사용** | - 직관적이고 이해하기 쉬움<br>- 성능이 우수 (순환 참조 한 번만 생성) | - 가변 상태를 사용 (함수형 순수성 위배) |
| **mkRec** | - 순수 함수형 방식<br>- 언어 수정 없이 구현 가능 | - 복잡하고 이해하기 어려움<br>- 성능 오버헤드 (함수 호출 중첩) |

실제 프로그래밍 언어 구현에서는 대부분 **var 방식**을 사용합니다. 인터프리터/컴파일러 구현 단계에서 가변 상태를 사용하는 것은 문제가 되지 않으며, 사용자에게 제공되는 언어는 여전히 순수 함수형일 수 있습니다.

반면 **mkRec 방식**은 학습 목적으로 유용합니다. 언어 차원의 지원 없이도 재귀를 구현할 수 있다는 것을 보여주며, λ-calculus의 표현력을 이해하는 데 도움이 됩니다.

## 6. 전체 인터프리터

RFAE의 전체 인터프리터 코드입니다:

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

  case If0(c, t, f) => interp(c, env) match {
    case NumV(0) => interp(t, env)
    case _ => interp(f, env)
  }

  case Rec(f, x, b, e) =>
    val cloV = CloV(x, b, Map.empty)  // 초기 환경은 빈 맵
    val nenv = env + (f -> cloV)
    cloV.fenv = nenv
    interp(e, nenv)
}
```

## 7. 정리

RFAE는 FAE에 두 가지 핵심 기능을 추가합니다:

1. **조건 분기 (if0)**: 0인지 검사하여 분기
2. **재귀 함수 (Rec)**: 순환 참조를 통한 재귀 구현

재귀 함수를 구현하는 방법은 두 가지입니다:

1. **var 키워드**: 가변 참조를 이용한 순환 참조 (실용적)
2. **mkRec 패턴**: 불동점 조합자를 이용한 desugaring (이론적)

두 방법 모두 같은 문제를 해결하지만, var 방식이 더 직관적이고 효율적이어서 실제 언어 구현에 주로 사용됩니다. mkRec 방식은 재귀가 언어의 필수 요소가 아니라 λ-calculus의 표현력만으로도 구현 가능하다는 것을 보여주는 학습 도구로서 가치가 있습니다.

다음 글에서는 RFAE를 더 확장하여 다양한 언어 기능들을 추가해보겠습니다.
