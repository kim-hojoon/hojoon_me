---
title: "[CS320] 7. 가비지 컬렉션"
date: 2023-06-13
draft: false
tags: ["CS320", "Programming Languages", "KAIST", "Garbage Collection", "Memory Management"]
categories: ["Computer Science"]
series: ["CS320 프로그래밍언어"]
summary: "Reference Counting, Mark & Sweep, Two-Space Collection 세 가지 가비지 컬렉션 알고리즘의 원리와 장단점을 비교합니다."
aliases: ["/posts/cs320-7-garbage-collection/"]
ShowToc: true
TocOpen: true
---

> KAIST CS320 프로그래밍언어 (Spring 2023)
> 교재: 수업 자료 기반

---

## 가비지 컬렉션이란?

**가비지 컬렉션(Garbage Collection, GC)**은 프로그램이 동적으로 할당한 메모리를 자동으로 관리하는 기술이다. 메모리를 효율적으로 사용하기 위해서는 더 이상 사용하지 않는 할당된 메모리를 해제해야 한다. 가비지 컬렉션은 프로그램에서 더 이상 접근할 수 없는(unreachable) 객체를 자동으로 해제하는 메커니즘을 제공한다.

### 핵심 용어

- **Record**: 메모리 객체에 대한 정보를 저장하는 자료구조
- **Live Object**: 프로그램에서 아직 사용 중인 메모리 객체 (접근 가능한 객체)
- **Dead Object (Garbage)**: 자신을 참조하고 있는 live object가 없는 객체

### Garbage 식별 방법

가비지를 식별하는 기본 원리는 다음과 같다:

1. **Stack과 Register**에서 참조하고 있는 record는 live하다 (이들을 **root**라고 부른다)
2. Live record가 참조하고 있는 record는 live하다 (재귀적으로 적용)
3. Live record가 아닌 모든 record는 garbage다 (프로그램에서 접근 불가능)

```text
[Stack/Register (Roots)]
    |
    v
[Live Object A] --> [Live Object B] --> [Live Object C]
    |
    v
[Live Object D]

[Dead Object X] --> [Dead Object Y]  (아무도 참조하지 않음)
```

명확하게 live한 root에서 시작해 참조할 수 있는 모든 record를 찾고, 나머지를 모두 garbage로 처리하면 된다.

---

## 가비지 컬렉션 알고리즘

가비지 컬렉션을 구현하는 대표적인 세 가지 알고리즘을 살펴보자.

---

## 1. Reference Counting

**Reference Counting**은 각 객체마다 자신을 참조하는 포인터의 개수를 세는 방식이다.

### 동작 원리

1. 모든 record의 `count`를 0으로 초기화
2. 해당 record를 가리키는 포인터가 생기면 → `count++`
3. 해당 record를 가리키던 포인터가 사라지면 → `count--`
4. Record의 count가 0이 되면 아무도 사용하지 않는다는 신호!
   - 해당 record가 참조하던 모든 record의 count를 1씩 감소
   - Count가 0인 record를 해제(free)

### 동작 예시

```text
초기 상태:
Root --> [A:1] --> [B:1] --> [C:1]

Root가 A 대신 B를 직접 참조하도록 변경:
Root --> [A:0]
    \
     --> [B:2] --> [C:1]

A의 count가 0이 되어 해제:
Root --> [B:1] --> [C:1]
```

### Reference Counting의 순환 참조 문제

```text
Root --> [A:1] --> [B:1]
               <--/

Root가 A를 해제:
[A:1] --> [B:1]  (서로를 참조하여 count가 0이 안 됨!)
    <--/
```

A와 B가 서로를 참조하면 count가 절대 0이 되지 않아 메모리 누수(Memory Leak)가 발생한다.

### 장단점

**장점:**
- Update를 기반으로 점진적으로 메모리를 해제하므로 전체 heap을 scan할 필요가 없다
- 프로그램을 중지시킬 필요가 없다 (incremental)

