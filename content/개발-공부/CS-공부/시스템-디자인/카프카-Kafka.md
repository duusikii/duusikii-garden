---
title: "카프카 (Apache Kafka)"
tags:
  - Kafka
  - 메시지큐
  - 시스템디자인
  - 면접
date: 2026-03-15
related:
  - "[[메시지큐]]"
  - "[[분산시스템]]"
---

# 카프카 (Apache Kafka)

## Kafka란?

**분산 이벤트 스트리밍 플랫폼.** 대용량 데이터를 실시간으로 안전하게 전달하는 파이프라인.

LinkedIn에서 만들고 Apache에 기증. Producer가 메시지를 Topic에 발행하고 Consumer가 구독하는 Pub/Sub 모델.

---

## 왜 Kafka를 쓰는가?

### 직접 통신의 문제

```
주문서비스 → 결제서비스
주문서비스 → 재고서비스
주문서비스 → 알림서비스

서비스가 10개면 → 10 × 9 = 90개 연결
하나 죽으면 → 데이터 유실
트래픽 폭증 → 받는 쪽 과부하
```

### Kafka 도입 후

```
주문서비스 → [Kafka] → 결제서비스
                     → 재고서비스
                     → 알림서비스

Producer      Broker      Consumer
```

**핵심 이점:**
- **디커플링** — Producer/Consumer가 서로 몰라도 됨
- **버퍼링** — Consumer가 느려도 Kafka가 들고 있음
- **내구성** — 디스크에 저장, 서버 죽어도 데이터 안 날아감
- **확장성** — 브로커 추가하면 처리량 증가

---

## 핵심 구성 요소

```
┌─────────────────────────────────────────────┐
│               Kafka Cluster                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │ Broker 1 │ │ Broker 2 │ │ Broker 3 │    │
│  │ Part 0   │ │ Part 1   │ │ Part 2   │    │
│  │ (Leader) │ │ (Leader) │ │ (Leader) │    │
│  └──────────┘ └──────────┘ └──────────┘    │
│          ZooKeeper / KRaft                   │
└─────────────────────────────────────────────┘
```

- **Producer** — 메시지를 Topic에 보내는 쪽
- **Consumer** — Topic에서 메시지를 가져가는 쪽
- **Broker** — Kafka 서버. 메시지를 저장하고 전달
- **Topic** — 메시지의 카테고리 (예: `order-events`)
- **Partition** — Topic을 쪼갠 단위. 병렬 처리의 핵심

---

## Partition

### 왜 필요한가?

```
파티션 1개: Producer → [msg1|msg2|msg3] → Consumer 1개 (처리량 제한)

파티션 3개: Producer → [Part 0] → Consumer A
                    → [Part 1] → Consumer B
                    → [Part 2] → Consumer C
                    (병렬 처리, 처리량 3배)
```

### 파티션 내부 구조

```
Partition 0:
┌─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │ ← Offset (불변의 순번)
└─────┴─────┴─────┴─────┴─────┘
                          ↑ Consumer가 여기까지 읽음
```

- **Offset** — 각 메시지의 고유 순번. 절대 바뀌지 않음
- **같은 파티션 안에서만 순서 보장**

### 어떤 파티션에 들어가는가?

```java
producer.send(new ProducerRecord<>("topic", key, value));

// key가 있으면: hash(key) % partitionCount
//   → 같은 key = 항상 같은 파티션 = 순서 보장!

// key가 없으면: Round-Robin
```

> "주문 ID를 파티션 키로 사용하면 같은 주문의 이벤트는 항상 같은 파티션에 들어가서 순서가 보장됩니다"

---

## Consumer Group

```
Topic: order-events (파티션 3개)

Consumer Group A (주문처리팀)
  Consumer A-1 ← Part 0
  Consumer A-2 ← Part 1
  Consumer A-3 ← Part 2
  → 각자 다른 파티션 담당, 병렬 처리!

Consumer Group B (분석팀)
  Consumer B-1 ← Part 0, 1, 2
  → 같은 데이터를 독립적으로 소비
```

**핵심 규칙:**
- 같은 Group 내에서 하나의 파티션은 **하나의 Consumer만** 읽음
- 다른 Group은 같은 데이터를 **독립적으로** 읽을 수 있음
- Consumer 수 > 파티션 수 → 놀고 있는 Consumer 발생

