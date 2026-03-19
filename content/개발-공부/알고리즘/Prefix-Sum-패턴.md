---
title: "Prefix Sum 패턴"
tags:
  - 알고리즘
  - 배열
  - HashMap
  - PrefixSum
  - 코딩테스트
date: 2026-03-19
related:
  - "[[TreeMap-활용-LIS]]"
---

# Prefix Sum 패턴

## 핵심 아이디어

연속 부분배열(subarray)의 합을 구할 때, 매번 다시 더하지 않고 **누적합의 차이**로 O(1)에 구한다.

```
prefixSum[j] - prefixSum[i] = 구간 [i+1 ~ j]의 합

합이 k인 부분배열을 찾으려면:
  prefixSum[j] - prefixSum[i] = k
  → prefixSum[i] = prefixSum[j] - k
  → HashMap에서 (prefixSum[j] - k)가 몇 번 나왔는지 조회!
```

---

## 문제: Subarrays with Sum k and Max bounded by M

### 문제 설명

정수 배열 `nums`와 정수 `k`, `M`이 주어질 때, **합이 정확히 k이고 최댓값이 M 이하**인 연속 부분배열의 개수를 구하라.

### 예시

```
nums = [2, -1, 2, 1, -2, 3], k = 3, M = 2
답: 2
```

### 풀이

**Step 1: 배열 분리**

`nums[i] > M`인 원소를 포함하는 부분배열은 절대 valid할 수 없다. 해당 원소를 기준으로 배열을 여러 구간으로 분리한다.

```
M = 2일 때, nums = [2, -1, 2, 1, -2, 3]
값이 M(2) 초과인 원소: 3 (index 5)
→ 구간 분리: [2, -1, 2, 1, -2] / [3]
→ [3]은 무시, [2, -1, 2, 1, -2]에서만 탐색
```

**Step 2: Prefix Sum + HashMap**

```java
int count = 0;
int prefixSum = 0;
Map<Integer, Integer> map = new HashMap<>();
map.put(0, 1); // 빈 구간 (합 0)

for (int num : segment) {
    prefixSum += num;
    // 먼저 조회 (i < j 보장)
    count += map.getOrDefault(prefixSum - k, 0);
    // 그 다음 추가
    map.merge(prefixSum, 1, Integer::sum);
}
```

### 주의 포인트 (함정들)

| 함정 | 설명 |
|------|------|
| Set vs HashMap | Set은 존재 여부만 판별. 동일 prefix sum이 여러 번 나오면 개수를 놓침 → HashMap 필수 |
| 순서 중요 | map을 미리 전부 만들면 i > j인 쌍까지 카운트됨. **조회 먼저, 추가 나중에!** |
| 초기값 | `map.put(0, 1)` 빼먹으면 전체 구간 합이 k인 경우를 놓침 |

### 복잡도

- 시간: O(N)
- 공간: O(N)

---

## Prefix Sum 변형 문제들

### 기본: Subarray Sum Equals K (LeetCode 560)

```java
// 합이 k인 부분배열 개수
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> map = new HashMap<>();
    map.put(0, 1);
    int sum = 0, count = 0;
    
    for (int num : nums) {
        sum += num;
        count += map.getOrDefault(sum - k, 0);
        map.merge(sum, 1, Integer::sum);
    }
    return count;
}
```

### 응용: 합이 k의 배수인 부분배열 (LeetCode 523)

```
prefixSum[j] - prefixSum[i]가 k의 배수
→ prefixSum[j] % k == prefixSum[i] % k
→ HashMap에 나머지를 저장!
```

### 응용: 0과 1이 같은 개수인 부분배열 (LeetCode 525)

```
0을 -1로 바꾸면 → 합이 0인 부분배열 찾기
→ Prefix Sum + HashMap 동일 패턴!
```

---

## 패턴 정리

```
"연속 부분배열" + "합 조건" = Prefix Sum + HashMap

1. prefixSum 계산
2. map에서 (prefixSum - target) 조회
3. prefixSum을 map에 추가
4. 순서: 조회 먼저, 추가 나중에!
```
