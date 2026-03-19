---
title: "TreeMap 활용 — LIS (최장 증가 부분수열)"
tags:
  - 알고리즘
  - TreeMap
  - DP
  - 이분탐색
  - 코딩테스트
date: 2026-03-19
related:
  - "[[Prefix-Sum-패턴]]"
---

# TreeMap 활용 — LIS (최장 증가 부분수열)

## 문제

정수 배열이 주어질 때, **가장 긴 순 증가 부분수열**의 길이를 구하라.

```
[10, 9, 2, 5, 3, 7, 101, 18] → 답: 4 (예: [2, 5, 7, 101])
```

---

## 풀이 1: DP — O(N²)

```
dp[i] = i번째 원소를 마지막으로 하는 LIS 길이

각 원소마다 "나보다 앞에 있고, 나보다 작은 것 중 최대 dp값 + 1"
```

```java
public int lengthOfLIS(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
    }
    return Arrays.stream(dp).max().getAsInt();
}
```

---

## 풀이 2: tails 배열 + 이분탐색 — O(N log N)

```
tails[i] = 길이가 (i+1)인 증가 부분수열의 마지막 원소 중 최솟값

규칙:
  num > tails 마지막 → 뒤에 붙임 (LIS 길이 증가!)
  그 외 → tails에서 num 이상인 첫 위치를 찾아 교체
```

```java
public int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    
    for (int num : nums) {
        int lo = 0, hi = tails.size();
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (tails.get(mid) < num) lo = mid + 1;
            else hi = mid;
        }
        if (lo == tails.size()) tails.add(num);
        else tails.set(lo, num);
    }
    return tails.size();
}
```

마지막 값을 최소로 유지 → 미래의 가능성을 넓힘.

주의: tails 배열의 내용이 항상 실제 LIS는 아님. 길이만 정확함.

---

## 풀이 3: TreeMap — O(N log N)

### 아이디어

TreeMap의 key에 원소 값, value에 LIS 길이를 저장.

핵심: **key가 증가하면 value도 증가하도록 pruning** 하면, `lowerEntry(num)` 한 번으로 나보다 작은 원소의 최대 LIS 길이를 O(log N)에 가져올 수 있다.

### 코드

```java
public int lengthOfLIS(int[] nums) {
    TreeMap<Integer, Integer> map = new TreeMap<>();
    int output = 0;
    
    for (int num : nums) {
        Map.Entry<Integer, Integer> lower = map.lowerEntry(num);
        int val = lower == null ? 1 : lower.getValue() + 1;
        
        map.put(num, val);
        
        // pruning: 나보다 key가 크면서 value가 같거나 작은 엔트리 제거
        while (true) {
            Map.Entry<Integer, Integer> higher = map.higherEntry(num);
            if (higher != null && higher.getValue() <= val) {
                map.remove(higher.getKey());
            } else {
                break;
            }
        }
        
        output = Math.max(output, val);
    }
    return output;
}
```

### 트레이싱

```
[5, 6, 7, 1, 2, 3, 4, 8]

5:  map={5:1}
6:  map={5:1, 6:2}
7:  map={5:1, 6:2, 7:3}
1:  prune 5:1 → map={1:1, 6:2, 7:3}
2:  prune 6:2 → map={1:1, 2:2, 7:3}
3:  prune 7:3 → map={1:1, 2:2, 3:3}
4:  map={1:1, 2:2, 3:3, 4:4}
8:  map={1:1, 2:2, 3:3, 4:4, 8:5}

답 = 5 ✅
```

### pruning에서 주의

나보다 key가 크면서 **value가 더 큰 건 살려둬야 한다!** 그 뒤에 더 긴 LIS가 이어질 수 있으므로.

### 복잡도

- 시간: O(N log N) — 각 원소가 map에 최대 1번 삽입, 1번 제거 (상각)
- 공간: O(N)

---

## 풀이 비교

| 풀이 | 시간 | 공간 | 특징 |
|------|------|------|------|
| DP | O(N²) | O(N) | 직관적, 느림 |
| tails + 이분탐색 | O(N log N) | O(N) | 가장 일반적 |
| TreeMap | O(N log N) | O(N) | TreeMap 연습에 좋음 |