**단점:**
- **Memory Leak**: 순환 참조(cycle)를 해제할 수 없음
- Count를 유지하는 데 시간이 소모됨
- **Bad Locality (Fragmentation)**: 메모리가 점점 조각화되어 locality가 떨어짐
- Free list를 사용하여 사용 가능한 메모리를 추적해야 함

---

## 2. Mark & Sweep

**Mark & Sweep**은 접근 가능한 객체를 표시(mark)하고, 표시되지 않은 객체를 해제(sweep)하는 방식이다.

### 동작 원리

1. 모든 record를 **white**로 칠함
2. Root에서 참조하는 record를 **gray**로 칠함
3. Gray record가 모두 사라질 때까지 다음 과정 반복:
   - 임의의 gray record 하나를 선택
   - 해당 record가 가리키는 white record를 모두 gray로 칠함
   - 선택한 gray record를 **black**으로 칠함
4. 모든 white record를 해제

### 동작 예시

```text
Step 1: 모든 record를 white로 초기화
[Root]
  |
  v
[A:W] --> [B:W] --> [D:W]
  |
  v
[C:W]

[E:W] --> [F:W]  (unreachable)


Step 2: Root가 참조하는 A를 gray로 칠함
[Root]
  |
  v
[A:G] --> [B:W] --> [D:W]
  |
  v
[C:W]

[E:W] --> [F:W]


Step 3: A를 처리 (A가 가리키는 B, C를 gray로, A를 black으로)
[Root]
  |
  v
[A:B] --> [B:G] --> [D:W]
  |
  v
[C:G]

[E:W] --> [F:W]


Step 4: B를 처리 (B가 가리키는 D를 gray로, B를 black으로)
[Root]
  |
  v
[A:B] --> [B:B] --> [D:G]
  |
  v
[C:G]

[E:W] --> [F:W]


Step 5: C를 처리 (C는 아무것도 가리키지 않음, C를 black으로)
[Root]
  |
  v
[A:B] --> [B:B] --> [D:G]
  |
  v
[C:B]

[E:W] --> [F:W]


Step 6: D를 처리 (D는 아무것도 가리키지 않음, D를 black으로)
[Root]
  |
  v
[A:B] --> [B:B] --> [D:B]
  |
  v
[C:B]

[E:W] --> [F:W]


Step 7: Gray가 없으므로 종료. White인 E, F를 해제
[Root]
  |
  v
[A:B] --> [B:B] --> [D:B]
  |
  v
[C:B]
```

### 순환 참조 처리

```text
Root --> [A] <--> [B]  (cycle)
         |
         v
        [C]

Mark 단계:
- A를 gray로 칠함
- A가 B와 C를 gray로 칠하고 A를 black으로
- B가 A를 가리키지만 이미 black이므로 무시, B를 black으로
- C를 black으로

결과: A, B, C 모두 black → 모두 보존됨 (정상 처리)
```

Mark & Sweep은 순환 참조를 올바르게 처리한다!

### 장단점

**장점:**
- 순환 참조(cycle) 문제를 해결함

**단점:**
- 전체 heap을 scan해야 함 → 프로그램을 중지시켜야 함 (stop-the-world)
- **Bad Locality (Fragmentation)**: 메모리 조각화 발생
- Free list를 사용하여 사용 가능한 메모리를 추적해야 함

---

## 3. Two-Space Collection

**Two-Space Collection** (Copy Collection)은 메모리를 두 개의 공간으로 나누어 live object를 한쪽에서 다른 쪽으로 복사하는 방식이다.

### 동작 원리

1. 메모리를 **from-space**와 **to-space** 두 영역으로 나눔
2. 프로그램은 from-space에서 객체를 할당
3. GC 실행 시:
   - Root에서 참조하는 live object를 to-space로 복사
   - 복사된 객체가 참조하는 객체도 재귀적으로 to-space로 복사
   - 포인터를 업데이트하여 새 주소를 가리키도록 함
