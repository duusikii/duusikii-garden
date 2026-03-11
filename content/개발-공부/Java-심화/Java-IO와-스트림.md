---
title: Java IO와 스트림
tags: [Java, IO, 면접, CS]
---

# Java IO와 스트림

## 핵심 질문부터: BufferedReader를 쓰는 이유?

```java
// ❌ 느림 — 1바이트씩 읽음
InputStream is = System.in;
int data = is.read();  // 매번 OS에 시스템 콜

// ⬆ 한 단계 업 — 바이트→문자 변환
InputStreamReader isr = new InputStreamReader(System.in);
int ch = isr.read();  // 여전히 한 문자씩 → 시스템 콜 많음

// ✅ 빠름 — 버퍼로 한 번에 묶어 읽음
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
String line = br.readLine();  // 버퍼(8KB)에 미리 읽어두고 꺼내 씀
```

**한 줄 요약:**
> **BufferedReader = InputStreamReader + 버퍼(8192자)**
> 시스템 콜 횟수를 대폭 줄여서 빠르다.

---

## Java IO 전체 구조

### 바이트 스트림 vs 문자 스트림

```
바이트 스트림 (1 byte 단위)          문자 스트림 (2 byte = char 단위)
┌──────────────────┐              ┌──────────────────┐
│   InputStream    │              │     Reader       │
│   OutputStream   │              │     Writer       │
└──────────────────┘              └──────────────────┘
이미지, 동영상, 바이너리              텍스트, 문자열

구현체:                            구현체:
FileInputStream                   FileReader
ByteArrayInputStream              StringReader
BufferedInputStream               BufferedReader
                                  InputStreamReader ← 브릿지!
```

### 핵심 클래스 관계

```
바이트 세계                        문자 세계
───────────                      ─────────
System.in (InputStream)
    │
    │  바이트 → 문자 변환 (디코딩)
    ▼
InputStreamReader ──── 브릿지(Bridge) ────┐
    │                                    │
    │  + 버퍼 추가                         │
    ▼                                    ▼
BufferedReader                        Reader 세계
    │
    │  readLine() 가능!
    ▼
한 줄씩 편하게 읽기
```

---

## 클래스별 역할 정리

### InputStream
```java
// 바이트(byte) 단위로 읽는 최상위 추상 클래스
// System.in이 대표적인 InputStream

int b = System.in.read();  // 1바이트 읽기 (0~255, EOF=-1)
// 문제: 한글 같은 멀티바이트 문자를 제대로 못 읽음
```

### InputStreamReader (브릿지)
```java
// 바이트 스트림 → 문자 스트림 변환 (디코딩)
// 인코딩 지정 가능

InputStreamReader isr = new InputStreamReader(System.in, "UTF-8");
int ch = isr.read();  // char 단위로 읽기 (한글 OK)

// 역할: 바이트를 문자로 "번역"
// 한계: 여전히 한 문자씩 → 느림
```

### BufferedReader (버퍼 + 편의기능)
```java
// InputStreamReader를 감싸서 버퍼 추가
// 기본 버퍼: 8192자 (8KB)

BufferedReader br = new BufferedReader(
    new InputStreamReader(System.in)
);

// 핵심 메서드
String line = br.readLine();  // 한 줄 읽기 ← 이게 핵심!
int ch = br.read();           // 한 문자 읽기
br.ready();                   // 읽을 수 있는지 확인
br.close();                   // 자원 해제
```

---

## 왜 BufferedReader가 빠른가?

### 시스템 콜 비용

```
InputStreamReader (버퍼 없음):
  read() → OS 시스템 콜 → 1문자 반환
  read() → OS 시스템 콜 → 1문자 반환
  read() → OS 시스템 콜 → 1문자 반환
  ...100글자 → 시스템 콜 100번 😱

BufferedReader (8KB 버퍼):
  read() → OS 시스템 콜 → 8192문자를 버퍼에 미리 읽음
  read() → 버퍼에서 꺼냄 (시스템 콜 X)
  read() → 버퍼에서 꺼냄 (시스템 콜 X)
  ...8192글자까지 시스템 콜 1번! 🚀
```

```
비유:
  InputStreamReader = 편의점에서 물건 1개씩 사오기
  BufferedReader    = 장바구니에 한 번에 담아오기
```

---

## Scanner vs BufferedReader

