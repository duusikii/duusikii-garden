---
title: "URL 단축기 설계"
tags:
  - 시스템디자인
  - Base62
  - Redis
  - URL단축
created: 2026-04-02
modified: 2026-04-02
---

# URL 단축기 설계

TinyURL, bit.ly 같은 URL 단축 서비스의 핵심 설계를 정리한다.

## 핵심 개념

긴 URL을 **짧은 고유 키로 매핑**하고, 짧은 키로 접근하면 원본 URL로 리다이렉트하는 서비스.

```
입력: https://www.example.com/very/long/path?param=value
출력: https://short.url/3d7
```

문자열 자체를 압축하는 게 아니라, **고유 키를 생성해서 원본과 매핑**하는 구조.

## 핵심 기능

| 기능 | 설명 |
|------|------|
| `shorten(url)` | 긴 URL → 짧은 키 반환 |
| `resolve(shortKey)` | 짧은 키 → 원본 URL 반환 |

## 키 생성 전략

### 1. Auto Increment + Base62

가장 일반적인 방식. 순차 증가하는 숫자를 Base62로 변환.

- **충돌 0**: 순차 증가라 중복 불가
- **짧음**: Base62로 변환하면 자릿수 절약
- **단점**: 순차적이라 다음 URL 추측 가능 (보안 이슈)

### 2. UUID

- 충돌 확률 사실상 0
- 단점: 36자로 "짧은 URL"이라기엔 김

### 3. Hash (MD5/SHA256) + 앞 N자리 자르기

- 단점: 잘라서 쓰면 충돌 가능, 충돌 처리 로직 필요

## Base62 인코딩

### Base62 vs Base64 차이

| | Base62 | Base64 |
|---|--------|--------|
| **입력** | 숫자 | 바이트 데이터(문자열/바이너리) |
| **목적** | 숫자를 짧은 문자열로 변환 | 바이너리를 텍스트로 안전하게 변환 |
| **결과** | 입력보다 짧아짐 | 입력보다 ~33% 길어짐 |
| **문자셋** | `0-9, a-z, A-Z` (62개) | `0-9, a-z, A-Z, +, /` (64개) |
| **용도** | URL 단축, 짧은 ID 생성 | 이메일 첨부, JWT, 이미지 인코딩 |

### 왜 Base64로 URL을 단축할 수 없는가?

```
Base64.encode("https://www.example.com")
→ "aHR0cHM6Ly93d3cuZXhhbXBsZS5jb20="
→ 원본보다 길어짐! (3바이트 → 4문자 변환이라 33% 증가)
→ 이건 "인코딩"이지 "단축"이 아님
```

### Base62 변환 원리

10진수를 2진수로 바꾸는 것과 동일한 원리. 62로 나누면서 나머지를 문자로 변환.

```
예시: 999,999를 Base62로 변환

999999 ÷ 62 = 16129 ... 나머지 7   → chars[7]  = '7'
 16129 ÷ 62 =   260 ... 나머지 9   → chars[9]  = '9'
   260 ÷ 62 =     4 ... 나머지 12  → chars[12] = 'c'
     4 ÷ 62 =     0 ... 나머지 4   → chars[4]  = '4'

결과: "4c97" (4자리!)
10진수로는 "999999" (6자리)
```

### Base62 구현 (Java)

```java
private static final String BASE62 = 
    "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

private String toBase62(long number) {
    StringBuilder sb = new StringBuilder();
    while (number > 0) {
        sb.append(BASE62.charAt((int)(number % 62)));
        number /= 62;
    }
    return sb.reverse().toString();
}
```

자릿수별 표현 가능 범위:

| Base62 자릿수 | 표현 가능 수 |
|--------------|-------------|
| 1자리 | 62 |
| 2자리 | 3,844 |
| 4자리 | 약 1,477만 |
| 6자리 | 약 568억 |
| 7자리 | 약 3.5조 |

→ **6~7자리면 대부분의 서비스에 충분**

## 저장 구조 설계

### Redis + DB 이중 구조

```
[요청] → Redis 조회 (캐시, TTL로 만료 관리)
           ↓ 없으면
        DB 조회 (영구 저장, fallback)
```

