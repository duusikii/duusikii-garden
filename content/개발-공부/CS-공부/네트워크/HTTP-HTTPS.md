---
title: HTTP & HTTPS
tags: [CS, 네트워크, 면접]
---

# HTTP & HTTPS

## HTTP 메서드
| 메서드 | 용도 | 멱등성 |
|--------|------|--------|
| GET | 조회 | ✅ |
| POST | 생성 | ❌ |
| PUT | 전체 수정 | ✅ |
| PATCH | 일부 수정 | ❌ |
| DELETE | 삭제 | ✅ |

## HTTP 상태코드
- **2xx**: 성공 (200 OK, 201 Created, 204 No Content)
- **3xx**: 리다이렉트 (301 영구, 302 임시)
- **4xx**: 클라이언트 오류 (400 Bad Request, 401 인증, 403 권한, 404 없음)
- **5xx**: 서버 오류 (500 Internal, 502 Bad Gateway, 503 Service Unavailable)

## HTTP vs HTTPS
- HTTPS = HTTP + TLS(SSL) 암호화
- 공개키/개인키 암호화 → 도청/변조 방지

## HTTP 버전
- **HTTP/1.1**: 기본, HOL(Head of Line) 블로킹 문제
- **HTTP/2**: 멀티플렉싱, 헤더 압축, 빠름
- **HTTP/3**: UDP 기반 QUIC 프로토콜

## ❓ 면접 질문
**Q. GET과 POST의 차이는?**
> GET: URL에 파라미터, 캐싱 가능, 멱등.
> POST: Body에 데이터, 캐싱 안 됨, 비멱등.

**Q. 쿠키 vs 세션 vs JWT?**
> 쿠키: 브라우저 저장, 서버 부담 없음, 보안 취약.
> 세션: 서버 저장, 안전하지만 확장성 문제.
> JWT: 토큰 자체에 정보, Stateless, 만료 처리 주의.