```java
// Scanner — 편하지만 느림
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
String s = sc.nextLine();

// BufferedReader — 빠르지만 파싱 직접
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
int n = Integer.parseInt(br.readLine());
StringTokenizer st = new StringTokenizer(br.readLine());
int a = Integer.parseInt(st.nextToken());
```

| | Scanner | BufferedReader |
|-|---------|----------------|
| 속도 | **느림** | **빠름** ✅ |
| 파싱 | 자동 (nextInt 등) | 수동 (parseInt) |
| 버퍼 | 1024자 | **8192자** |
| 정규식 | 내부에서 사용 (느린 원인) | 사용 안 함 |
| 코딩테스트 | 소규모 OK | **대량 입력 필수** ✅ |

**🔑 코딩테스트 팁:**
> HackerRank에서 입력이 많으면 Scanner가 시간초과(TLE) 날 수 있다. **BufferedReader + StringTokenizer 조합**이 안전하다.

---

## 코딩테스트 표준 입출력 템플릿

```java
import java.io.*;
import java.util.*;

public class Solution {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(System.out));
        
        int n = Integer.parseInt(br.readLine().trim());
        StringTokenizer st = new StringTokenizer(br.readLine());
        
        int[] arr = new int[n];
        for (int i = 0; i < n; i++) {
            arr[i] = Integer.parseInt(st.nextToken());
        }
        
        // 풀이...
        
        bw.write(result + "\n");
        bw.flush();
        bw.close();
        br.close();
    }
}
```

---

## NIO (New IO) — 간단히

Java 1.4에서 추가된 **비동기/논블로킹 IO**.

```
기존 IO (java.io)           NIO (java.nio)
─────────────              ──────────────
스트림 기반                  채널(Channel) + 버퍼(Buffer) 기반
블로킹                      논블로킹 가능
단방향                      양방향
바이트/문자 단위              Buffer 단위
```

| | IO | NIO |
|-|----|-----|
| 데이터 흐름 | Stream (단방향) | Channel (양방향) |
| 버퍼 | 없음 (BufferedXxx 래핑) | **기본 제공** |
| 블로킹 | 항상 블로킹 | **논블로킹 가능** |
| Selector | 없음 | ✅ (1스레드로 여러 채널 관리) |
| 용도 | 일반 파일 IO | **네트워크 서버, 대용량 파일** |

**🔑 면접 포인트:**
> "IO는 스트림 기반으로 블로킹 방식이고, NIO는 채널+버퍼 기반으로 논블로킹이 가능합니다. Netty 같은 고성능 네트워크 프레임워크가 NIO 기반입니다."

---

## 🎤 면접 예상 질문

**Q: BufferedReader를 쓰는 이유는?**
> "내부에 8KB 버퍼를 두고 한 번에 많은 데이터를 읽어 시스템 콜 횟수를 줄입니다. InputStreamReader만 쓰면 한 문자마다 시스템 콜이 발생해 느리지만, BufferedReader로 감싸면 버퍼에서 꺼내 쓰므로 훨씬 빠릅니다."

**Q: InputStreamReader의 역할은?**
> "바이트 스트림(InputStream)을 문자 스트림(Reader)으로 변환하는 브릿지 역할입니다. 바이트를 지정된 인코딩(UTF-8 등)에 따라 문자로 디코딩합니다."

**Q: Scanner와 BufferedReader 차이?**
> "Scanner는 내부에서 정규식을 사용해 파싱해 편리하지만 느립니다. BufferedReader는 버퍼가 크고 정규식을 쓰지 않아 빠릅니다. 대량 입력 처리 시 BufferedReader가 유리합니다."

**Q: 바이트 스트림과 문자 스트림의 차이?**
> "바이트 스트림(InputStream/OutputStream)은 1바이트 단위로 처리해 이미지 등 바이너리에 적합합니다. 문자 스트림(Reader/Writer)은 char(2바이트) 단위로 처리해 텍스트에 적합하며, 인코딩 변환을 자동으로 처리합니다."

**Q: IO와 NIO의 차이?**
> "IO는 스트림 기반 블로킹 방식이고, NIO는 채널+버퍼 기반으로 논블로킹과 Selector를 지원합니다. NIO는 하나의 스레드로 여러 연결을 관리할 수 있어 네트워크 서버에 적합합니다."
