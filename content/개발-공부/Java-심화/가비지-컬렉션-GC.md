---
title: 가비지 컬렉션 (GC)
tags:
  - Java
  - JVM
  - GC
  - G1
  - 면접
created: 2026-03-12
modified: 2026-03-12
related:
  - "[[JVM-구조와-동작원리]]"
  - "[[힙-덤프와-메모리-릭]]"
---

# 가비지 컬렉션 (GC)

## GC란?

- **Heap 메모리에서 더 이상 참조되지 않는 객체를 자동으로 해제**하는 메커니즘
- C/C++은 수동 (`free`/`delete`) → Java는 GC가 알아서
- `System.gc()` 호출 가능하지만 JVM이 무시할 수 있음 → 프로덕션에서 사용 X

---

## GC의 기본 원리 — Reachability Analysis

GC Root에서 **참조 체인(Reference Chain)** 을 따라가서 도달 가능하면 살아있는 객체, 도달 불가능하면 수거 대상.

```
GC Root → A → B → C  ✅ 도달 가능 = 살아있음
GC Root    X → Y → Z  ❌ 도달 불가 = 수거 대상
```

### GC Root가 될 수 있는 것

- JVM Stack의 지역변수/파라미터 참조
- Method Area의 static 변수 참조
- Method Area의 상수풀 참조
- JNI (Native Method)의 참조
- 활성 스레드 객체
- synchronized로 잡힌 모니터 객체
- JVM 내부 참조 (클래스로더, 예외 객체 등)

### 힙 객체는 GC Root가 될 수 있는가?

기본적으로 **GC Root는 힙 바깥에 있는 참조들**이다. 다만 힙 안의 객체가 사실상 Root처럼 동작하는 경우가 있다:

- **ClassLoader 객체** — 힙에 있지만, 로딩한 클래스들의 생명주기를 좌우
- **활성 Thread 객체** — 힙에 있지만, 실행 중이면 GC Root 취급
- **java.lang.Class 인스턴스** — 해당 클래스의 static 필드를 참조

### Reference Counting vs Tracing

| 방식 | 설명 | 문제 |
|------|------|------|
| Reference Counting | 참조 수 카운트, 0이면 제거 | **순환 참조** 감지 불가 |
| **Tracing (Mark & Sweep)** | GC Root부터 추적 | Java가 사용하는 방식 ✅ |

---

## Heap 메모리 구조

```
Young Generation                    Old Generation
┌────────┬─────────┬─────────┐     ┌──────────────┐
│  Eden  │  S0     │  S1     │     │              │
│        │(From)   │ (To)    │     │              │
│ new 객체│ GC 생존  │         │     │  오래된 객체   │
│  할당   │  객체    │         │     │              │
└────────┴─────────┴─────────┘     └──────────────┘
         ← Minor GC →                ← Major GC →
```

### 객체의 일생

1. `new Object()` → **Eden에 할당**
2. Eden 가득 참 → **Minor GC** 발동 → 살아남은 객체 S0으로, age = 1
3. 다음 Minor GC → Eden + S0 살아남은 객체 → S1으로, S0/S1 역할 교대, age++
4. age가 임계값 초과 (기본 15) → **Old Generation으로 승격(Promotion)**
5. Old 가득 참 → **Major GC(Full GC)** 발동

### GC가 발생하는 시점

**Minor GC:** Eden 영역이 꽉 찼을 때
**Major GC / Full GC:**
- Old Generation이 꽉 찼을 때
- Minor GC 시 Old로 승격할 공간이 부족할 때
- `System.gc()` 호출 시 (보장은 안 됨)
- Metaspace 부족 시

---

## STW (Stop-The-World)

- GC 실행 시 **모든 애플리케이션 스레드가 멈춤**
- Minor GC: STW 짧음 (밀리초)
- Major GC: STW 길 수 있음 (수백ms ~ 초)
- GC 튜닝의 핵심 = STW 시간 최소화

---

## GC 알고리즘 종류

### Serial GC

싱글 스레드로 GC 수행. 단순하고 오버헤드 적지만 STW가 길다. 소규모 앱용.

### Parallel GC (Java 8 기본)

멀티 스레드로 GC 수행. Serial보다 빠르지만 여전히 STW 발생. 처리량(Throughput) 중시 서버에 적합.

### CMS GC (deprecated)

대부분의 GC 작업을 앱 스레드와 동시(Concurrent) 수행. STW 짧지만 CPU 많이 사용하고 메모리 단편화 문제. Java 9부터 deprecated.

### G1 GC (Java 9+ 기본) ⭐

Heap을 동일 크기 **Region**으로 분할하고, 쓰레기가 많은 Region부터 우선 수거.

---

## G1 GC 심화

### Region 기반 구조

기존 GC가 Young/Old를 연속된 물리 공간으로 나누는 것과 달리, G1은 힙을 **동일 크기의 Region**(1~32MB)으로 쪼갠다. 각 Region이 Eden/Survivor/Old 역할을 **유동적으로** 맡는다.

