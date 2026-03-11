---
title: 가비지 컬렉션 (GC)
tags: [Java, JVM, GC, 면접, CS]
---

# 가비지 컬렉션 (GC)

## GC란?
- **Heap 메모리에서 더 이상 참조되지 않는 객체를 자동으로 해제**하는 메커니즘
- C/C++은 수동 (`free`/`delete`) → Java는 GC가 알아서
- 개발자가 `System.gc()` 호출 가능하지만 **권장 X** (JVM이 무시할 수 있음)

---

## GC의 기본 원리

### 어떤 객체를 수거할까? — Reachability

```
GC Root에서 참조 체인(Reference Chain)으로 
도달할 수 있으면 → 살아있음 (Live)
도달할 수 없으면 → 쓰레기 (Garbage) → 수거 대상
```

**GC Root가 될 수 있는 것:**
- Stack의 지역변수
- Static 변수
- JNI 참조
- 활성 스레드

### Reference Counting vs Tracing
| 방식 | 설명 | 문제 |
|------|------|------|
| Reference Counting | 참조 수 카운트, 0이면 제거 | **순환 참조** 감지 불가 |
| **Tracing (Mark & Sweep)** | GC Root부터 추적 | Java가 사용하는 방식 ✅ |

---

## Heap 메모리 구조와 GC 흐름

```
Young Generation                    Old Generation
┌────────┬─────────┬─────────┐     ┌──────────────┐
│  Eden  │  S0     │  S1     │     │              │
│        │(From)   │ (To)    │     │              │
│ new 객체│ GC 생존  │         │     │  오래된 객체    │
│  할당   │  객체    │         │     │              │
└────────┴─────────┴─────────┘     └──────────────┘
         ← Minor GC →                ← Major GC →
                                   (= Full GC)
```

### 객체의 일생

```
1. new Object() → Eden에 할당

2. Eden 가득 참 → Minor GC 발동
   - 살아있는 객체 → S0(From)으로 이동
   - 나머지 → 삭제
   - age = 1

3. 다음 Minor GC
   - Eden + S0 살아있는 객체 → S1(To)로 이동
   - S0, S1 역할 교대 (From ↔ To)
   - age++

4. age가 임계값 초과 (기본 15)
   → Old Generation으로 승격 (Promotion)

5. Old Generation 가득 참
   → Major GC (Full GC) 발동 — 느림! STW 김!
```

---

## STW (Stop-The-World)

- GC 실행 시 **모든 애플리케이션 스레드가 멈춤**
- Minor GC: STW 짧음 (밀리초)
- Major GC: STW 길 수 있음 (수백ms ~ 초)
- **GC 튜닝의 핵심 = STW 시간 최소화**

---

## GC 알고리즘 종류

### 1. Serial GC (`-XX:+UseSerialGC`)
```
싱글 스레드로 GC 수행
장점: 단순, 오버헤드 적음
단점: STW 김
용도: 소규모 앱, 클라이언트
```

### 2. Parallel GC (`-XX:+UseParallelGC`)
```
멀티 스레드로 GC 수행 (Java 8 기본)
장점: Serial보다 빠름
단점: 여전히 STW 발생
용도: 처리량(Throughput) 중시 서버
```

### 3. CMS GC (`-XX:+UseConcMarkSweepGC`)
```
대부분의 GC 작업을 앱 스레드와 동시(Concurrent) 수행
장점: STW 짧음
단점: CPU 많이 사용, 메모리 단편화
용도: 지연시간 중시 앱
상태: Java 9부터 deprecated ⚠️
```

### 4. G1 GC (`-XX:+UseG1GC`) ⭐
```
Java 9+ 기본 GC
Heap을 동일 크기 Region으로 분할
쓰레기가 많은 Region부터 우선 수거 (Garbage First)

장점: 
  - 예측 가능한 STW 시간 (-XX:MaxGCPauseMillis=200)
  - 대용량 Heap에 적합
  - CMS의 메모리 단편화 문제 해결

Region 구조:
┌───┬───┬───┬───┬───┬───┐
│ E │ S │ O │ E │ H │ O │  E=Eden, S=Survivor
├───┼───┼───┼───┼───┼───┤  O=Old, H=Humongous
│ O │ E │   │ O │ E │ S │  (빈칸=Free)
└───┴───┴───┴───┴───┴───┘
```

### 5. ZGC / Shenandoah (최신)
```
Java 15+
STW 10ms 이하 목표
대용량 Heap (수 TB)에서도 저지연
아직 실무 도입은 점진적
```

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

### 모니터링 먼저
```bash
# GC 로그 활성화
-verbose:gc
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/path/to/gc.log

# Java 9+
-Xlog:gc*:file=gc.log
```

### 튜닝 포인트
1. **Heap 크기 적절히** (`-Xms`, `-Xmx` 동일하게 설정 권장)
2. **Young:Old 비율** (`-XX:NewRatio`)
3. **GC 알고리즘 선택** (보통 G1이면 충분)
4. **목표 STW 시간** (`-XX:MaxGCPauseMillis`)

---

## 🎤 면접 예상 질문

**Q: GC가 뭔지 설명해주세요.**
> "Heap에서 더 이상 참조되지 않는 객체를 자동으로 메모리 해제하는 메커니즘입니다. GC Root에서 참조 체인으로 도달할 수 없는 객체를 Mark & Sweep 방식으로 수거합니다."

**Q: Minor GC와 Major GC(Full GC)의 차이?**
> "Minor GC는 Young Generation(Eden + Survivor)을 대상으로 하며 빠릅니다. Major GC(Full GC)는 Old Generation을 포함한 전체 Heap을 대상으로 하며, STW 시간이 길어 성능에 영향을 줍니다."

**Q: G1 GC의 특징은?**
> "Heap을 동일 크기 Region으로 나누고, 쓰레기가 많은 Region부터 우선 수거합니다. 목표 STW 시간을 설정할 수 있어 예측 가능한 지연 시간을 제공합니다. Java 9부터 기본 GC입니다."

**Q: GC 때문에 서비스가 느려졌다면 어떻게 대응?**
> "1) GC 로그를 먼저 확인해서 Full GC 빈도와 STW 시간을 파악합니다. 2) 메모리 누수가 있는지 Heap Dump로 분석합니다. 3) 힙 크기 조정이나 GC 알고리즘 변경(G1, ZGC)을 검토합니다. 4) 코드 레벨에서 불필요한 객체 생성을 줄입니다."

**Q: `System.gc()`를 호출하면 GC가 바로 실행되나요?**
> "아닙니다. `System.gc()`는 JVM에 GC를 요청하는 것일 뿐, JVM이 무시할 수 있습니다. 프로덕션 코드에서 직접 호출하는 것은 권장하지 않습니다."