4. 복사가 끝나면 from-space 전체를 해제
5. From-space와 to-space의 역할을 교체

### 동작 예시

```text
초기 상태 (from-space에서 할당):
From-space:                  To-space:
[Root]                       [empty]
  |
  v
[A] --> [B] --> [D]
  |
  v
[C]

[E] --> [F]  (unreachable)


Step 1: Root가 참조하는 A를 to-space로 복사
From-space:                  To-space:
[Root] ----+                 [A']
  |        |
  v        v
[A] --> [B] --> [D]
  |
  v
[C]


Step 2: A가 참조하는 B, C를 to-space로 복사
From-space:                  To-space:
[Root] ----+                 [A'] --> [B']
           |                   |
[A] --> [B] --> [D]            v
  |                           [C']
  v
[C]


Step 3: B가 참조하는 D를 to-space로 복사
From-space:                  To-space:
[Root] ----+                 [A'] --> [B'] --> [D']
           |                   |
[A] --> [B] --> [D]            v
  |                           [C']
  v
[C]

E, F는 unreachable이므로 복사되지 않음


Step 4: From-space 전체 해제, 역할 교체
From-space:                  To-space (now from-space):
[empty]                      [A'] --> [B'] --> [D']
                               |
                               v
                              [C']
```

### 메모리 레이아웃

```text
전체 메모리:
+------------------+------------------+
|   From-space     |    To-space      |
| (현재 사용 중)    | (다음 GC 대기)    |
+------------------+------------------+
      50%                 50%

GC 후 역할 교체:
+------------------+------------------+
|    To-space      |   From-space     |
| (다음 GC 대기)    | (현재 사용 중)    |
+------------------+------------------+
      50%                 50%
```

### 장단점

**장점:**
- **Fragmentation 문제 해결**: Live object를 compact하게 복사하여 연속된 메모리 공간 확보
- **Allocation이 빠름**: Bump pointer 방식으로 메모리 할당 (포인터만 증가시키면 됨)
- 순환 참조 문제 해결

**단점:**
- **메모리 효율 50%**: 전체 메모리의 절반만 사용 가능
- 복사 비용: Live object를 모두 복사해야 하므로 live object가 많으면 비용이 큼

---

## 알고리즘 비교

### 종합 비교표

| 특성 | Reference Counting | Mark & Sweep | Two-Space Collection |
|------|-------------------|--------------|---------------------|
| **순환 참조 처리** | X (Memory Leak) | O | O |
| **프로그램 중지** | 불필요 (incremental) | 필요 (stop-the-world) | 필요 (stop-the-world) |
| **Fragmentation** | 발생 | 발생 | 해결 (compaction) |
| **메모리 효율** | 높음 (100%) | 높음 (100%) | 낮음 (50%) |
| **Allocation 속도** | 보통 (free list) | 보통 (free list) | 빠름 (bump pointer) |
| **추가 오버헤드** | Count 유지 | Color 정보 | To-space 필요 |

### 알고리즘 선택 기준

- **Reference Counting**: 순환 참조가 없고, 실시간 반응이 중요한 시스템
- **Mark & Sweep**: 메모리가 제한적이고, 순환 참조를 처리해야 하는 경우
- **Two-Space Collection**: 메모리가 충분하고, fragmentation을 피하고 싶은 경우

---

## 마치며

가비지 컬렉션은 현대 프로그래밍 언어의 핵심 기능 중 하나다. 각 알고리즘은 서로 다른 trade-off를 가지고 있으며, 실제 언어 구현에서는 이들을 조합하거나 변형한 알고리즘을 사용한다. 예를 들어:

- **Java, C#**: Generational GC (young/old generation을 나누어 처리)
- **Python**: Reference Counting + Cycle Detector
- **Go**: Concurrent Mark & Sweep

프로그래밍 언어를 설계하거나 성능을 최적화할 때, 해당 언어의 GC 알고리즘을 이해하는 것은 매우 중요하다.
