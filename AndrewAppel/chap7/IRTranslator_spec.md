# IRTranslation: 수학적 명세

이 문서는 구현 코드와 무관하게, MiniJava AST를 Tree IR로 변환하는 의미론을 수학적으로 기술한 명세이다.

---

## 1. 목적

MiniJava 프로그램은 문법적 AST로 표현되며, 이를 Tree IR로 변환하는 함수 `T`를 정의한다.

- 입력: 프로그램 AST `P`
- 출력: procedure 집합 `Proc` 와 내부 표현식/문장으로 구성된 Tree IR

목적은 다음과 같다.

1. 문법 구조를 실행 가능한 중간 표현으로 낮추기
2. 제어 흐름을 `CJUMP`, `LABEL`, `JUMP`로 정규화하기
3. 메모리 접근과 메소드 호출을 주소 기반 표현으로 캡처하기
4. 이후 코드 생성 단계에서 실제 기계어로 변환 가능하도록 구조화하기

---

## 2. 도메인

### 2.1 MiniJava AST

문법은 다음과 같은 집합으로 정의한다.

- `Program = MainClass × ClassDecl*`
- `Stmt ::= Block | Assign | If | While | Print | ArrayAssign`
- `Expr ::= Int | True | False | Var | This | BinOp(Expr, Expr) | Not(Expr) | ArrayLookup | ArrayLength | Call | NewObject | NewArray`

### 2.2 Tree IR

Tree IR은 다음 범주로 구성된다.

- `Stm`: `MOVE`, `EXP`, `JUMP`, `CJUMP`, `SEQ`, `LABEL`
- `Exp`: `CONST`, `TEMP`, `MEM`, `BINOP`, `CALL`, `ESEQ`
- `Label`, `Temp` 는 고유 식별자

IR은 문장 중심으로 조직되며, 표현식은 값으로 평가된다.

---

## 3. 환경

### 3.1 전역 환경

컴파일 시간 환경은 다음 세 가지로 구성된다.

- `Env_v : Var -> Temp ∪ FieldLoc`
- `Env_t : Var -> Type`
- `ClassEnv : ClassName -> Field*`
- `MethodEnv : (ClassName, MethodName) -> MethodSig`

여기서 `FieldLoc`는 객체 필드의 위치를 의미한다.

### 3.2 필드 배치

각 클래스 `C`의 필드 목록을

`Fields(C) = [f1, f2, ..., fn]`

라 하자. 해당 필드들의 오프셋은 다음과 같이 정의한다.

`offset(C, fi) = (i - 1) × wordSize`

여기서 `wordSize = 4`이다.

상속이 있는 경우 부모 필드를 먼저 나열하고 자식 필드를 이어 붙인 전체 목록을 사용한다.

---

## 4. 의미 함수

Tree IR 변환은 함수 `⟦·⟧`로 정의된다.

- `⟦S⟧` : statement 의미
- `⟦E⟧` : expression 의미
- `⟦M⟧` : method 의미
- `⟦P⟧` : program 의미

정의는 다음과 같이 다루어진다.

- `⟦S⟧ ∈ Stm`
- `⟦E⟧ ∈ Exp`
- `⟦M⟧ ∈ Proc`
- `⟦P⟧ ∈ Proc*`

---

## 5. Statement 번역

### 5.1 블록

`Block([s1, s2, ..., sn])`는 순차적으로 번역한다.

`⟦Block([s1,...,sn])⟧ = SEQ(⟦s1⟧, SEQ(⟦s2⟧, ..., ⟦sn⟧))`

즉, 문장은 순차 실행 문맥으로 연결된다.

### 5.2 할당

변수 할당은 로컬 변수와 필드 접근을 구분한다.

- 로컬 변수 x: `MOVE(TEMP(vx), ⟦e⟧)`
- 필드 f: `MOVE(MEM(this + offset(C,f)), ⟦e⟧)`

따라서

`⟦x = e⟧ = MOVE(Loc(x), ⟦e⟧)`

여기서

