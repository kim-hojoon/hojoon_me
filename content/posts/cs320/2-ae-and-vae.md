---
title: "[CS320] 2. 산술식에서 변수까지 — AE와 VAE"
date: 2023-06-03
draft: false
tags: ["CS320", "Programming Languages", "KAIST", "AE", "VAE", "Interpreter"]
categories: ["Computer Science"]
series: ["CS320 프로그래밍언어"]
summary: "가장 단순한 산술식 언어(AE)부터 변수 바인딩을 지원하는 VAE까지, 인터프리터를 단계적으로 확장하며 프로그래밍 언어의 기초를 쌓습니다."
aliases: ["/posts/cs320-2-ae-and-vae/"]
ShowToc: true
TocOpen: true
---

> KAIST CS320 프로그래밍언어 (Spring 2023)
> 교재: 수업 자료 기반

---

프로그래밍 언어를 설계하고 구현하는 과정을 이해하기 위해, 가장 단순한 형태의 언어부터 시작해봅시다. 이번 글에서는 **AE**(Arithmetic Expressions)와 **VAE**(Variable Arithmetic Expressions)라는 두 가지 간단한 언어를 통해 인터프리터의 기본 구조와 변수 바인딩의 개념을 배웁니다.

## 언어 설계의 3요소

프로그래밍 언어를 정의할 때는 다음 세 가지 측면을 고려해야 합니다:

1. **Concrete Syntax (구체적 문법)**: BNF(Backus-Naur Form)로 표현되는, 실제로 작성하는 프로그램의 형태
2. **Abstract Syntax (추상 문법)**: 프로그램의 구조를 나타내는 추상 구문 트리(AST)
3. **Semantics (의미론)**: 프로그램이 실행될 때의 동작을 정의하는 인터프리터

이 세 가지 요소는 서로 밀접하게 연결되어 있으며, 각각의 역할이 명확합니다. Concrete Syntax는 사람이 읽고 쓰는 형태, Abstract Syntax는 컴퓨터가 처리하기 좋은 형태, Semantics는 실제 실행 규칙을 나타냅니다.

## AE: 산술식 언어

AE(Arithmetic Expressions)는 정수와 덧셈, 뺄셈만을 지원하는 가장 단순한 언어입니다. 변수도 없고, 함수도 없지만, 프로그래밍 언어의 핵심 개념을 배우기에 충분합니다.

### Concrete Syntax

AE의 구체적 문법은 다음과 같이 BNF로 정의됩니다:

```text
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
```

이 문법에 따르면, 다음과 같은 표현식들이 유효한 AE 프로그램입니다:

- `42`
- `(1 + 2)`
- `((3 - 1) + (5 + 6))`

### Abstract Syntax

Concrete Syntax는 사람이 읽기 편하지만, 프로그램으로 처리하기에는 불편합니다. 괄호와 공백 같은 정보는 구조를 나타낼 뿐 의미에는 영향을 주지 않습니다. 따라서 우리는 프로그램의 본질적인 구조만을 담은 **추상 구문 트리**(Abstract Syntax Tree, AST)를 사용합니다.

Scala로 표현한 AE의 추상 문법:

```scala
trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
```

- `Num(n)`: 정수 값 `n`을 나타냅니다
- `Add(l, r)`: 두 식 `l`과 `r`의 덧셈을 나타냅니다
- `Sub(l, r)`: 두 식 `l`과 `r`의 뺄셈을 나타냅니다

예를 들어, `((3 - 1) + (5 + 6))`는 다음과 같은 AST로 표현됩니다:

```scala
Add(
  Sub(Num(3), Num(1)),
  Add(Num(5), Num(6))
)
```

수학적 표기법으로는 다음과 같이 간결하게 쓸 수 있습니다:

```text
e ::= n
    | e + e
    | e - e
```

### Interpreter Semantics

이제 AE 프로그램이 어떻게 실행되는지 정의해야 합니다. 인터프리터는 AST를 입력받아 계산 결과를 반환하는 함수입니다.

```scala
def interp(expr: Expr): Int = expr match {
  case Num(n) => n
  case Add(l, r) => interp(l) + interp(r)
  case Sub(l, r) => interp(l) - interp(r)
}
```

각 case의 의미:

- `Num(n)`: 정수 `n`을 그대로 반환합니다
- `Add(l, r)`: 왼쪽 식 `l`을 계산한 결과와 오른쪽 식 `r`을 계산한 결과를 더합니다
- `Sub(l, r)`: 왼쪽 식 `l`을 계산한 결과에서 오른쪽 식 `r`을 계산한 결과를 뺍니다

