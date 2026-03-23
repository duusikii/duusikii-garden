---
title: "문자열 매칭 — KMP, 아호코라식"
tags:
  - 알고리즘
  - 문자열
  - KMP
  - 아호코라식
  - 코딩테스트
created: 2026-03-21
modified: 2026-03-24
related:
  - "[[슬라이딩-윈도우-패턴]]"
---

# 문자열 매칭 — KMP, 아호코라식

## 브루트 포스 매칭 — O(N × M)

```
S = "AAAAAAAAB"
P = "AAAAB"

시도 1:
  S: A A A A A A A A B
  P: A A A A B
              ↑ 불일치! (4번 비교 후 실패)
  → 시작점을 1칸만 오른쪽으로

시도 2:
  S: A A A A A A A A B
  P:   A A A A B
                ↑ 불일치! (또 4번 비교)

→ 이미 비교한 "AAAA"를 매번 다시 비교 → 낭비!
→ O(N × M)
```

---

## KMP 알고리즘 — O(N + M)

### 핵심 아이디어

불일치 발생 시, **이미 일치한 부분의 정보를 활용**해서 불필요한 비교를 건너뜀.

```
S = "ABCABCABD"
P = "ABCABD"

비교:
  S: A B C A B C A B D
  P: A B C A B D
              ↑ i=5에서 불일치! (S[5]='C', P[5]='D')

이 시점에서 우리가 아는 것:
  S[0..4] = "ABCAB" = P[0..4]
  → 5글자가 일치했다는 정보를 이미 알고 있음!

브루트 포스: i=1부터 처음부터 다시 비교 😩
KMP:        이 정보를 활용해서 건너뜀! 🎯
```

### 어떻게 건너뛰는가?

```
일치했던 부분: "ABCAB"

이 문자열에서 "접두사 = 접미사"인 부분을 찾자:
  접두사: A, AB, ABC, ABCA
  접미사: B, AB, CAB, BCAB

  겹치는 것: "AB" (길이 2)

의미:
  S에서 일치한 부분의 끝 "AB"가
  P의 시작 "AB"와 같다!

  → P를 통째로 밀 필요 없이
  → "AB"는 이미 맞으니까 P[2]부터 비교!
```

```
시각화:

불일치 시점:
  S: A B C [A B] C A B D
  P: A B C [A B] D
                  ↑ 여기서 실패

건너뛴 후:
  S: A B C [A B] C A B D
           [A B] C A B D   ← P를 여기로!
                  ↑ P[2]='C'부터 비교!

→ S의 포인터(i)는 뒤로 안 감! 5에서 그대로!
→ P의 포인터(j)만 2로 이동!

이것이 KMP가 빠른 이유: i가 절대 뒤로 안 감!
```

---

### LPS 배열 (실패 함수)

```
LPS[i] = 패턴[0..i]에서 "접두사 = 접미사"인 최대 길이

미리 계산해두면 불일치 시 "j를 어디로 보낼지" 바로 알 수 있음
```

```
예시: P = "ABCABD"

i=0: "A"
  접두사: (없음)  접미사: (없음)
  → LPS[0] = 0  (항상 0)

i=1: "AB"
  접두사: A     접미사: B
  → 같은 게 없음 → LPS[1] = 0

i=2: "ABC"
  접두사: A, AB    접미사: C, BC
  → 같은 게 없음 → LPS[2] = 0

i=3: "ABCA"
  접두사: A, AB, ABC
  접미사: A, CA, BCA
  → "A" = "A" ✅ → LPS[3] = 1

i=4: "ABCAB"
  접두사: A, AB, ABC, ABCA
  접미사: B, AB, CAB, BCAB
  → "AB" = "AB" ✅ → LPS[4] = 2

i=5: "ABCABD"
  접두사: A, AB, ABC, ABCA, ABCAB
  접미사: D, BD, ABD, CABD, BCABD
  → 같은 게 없음 → LPS[5] = 0

결과: LPS = [0, 0, 0, 1, 2, 0]
```

```
LPS가 알려주는 것:

P[5]에서 불일치하면?
  → LPS[5-1] = LPS[4] = 2
  → j를 2로 보내! (P[2]부터 다시 비교)
  → 왜? "AB"는 이미 일치하니까!
```

