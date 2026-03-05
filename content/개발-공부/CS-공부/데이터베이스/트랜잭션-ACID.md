---
title: 트랜잭션 & ACID
tags: [CS, DB, 면접]
---

# 트랜잭션 & ACID

## ACID 속성
| 속성 | 설명 |
|------|------|
| **Atomicity** (원자성) | 전부 성공 or 전부 실패 |
| **Consistency** (일관성) | 트랜잭션 전후 DB 무결성 유지 |
| **Isolation** (격리성) | 동시 트랜잭션이 서로 영향 없음 |
| **Durability** (지속성) | 커밋된 데이터는 영구 저장 |

## 격리 수준 (Isolation Level)
| 레벨 | Dirty Read | Non-repeatable | Phantom |
|------|-----------|----------------|---------|
| READ UNCOMMITTED | ✅ 발생 | ✅ | ✅ |
| READ COMMITTED | ❌ | ✅ 발생 | ✅ |
| REPEATABLE READ | ❌ | ❌ | ✅ 발생 |
| SERIALIZABLE | ❌ | ❌ | ❌ |

## 락 (Lock)
- **공유락(Shared Lock)**: 읽기 시 — 다른 읽기 허용
- **배타락(Exclusive Lock)**: 쓰기 시 — 모든 접근 차단
- **낙관적 락**: 충돌 적다고 가정, 커밋 시 버전 확인
- **비관적 락**: 충돌 많다고 가정, SELECT FOR UPDATE

## ❓ 면접 질문
**Q. Dirty Read란?**
> 커밋 안 된 데이터를 다른 트랜잭션이 읽는 것.

**Q. MySQL 기본 격리 수준은?**
> REPEATABLE READ. Phantom Read는 MVCC로 어느 정도 방지.