---

## Replication — 데이터 안전성

```
Partition 0 (Replication Factor = 3)

Broker 1: [Part 0 - Leader]   ← 읽기/쓰기 여기서
Broker 2: [Part 0 - Follower] ← Leader 복제
Broker 3: [Part 0 - Follower] ← Leader 복제

Broker 1 죽으면 → Follower가 새 Leader로 승격
```

### ISR (In-Sync Replicas)

Leader와 동기화가 잘 되고 있는 Follower 목록.

```
Producer의 acks 설정:
  acks=0   → 응답 안 기다림 (빠르지만 유실 가능)
  acks=1   → Leader만 확인
  acks=all → 모든 ISR 확인 (느리지만 안전)
```

---

## 메시지 전달 보장

| 보장 수준 | 설명 | 사용처 |
|-----------|------|--------|
| At Most Once | 최대 1번 (유실 가능) | 로그 |
| At Least Once | 최소 1번 (중복 가능) | 일반적 |
| Exactly Once | 정확히 1번 | 금융/결제 |

---

## Idempotent Producer — 단일 파티션 중복 방지

`enable.idempotence=true` 설정하면 PID + Sequence Number로 중복을 감지한다.

### PID (Producer ID)
- **Broker가 부여**. Producer가 최초 연결할 때 자동으로 받음
- Producer 재시작하면 새 PID 부여됨

### Sequence Number
- **Producer가 파티션별로 0부터 순차적으로 매김**
- Sender Thread(단일 스레드)가 Batch 전송 시 부여하므로 충돌 없음

### 중복 감지 원리

```
Producer (PID=5):
  msg + Seq=0 → Broker: "저장!" (마지막 Seq=0 기록)
  msg + Seq=1 → Broker: "저장!" (마지막 Seq=1 기록)
  msg + Seq=1 → Broker: "이미 있네? 무시!" (중복 감지)
  msg + Seq=5 → Broker: "1 다음인데 5? 에러!" (유실 감지)
```

Broker는 Seq를 만들지 않고, **마지막에 받은 Seq를 기억하고 검증만 한다.**

### Producer 내부 구조 — Seq 동기화

```
Thread A ──send()──┐
Thread B ──send()──┤→ RecordAccumulator (파티션별 Batch)
Thread C ──send()──┘           │
                          Sender Thread (단일 스레드)
                          Seq 부여 + Broker 전송

→ 여러 스레드가 send() 해도
→ Batch에 모여서 Sender Thread 하나가 Seq 매김
→ 구조 자체가 충돌이 안 나게 설계됨
```

### Idempotence의 한계

- **단일 파티션 내에서만** 중복 방지
- 여러 파티션에 걸친 원자적 쓰기 보장 못 함
- Producer **재시작하면 새 PID** → 이전 세션과 중복 감지 불가

---

## Transaction — Exactly Once 보장

Idempotence의 한계를 해결하기 위해 Transaction을 사용한다.

### At Least Once vs Exactly Once 설정 비교

| 설정 | At Least Once | Exactly Once |
|------|---------------|--------------|
| acks | all | all |
| enable.idempotence | false | **true** |
| transactional.id | 없음 | **설정** |
| enable.auto.commit | true | **false** |
| isolation.level | (기본) | **read_committed** |

### 원자적 쓰기 원리

Kafka 내부에 `__transaction_state`라는 특수 토픽이 있어서, 여기에 트랜잭션 상태를 기록하면서 원자성을 보장한다.

```java
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(topicA, 메시지A);
    producer.send(topicB, 메시지B);
    producer.commitTransaction();   // 둘 다 COMMIT
} catch (Exception e) {
    producer.abortTransaction();    // 둘 다 ABORT
}
```

### 동작 단계

```
① beginTransaction()
   → __transaction_state: { txId: "order-tx-1", status: ONGOING }

② send()
   → 파티션에 메시지가 쓰이지만 "미확정" 상태 [🔒]

③-A commitTransaction()
   → __transaction_state: PREPARE_COMMIT
   → 각 파티션에 COMMIT 마커 기록 [✅]
   → Consumer가 읽을 수 있게 됨

③-B abortTransaction()
   → 각 파티션에 ABORT 마커 기록 [❌]
   → Consumer가 무시함
```