### LPS 배열 구축 코드

```java
int[] buildLPS(String pattern) {
    int m = pattern.length();
    int[] lps = new int[m];
    int len = 0;   // 이전까지 일치한 접두사 길이
    int i = 1;     // 0은 항상 0이니까 1부터

    while (i < m) {
        if (pattern.charAt(i) == pattern.charAt(len)) {
            // 일치! 접두사=접미사 길이 증가
            len++;
            lps[i] = len;
            i++;
        } else {
            if (len != 0) {
                // 핵심! 처음으로 안 돌아감!
                // 이전 LPS를 활용해서 건너뜀
                len = lps[len - 1];
                // i는 증가 안 함! len만 조정!
            } else {
                lps[i] = 0;
                i++;
            }
        }
    }
    return lps;
}
```

```
트레이싱: P = "ABCABD"

i=1, len=0: B vs A → 불일치, len==0 → lps[1]=0, i=2
i=2, len=0: C vs A → 불일치, len==0 → lps[2]=0, i=3
i=3, len=0: A vs A → 일치!  → len=1, lps[3]=1, i=4
i=4, len=1: B vs B → 일치!  → len=2, lps[4]=2, i=5
i=5, len=2: D vs C → 불일치, len!=0 → len=lps[1]=0
i=5, len=0: D vs A → 불일치, len==0 → lps[5]=0, i=6

결과: [0, 0, 0, 1, 2, 0] ✅
```

---

### KMP 검색 코드

```java
List<Integer> kmpSearch(String s, String pattern) {
    int[] lps = buildLPS(pattern);
    List<Integer> result = new ArrayList<>();
    int i = 0; // S 포인터
    int j = 0; // P 포인터

    while (i < s.length()) {
        if (s.charAt(i) == pattern.charAt(j)) {
            // 일치 → 둘 다 전진
            i++;
            j++;
        }

        if (j == pattern.length()) {
            // 패턴 전체 매칭 성공!
            result.add(i - j);  // 시작 위치 기록
            j = lps[j - 1];    // 다음 매칭 탐색 계속!
        } else if (i < s.length() && s.charAt(i) != pattern.charAt(j)) {
            // 불일치!
            if (j != 0) {
                j = lps[j - 1]; // LPS만큼 건너뜀! i는 안 움직임!
            } else {
                i++;             // j가 0이면 i만 전진
            }
        }
    }
    return result;
}
```

### KMP 검색 트레이싱

```
S = "ABCABCABD", P = "ABCABD"
LPS = [0, 0, 0, 1, 2, 0]

i=0,j=0: A==A ✅ → i=1,j=1
i=1,j=1: B==B ✅ → i=2,j=2
i=2,j=2: C==C ✅ → i=3,j=3
i=3,j=3: A==A ✅ → i=4,j=4
i=4,j=4: B==B ✅ → i=5,j=5
i=5,j=5: C!=D ❌ → j=lps[4]=2  (i는 그대로 5!)

  → "AB"는 이미 맞으니까 j=2부터!

i=5,j=2: C==C ✅ → i=6,j=3
i=6,j=3: A==A ✅ → i=7,j=4
i=7,j=4: B==B ✅ → i=8,j=5
i=8,j=5: D==D ✅ → i=9,j=6

j==6 == pattern.length() → 매칭! 위치 = 9-6 = 3
result = [3] ✅
```

---

## 아호-코라식 (Aho-Corasick) — O(N + M + Z)

### 핵심 아이디어

**여러 패턴을 동시에** 찾을 때 사용. 트라이(Trie) + KMP의 실패 함수를 결합.

```
패턴이 10만 개일 때:
  KMP 10만 번 → O(N × 패턴 수) 💥
  아호코라식   → O(N + 전체 패턴 길이 + 매칭 수) ✅
```

### 구조

