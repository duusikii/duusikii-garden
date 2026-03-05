---
title: TCP & UDP
tags: [CS, 네트워크, 면접]
---

# TCP & UDP

## TCP (Transmission Control Protocol)
- **연결 지향** (3-way handshake)
- 순서 보장, 재전송, 흐름제어
- 느리지만 **신뢰성 보장**
- 사용: HTTP, FTP, 이메일

### 3-way Handshake
```
Client → SYN     → Server
Client ← SYN-ACK ← Server
Client → ACK     → Server
(연결 수립)
```

## UDP (User Datagram Protocol)
- **비연결 지향**
- 순서/신뢰성 보장 없음
- 빠름
- 사용: DNS, 스트리밍, 게임

## ❓ 면접 질문
**Q. TCP vs UDP 언제 쓰나요?**
> 신뢰성 필요 → TCP (결제, 로그인).
> 속도 중요, 일부 손실 허용 → UDP (실시간 영상, 게임).

**Q. 4-way Handshake란?**
> TCP 연결 종료 과정. FIN → ACK → FIN → ACK.