`Loc(x) = TEMP(Env_v(x))` if x is local
`Loc(x) = MEM(this + offset(currentClass, x))` if x is field

### 5.3 조건문

`if e then s1 else s2`는 다음 규칙으로 변환한다.

`⟦if e then s1 else s2⟧ =`

```text
SEQ(
  CJUMP(NE, ⟦e⟧, CONST(0), l_true, l_false),
  LABEL(l_true),
  ⟦s1⟧,
  JUMP(l_end),
  LABEL(l_false),
  ⟦s2⟧,
  LABEL(l_end)
)
```

조건이 0이 아니면 then 분기, 0이면 else 분기한다.

### 5.4 반복문

`while e do s`는 다음 규칙으로 변환한다.

`⟦while e do s⟧ =`

```text
SEQ(
  LABEL(l_cond),
  CJUMP(NE, ⟦e⟧, CONST(0), l_body, l_end),
  LABEL(l_body),
  ⟦s⟧,
  JUMP(l_cond),
  LABEL(l_end)
)
```

조건 값이 참일 때만 본문을 실행하고, 거짓이면 종료한다.

### 5.5 출력

`print(e)`는 외부 함수 호출로 번역한다.

`⟦print(e)⟧ = EXP(CALL(NAME(print), [⟦e⟧]))`

여기서 `print`는 전역 함수 심볼이다.

### 5.6 배열 할당

`a[i] = v`는 다음으로 변환한다.

`⟦a[i] = v⟧ = MOVE(MEM(base(a) + 4 + i × 4), ⟦v⟧)`

이 규칙은 배열이 시작 주소와 길이 정보를 함께 가지고 있다는 단순화된 메모리 모델을 가정한다.

---

## 6. Expression 번역

### 6.1 상수와 논리 상수

`n ∈ Int`에 대해

`⟦n⟧ = CONST(n)`

`true`는 정수 1로 해석하고, `false`는 정수 0으로 해석한다.

`⟦true⟧ = CONST(1)`
`⟦false⟧ = CONST(0)`

### 6.2 변수

변수식은 로컬/필드 여부에 따라 다르게 번역한다.

`⟦x⟧ = TEMP(Env_v(x))`          if x is local
`⟦x⟧ = MEM(this + offset(C, x))` if x is field

### 6.3 `this`

`⟦this⟧ = TEMP(thisTemp)`

### 6.4 산술 연산

다음이 성립한다.

`⟦e1 + e2⟧ = BINOP(PLUS, ⟦e1⟧, ⟦e2⟧)`
`⟦e1 - e2⟧ = BINOP(MINUS, ⟦e1⟧, ⟦e2⟧)`
`⟦e1 × e2⟧ = BINOP(MUL, ⟦e1⟧, ⟦e2⟧)`

### 6.5 비교 연산

`e1 < e2` 표현은 bool 값을 0/1 정수로 계산한다.

`⟦e1 < e2⟧ = ESEQ(S, TEMP(t))`

여기서 `S`는 다음 구조를 갖는다.

```text
SEQ(
  MOVE(TEMP(t), CONST(0)),
  CJUMP(LT, ⟦e1⟧, ⟦e2⟧, l_true, l_false),
  LABEL(l_true),
  MOVE(TEMP(t), CONST(1)),
  JUMP(l_end),
  LABEL(l_false),
  LABEL(l_end)
)
```

즉 결과값은 1 또는 0이다.

### 6.6 논리식

`and`는 정수 비트 연산으로 처리한다.

`⟦e1 && e2⟧ = BINOP(AND, ⟦e1⟧, ⟦e2⟧)`

`not e`는 XOR 1로 처리한다.

`⟦!e⟧ = BINOP(XOR, ⟦e⟧, CONST(1))`

### 6.7 배열 접근

배열의 원소 접근은 주소 계산 방식으로 번역한다.

`⟦a[i]⟧ = MEM(base(a) + 4 + i × 4)`

여기서 `base(a)`는 배열의 시작 주소이다.

