---
title: String vs StringBuffer vs StringBuilder
tags: [Java, 면접, CS]
---

# String vs StringBuffer vs StringBuilder

## 핵심 비교표

| | String | StringBuffer | StringBuilder |
|-|--------|-------------|---------------|
| 가변성 | **불변(Immutable)** | 가변(Mutable) | 가변(Mutable) |
| 스레드 안전 | ✅ (불변이니까) | ✅ **(synchronized)** | ❌ |
| 성능 | 연결 많으면 느림 | StringBuilder보다 느림 | **가장 빠름** ✅ |
| 사용 시점 | 변경 적을 때 | 멀티스레드 + 문자열 변경 | **싱글스레드 + 문자열 변경** |

---

## 1. String — 불변 객체

```java
String s = "hello";
s = s + " world";  // "hello"는 버려지고 새 객체 "hello world" 생성
```

```
Heap:
  "hello"        ← 버려짐 (GC 대상)
  " world"       ← 버려짐
  "hello world"  ← s가 가리킴 (새 객체)
```

**문자열 연결을 100번 하면?**
```java
String result = "";
for (int i = 0; i < 100; i++) {
    result += i;  // 매번 새 String 객체 생성! → 100개 쓰레기 객체
}
// 시간복잡도: O(n²) — 매번 전체 복사하니까
```

### String이 불변인 이유
1. **String Pool** — 같은 문자열 재사용 (메모리 절약)
2. **보안** — DB 비밀번호, URL 등이 외부에서 변경 불가
3. **스레드 안전** — 공유해도 변경 안 되니까 락 불필요
4. **hashCode 캐싱** — HashMap 키로 쓸 때 해시값 재계산 불필요

```java
String a = "hello";
String b = "hello";
System.out.println(a == b);  // true! (String Pool에서 같은 객체)

String c = new String("hello");
System.out.println(a == c);  // false (new는 힙에 새로 생성)
System.out.println(a.equals(c));  // true (값은 같음)
```

---

## 2. StringBuilder — 가변, 빠름, 스레드 안전 X

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100; i++) {
    sb.append(i);  // 같은 객체의 내부 배열에 추가 (새 객체 X)
}
String result = sb.toString();
// 시간복잡도: O(n)
```

```
내부 구조:
  char[] value = ['h','e','l','l','o','_','_','_','_','_']
                                       ↑ 여유 공간
  
  append("!") → ['h','e','l','l','o','!','_','_','_','_']
  새 객체 생성 없이 배열에 추가!
  
  용량 초과 시: 기존 * 2 + 2 로 확장 (한 번만)
```

### 주요 메서드
```java
StringBuilder sb = new StringBuilder("hello");
sb.append(" world");      // 뒤에 추가    → "hello world"
sb.insert(5, ",");        // 위치에 삽입   → "hello, world"
sb.delete(5, 6);          // 범위 삭제     → "hello world"
sb.reverse();             // 뒤집기        → "dlrow olleh"
sb.replace(0, 5, "hi");  // 범위 교체     → "hi olleh"
sb.charAt(0);             // 인덱스 접근   → 'h'
sb.length();              // 길이
sb.toString();            // String 변환
```

---

## 3. StringBuffer — 가변, 느림, 스레드 안전 O

```java
// StringBuilder와 API 완전히 동일
// 차이: 모든 메서드에 synchronized 키워드

// StringBuffer 내부 (실제 코드)
@Override
public synchronized StringBuffer append(String str) {
    // synchronized → 한 번에 한 스레드만 접근 가능
    super.append(str);
    return this;
}
```

**synchronized의 비용:**
```
StringBuilder.append()  → 그냥 실행
StringBuffer.append()   → 락 획득 → 실행 → 락 해제
                          ↑ 이 오버헤드가 10~15% 느린 이유
```

---

## 언제 뭘 쓸까?

```
문자열 변경이 거의 없다
  → String ✅

문자열 변경이 잦다 + 싱글스레드 (대부분의 경우)
  → StringBuilder ✅

문자열 변경이 잦다 + 멀티스레드에서 공유
  → StringBuffer ✅

코딩테스트
  → StringBuilder ✅ (항상)
```

**실무에서 StringBuffer 쓸 일?**
> 거의 없다. 문자열을 멀티스레드에서 공유하는 설계 자체가 드물다. 99%는 StringBuilder.

---

## 코딩테스트 활용

```java
// 문자열 뒤집기
String reversed = new StringBuilder("hello").reverse().toString();

// 빠른 문자열 조합 (for문)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) {
    sb.append(arr[i]);
    if (i < n - 1) sb.append(" ");
}
System.out.println(sb);

// 팰린드롬 체크
String s = "racecar";
boolean isPalindrome = s.equals(new StringBuilder(s).reverse().toString());
```

---

## 성능 비교 (10만 번 연결)

```
String   += 연결:  ~5000ms  (매번 새 객체)
StringBuffer:      ~15ms    (synchronized 오버헤드)
StringBuilder:     ~10ms    (가장 빠름)
```

---

## 🎤 면접 예상 질문

**Q: String이 불변인 이유는?**
> "String Pool을 통한 메모리 재사용, 보안(비밀번호·URL 변경 방지), 스레드 안전성, hashCode 캐싱 등의 이점이 있기 때문입니다."

**Q: StringBuffer와 StringBuilder의 차이?**
> "둘 다 가변 문자열이고 API도 동일합니다. 차이는 **스레드 안전성**입니다. StringBuffer는 모든 메서드에 synchronized가 걸려 있어 멀티스레드에서 안전하지만, 그만큼 느립니다. StringBuilder는 동기화가 없어 빠르고, 싱글스레드 환경에서 사용합니다. 실무에서는 대부분 StringBuilder를 씁니다."

**Q: String + 연결이 느린 이유?**
> "String은 불변이라 +를 할 때마다 새 String 객체를 생성하고 기존 내용을 복사합니다. n번 연결하면 O(n²)입니다. 컴파일러가 내부적으로 StringBuilder로 최적화하기도 하지만 반복문 안에서는 최적화가 안 됩니다."

**Q: `==`와 `equals()`의 차이? (String 관련)**
> "`==`는 참조(주소) 비교, `equals()`는 값 비교입니다. String Pool의 리터럴은 `==`로도 true이지만, `new String()`으로 만들면 다른 객체라 `==`는 false입니다. 문자열 비교는 항상 `equals()`를 써야 합니다."

**Q: String s = "abc"와 String s = new String("abc")의 차이?**
> "리터럴 `\"abc\"`는 String Pool에 저장되어 같은 문자열이면 같은 객체를 참조합니다. `new String(\"abc\")`는 힙에 별도 객체를 강제 생성합니다. 그래서 리터럴끼리 `==`는 true이지만, new와 리터럴의 `==`는 false입니다."

**Q: StringBuilder의 초기 용량과 확장 방식?**
> "기본 용량은 16입니다. 인자로 String을 넣으면 문자열 길이 + 16입니다. 용량 초과 시 `기존 용량 × 2 + 2`로 확장하며, 내부 배열을 새로 만들고 복사합니다."
