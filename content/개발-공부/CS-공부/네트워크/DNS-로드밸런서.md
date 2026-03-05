---
title: DNS & 로드밸런서
tags: [CS, 네트워크, 면접]
---

# DNS & 로드밸런서

## DNS (Domain Name System)
- 도메인 → IP 변환
- 계층 구조: Root → TLD(.com) → 도메인
- TTL: 캐시 유지 시간

### DNS 조회 순서
```
브라우저 캐시 → OS 캐시 → /etc/hosts
→ Local DNS → Root NS → TLD NS → 권한 NS
```

## 로드밸런서 알고리즘
| 방식 | 설명 |
|------|------|
| Round Robin | 순서대로 분배 |
| Least Connection | 연결 수 가장 적은 서버 |
| IP Hash | 클라이언트 IP 기반 고정 |
| Weighted | 서버 성능 비율로 분배 |

## ❓ 면접 질문
**Q. L4 vs L7 로드밸런서 차이는?**
> L4: TCP/UDP 레벨 (IP + Port 기반).
> L7: HTTP 레벨 (URL, 헤더, 쿠키 기반 라우팅 가능).
