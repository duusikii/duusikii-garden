---
title: "Redis 심화"
tags:
  - CS
  - Redis
  - 캐시
  - 면접
date: 2026-03-20
related:
  - "[[캐시전략]]"
  - "[[카프카-Kafka]]"
---

# Redis 심화

## Redis란?

Remote Dictionary Server. 인메모리 키-값 저장소.

```
DB 조회: 디스크 I/O → 수~수십 ms
Redis:   메모리 접근 → 수십~수백 μs
→ 약 100배 빠름!
```

---

## 핵심 특징

- 인메모리 → 빠름
- 싱글 스레드 (명령 실행) → 원자성 보장
- 다양한 자료구조 → String, List, Set, Hash, Sorted Set
- 영속성 옵션 → RDB, AOF
- TTL 지원 → 키 만료 시간 설정

---

## 자료구조별 활용

### String

```
SET user:1:name "시은"
GET user:1:name → "시은"
INCR page:view:count → 원자적 증가
SETEX session:abc 3600 "userData" → 1시간 후 만료
```

활용: 세션, API Rate Limiting, 조회수 카운터

### Hash

```
HSET user:1 name "시은" age 25 role "backend"
HGET user:1 name → "시은"
HGETALL user:1 → {name, age, role}
```

활용: 유저 프로필, 상품 정보 (부분 수정 가능!)

### List

```
LPUSH queue:orders "order1"
RPOP queue:orders → FIFO 큐
```

활용: 메시지 큐, 최근 본 상품

### Set

```
SADD likes:post:1 "user:1" "user:2"
SISMEMBER likes:post:1 "user:1" → true
SINTER likes:post:1 likes:post:2 → 교집합
```

활용: 좋아요, 중복 방지, 친구 목록

### Sorted Set (ZSet)

```
ZADD leaderboard 100 "시은"
ZADD leaderboard 200 "두식"
ZREVRANGE leaderboard 0 2 → 점수 높은 순
```

활용: 리더보드, 실시간 랭킹, 인기 검색어

---

## 싱글 스레드인데 왜 빠른가?

### I/O Multiplexing (epoll)

```
커넥션 1만 개가 연결되어 있어도
→ 실제 데이터가 도착한 커넥션만 처리!

OS 커널이 소켓을 감시하다가
→ 데이터 도착하면 Redis에게 알려줌
→ Redis는 준비된 것만 순차 처리
```

### epoll 동작 원리

```
Redis: epoll_wait() 호출 → sleep (CPU 안 씀!)

... 시간 흐름 ...

NIC에 패킷 도착 → 하드웨어 인터럽트
→ 커널이 TCP 처리 → 소켓 버퍼에 저장
→ epoll ready list에 추가 → Redis 깨움!

Redis: 깨어남! → read → execute → write
→ 다시 epoll_wait() → sleep

→ busy waiting이 아니라 이벤트 기반!
  이름이 epoll이지만 실제로는 polling 아님
```

### 커넥션 유지는 누가?

```
TCP 핸드셰이크: OS 커널이 인터럽트로 처리
소켓 감시: OS 커널이 epoll로 관리
버퍼 관리: OS 커널 메모리

→ Redis 스레드는 "명령 실행"만!
→ 나머지는 전부 OS 커널!
→ 커넥션 1만 개 유지해도 Redis CPU ≈ 0%
```

### Redis 6.0 이후 변경점

```
6.0 이전: read + execute + write 전부 메인 스레드
6.0 이후: read/write는 I/O 스레드, execute만 메인 스레드

→ 네트워크 I/O만 멀티스레드로 분리
→ 명령 실행은 여전히 싱글 → 원자성 보장!
→ 성능 약 2배 향상 (~20만 TPS)
```

### 싱글 스레드 주의점

```
O(N) 명령이 전체를 블로킹!

KEYS * → 프로덕션에서 절대 금지!
→ 대안: SCAN (커서 기반 점진적 스캔)

SCAN은 해시 테이블 버킷을 조금씩 방문
→ 중간에 다른 요청 처리 가능
→ 총 시간은 O(N)이지만 블로킹 없음
```

---

## 영속성 (Persistence)

### RDB (Redis Database)

```
특정 시점의 스냅샷을 디스크에 저장
save 900 1 (900초 동안 1번 이상 변경 시)

장점: 복구 빠름, 파일 크기 작음
단점: 스냅샷 사이 데이터 유실 가능
```

### AOF (Append Only File)

```
모든 쓰기 명령을 로그로 기록
appendfsync everysec (매초 동기화)

장점: 유실 최소화
단점: 파일 크기 큼, 복구 느림
```

실무에서는 **RDB + AOF 둘 다 사용**.

---

## 효율적인 키 조회

```
KEYS * → O(N) 절대 금지
SCAN → O(N) 블로킹 없음, 어쩔 수 없을 때
GET → O(1) 키 이름 알 때
Set → O(M) 특정 그룹 목록
Sorted Set → O(logN+M) 범위/랭킹 조회

핵심: "검색하지 말고, 찾을 수 있게 설계하라!"
  → 키 네이밍 규칙, Set 인덱스, Hash 그룹핑
```

---

## 부하 대응

### Redis Cluster (데이터 분산)

