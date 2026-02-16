---
title: "[CS320] 1. Scala 기초와 프로그램 설계 방법론"
date: 2023-06-01
draft: false
tags: ["CS320", "Programming Languages", "KAIST", "Scala", "BNF", "Semantics"]
categories: ["Computer Science"]
series: ["CS320 프로그래밍언어"]
summary: "Scala의 Trait, Class, Case Class 차이점과 프로그래밍 언어를 설계하는 체계적인 방법론(Syntax, Semantics)을 정리합니다."
aliases: ["/posts/cs320-1-scala-basics-and-program-design/"]
ShowToc: true
TocOpen: true
---

> KAIST CS320 프로그래밍언어 (Spring 2023)
> 교재: 수업 자료 기반

---

## Scala 기초

### Trait vs Class vs Case Class

Scala에서는 객체 지향 프로그래밍을 위해 `trait`, `class`, `case class`라는 세 가지 핵심 개념을 제공합니다. 각각의 특징과 차이점을 이해하는 것이 중요합니다.

#### Single Inheritance (단일 상속)

먼저 공통점부터 살펴보겠습니다. Scala에서 `class`나 `trait`은 **단일 상속** 원칙을 따릅니다. 즉, 다른 `class`나 `trait`을 최대 하나까지만 상속받을 수 있으며, `extends` 키워드는 한 번에 하나만 사용할 수 있습니다.

```scala
// 단일 상속 예시
class Animal
class Dog extends Animal  // OK: Animal 하나만 상속
```

#### Trait vs Class

`trait`과 `class`의 주요 차이점은 다음과 같습니다:

**1. Multiple Inheritance (다중 상속)**
- `class`는 동시에 오직 하나의 클래스만 상속(inherit)할 수 있습니다
- 여러 `trait`은 동시에 상속할 수 있습니다

```scala
trait Walkable {
  def walk(): String = "walking"
}

trait Swimmable {
  def swim(): String = "swimming"
}

// Dog는 Animal 클래스 하나만 상속하고, 두 개의 trait을 동시에 믹스인
class Dog extends Animal with Walkable with Swimmable
```

**2. Constructor (생성자)**
- `class`는 constructor를 가질 수 있습니다
- `trait`은 constructor를 가질 수 없습니다 (단, constructor parameter는 가질 수 있음)

```scala
// class는 constructor 가능
class Person(name: String, age: Int) {
  println(s"Person created: $name")
}

// trait은 constructor 불가 (하지만 parameter는 가능)
trait Named(val name: String)  // Scala 3 스타일
```

**3. Instantiation (인스턴스화)**
- `class`는 `new` 키워드를 통해 직접 인스턴스를 생성할 수 있습니다
- `trait`은 직접 인스턴스화할 수 없으며, 항상 `class`와 함께 사용됩니다

```scala
val person = new Person("Alice", 25)  // OK
// val walkable = new Walkable  // 컴파일 에러!
```

**4. Polymorphism (다형성)**
- `class`와 `trait` 모두 다형성(polymorphism)을 지원합니다
- 하지만 `trait`에서 더 유연하게 지원됩니다

#### Case Class

`case class`는 Scala에서 불변(immutable) 데이터 구조를 간편하게 만들기 위한 특별한 클래스입니다.

**일반 Class vs Case Class:**

일반적인 `class`에서는 **클래스 정의**와 **클래스의 constructor 정의**가 구분됩니다. 하지만 constructor를 선언하는 것만으로는 클래스가 완전히 정의되지 않습니다.

반면 `case class`는 **클래스 정의**와 **constructor 정의**가 구분되지 않습니다. 즉, `case class`는 constructor를 정의하는 것만으로도 클래스를 완전히 정의할 수 있습니다.

```scala
// 일반 class: field 선언 필요
class Point(x: Int, y: Int)  // x, y는 field가 아님!

class Point2(val x: Int, val y: Int)  // val을 명시해야 field가 됨

// case class: 자동으로 val이 붙음
case class Point3(x: Int, y: Int)  // x, y가 자동으로 field가 됨
```

**Case Class의 주요 특징:**

1. **자동 val 처리**: constructor parameter 앞에 아무것도 붙이지 않아도 자동으로 `val`이 붙은 것으로 처리되므로, constructor parameter를 바로 field로 사용할 수 있습니다

2. **new 없이 생성**: `new` 키워드 없이 인스턴스 생성이 가능합니다

```scala
val p1 = Point3(1, 2)  // new 없이 생성 가능
```

3. **자동 메서드 생성**: pattern matching, immutability를 지원하는 메서드들(`copy`, `equals`, `hashCode`, `toString` 등)이 자동으로 생성됩니다