```
패턴: ["he", "she", "his", "hers"]

1. 트라이 구축:
        root
       / | \
      h  s   (fail links 생략)
     / \  \
    e   i   h
    |   |   |
    r   s   e
    |
    s

2. 실패 링크 추가 (BFS):
   "she"의 "he" → "he" 패턴의 노드로 실패 링크!
   → KMP의 LPS와 같은 역할을 트라이 위에서!

3. 문자열 순회 (1번만!):
   S의 각 문자를 트라이에서 따라가면서
   매칭되는 모든 패턴을 동시에 찾음
```

### 간략 구현

```java
class AhoCorasick {
    int[][] go;       // 트라이 전이
    int[] fail;       // 실패 링크
    int[] output;     // 매칭되는 패턴 인덱스
    int size = 0;

    void build(List<String> patterns) {
        // 1. 트라이에 모든 패턴 삽입
        for (String p : patterns) {
            int cur = 0;
            for (char c : p.toCharArray()) {
                if (go[cur][c - 'a'] == 0) {
                    go[cur][c - 'a'] = ++size;
                }
                cur = go[cur][c - 'a'];
            }
            output[cur] = patternIndex;
        }

        // 2. BFS로 실패 링크 구축
        Queue<Integer> queue = new LinkedList<>();
        for (int c = 0; c < 26; c++) {
            if (go[0][c] != 0) {
                fail[go[0][c]] = 0;
                queue.add(go[0][c]);
            }
        }
        while (!queue.isEmpty()) {
            int u = queue.poll();
            for (int c = 0; c < 26; c++) {
                if (go[u][c] != 0) {
                    fail[go[u][c]] = go[fail[u]][c];
                    queue.add(go[u][c]);
                } else {
                    go[u][c] = go[fail[u]][c];
                }
            }
        }
    }

    // 3. 문자열 S를 한 번만 순회!
    List<int[]> search(String s) {
        int cur = 0;
        for (int i = 0; i < s.length(); i++) {
            cur = go[cur][s.charAt(i) - 'a'];
            // cur에서 output/fail 따라가며 매칭된 패턴 수집
        }
    }
}
```

---

## 비교

| | 시간복잡도 | 패턴 수 | 적합한 경우 |
|---|---|---|---|
| indexOf (브루트포스) | O(N × M) | 1개씩 | 간단한 경우, 패턴 짧을 때 |
| KMP | O(N + M) | 1개씩 | 패턴 1개, 긴 문자열 |
| 아호코라식 | O(N + M + Z) | 동시에 여러 개 | 패턴 수천~수만 개 |

> N = 텍스트 길이, M = 패턴 길이(합), Z = 매칭 수

### 실전 문제에 적용할 때

```
예: "Smallest Substring Containing All Patterns"

S = 10^6, 패턴 수 = 10^5, 패턴 길이 = 10^5

indexOf: O(N × L × m) = 10^16 💥
KMP:     O(m × (N + L)) = 10^11 ⚠️
아호코라식: O(N + M + Z) = ~10^6 ✅

현실적으로: 전체 패턴 길이 합에 암묵적 제한이 있으면
indexOf/KMP로도 통과 가능. 안 되면 아호코라식!
```

---

## 면접 포인트

**Q: KMP 알고리즘이란?**
> "불일치 시 이미 일치한 접두사/접미사 정보(LPS 배열)를 활용해 불필요한 비교를 건너뜁니다. 핵심은 텍스트 포인터가 절대 뒤로 안 가서 O(N+M)입니다."

**Q: LPS 배열이 뭔가요?**
> "Longest Proper Prefix which is also Suffix. 패턴의 각 위치에서 접두사와 접미사가 같은 최대 길이를 저장합니다. 불일치 시 이 값으로 패턴 포인터를 이동해 이미 일치한 부분을 재활용합니다."

**Q: 패턴이 여러 개일 때는?**
> "아호코라식을 사용합니다. 트라이에 모든 패턴을 넣고 실패 링크(KMP의 LPS와 같은 역할)를 구축하면, 텍스트를 한 번만 순회하면서 모든 패턴을 동시에 매칭합니다."

**Q: indexOf 대신 KMP를 쓰는 이유?**
> "indexOf는 불일치 시 텍스트를 되돌아가서 O(N×M). KMP는 되돌아가지 않아 O(N+M). 패턴이 길수록 차이가 커집니다."