### 6.8 배열 길이

`length(a)`는 배열의 헤더에 저장된 길이 값을 읽는 것으로 모델링한다.

`⟦length(a)⟧ = MEM(base(a))`

### 6.9 메소드 호출

`recv.m(arg1, ..., argn)`는 다음과 같이 번역한다.

`⟦recv.m(arg1,...,argn)⟧ = CALL(NAME(label(C, m)), [⟦recv⟧, ⟦arg1⟧, ..., ⟦argn⟧])`

여기서

`label(C, m) = C + "_" + m`

수신 객체 `recv`는 첫 인자로 전달되며, 나머지 실제 인자는 뒤에 이어진다.

### 6.10 객체 생성

`new C()`는 heap allocation 및 필드 초기화로 번역한다.

`⟦new C()⟧ = ESEQ(S, TEMP(t))`

여기서 `S`는 다음 의미를 가진다.

1. `size = |Fields(C)| × wordSize`
2. `t = heapAlloc(size)`
3. 각 필드 `fi`를 0으로 초기화

즉,

```text
S =
  SEQ(
    MOVE(TEMP(t), CALL(NAME(heapAlloc), [CONST(size)])),
    MOVE(MEM(TEMP(t) + offset(C, f1)), CONST(0)),
    ...,
    MOVE(MEM(TEMP(t) + offset(C, fn)), CONST(0))
  )
```

### 6.11 배열 생성

`new T[e]`는 런타임 allocator에게 크기를 넘겨 실제 배열 메모리를 할당받는 식으로 모형화한다.

`⟦new T[e]⟧ = ESEQ(MOVE(TEMP(t), CALL(NAME(allocArray), [⟦e⟧])), TEMP(t))`

---

## 7. Method 번역

메소드 `m`를 클래스 `C`에 대해 번역할 때, 다음과 같은 의미를 가진다.

`⟦C.m(x1,...,xn) { var v1; ...; return e; }⟧ = Proc(label(C,m), frame, body)`

여기서

- `frame`은 함수 호출 규약과 로컬 크기를 반영한 프레임 구조
- `body`는 지역 변수 환경을 반영한 문장 시퀀스
- 반환값은 `RV`(예: $v0)에 저장된다.

형식적으로

`body = SEQ( ... , MOVE(TEMP(RV), ⟦e⟧) )`

메소드의 최종 반환값은 호출자에게 전달된다.

---

## 8. Program 번역

프로그램 `P = (main, classes)`의 의미는 각 procedure의 집합이다.

`⟦P⟧ = {⟦main⟧} ∪ {⟦C.m⟧ | C ∈ classes, m ∈ methods(C)}`

여기서 `main`은 인자 없는 엔트리 포인트로 취급된다.

---

## 9. 점검 조건

다음 조건을 만족해야 변환이 유효하다고 본다.

1. 모든 로컬 변수는 고유한 `Temp`에 대응한다.
2. 모든 필드는 클래스별로 고유 오프셋을 갖는다.
3. 모든 메소드 호출은 클래스 기반 이름으로 디스패치된다.
4. 모든 제어문은 `LABEL`/`JUMP`/`CJUMP` 구조로 표현된다.
5. 모든 표현식은 값으로 평가된다.
6. 모든 메모리 접근은 주소 계산과 `MEM` 연산으로 표현된다.

---

## 10. 요약

MiniJava AST를 Tree IR로 변환하는 수학적 명세는 다음과 같이 요약된다.

- 문장은 `Stm`으로 번역된다.
- 식은 `Exp`로 번역된다.
- 변수는 `Temp` 또는 객체 메모리 주소로 표현된다.
- 제어 흐름은 분기 구조로 번역된다.
- 메소드 호출은 함수 이름 + 인자 리스트로 번역된다.
- 객체/배열 생성은 heap allocation과 메모리 초기화를 포함한다.

이 명세는 구현 세부사항(Visitor, Java 클래스 구조)에 의존하지 않고, 컴파일러가 보장해야 할 의미론만 표현한다.