```scala
val p1 = Point3(1, 2)
val p2 = p1.copy(x = 3)  // 불변 객체의 복사본 생성
println(p1)  // Point3(1, 2) - toString 자동 생성
```

### OOP Terminology (객체 지향 용어)

프로그래밍 언어 이론에서 자주 사용되는 객체 지향 용어들을 정리합니다.

**Polymorphism (다형성)**
- 여러 subclass가 공통의 superclass를 상속받을 수 있는 것
- 하나의 인터페이스로 여러 타입을 다룰 수 있는 능력

**Variant (변종)**
- Polymorphism과 관련된 용어
- 여러 subclass가 공통의 superclass를 상속받을 때, 각 subclass는 superclass의 variant로 간주됩니다

**Member (멤버)**
- class의 구성 요소를 나타내는 일반적인 용어
- class 내부에 정의된 변수, 메서드 등을 포함
- field와 method 모두 member의 하위 카테고리입니다

**Field (필드)**
- class의 속성이나 state를 저장하는 variable
- class variable과 instance variable로 구분됩니다
  - **class variable**: class의 모든 instance에서 공유되는 값을 가지는 variable
  - **instance variable**: 각 instance마다 고유한 값을 가지는 variable

```scala
class Counter {
  var count: Int = 0  // instance variable (instance field)
}

object Counter {
  var totalCount: Int = 0  // class variable (companion object를 통한 공유 상태)
}
```

### Type & Object

Scala의 타입 시스템과 객체에 대해 알아봅니다.

#### Type의 종류

**Primitive Type (기본 타입)**
- `Int`, `Long`, `Double`, `Float` 등과 같은 기본 데이터 타입
- JVM의 primitive type과 대응됩니다

**Reference Type (참조 타입)**
- `class`, `object`, `trait`, `array` 등의 복잡한 데이터 구조
- 메모리 힙에 할당되고 참조를 통해 접근합니다

**Function Type (함수 타입)**
- parameter type과 return parameter type을 기반으로 하는 function과 method 타입
- 예: `Int => String`, `(Int, Int) => Int`

**Generic Type (제네릭 타입)**
- type parameter를 사용하여 범용적인 코드를 작성할 수 있는 타입
- 예: `List[T]`, `Option[A]`

**Type Alias (타입 별칭)**
- 기존 타입에 대한 새로운 이름을 부여하여 코드의 가독성을 높이는 방법
- `type` keyword를 사용하여 정의합니다

```scala
type Name = String
type Age = Int
type Person = (Name, Age)

val alice: Person = ("Alice", 25)
```

#### Object Keyword

`object` keyword는 **Singleton object**를 정의하는 데 사용됩니다. Singleton pattern을 언어 차원에서 지원하는 것으로, 프로그램 전체에서 단 하나의 인스턴스만 존재합니다.

```scala
object DatabaseConnection {
  private var connection: String = "disconnected"

  def connect(): Unit = {
    connection = "connected"
    println("Database connected")
  }
}

// 어디서든 동일한 인스턴스에 접근
DatabaseConnection.connect()
```

---

## How to Design Program (프로그램 설계 방법론)

프로그래밍 언어를 설계하고 구현하는 체계적인 방법론을 배웁니다. 언어 설계는 크게 **Syntax** (문법)와 **Semantics** (의미론) 두 부분으로 나뉩니다.

### Syntax (= Grammar)

프로그래밍 언어의 문법은 두 가지 형태로 표현됩니다.

#### Concrete Syntax (구체적 문법)

- **BNF (Backus-Naur Form)**: 사람이 읽고 쓰기 위한 문법 표기법
- 괄호 `()`가 아주 중요합니다
- `""`로 literal이 구체적으로 표기됩니다

```text
<expr> ::= <num>
         | "(" <expr> "+" <expr> ")"
         | "(" <expr> "*" <expr> ")"

<num>  ::= digit+
```

위 BNF 문법에서:
- `""`로 감싸진 것(`"("`, `"+"`, `"*"`, `")"`)은 **literal** (실제 문자)
- 감싸지지 않은 것(`<expr>`, `<num>`)은 **non-terminal** (문법 규칙)

Scala에서 구현한 `parser` function을 사용하면, input으로 받은 concrete syntax로 쓰인 string을 Abstract Syntax로 변환해줍니다.

```scala
// 예시: "(1 + 2)" 문자열을 파싱
val input = "(1 + 2)"
val ast = parser(input)  // Add(Num(1), Num(2))로 변환
```

#### Abstract Syntax (추상 문법)

프로그래밍 언어의 구조를 컴퓨터가 처리하기 쉬운 형태로 표현합니다.