예를 들어, `Add(Sub(Num(3), Num(1)), Add(Num(5), Num(6)))`를 계산하면:

1. `Sub(Num(3), Num(1))`를 계산 → `3 - 1 = 2`
2. `Add(Num(5), Num(6))`를 계산 → `5 + 6 = 11`
3. `2 + 11 = 13`

이 인터프리터는 **재귀적 구조**를 따릅니다. 복잡한 식을 부분 식으로 분해하고, 각 부분을 계산한 후 결과를 조합합니다. 이는 프로그래밍 언어 인터프리터의 가장 기본적이면서도 강력한 패턴입니다.

## VAE: 변수를 가진 산술식 언어

AE는 단순하지만, 같은 값을 여러 번 사용하려면 매번 똑같은 식을 반복해야 합니다. 실제 프로그래밍 언어에서는 **변수**(variable)를 사용해 값에 이름을 붙이고 재사용할 수 있습니다. VAE(Variable Arithmetic Expressions)는 AE에 변수 바인딩 기능을 추가한 언어입니다.

### Concrete Syntax

```text
expr ::= num
       | "(" expr "+" expr ")"
       | "(" expr "-" expr ")"
       | "{" "val" id "=" expr ";" expr "}"
       | id
```

새로 추가된 부분:

- `{ val id = expr; expr }`: 변수 바인딩 (variable binding)
- `id`: 변수 참조 (variable reference)

예를 들어, 다음은 유효한 VAE 프로그램입니다:

```text
{ val x = 5; (x + 3) }
```

이 프로그램은 `x`에 `5`를 바인딩하고, `x + 3`을 계산하여 `8`을 반환합니다.

더 복잡한 예:

```text
{ val x = 5; { val y = (x + 3); (x + y) } }
```

이 프로그램은:
1. `x`에 `5`를 바인딩
2. `y`에 `x + 3 = 8`을 바인딩
3. `x + y = 5 + 8 = 13`을 계산

### Abstract Syntax

Scala로 표현한 VAE의 추상 문법:

```scala
trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
case class Val(x: String, i: Expr, b: Expr) extends Expr
case class Id(x: String) extends Expr

type Env = Map[String, Int]
```

새로 추가된 부분:

- `Val(x, i, b)`: 변수 `x`에 식 `i`의 값을 바인딩하고, 본문 `b`를 계산합니다
- `Id(x)`: 변수 `x`의 값을 참조합니다
- `Env`: **환경**(Environment)으로, 변수 이름과 그 값의 매핑을 저장합니다

수학적 표기법:

```text
e ::= n
    | e + e
    | e - e
    | val x = e; e
    | x
```

### Environment의 개념

변수를 지원하려면 **환경**(Environment)이라는 개념이 필요합니다. 환경은 변수 이름(String)을 값(Int)으로 매핑하는 사전(dictionary)입니다.

```scala
type Env = Map[String, Int]
```

예를 들어:
- `Map()`: 빈 환경
- `Map("x" -> 5)`: `x`가 `5`로 바인딩된 환경
- `Map("x" -> 5, "y" -> 8)`: `x`가 `5`, `y`가 `8`로 바인딩된 환경

환경은 프로그램 실행 중에 현재 스코프에서 접근 가능한 모든 변수의 정보를 담고 있습니다. 새로운 변수를 바인딩하면 환경에 새 항목이 추가되고, 변수를 참조하면 환경에서 해당 변수의 값을 찾습니다.

### Interpreter Semantics

VAE의 인터프리터는 식뿐만 아니라 환경도 함께 받아서 계산합니다:

```scala
def interp(expr: Expr, env: Env): Int = expr match {
  case Num(n) => n
  case Add(l, r) => interp(l, env) + interp(r, env)
  case Sub(l, r) => interp(l, env) - interp(r, env)
  case Val(x, i, b) => interp(b, env + (x -> interp(i, env)))
  case Id(x) => lookup(x, env)
}
```

각 case의 의미:

- `Num(n)`: 정수 `n`을 그대로 반환 (환경은 사용하지 않음)
- `Add(l, r)`, `Sub(l, r)`: 양쪽 식을 같은 환경에서 계산하고 결과를 조합
- `Val(x, i, b)`: **핵심 부분!**
  1. 먼저 초기화 식 `i`를 현재 환경 `env`에서 계산
  2. 계산 결과를 변수 `x`에 바인딩하여 새로운 환경 생성: `env + (x -> interp(i, env))`
  3. 본문 `b`를 새로운 환경에서 계산