```
┌────┬────┬────┬────┬────┬────┐
│ E  │ S  │ O  │ E  │ H  │ O  │
├────┼────┼────┼────┼────┼────┤
│ O  │ E  │ O  │ S  │ O  │ E  │
├────┼────┼────┼────┼────┼────┤
│ O  │ O  │ E  │ O  │ O  │ O  │
└────┴────┴────┴────┴────┴────┘
E=Eden  S=Survivor  O=Old  H=Humongous
```

- Region 크기: `-XX:G1HeapRegionSize` (기본: 힙을 약 2048개 Region으로 나누는 크기)
- **Humongous Region**: Region 크기의 50% 넘는 큰 객체용. 연속된 여러 Region을 차지할 수 있음

### Region 내부 구조

각 Region은 다음 정보를 가진다:

- **bottom / top / end**: 시작 주소, 현재 할당 위치, 끝 주소
- **type**: Eden / Survivor / Old / Humongous / Free
- **RSet (Remembered Set)**: 나를 참조하는 외부 Region 기록
- **TAMS (Top at Mark Start)**: 마킹 시작 시점의 top 위치

사용량은 `top - bottom`으로 O(1)에 확인 가능. `top == end`면 Region이 꽉 찬 것.

### Region의 생애주기

```
Free → Eden으로 지정 (객체 할당)
     → 가득 참 → Young GC로 수거
     → 살아남은 객체는 다른 Region으로 복사
     → 다시 Free로 반환 → 재활용
```

### 객체 할당과 TLAB

```
new Object()
  → TLAB (Thread Local Allocation Buffer) 확인
    → TLAB 여유 있음 → 바로 할당 (Bump-the-Pointer, Lock 없음!)
    → TLAB 부족 → Eden Region에서 새 TLAB 할당
    → Eden Region 부족 → Free Region을 Eden으로 지정
    → Free Region 없음 → Young GC 발동!
```

**TLAB**: 각 스레드가 Eden Region 안에 자기만의 작은 버퍼를 갖는다. Lock 없이 포인터만 이동하므로 할당이 매우 빠르다.

**Eden이 다 찼는지 감지하는 별도 모니터링 스레드는 없다.** 할당을 시도한 App Thread가 실패하면 GC가 트리거된다.

### G1의 스레드 구조

G1은 멀티스레드로 동작하며 3종류의 GC 스레드가 있다:

**1) Parallel GC Workers** — STW 구간에서 Region을 병렬로 수거
- `-XX:ParallelGCThreads=N`
- 기본값: CPU 코어 8개 이하 → 코어 수, 이상 → 8 + (코어-8) × 5/8

**2) Concurrent Mark Threads** — 앱과 동시에 실행되며 Region별 쓰레기 비율 계산
- `-XX:ConcGCThreads=N`
- 기본값: ParallelGCThreads의 약 1/4

**3) Refinement Threads** — Region 간 참조 관계(RSet)를 비동기적으로 갱신
- `-XX:G1ConcRefinementThreads=N`
- 부하에 따라 스레드 수 자동 조절

### Remembered Set (RSet)과 Write Barrier

Old Region이 Young Region을 참조하면, Young GC 시 Old 전체를 스캔해야 하는 문제가 생긨다. 이를 해결하기 위해 **RSet**을 사용한다.

각 Region마다 **"나를 참조하는 외부 Region 정보"** 를 RSet에 기록한다.

```
참조 변경 발생 (obj.field = newObj)
  → Write Barrier 발동
  → Card Table에 dirty 표시
  → Dirty Card Queue에 추가
  → Refinement Thread가 비동기적으로 RSet 업데이트
```

### Young GC 동작

1. Eden Region이 가득 차면 STW 진입
2. GC Root + RSet 스캔으로 살아있는 객체 식별
3. 살아있는 객체를 새 Survivor Region으로 복사 (Evacuation)
4. age 임계값 넘은 객체는 Old Region으로 승격
5. 원래 Eden/Survivor Region은 Free로 반환
6. STW 해제

### Concurrent Marking 단계

힙 사용률이 임계값(기본 45%)을 넘으면 시작된다.

1. **Initial Mark (STW)** — GC Root만 마킹 (짧음, Young GC에 편승)
2. **Root Region Scan** — Survivor Region 스캔
3. **Concurrent Mark** — 앱과 동시에, Root에서 따라가며 마킹
4. **Remark (STW)** — 마킹 중 변경된 참조 보정
5. **Cleanup** — Region별 쓰레기 비율 확정

### Mixed GC — G1의 핵심

Concurrent Marking으로 계산한 **쓰레기 비율이 높은 Old Region을 우선 수거**한다. "Garbage First" 이름의 유래.

```
목표 pause time: 200ms (-XX:MaxGCPauseMillis)

Collection Set = {
  모든 Young Region (필수)
  + 쓰레기가 가장 많은 Old Region들 (시간 안에 처리 가능한 만큼)
}
```

