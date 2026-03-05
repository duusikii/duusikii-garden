---
title: "실전: Flash Sale 설계"
tags: [CS, 시스템디자인, Redis, 면접]
---

# 실전: Flash Sale 설계

100개 한정 초특가 이벤트 시스템.

## 핵심 문제
동시에 1,000명이 구매 요청 → 100개만 팔아야 함 → Race Condition 방지

## 아키텍처

```
사용자 1000명
→ Rate Limiter (봇/도배 차단)
→ Load Balancer
→ API 서버 (n대)
→ Redis (재고 제어 핵심)
  ① SISMEMBER sold_users — 중복 체크
  ② DECR flash:stock — 원자적 재고 감소
  ③ SET lock:user:{id} nx ex=5 — 분산 락
→ stock ≥ 0: DB 주문 생성 → 결제
→ stock < 0: 즉시 "품절" 거절
```

## Redis가 핵심인 이유
- `DECR`는 **원자적 연산** → 동시 요청도 정확히 1씩 감소
- 분산 락으로 같은 유저 동시 요청 차단
- 빠른 인메모리 처리

## 결제 실패 롤백
```
실패 시:
→ DB status = failed
→ INCR flash:stock (+1 복구)
→ SREM sold_users (유저 제거)
→ 사용자 실패 알림
```

## 큐 버전과의 차이
- **락만**: 즉시 거절, 구현 단순
- **락 + 큐**: 대기열 UX 제공 가능, 구현 복잡

## 관련 노트
- [[캐시전략]]
- [[분산시스템]]