- `Id(x)`: 환경에서 변수 `x`의 값을 찾아 반환 (`lookup` 함수 사용)

여기서 중요한 점은 **`Val` case에서만 환경이 변경된다**는 것입니다. 변수 바인딩이 일어나는 곳이 바로 `Val`이기 때문입니다.

### 실행 예제

`{ val x = 5; { val y = (x + 3); (x + y) } }`를 AST로 표현하면:

```scala
Val("x", Num(5),
  Val("y", Add(Id("x"), Num(3)),
    Add(Id("x"), Id("y"))
  )
)
```

빈 환경 `Map()`에서 이 식을 계산하는 과정:

1. `Val("x", Num(5), ...)`
   - `Num(5)`를 계산 → `5`
   - 환경을 `Map("x" -> 5)`로 업데이트
   - 본문을 새 환경에서 계산

2. `Val("y", Add(Id("x"), Num(3)), ...)` (환경: `Map("x" -> 5)`)
   - `Add(Id("x"), Num(3))`를 계산
   - `Id("x")` → 환경에서 `"x"`를 찾아 `5` 반환
   - `Num(3)` → `3`
   - `5 + 3 = 8`
   - 환경을 `Map("x" -> 5, "y" -> 8)`로 업데이트
   - 본문을 새 환경에서 계산

3. `Add(Id("x"), Id("y"))` (환경: `Map("x" -> 5, "y" -> 8)`)
   - `Id("x")` → `5`
   - `Id("y")` → `8`
   - `5 + 8 = 13`

최종 결과: `13`

### Shadowing (변수 가리기)

VAE에서는 같은 이름의 변수를 중첩해서 선언할 수 있습니다:

```text
{ val x = 5; { val x = 3; x } }
```

AST:

```scala
Val("x", Num(5),
  Val("x", Num(3),
    Id("x")
  )
)
```

실행 과정:

1. 외부 `Val`: 환경을 `Map("x" -> 5)`로 설정
2. 내부 `Val`: 환경을 `Map("x" -> 3)`으로 설정 (기존 `"x"` 바인딩을 덮어씀)
3. `Id("x")`: 환경에서 `"x"`를 찾으면 `3`

결과는 `3`입니다. 내부의 `x`가 외부의 `x`를 **가린다**(shadow)고 표현합니다. 내부 스코프가 끝나면 다시 외부의 바인딩이 유효해집니다.

## AE에서 VAE로의 확장

AE에서 VAE로 넘어오면서 무엇이 달라졌을까요?

1. **Abstract Syntax**: `Val`과 `Id` 두 가지 case class가 추가되었습니다
2. **Environment**: 변수 바인딩을 저장하기 위한 `Map[String, Int]` 타입이 도입되었습니다
3. **Interpreter**: 환경을 인자로 받고, `Val`에서 환경을 업데이트하며, `Id`에서 환경을 조회합니다

이 과정은 프로그래밍 언어 설계의 일반적인 패턴을 보여줍니다:

1. 새로운 기능(변수)이 필요하다
2. Abstract Syntax에 새로운 노드를 추가한다
3. 그 기능을 구현하기 위해 필요한 런타임 정보(환경)를 정의한다
4. Interpreter에서 새 노드를 처리하는 case를 추가한다

앞으로 우리가 배울 더 복잡한 언어들도 이 패턴을 따릅니다. 함수를 추가하고 싶다면? Abstract Syntax에 함수 정의와 호출 노드를 추가하고, 환경을 확장하고, 인터프리터에 case를 추가하면 됩니다.

## 마치며

이번 글에서는 가장 단순한 언어 AE부터 시작해서, 변수 바인딩을 지원하는 VAE까지 확장하는 과정을 살펴보았습니다. 핵심 개념들을 정리하면:

- **Concrete Syntax**: 사람이 작성하는 프로그램의 형태 (BNF)
- **Abstract Syntax**: 프로그램의 구조를 나타내는 추상 구문 트리 (Scala case class)
- **Semantics**: 프로그램의 실행 규칙을 정의하는 인터프리터 함수
- **Environment**: 변수 이름과 값의 매핑을 저장하는 사전 구조

다음 글에서는 VAE에 함수를 추가한 FAE(First-class Arithmetic Expressions)를 다루며, 클로저(closure)와 일급 함수(first-class function)의 개념을 배우게 됩니다.