### Consumer의 isolation.level

```
read_uncommitted (기본): 미확정 메시지도 읽음
read_committed:          COMMIT 마커가 있는 메시지만 읽음
```

### Producer가 죽으면?

| 죽는 시점 | 결과 |
|-----------|------|
| send 도중 | ABORT (전부 취소) |
| commit 전 | ABORT (전부 취소) |
| commit 도중 | Coordinator가 이어서 COMMIT 완료 |
| commit 후 | 이미 끝남, 문제 없음 |

어느 시점에 죽어도 "반쪽짜리 상태"가 안 생긴다.

### Zombie Fencing — 좀비 Producer 방지

```
Producer A (TxId="order-tx", Epoch=1): 느려져서 응답 없음
→ 새 Producer B (TxId="order-tx", Epoch=2) 시작
→ Producer A가 살아나서 메시지 보내려 함
→ Broker: "Epoch=1? 지금 Epoch=2인데? 거부!"
→ 좀비 Producer의 중복 쓰기 방지
```

---

## Kafka vs RabbitMQ vs Redis Pub/Sub

| | Kafka | RabbitMQ | Redis PubSub |
|--|-------|----------|-------------|
| 모델 | Pull | Push | Push |
| 저장 | 디스크 | 메모리+디스크 | 메모리만 |
| 재소비 | 가능 | 불가 | 불가 |
| 처리량 | 초당 수백만 | 초당 수만 | 초당 수십만 |
| 순서보장 | 파티션 내 | 큐 내 | X |
| 적합 | 이벤트 스트림 | 작업 큐 | 실시간 알림 |

---

## 실무 활용 패턴

1. **이벤트 드리븐 아키텍처** — 주문 생성 → [order-created] → 결제/재고/알림 각자 처리
2. **CDC (Change Data Capture)** — DB 변경 → Kafka → 검색엔진(ES) 동기화
3. **로그 수집** — 서버 로그 → Kafka → ELK 스택
4. **실시간 데이터 파이프라인** — 유저 행동 → Kafka → 실시간 분석/추천

---

## 면접 예상 질문

**Q: Kafka란?**
> "분산 이벤트 스트리밍 플랫폼으로, Producer가 메시지를 Topic에 발행하고 Consumer가 구독하는 Pub/Sub 모델입니다. 디스크에 메시지를 보관하므로 재소비가 가능하고, 파티션을 통해 수평 확장합니다."

**Q: 파티션이 왜 필요한가요?**
> "병렬 처리를 위해서입니다. 파티션 수만큼 Consumer를 늘릴 수 있어 처리량이 증가합니다. 같은 파티션 키의 메시지는 같은 파티션에 들어가서 순서가 보장됩니다."

**Q: Exactly Once를 어떻게 보장하나요?**
> "Idempotent Producer로 단일 파티션 내 중복을 방지하고, Transaction으로 여러 파티션에 원자적 쓰기를 보장합니다. Consumer는 read_committed로 커밋된 메시지만 읽고 수동 커밋합니다."

**Q: Idempotent Producer만으로 Exactly Once가 안 되나요?**
> "Idempotent Producer는 PID+Sequence로 단일 파티션 내 중복만 방지합니다. 여러 파티션에 걸친 원자적 쓰기는 보장하지 못하고, Producer 재시작 시 새 PID가 부여되어 이전 세션과 중복 감지가 불가능합니다."

**Q: Transaction이 원자적 쓰기를 어떻게 보장하나요?**
> "Transaction Coordinator가 __transaction_state 토픽에 상태를 기록합니다. 메시지는 파티션에 미확정 상태로 먼저 쓰이고, 커밋 시 각 파티션에 COMMIT 마커를 기록합니다. read_committed Consumer는 COMMIT 마커가 있는 메시지만 읽어서 원자성이 보장됩니다."

**Q: Kafka와 RabbitMQ 차이?**
> "Kafka는 이벤트 로그를 디스크에 보관해서 재소비가 가능하고, Consumer가 Pull 방식으로 자기 속도에 맞게 처리합니다. RabbitMQ는 메시지 큐로 한 번 소비하면 사라지고, Broker가 Push합니다."