**Scala (grammar for computer)**

```scala
// Abstract Syntax Tree를 위한 데이터 구조
sealed trait Expr
case class Num(value: Int) extends Expr
case class Add(left: Expr, right: Expr) extends Expr
case class Mul(left: Expr, right: Expr) extends Expr
```

**Mathematical Notation (수학적 표기법)**

괄호 `()`는 크게 중요하지 않으며, 편의를 위해 자유롭게 사용합니다:

```text
e ∈ Expr ::= n | e + e | e * e
n ∈ ℤ
```

### BNF (Backus-Naur Form) 상세

BNF는 프로그래밍 언어의 문법을 정의하는 표준 표기법입니다.

#### BNF의 핵심 개념

**1. Terminal vs Non-terminal**
- **Terminal**: `""`로 감싸진 문자는 literal로, 실제 프로그램에 나타나는 문자
- **Non-terminal**: `<>`로 감싸진 기호로, 다른 규칙으로 확장되는 문법 요소

**2. Production Rule (생성 규칙)**
- `::=` 기호는 "다음과 같이 정의된다"는 의미
- `|` 기호는 "또는"을 의미

**3. 재귀적 정의**
- BNF는 재귀적으로 문법을 정의할 수 있어 복잡한 중첩 구조 표현 가능

예시:

```text
<if-expr> ::= "if" "(" <expr> ")" <expr> "else" <expr>
<expr>    ::= <num>
            | <id>
            | <if-expr>
            | <expr> "+" <expr>
```

이 BNF는 다음과 같은 프로그램을 허용합니다:

```text
if (x) 1 + 2 else 3 * 4
```

### Semantics (의미론)

Syntax가 프로그램의 "형태"를 정의한다면, Semantics는 프로그램의 "의미"를 정의합니다.

#### Semantics 표현 방법

**1. Operational Semantics (작동 의미론)**

프로그램이 어떻게 실행되는지를 수학적으로 정의하는 방법입니다.

- **Natural Semantics (Big-step)**: 표현식이 최종 값으로 한 번에 평가되는 과정
- Natural Semantics ⊂ Operational Semantics

```text
Mathematical notation 예시:

    n ∈ ℤ
  ───────────  [NumRule]
    n ⇒ n

  e1 ⇒ n1    e2 ⇒ n2
  ───────────────────  [AddRule]
    e1 + e2 ⇒ n1+n2
```

위 표기법은:
- 줄 위는 전제 조건 (premise)
- 줄 아래는 결론 (conclusion)
- 오른쪽은 규칙 이름

**2. Interpreter Semantics (인터프리터 의미론)**

Scala로 인터프리터를 작성하여 언어의 의미를 설명하는 방법입니다. "언어를 설명하려면, 그 언어를 위한 인터프리터를 작성하라."

```scala
sealed trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Mul(l: Expr, r: Expr) extends Expr

// Value는 평가 결과
type Value = Int

// 인터프리터 함수
def interp(e: Expr): Value = e match {
  case Num(n) => n
  case Add(l, r) => interp(l) + interp(r)
  case Mul(l, r) => interp(l) * interp(r)
}

// 사용 예시
val expr = Add(Num(1), Mul(Num(2), Num(3)))
val result = interp(expr)  // 7
```

### Key Insights (핵심 통찰)

프로그래밍 언어 설계에서 중요한 개념들을 정리합니다:

1. **Abstract Syntax와 Scala Type**
   - Scala에서 BNF에 없는 `Value` 같은 것을 새로운 Type으로 정의하는 것은 Abstract Syntax를 정의하는 것입니다
   - Interpreter Semantics에서 사용하는 모든 Type은 Abstract Syntax로 표현된 것입니다

2. **Mathematical Notation의 역할**
   - Mathematical Notation으로 Abstract Syntax를 표현하는 것은 Data Type, 즉 Expression과 Value의 종류를 수학적으로 표현하는 것입니다

3. **Natural Semantics의 역할**
   - Natural Semantics (Operational Semantics)로 Semantics를 표현하는 것은 Expression으로부터 Value가 도출되는 과정을 수학적으로 표현하는 것입니다

```scala
// Abstract Syntax 확장 예시
sealed trait Expr
case class Num(n: Int) extends Expr
case class Add(l: Expr, r: Expr) extends Expr

// Value도 Abstract Syntax의 일부
sealed trait Value
case class NumV(n: Int) extends Value

// interp: Expr => Value
def interp(e: Expr): Value = e match {
  case Num(n) => NumV(n)
  case Add(l, r) =>
    val NumV(n1) = interp(l)
    val NumV(n2) = interp(r)
    NumV(n1 + n2)
}
```