```
16384개 슬롯을 여러 노드가 나눠 저장 (샤딩)
키 → CRC16 해시 → 슬롯 번호 → 해당 노드

→ 쓰기/읽기 모두 분산, 저장 용량 N배
→ 잘못된 노드 요청 시 MOVED 리다이렉트
→ Hash Tag {user}:1로 같은 슬롯 배치 가능
```

### Read Replica (읽기 분산)

```
같은 데이터를 복사해서 읽기 전용 노드 생성

Master: 쓰기 + 읽기
Replica 1~N: 읽기만

→ 읽기 부하가 N배 분산
→ Master 죽으면 Replica가 승격
```

### Cluster + Replica 조합

```
실무에서는 둘 다 씀!
각 Cluster 노드마다 Replica를 둠
→ 데이터 분산 + 읽기 분산 + 장애 대비
```

### 기타

- **로컬 캐시 (L1+L2)**: Caffeine → Redis → DB 순서로 조회. Redis 부하 80% 감소
- **Pipeline**: 여러 명령을 한 번에 전송. 네트워크 왕복 횟수 줄임. 100개 명령 시 50배 차이
- **데이터 구조 최적화**: 개별 키 대신 Hash로 묶으면 메모리 최대 70% 절약

---

## Redis Pub/Sub

```
발행/구독 메시징. 구독자 전원에게 메시지 전달.

PUBLISH "order-events" "주문 #123 생성"
→ 구독 중인 모든 서버에 전달

주의: 메시지 저장 안 함!
→ 구독자 없으면 유실
→ 실시간 알림, 캐시 무효화 브로드캐스트에 적합
→ 안정적 처리가 필요하면 Kafka 사용
```

## Redis Streams

```
Kafka처럼 메시지를 저장하는 자료구조 (5.0+)
Consumer Group 지원, 재처리 가능

Pub/Sub: 저장 ❌, 재처리 ❌
Streams: 저장 ✅, 재처리 ✅ (메모리 기반이라 대용량은 Kafka)
```

---

## Lua 스크립트

```
여러 명령을 하나의 원자적 연산으로 실행

싱글 스레드라 스크립트 실행 중 다른 명령이 끼어들 수 없음!

활용:
  Rate Limiting: GET + INCR + EXPIRE 원자적으로
  재고 차감: 확인 + 차감 원자적으로
  분산 락 해제: 내 락인지 확인 + 삭제 원자적으로

⚠️ 스크립트가 오래 걸리면 전체 블로킹!
```

---

## Redisson

Java용 고수준 Redis 클라이언트 라이브러리.

### Lettuce/Jedis vs Redisson

```
Lettuce: 저수준 (GET, SET 직접 호출)
Redisson: 고수준 (Java 자료구조처럼 사용)
  → RLock, RMap, RQueue, RAtomicLong 등 제공
```

### Watchdog (분산 락 핵심!)

```
문제: 락 TTL=30초, 작업이 40초 걸리면 락 만료!

Watchdog: 매 10초마다 TTL을 30초로 갱신
→ 작업 중에는 락 유지
→ 프로세스 죽으면 갱신 멈춤 → TTL 만료 → 자동 해제
→ 데드락 방지!
```

---

## 현업 주의사항 체크리스트

| # | 항목 | 확인 |
|---|------|------|
| 1 | maxmemory + eviction 정책 설정 | allkeys-lru 권장 |
| 2 | Big Key 금지 | String 1MB, Collection 1만 개 이상 주의 |
| 3 | Hot Key 대응 | 로컬 캐시 병행, 키 분산 |
| 4 | 커넥션 풀 설정 | Lettuce pool 설정 |
| 5 | JSON 직렬화 | Java 기본 직렬화 ❌ |
| 6 | 장애 대비 | Redis 없어도 서비스 동작 (Graceful Degradation) |
| 7 | 타임아웃 | 200ms로 짧게 |
| 8 | KEYS * | 프로덕션에서 절대 금지 |
| 9 | 네이밍 컨벤션 | 서비스:엔티티:ID 통일 |
| 10 | TTL | 모든 캐시에 설정 |

---

## 면접 예상 질문

**Q: Redis가 싱글 스레드인데 어떻게 수만 커넥션을 처리하나요?**
> "I/O Multiplexing의 epoll을 사용합니다. 데이터가 도착한 커넥션만 OS가 알려주고, Redis는 그것만 순차 처리합니다. 6.0부터는 네트워크 I/O를 멀티스레드로 분리해 처리량을 높였습니다."

**Q: Redis Cluster와 Read Replica의 차이?**
> "Cluster는 데이터를 여러 노드에 분산 저장(샤딩)하여 쓰기/읽기 모두 분산합니다. Replica는 같은 데이터를 복사하여 읽기만 분산합니다. 실무에서는 둘을 조합합니다."

**Q: Redisson의 Watchdog이란?**
> "분산 락의 TTL을 주기적으로 갱신하여 작업 중 락 만료를 방지합니다. 프로세스가 죽으면 갱신이 멈춰 TTL 만료 후 자동 해제됩니다."

**Q: KEYS 명령을 프로덕션에서 왜 쓰면 안 되나요?**
> "싱글 스레드라 O(N) 스캔 중 다른 모든 요청이 블로킹됩니다. SCAN을 사용하면 점진적으로 처리하면서 다른 요청도 수행할 수 있습니다."
