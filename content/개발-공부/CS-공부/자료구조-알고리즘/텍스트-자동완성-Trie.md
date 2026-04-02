---
title: "텍스트 자동완성 (Trie)"
tags:
  - 자료구조
  - Trie
  - 자동완성
  - 코딩테스트
created: 2026-04-02
modified: 2026-04-02
---

# 텍스트 자동완성 (Trie)

입력 글자에 맞는 단어를 실시간으로 추천하는 시스템. Trie(접두사 트리) 기반 구현.

## Trie 구조

각 노드가 자식 문자를 Map으로 관리하는 트리.

```
        root
       /    \
      c      d
      |      |
      h      o
     / \     |
    a    e   g
    |    |
    t    c
   (chat) k
         (check)

prefix "ch" → 노드 h까지 이동 → 하위 전부 수집 → [chat, check]
```

## 구현

```java
import java.util.*;

public class AutoComplete {

    private class TrieNode {
        Map<Character, TrieNode> children = new HashMap<>();
        boolean isEnd = false;   // 단어의 끝인가?
        String word = null;      // 끝이면 전체 단어 저장
        int count = 0;           // 검색/등록 횟수 (인기도)
    }

    private TrieNode root = new TrieNode();

    // 단어 등록
    public void insert(String word) {
        TrieNode node = root;
        for (char c : word.toLowerCase().toCharArray()) {
            node = node.children.computeIfAbsent(c, k -> new TrieNode());
        }
        node.isEnd = true;
        node.word = word.toLowerCase();
        node.count++;
    }

    // 자동완성 추천 (최대 limit개, 인기순)
    public List<String> search(String prefix, int limit) {
        TrieNode node = root;

        // prefix 끝 노드까지 이동
        for (char c : prefix.toLowerCase().toCharArray()) {
            node = node.children.get(c);
            if (node == null) {
                return new ArrayList<>();
            }
        }

        // 하위 모든 단어 수집 (인기순 정렬)
        PriorityQueue<TrieNode> pq = new PriorityQueue<>(
            (a, b) -> b.count - a.count
        );
        collectWords(node, pq);

        List<String> result = new ArrayList<>();
        while (!pq.isEmpty() && result.size() < limit) {
            result.add(pq.poll().word);
        }
        return result;
    }

    // DFS로 하위 단어 수집
    private void collectWords(TrieNode node, PriorityQueue<TrieNode> pq) {
        if (node.isEnd) {
            pq.offer(node);
        }
        for (TrieNode child : node.children.values()) {
            collectWords(child, pq);
        }
    }

    public static void main(String[] args) {
        AutoComplete ac = new AutoComplete();

        ac.insert("channel");
        ac.insert("channel");
        ac.insert("channel");  // 3번 → 인기 높음
        ac.insert("chat");
        ac.insert("chat");     // 2번
        ac.insert("check");    // 1번
        ac.insert("customer"); // 1번

        System.out.println(ac.search("ch", 3));
        // [channel, chat, check] (인기순)

        System.out.println(ac.search("cu", 3));
        // [customer]

        System.out.println(ac.search("z", 3));
        // [] (없음)
    }
}
```

## 시간 복잡도

- insert: **O(L)** — L = 단어 길이
- search: **O(L + N)** — L = prefix 길이, N = 하위 단어 수

## 현업에서는?

- **실시간 소규모**: Trie (메모리 내)
- **실시간 대규모**: Redis Sorted Set (prefix → 단어들)
- **검색 엔진**: Elasticsearch prefix query, completion suggester

## 면접 포인트

- **"왜 Trie?"** → prefix 기반 탐색에 최적, O(L)로 prefix 노드 도달
- **"인기순 어떻게?"** → 각 단어 노드에 count 저장, PriorityQueue로 정렬
- **"HashMap으로 안 되나?"** → 모든 키를 순회해야 prefix 매칭 가능 → O(N), Trie는 O(L)
- **"한글은?"** → 자모 분리 후 Trie 적용 가능, 또는 Elasticsearch 활용