---

## Steps for Language Implementation (언어 구현 단계)

실제로 프로그래밍 언어를 구현할 때는 다음 5단계를 따릅니다:

### 1. Determine the Data Representation (데이터 표현 결정)

Scala에서 `trait`과 `case class`를 정하는 단계입니다.

```scala
// AE (Arithmetic Expression) 언어 예시
sealed trait AE
case class Num(value: Int) extends AE
case class Add(left: AE, right: AE) extends AE
case class Sub(left: AE, right: AE) extends AE
```

### 2. Write Tests (테스트 작성)

구현하기 전에 테스트를 먼저 작성합니다 (TDD - Test-Driven Development).

```scala
// ScalaTest 예시
class AETest extends AnyFunSuite {
  test("Num(5) should evaluate to 5") {
    assert(interp(Num(5)) == 5)
  }

  test("Add(Num(1), Num(2)) should evaluate to 3") {
    assert(interp(Add(Num(1), Num(2))) == 3)
  }

  test("Sub(Num(5), Num(3)) should evaluate to 2") {
    assert(interp(Sub(Num(5), Num(3))) == 2)
  }
}
```

### 3. Create a Template for the Implementation (구현 템플릿 생성)

Scala에서 pattern matching으로 모든 case를 나열하는 단계입니다.

```scala
def interp(e: AE): Int = e match {
  case Num(n) => ???      // TODO
  case Add(l, r) => ???   // TODO
  case Sub(l, r) => ???   // TODO
}
```

### 4. Finish Implementation Case-by-Case (케이스별 구현 완성)

각 case를 하나씩 구현합니다.

```scala
def interp(e: AE): Int = e match {
  case Num(n) => n
  case Add(l, r) => interp(l) + interp(r)
  case Sub(l, r) => interp(l) - interp(r)
}
```

### 5. Run Tests (테스트 실행)

테스트를 실행하여 구현이 올바른지 확인합니다.

```scala
// sbt test 또는 IDE에서 테스트 실행
// All tests should pass!
```

---

## Language Model Roadmap (언어 모델 로드맵)

이 수업에서 다룰 언어 모델들의 발전 과정을 도식으로 나타냅니다. 각 단계는 이전 언어에 새로운 기능을 추가하여 점진적으로 복잡해집니다.

```text
기본 산술 언어:
  AE → VAE → F1VAE → FVAE → FAE → RFAE

변종 언어들:
  BFAE = FAE + Boxes (참조와 박스)
  MFAE = BFAE + Variables (가변 상태)
  LFAE = Lazy FAE (지연 평가)
```

각 언어의 의미:
- **AE** (Arithmetic Expression): 기본 산술 연산
- **VAE** (Variables + AE): 변수 추가
- **F1VAE** (First-class Functions + VAE): 일급 함수 (single parameter)
- **FVAE** (Functions with multiple parameters + VAE): 다중 매개변수 함수
- **FAE** (Functions + AE): 함수형 프로그래밍 기초
- **RFAE** (Recursive FAE): 재귀 함수 지원
- **BFAE** (Boxes + FAE): 참조와 박스 (간접 참조)
- **MFAE** (Mutable FAE): 가변 상태 (mutation)
- **LFAE** (Lazy FAE): 지연 평가 (lazy evaluation)

이 로드맵을 따라가며, 작은 언어부터 시작해서 점진적으로 실제 프로그래밍 언어의 핵심 기능들을 구현하게 됩니다.

---

## 정리

이번 글에서는 Scala의 기초 개념과 프로그래밍 언어를 체계적으로 설계하는 방법론을 배웠습니다.

**핵심 요점:**

1. **Scala 기초**
   - `trait`: 다중 상속 가능, constructor 불가, 직접 인스턴스화 불가
   - `class`: 단일 상속, constructor 가능, 인스턴스화 가능
   - `case class`: 불변 데이터 구조를 위한 특별한 클래스, 많은 기능 자동 생성

2. **언어 설계 방법론**
   - **Syntax**: Concrete Syntax (BNF) → Abstract Syntax (Scala types)
   - **Semantics**: Natural Semantics (수학적) 또는 Interpreter Semantics (Scala 구현)

3. **구현 프로세스**
   - 데이터 표현 결정 → 테스트 작성 → 템플릿 생성 → 케이스별 구현 → 테스트 실행

4. **언어 모델 발전**
   - AE에서 시작해서 점진적으로 변수, 함수, 재귀, 가변 상태 등을 추가

다음 글에서는 기본 산술 언어 **AE (Arithmetic Expression)**부터 시작하여 실제로 인터프리터를 구현해보겠습니다.
