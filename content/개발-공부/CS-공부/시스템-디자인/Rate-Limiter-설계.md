---
title: "Rate Limiter 설계"
tags:
  - 시스템디자인
  - RateLimiter
  - Redis
  - SlidingWindow
  - TokenBucket
created: 2026-04-02
modified: 2026-04-02
---

# Rate Limiter 설계

특정 사용자의 API 호출 횟수를 제한하는 시스템. 제한 초과 시 요청을 거부한다.

## 4가지 구현 방식

### 1. Fixed Window Counter (가장 단순)

시간을 고정된 윈도우(예: 1분)로 나누고, 각 윈도우마다 카운터 1개로 관리.

```
|--- 17:44 ---|--- 17:45 ---|--- 17:46 ---|
  count: 7       count: 3       count: 0
```

```java
public class FixedWindowLimiter {
    private int LIMIT = 10;
    private HashMap<String, Integer> counters = new HashMap<>();

    public boolean isAllowed(String userId) {
        String key = userId + ":" + LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyyMMddHHmm"));
        
        int count = counters.getOrDefault(key, 0);
        
        if (count >= LIMIT) {
            return false;
        }
        
        counters.put(key, count + 1);
        return true;
    }
}

// Redis라면:
// INCR  user1:202604021744
// EXPIRE user1:202604021744 60
```

- 장점: 구현 초간단, 메모리 적음
- 단점: **경계 문제** — 17:44:50에 10번 + 17:45:10에 10번 = 20초에 20번 통과

### 2. Sliding Window Log (정확한 방식)

요청 시간을 전부 저장하고, 현재 시간 기준 1분 이내 요청만 카운트.

```
현재: 17:45:30
|---------- 1분 윈도우 ----------|
17:44:30                    17:45:30(now)

이 범위 안에 있는 요청만 셈 → 윈도우가 "슬라이딩"
```

```java
public class SlidingWindowLogLimiter {
    private int LIMIT = 10;
    private int WINDOW_MINUTES = 1;
    private HashMap<String, Deque<LocalDateTime>> map = new HashMap<>();

    public boolean isAllowed(String userId) {
        Deque<LocalDateTime> que = map.computeIfAbsent(
            userId, k -> new ArrayDeque<>());

        LocalDateTime now = LocalDateTime.now();

        // 1분 넘은 오래된 요청 제거
        while (!que.isEmpty() && 
               que.peekLast().plusMinutes(WINDOW_MINUTES).isBefore(now)) {
            que.pollLast();
        }

        boolean output = false;
        if (que.size() < LIMIT) {
            que.addFirst(now);
            output = true;
        }
        map.put(userId, que);
        return output;
    }
}

// Redis라면 Sorted Set:
// ZADD user1 {timestamp} {timestamp}
// ZREMRANGEBYSCORE user1 0 {1분전 timestamp}
// ZCARD user1
```

- 장점: 정확함, 경계 문제 없음
- 단점: 요청마다 시간 저장 → 메모리 많이 씀

### 3. Sliding Window Counter (절충안)

Fixed Window 2개의 카운터로 비율 계산하여 근사치를 구함.

```
|--- 이전 분 ---|--- 현재 분 ---|
    count: 8        count: 3

현재 시점이 현재 분의 40% 지점이면?
→ 이전 분의 나머지 60% + 현재 분 100%
→ 8 × 0.6 + 3 × 1.0 = 7.8
→ 10 미만이니 허용!
```

```java
public class SlidingWindowCounterLimiter {
    private int LIMIT = 10;
    private HashMap<String, Integer> prevWindow = new HashMap<>();
    private HashMap<String, Integer> currWindow = new HashMap<>();
    private HashMap<String, Long> windowStart = new HashMap<>();

    public boolean isAllowed(String userId) {
        long now = System.currentTimeMillis();
        long windowSize = 60_000;
        long currentWindowStart = (now / windowSize) * windowSize;

        Long lastStart = windowStart.get(userId);
        if (lastStart == null || lastStart < currentWindowStart) {
            prevWindow.put(userId, currWindow.getOrDefault(userId, 0));
            currWindow.put(userId, 0);
            windowStart.put(userId, currentWindowStart);
        }

        long elapsed = now - currentWindowStart;
        double prevWeight = 1.0 - ((double) elapsed / windowSize);

        double estimatedCount = 
            prevWindow.getOrDefault(userId, 0) * prevWeight
            + currWindow.getOrDefault(userId, 0);

        if (estimatedCount >= LIMIT) {
            return false;
        }

        currWindow.merge(userId, 1, Integer::sum);
        return true;
    }
}
```

- 장점: 메모리 적음(카운터 2개만), 경계 문제 완화
- 단점: 근사치라 100% 정확하진 않음 (실무에서는 충분)

### 4. Token Bucket (대기업 표준)

버킷에 토큰이 일정 속도로 충전되고, 요청 시 토큰 1개를 소비. 토큰 없으면 거부.

```
버킷 최대: 10개
충전 속도: 6초마다 1개 (= 1분에 10개)

|  🪙🪙🪙🪙🪙🪙🪙  |  ← 토큰 7개 남음

요청 → 🪙 1개 소비 → 6개 → 허용!
토큰 0개 → 거부!
6초 후 → 🪙 1개 자동 충전
```

```java
public class TokenBucketLimiter {
    private int MAX_TOKENS = 10;
    private double REFILL_RATE = 10.0 / 60.0; // 초당 충전량

    private HashMap<String, Double> tokens = new HashMap<>();
    private HashMap<String, Long> lastRefill = new HashMap<>();

    public boolean isAllowed(String userId) {
        long now = System.currentTimeMillis();

        double currentTokens = tokens.getOrDefault(userId, (double) MAX_TOKENS);
        long lastTime = lastRefill.getOrDefault(userId, now);
        
        double elapsed = (now - lastTime) / 1000.0;
        currentTokens = Math.min(
            MAX_TOKENS, 
            currentTokens + elapsed * REFILL_RATE
        );

        if (currentTokens >= 1) {
            tokens.put(userId, currentTokens - 1);
            lastRefill.put(userId, now);
            return true;
        }

        tokens.put(userId, currentTokens);
        lastRefill.put(userId, now);
        return false;
    }
}
```

- 장점: 버스트 허용 가능, 메모리 적음, 균일한 처리율
- 단점: 구현이 상대적으로 복잡
- 사용처: AWS API Gateway, Stripe, GitHub API

## 방식별 비교 요약

- **Fixed Window**: 가장 단순. Redis INCR 한방. 정밀도 낮아도 될 때
- **Sliding Log**: 가장 정확. 메모리 많이 씀. 소규모에 적합
- **Sliding Counter**: 균형. 근사치지만 실무에 충분
- **Token Bucket**: 대기업 표준. 버스트 허용. 가장 유연

## 분산 환경 확장

단일 서버의 HashMap → Redis로 교체하면 분산 환경 대응 가능.

동시성 문제: 여러 서버에서 동시 요청 시 "조회 → 판단 → 추가" 사이에 Race Condition 발생 가능.

해결: **Redis Lua Script**로 전체 로직을 원자적으로 실행.

## 현업에서 실제로는?

1. **인프라 레벨**: API Gateway(AWS, Kong, Nginx) Rate Limiting 설정
2. **직접 구현**: Redis + Lua Script (Fixed Window 또는 Token Bucket)
3. **라이브러리**: Guava RateLimiter(단일), Bucket4j(분산), Resilience4j
