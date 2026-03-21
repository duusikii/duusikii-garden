---
title: "API 멱등성 (Idempotency)"
tags:
  - CS
  - API
  - 시스템디자인
  - 면접
created: 2026-03-19
modified: 2026-03-19
related:
  - "[[Redis-심화]]"
  - "[[카프카-Kafka]]"
---

# API 멱등성 (Idempotency)

같은 요청을 여러 번 보내도 결과가 동일한 것.

```
비유:
  엘리베이터 버튼: 1번 누르든 10번 누르든 결과 같음 = 멱등 ✅
  커피 주문 버튼: 누를 때마다 새 주문 = 멱등 아님 ❌
```

---

## HTTP 메서드별 멱등성

| 메서드 | 멱등? | 이유 |
|--------|-------|------|
| GET | ✅ | 조회만 함 |
| PUT | ✅ | 전체 덮어쓰기 → 몇 번 해도 결과 같음 |
| DELETE | ✅ | 이미 없으면 없는 거 |
| POST | ❌ | 할 때마다 새 리소스 생성 |
| PATCH | ❌ | 상대적 변경 가능 ("나이 +1") |

---

## 왜 중요한가?

```
결제 API → 서버 처리 완료 → 응답 중 네트워크 끊김
→ 클라이언트: "응답 안 왔네? 다시 보내자"
→ 멱등성 없으면: 2번 결제! 💥
→ 멱등성 있으면: 이미 처리된 거 확인 → 결과만 반환 ✅
```

---

## 구현 방법

### 1. Idempotency Key

```
클라이언트가 고유 키를 생성해서 헤더에 포함

POST /payments
Headers: Idempotency-Key: "abc-123-xyz"

서버:
  ① 키로 Redis/DB 조회 → 이미 처리했나?
  ② 없으면 → 처리 후 키+결과 저장
  ③ 있으면 → 저장된 결과 그대로 반환
```

### 2. DB 유니크 제약

```sql
CREATE TABLE payments (
    id BIGINT PRIMARY KEY,
    order_id VARCHAR(50) UNIQUE -- 같은 주문은 1번만!
);
```

### 3. 상태 확인

```java
public void completeOrder(Long orderId) {
    Order order = orderRepository.findById(orderId);
    if (order.getStatus() == COMPLETED) return; // 이미 처리됨
    order.setStatus(COMPLETED);
    orderRepository.save(order);
}
```

---

## 멱등키 생성 규칙

```
핵심: "하나의 유저 의도 = 하나의 멱등키"

생성: 유저가 버튼 클릭 시 UUID 생성
재사용: 네트워크 실패로 재시도 시 같은 키
만료: Redis에 TTL 24시간으로 저장

같은 상품을 다시 구매? → 새 멱등키 → 별개 주문
같은 요청 재시도? → 같은 멱등키 → 중복 방지
```

### 생성 방법

```
방법 1: UUID
  → 클라이언트가 random UUID 생성, 간단

방법 2: 비즈니스 키 조합
  → userId + orderId + amount
  → 비즈니스 로직에 맞는 중복 방지
```

---

## Kafka 멱등성과 연결

```
Kafka Idempotent Producer도 같은 원리!
  PID + Seq로 "이미 처리한 메시지인가?" 확인
  → 중복이면 무시

API 멱등성 = Idempotency Key
Kafka 멱등성 = PID + Sequence Number
→ 본질적으로 같은 개념!
```

---

## 면접 예상 질문

**Q: API 멱등성이란?**
> "같은 요청을 여러 번 보내도 결과가 동일한 성질입니다. 네트워크 장애로 재시도 시 중복 처리를 방지합니다."

**Q: POST의 멱등성을 어떻게 보장하나요?**
> "Idempotency-Key를 헤더에 포함하고, 서버가 Redis에 키와 결과를 저장합니다. 동일한 키로 재요청 시 저장된 결과를 반환합니다."
