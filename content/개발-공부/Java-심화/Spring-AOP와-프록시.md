---
title: "Spring AOP와 프록시"
tags:
  - Java
  - Spring
  - AOP
  - 면접
date: 2026-03-19
related:
  - "[[JVM-구조와-동작원리]]"
---

# Spring AOP와 프록시

## AOP 프록시 동작 원리

Spring은 빈을 생성할 때 프록시 객체로 감싼다.

```
호출 흐름:
  Controller → Proxy → 실제 객체
                 ↑
           여기서 @Async, @Transactional 등 감지
           → 부가 기능 실행!
```

---

## @Async가 private에서 안 되는 이유

```java
// ❌ private: 프록시가 오버라이드 못 함
@Async
private void sendEmail() { }

// ❌ 같은 클래스 내부 호출 (self-invocation)
public void process() {
    sendEmail(); // this.sendEmail() → 프록시 안 거침!
}

@Async
public void sendEmail() { }

// ✅ 다른 빈에서 호출
@Service
public class EmailService {
    @Async
    public void sendEmail() { } // 프록시를 거침 → 비동기!
}
```

---

## AOP 어노테이션 공통 규칙

```
@Transactional, @Cacheable, @Async 전부 같은 규칙!

동작 조건:
  ✅ public 메서드
  ✅ 다른 빈에서 호출
  ✅ Spring 컨텍스트에 등록된 빈

안 되는 경우:
  ❌ private/protected 메서드
  ❌ 같은 클래스 내부 호출 (self-invocation)
  ❌ new로 직접 생성한 객체
```

---

## self-invocation 해결법

```java
// 방법 1: 별도 서비스로 분리 (권장)
@Service
public class OrderService {
    @Autowired
    private NotificationService notificationService;
    
    public void createOrder() {
        // 주문 처리
        notificationService.sendNotification(); // 프록시 거침!
    }
}

// 방법 2: 자기 자신 주입
@Service
public class OrderService {
    @Autowired
    private OrderService self; // 프록시 객체 주입
    
    public void createOrder() {
        self.sendNotification(); // 프록시 거침!
    }
    
    @Async
    public void sendNotification() { }
}
```

---

## 면접 예상 질문

**Q: @Transactional이 동작하지 않는 경우는?**
> "private 메서드이거나, 같은 클래스 내부에서 호출하는 경우입니다. Spring AOP는 프록시 기반이라 프록시를 거치지 않으면 부가 기능이 적용되지 않습니다."

**Q: self-invocation 문제를 어떻게 해결하나요?**
> "해당 메서드를 별도 서비스로 분리하여 다른 빈에서 호출하도록 합니다."
