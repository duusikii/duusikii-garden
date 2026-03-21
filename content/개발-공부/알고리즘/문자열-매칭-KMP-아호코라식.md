---
title: "문자열 매칭 — KMP, 아호코라식"
tags:
  - 알고리즘
  - 문자열
  - KMP
  - 아호코라식
  - 코딩테스트
created: 2026-03-21
modified: 2026-03-21
related:
  - "[[슬라이딩-윈도우-패턴]]"
---

# 문자열 매칭 — KMP, 아호코라식

## 브루트 포스 매칭 — O(N × M)

```
S = "ABCABCABD"
P = "ABCABD"

i=0: A B C A B C A B D
     A B C A B D ← 5번째에서 불일치!
     
i=1: A B C A B C A B D
       A B C A B D ← 1번째에서 불일치!

→ 불일치 시 i를 1칸만 옮기고 처음부터 다시 비교
→ O(N × M)
```

---

## KMP 알고리즘 — O(N + M)

### 핵심 아이디어

불일치 발생 시, **이미 일치한 부분의 정보를 활용**해서 불필요한 비교를 건너뜀.

### 실패 함수 (Failure Function / LPS 배열)

```
패턴: A B C A B D
LPS:  0 0 0 1 2 0

LPS[i] = 패턴[0..i]에서 접두사와 접미사가 같은 최대 길이

"ABCAB"의 접두사: A, AB, ABC, ABCA, ABCAB
"ABCAB"의 접미사: B, AB, CAB, BCAB, ABCAB

같은 것: "AB" → LPS[4] = 2
```

### LPS 배열 구축

```java
int[] buildLPS(String pattern) {
    int m = pattern.length();
    int[] lps = new int[m];
    int len = 0;
    int i = 1;
    
    while (i < m) {
        if (pattern.charAt(i) == pattern.charAt(len)) {
            len++;
            lps[i] = len;
            i++;
        } else {
            if (len != 0) {
                len = lps[len - 1]; // 핵심: 처음으로 안 돌아감!
            } else {
                lps[i] = 0;
                i++;
            }
        }
    }
    return lps;
}
```

### KMP 검색

```java
List<Integer> kmpSearch(String s, String pattern) {
    int[] lps = buildLPS(pattern);
    List<Integer> result = new ArrayList<>();
    int i = 0, j = 0;
    
    while (i < s.length()) {
        if (s.charAt(i) == pattern.charAt(j)) {
            i++;
            j++;
        }
        
        if (j == pattern.length()) {
            result.add(i - j); // 매칭 위치!
            j = lps[j - 1];   // 다음 매칭 탐색
        } else if (i < s.length() && s.charAt(i) != pattern.charAt(j)) {
            if (j != 0) {
                j = lps[j - 1]; // LPS만큼 건너뜀!
            } else {
                i++;
            }
        }
    }
    return result;
}
```

### 왜 빠른가?

```
브루트 포스: 불일치 시 i를 1칸 뒤로 + j를 0으로 리셋
KMP:        불일치 시 i는 그대로! j만 LPS[j-1]로 이동

S = "ABCABCABD"
P = "ABCABD"

     A B C A B C A B D
     A B C A B D ← 불일치! (i=5, j=5)
     
브루트: i=1로 돌아감
KMP:    LPS[4]=2 → j=2로 이동, i=5 그대로!

     A B C A B C A B D
             A B C A B D ← "AB"는 이미 일치함을 아니까 건너뜀!
             
→ i가 절대 뒤로 안 감 → O(N + M)
```

---

## 아호-코라식 (Aho-Corasick) — O(N + M + Z)

### 핵심 아이디어

**여러 패턴을 동시에** 찾을 때 사용. 트라이(Trie) + KMP의 실패 함수를 결합.

```
패턴이 10만 개일 때:
  KMP 10만 번 → O(N × 패턴 수) = O(10^6 × 10^5) = 10^11 💥
  아호코라식 → O(N + 전체 패턴 길이 + 매칭 수) ✅
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
   → 한 번 순회하면서 여러 패턴을 동시에 매칭

3. 문자열 순회 (1번만!):
   S의 각 문자를 트라이에서 따라가면서
   매칭되는 모든 패턴을 찾음
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
            output[cur] = patternIndex; // 이 노드에서 패턴 매칭
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
            // cur에서 output 따라가며 매칭된 패턴 수집
        }
    }
}
```

---

## 비교

```
┌──────────────┬────────────┬───────────────┬──────────────────┐
│              │ 시간복잡도  │ 패턴 수        │ 적합한 경우       │
├──────────────┼────────────┼───────────────┼──────────────────┤
│ indexOf      │ O(N×M)     │ 1개씩         │ 간단한 경우       │
│ KMP          │ O(N+M)     │ 1개씩         │ 패턴 1개, 긴 문자열│
│ 아호코라식    │ O(N+M+Z)   │ 동시에 여러 개 │ 패턴 수천~수만 개 │
└──────────────┴────────────┴───────────────┴──────────────────┘

N = 텍스트 길이, M = 패턴 길이(합), Z = 매칭 수
```

---

## 면접에서의 활용

```
면접에서 직접 구현을 요구하는 경우는 드묾.

하지만 이렇게 말하면 가산점:
"indexOf로 기본 구현하고, 패턴이 많으면
 아호코라식으로 O(N + 전체 패턴 길이)에 최적화할 수 있습니다"

KMP를 물어보는 경우:
"불일치 시 LPS 배열을 활용해 불필요한 비교를 건너뛰어
 O(N + M)에 매칭합니다. 핵심은 텍스트 인덱스가 절대 뒤로 안 가는 것입니다"
```

---

## 면접 예상 질문

**Q: KMP 알고리즘이란?**
> "문자열 매칭 알고리즘으로, 불일치 시 이미 일치한 접두사/접미사 정보(LPS 배열)를 활용해 불필요한 비교를 건너뜁니다. 텍스트를 한 번만 순회하므로 O(N+M)입니다."

**Q: 패턴이 여러 개일 때는?**
> "아호코라식 알고리즘을 사용합니다. 트라이에 모든 패턴을 넣고 실패 링크를 구축하면, 텍스트를 한 번만 순회하면서 모든 패턴을 동시에 매칭할 수 있습니다."
