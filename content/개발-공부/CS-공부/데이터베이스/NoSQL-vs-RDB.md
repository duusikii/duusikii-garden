---
title: NoSQL vs RDB
tags: [CS, DB, 면접]
---

# NoSQL vs RDB

## 비교
| | RDB (MySQL/PG) | NoSQL (MongoDB/Redis) |
|-|----------------|----------------------|
| 스키마 | 고정 | 유연 |
| 관계 | JOIN 가능 | 어려움 |
| 확장 | 수직 확장 | 수평 확장 |
| ACID | 완전 지원 | 일부 지원 |
| 적합 | 정형 데이터, 복잡 쿼리 | 비정형, 대용량, 빠른 읽기 |

## NoSQL 종류
- **Document**: MongoDB — JSON 형태 저장
- **Key-Value**: Redis — 캐시, 세션
- **Column**: Cassandra — 대용량 시계열
- **Graph**: Neo4j — 관계 데이터

## CAP 정리
- **C**onsistency (일관성)
- **A**vailability (가용성)
- **P**artition Tolerance (분단 내성)
- 분산 시스템에서 3가지 동시 보장 불가, 2가지만 선택 가능

## ❓ 면접 질문
**Q. Redis를 캐시로 쓸 때 주의할 점은?**
> Cache Stampede: 캐시 만료 시 다수 요청이 DB에 몰림.
> → 캐시 갱신 Lock, TTL 랜덤화로 방지.

**Q. 어떤 상황에서 NoSQL을 선택하나요?**
> 스키마 변경이 잦거나, 수평 확장이 필요하거나, 읽기가 압도적으로 많을 때.