### DB 스키마

```sql
CREATE TABLE url_mapping (
    short_key   VARCHAR(10) PRIMARY KEY,
    original_url TEXT NOT NULL,
    created_at  DATETIME NOT NULL,
    expires_at  DATETIME NOT NULL,
    UNIQUE INDEX idx_url (original_url(255))  -- 동일 URL 중복 방지
);
```

- `original_url`에 UNIQUE INDEX → 동일 URL 요청 시 기존 키 반환
- `expires_at` → 만료 처리

### Redis 활용

```
SET short_key original_url EX 2592000  -- 30일 TTL
```

- 조회 성능: O(1)
- 만료: Redis TTL로 자동 처리
- 장애 시: DB fallback

## 동일 URL 반복 요청 처리

```
방법 1: DB의 original_url UNIQUE INDEX로 기존 키 조회
방법 2: Redis에 양방향 저장
  → SET key→url
  → SET url→key
```

## 만료 처리

```
1. Redis: TTL 설정으로 자동 만료
2. DB: expires_at 컬럼으로 조회 시 체크
3. 배치: 만료된 레코드 주기적 정리 (선택)
```

## 전체 구현 (Java)

```java
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.concurrent.atomic.AtomicLong;

public class UrlShortener {

    private static final String BASE62 = 
        "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    
    // DB 시뮬레이션
    private HashMap<String, String> dbUrl = new HashMap<>();
    private HashMap<String, LocalDateTime> dbTime = new HashMap<>();
    // Redis 시뮬레이션
    private HashMap<String, String> redis = new HashMap<>();
    // 역방향 인덱스 (DB UNIQUE INDEX 시뮬레이션)
    private HashMap<String, String> urlToKey = new HashMap<>();
    // Auto Increment ID
    private AtomicLong counter = new AtomicLong(1);
    
    private int TIMEOUT_DAYS = 30;

    private String toBase62(long number) {
        StringBuilder sb = new StringBuilder();
        while (number > 0) {
            sb.append(BASE62.charAt((int)(number % 62)));
            number /= 62;
        }
        return sb.reverse().toString();
    }

    public String shorten(String url) {
        // 동일 URL 체크 (DB UNIQUE INDEX 역할)
        if (urlToKey.containsKey(url)) {
            return urlToKey.get(url);
        }
        
        // 새 키 생성
        String key = toBase62(counter.getAndIncrement());
        
        // 저장
        dbUrl.put(key, url);
        dbTime.put(key, LocalDateTime.now());
        redis.put(key, url);
        urlToKey.put(url, key);
        
        return key;
    }

    public String resolve(String shortKey) {
        // 1차: Redis 조회
        String url = redis.get(shortKey);
        if (url != null) return url;

        // 2차: DB fallback
        url = dbUrl.get(shortKey);
        if (url == null) return null;

        // 만료 체크
        LocalDateTime created = dbTime.get(shortKey);
        if (created.plusDays(TIMEOUT_DAYS).isBefore(LocalDateTime.now())) {
            return null;
        }

        return url;
    }
}
```

## 확장 고려사항

### 대규모 트래픽

- **읽기**: Redis 캐시로 대부분 처리, DB 부하 최소화
- **쓰기**: Auto Increment ID 생성 시 분산 환경에서 충돌 주의
  - 해결: [[Snowflake ID]], DB 시퀀스, Redis INCR 활용

### 보안

- 순차 ID는 다음 URL 추측 가능
- 해결: ID에 랜덤 솔트 추가 후 Base62 변환

### 분석

- 클릭 수, 접속 국가, 리퍼러 등 통계 수집
- 별도 분석 테이블 또는 이벤트 스트림으로 처리

## 면접 포인트

- **"왜 Base62?"** → 짧은 URL 생성이 목적. Base64는 인코딩이라 길어짐
- **"왜 Redis + DB?"** → 읽기 성능(Redis) + 영속성/장애 대응(DB)
- **"동일 URL 처리?"** → DB UNIQUE INDEX로 기존 키 반환
- **"만료 처리?"** → Redis TTL + DB expires_at 이중 관리
- **"충돌 방지?"** → Auto Increment로 충돌 0, hashCode는 충돌 가능