Mixed GC는 여러 차례 반복되며, 쓰레기 비율이 임계값 이하면 중단된다.

### Region 단위인데 왜 STW가 필요한가?

객체를 다른 Region으로 **복사(Evacuate)** 하고 **참조 주소를 갱신**하는 과정에서, 앱이 동시에 접근하면 데이터 일관성이 깨지기 때문이다.

- 복사 중: 객체가 A에도 B에도 있는 상태 → 어디를 봐야 하나?
- 참조 갱신 중: 옛 주소로 접근하면 잘못된 데이터를 읽음

### G1 전체 사이클

```
Eden 할당 → 가득 참
  → Young GC (STW)
  → Old 비율 45% 초과?
    → Yes: Concurrent Marking → Mixed GC (STW) × 여러 회
    → No: 다시 Eden 할당
  → 할당 속도 > 수거 속도? → Full GC (최후의 수단)
```

---

## ZGC / Shenandoah

G1은 Evacuation 시 STW가 필수지만, ZGC와 Shenandoah는 **Load Barrier / Colored Pointer**로 이 문제를 해결했다.

앱이 이동된 객체에 접근하면 Barrier가 가로채서 자동으로 새 주소로 리다이렉트한다. STW 없이 처리 가능.

| GC | STW | 적합 환경 |
|-----|-----|----------|
| G1 | 수~수백ms | 범용 서버 |
| ZGC | < 1ms | 대용량 힙, 초저지연 |
| Shenandoah | < 10ms | 저지연 |

대신 Barrier 오버헤드로 전체 처리량(throughput)은 약간 떨어질 수 있다.

---

## GC 알고리즘 비교표

| GC | STW | 처리량 | 적합 환경 | Java 기본 |
|----|-----|--------|----------|-----------|
| Serial | 긺 | 낮음 | 소규모/클라이언트 | |
| Parallel | 중간 | **높음** | 배치/처리량 서버 | Java 8 |
| CMS | **짧음** | 중간 | 저지연 서버 | deprecated |
| **G1** | **조절가능** | 높음 | **범용 서버** | **Java 9+** |
| ZGC | **극히 짧음** | 높음 | 대용량/저지연 | Java 15+ |

---

## GC 튜닝 기본

### 모니터링

```bash
# GC 로그 활성화 (Java 9+)
-Xlog:gc*:file=gc.log

# Java 8
-verbose:gc -XX:+PrintGCDetails -Xloggc:/path/to/gc.log
```

### 주요 옵션

| 옵션 | 설명 |
|------|------|
| `-Xms`, `-Xmx` | 힙 크기 (동일하게 설정 권장) |
| `-XX:NewRatio` | Young:Old 비율 |
| `-XX:MaxGCPauseMillis` | G1 목표 STW 시간 |
| `-XX:G1HeapRegionSize` | G1 Region 크기 |

---

## 면접 예상 질문

**Q: GC가 뭔지 설명해주세요.**
> "Heap에서 더 이상 참조되지 않는 객체를 자동으로 해제하는 메커니즘입니다. GC Root에서 참조 체인으로 도달할 수 없는 객체를 수거합니다."

**Q: GC Root에서 도달 가능하다는 게 무슨 뜻인가요?**
> "Stack 지역변수, static 변수, 활성 스레드 등 GC Root에서 참조 체인을 따라갈 수 있으면 살아있는 객체이고, 따라갈 수 없으면 수거 대상입니다."

**Q: G1 GC의 동작 원리를 설명해주세요.**
> "힙을 동일 크기 Region으로 나누고, TLAB으로 빠르게 할당합니다. Concurrent Marking으로 각 Region의 쓰레기 비율을 계산하고, Mixed GC에서 쓰레기가 많은 Region부터 우선 수거합니다. 목표 pause time 안에서 최대한 많이 수거하는 것이 핵심입니다."

**Q: G1에서 Region 간 참조는 어떻게 관리하나요?**
> "각 Region마다 Remembered Set을 두고, Write Barrier와 Refinement Thread를 통해 Region 간 참조 변경을 비동기적으로 추적합니다. 덕분에 Young GC 시 Old 전체를 스캔하지 않아도 됩니다."

**Q: G1은 Region 단위인데 왜 STW가 필요한가요?**
> "객체를 다른 Region으로 복사하고 참조를 갱신하는 과정에서 앱이 동시에 접근하면 일관성이 깨지기 때문입니다. ZGC는 Load Barrier로 이 문제를 해결해 STW를 1ms 이하로 줄였습니다."

**Q: GC 때문에 서비스가 느려졌다면 어떻게 대응?**
> "1) GC 로그로 Full GC 빈도와 STW 시간 파악 2) Heap Dump로 메모리 릭 분석 3) 힙 크기 조정이나 GC 알고리즘 변경 검토 4) 코드에서 불필요한 객체 생성 줄이기"
