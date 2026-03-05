---
title: REST & WebSocket
tags: [CS, 네트워크, 면접]
---

# REST & WebSocket

## REST API 원칙
1. **Stateless**: 서버가 클라이언트 상태 저장 안 함
2. **Uniform Interface**: 일관된 URL 구조
3. **Client-Server**: 관심사 분리
4. **Cacheable**: 응답 캐싱 가능해야 함

## REST vs GraphQL
| | REST | GraphQL |
|-|------|---------|
| Over-fetching | 있음 | 없음 |
| Under-fetching | 있음 | 없음 |
| 버전 관리 | /v1, /v2 | 불필요 |
| 학습 곡선 | 낮음 | 있음 |

## WebSocket
- HTTP와 달리 **양방향 지속 연결**
- 실시간 채팅, 알림, 주식 시세에 사용
- 첫 연결은 HTTP Upgrade 요청

## ❓ 면접 질문
**Q. REST API URL 설계 원칙은?**
> 명사 사용, 소문자, 복수형.
> GET /users/1 (O) / GET /getUser?id=1 (X)

**Q. CORS란?**
> 다른 도메인 간 리소스 요청 제한 정책.
> 서버에서 Access-Control-Allow-Origin 헤더로 허용.
