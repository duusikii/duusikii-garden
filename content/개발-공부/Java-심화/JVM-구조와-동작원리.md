---
title: JVM 구조와 동작원리
tags: [Java, JVM, 면접, CS]
---

# JVM 구조와 동작원리

## JVM이란?
- **Java Virtual Machine** — Java 바이트코드를 실행하는 가상 머신
- "Write Once, Run Anywhere"의 핵심
- `.java` → (javac 컴파일) → `.class` (바이트코드) → (JVM 실행)

## JVM 전체 구조

```
┌─────────────────────────────────────────┐
│              JVM                        │
│                                         │
│  ┌─────────────────────────────┐        │
│  │   Class Loader Subsystem   │        │
│  │   .class 파일 → 메모리 적재     │        │
│  └──────────┬──────────────────┘        │
│             ▼                           │
│  ┌─────────────────────────────┐        │
│  │   Runtime Data Areas       │        │
│  │                             │        │
│  │  ┌───────┐ ┌───────┐       │        │
│  │  │Method │ │ Heap  │       │        │
│  │  │ Area  │ │       │       │        │
│  │  └───────┘ └───────┘       │        │
│  │  ┌───────┐ ┌───────┐       │        │
│  │  │ Stack │ │  PC   │       │        │
│  │  │       │ │Register│      │        │
│  │  └───────┘ └───────┘       │        │
│  │  ┌────────────────┐        │        │
│  │  │ Native Method  │        │        │
│  │  │    Stack       │        │        │
│  │  └────────────────┘        │        │
│  └──────────┬──────────────────┘        │
│             ▼                           │
│  ┌─────────────────────────────┐        │
│  │   Execution Engine         │        │
│  │  ┌────────┐ ┌───────────┐  │        │
│  │  │Interpreter│ │JIT Compiler│  │        │
│  │  └────────┘ └───────────┘  │        │
│  │  ┌────────────────┐        │        │
│  │  │  GC (Garbage   │        │        │
│  │  │   Collector)   │        │        │
│  │  └────────────────┘        │        │
│  └─────────────────────────────┘        │
└─────────────────────────────────────────┘
```

---

## 1. Class Loader (클래스 로더)

`.class` 파일을 **찾고 → 검증하고 → 메모리에 적재**하는 시스템.

### 로딩 3단계
```
Loading → Linking → Initialization
  ↓         ↓           ↓
파일 읽기  검증/준비    static 초기화
```

### 클래스 로더 계층 (위임 모델)
```
Bootstrap ClassLoader     ← rt.jar (java.lang.* 등 코어)
    ↓
Extension ClassLoader     ← jre/lib/ext/
    ↓
Application ClassLoader   ← classpath (우리가 짠 코드)
```

**🔑 면접 포인트:**
> "클래스 로더는 **부모 위임 모델(Parent Delegation)**을 따릅니다. 클래스를 로드할 때 먼저 부모에게 위임하고, 부모가 못 찾으면 자신이 로드합니다. 이렇게 하면 `java.lang.String` 같은 코어 클래스를 누군가 덮어쓸 수 없어 **보안**이 보장됩니다."

---

## 2. Runtime Data Areas (메모리 영역)

### Method Area (메서드 영역)
- 클래스 메타데이터, static 변수, 상수 풀
- **모든 스레드가 공유**
- Java 8+: PermGen → **Metaspace** (네이티브 메모리)

### Heap (힙)
- **객체 인스턴스가 생성되는 곳**
- **모든 스레드가 공유** → 동시성 이슈 발생 지점
- **GC의 주요 대상**
- 구조:

```
Heap
├── Young Generation
│   ├── Eden        ← new 객체가 먼저 여기에
│   ├── Survivor 0  ← Minor GC에서 살아남은 것
│   └── Survivor 1
└── Old Generation  ← 오래 살아남은 객체가 여기로 승격(Promotion)
```

### Stack (스택)
- **스레드마다 독립적** (공유 X)
- 메서드 호출 시 **Stack Frame** 생성
- Frame 구성: 지역변수, 피연산자 스택, 프레임 데이터

```java
void methodA() {
    int x = 10;        // Stack Frame A에 저장
    methodB();          // Stack Frame B 추가
}                       // Frame A로 돌아감
```

### PC Register
- 스레드마다 독립적
- 현재 실행 중인 바이트코드 주소 저장

### Native Method Stack
- C/C++ 네이티브 메서드 실행 시 사용

---

## 3. Execution Engine (실행 엔진)

### Interpreter (인터프리터)
- 바이트코드를 **한 줄씩** 해석 실행
- 장점: 빠른 시작
- 단점: 반복 실행 시 느림

### JIT Compiler (Just-In-Time)
- **자주 실행되는 코드(Hot Spot)**를 네이티브 코드로 컴파일
- 컴파일 후에는 인터프리터 거치지 않고 직접 실행 → **빠름**

```
첫 실행: 인터프리터로 해석
  ↓
반복 감지: "이 메서드 1000번 넘게 호출됐네?"
  ↓
JIT 컴파일: 네이티브 코드로 변환
  ↓
이후: 네이티브 코드 직접 실행 (빠름!)
```

**🔑 면접 포인트:**
> "JVM은 처음에 인터프리터로 실행하다가, 자주 호출되는 **핫스팟(Hot Spot)** 코드를 JIT 컴파일러가 네이티브 코드로 변환합니다. 이 덕분에 Java가 순수 인터프리터 언어보다 훨씬 빠릅니다."

---

## 4. 주요 JVM 옵션

| 옵션 | 설명 |
|------|------|
| `-Xms256m` | 초기 힙 크기 |
| `-Xmx1024m` | 최대 힙 크기 |
| `-Xss512k` | 스레드 스택 크기 |
| `-XX:MetaspaceSize=128m` | Metaspace 초기 크기 |
| `-XX:+UseG1GC` | G1 GC 사용 |

---

## 🎤 면접 예상 질문

**Q: JVM의 메모리 구조를 설명해주세요.**
> "JVM 메모리는 크게 Method Area, Heap, Stack, PC Register, Native Method Stack 5가지로 나뉩니다. Method Area와 Heap은 모든 스레드가 공유하고, Stack은 스레드마다 독립입니다. 객체는 Heap에 생성되며 GC의 대상이 됩니다. Java 8부터 PermGen이 제거되고 Metaspace로 대체되었습니다."

**Q: Stack과 Heap의 차이는?**
> "Stack은 스레드마다 독립적이고 메서드의 지역변수·매개변수가 저장됩니다. 후입선출(LIFO) 구조이고 메서드 종료 시 자동 해제됩니다. Heap은 모든 스레드가 공유하며 `new`로 생성된 객체가 저장됩니다. GC가 관리합니다."

**Q: JIT 컴파일러란?**
> "바이트코드를 실행 시점(Just-In-Time)에 네이티브 코드로 컴파일하는 컴파일러입니다. 자주 실행되는 Hot Spot 코드를 감지해서 컴파일하므로, 인터프리터 방식의 느린 속도를 보완합니다."

**Q: Java 8에서 PermGen이 Metaspace로 바뀐 이유?**
> "PermGen은 고정 크기여서 클래스가 많으면 `OutOfMemoryError: PermGen space`가 자주 발생했습니다. Metaspace는 네이티브 메모리를 사용해 자동으로 크기가 조절되므로 이 문제가 해소됩니다."
