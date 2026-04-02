---
title: "LRU Cache 구현"
tags:
  - 자료구조
  - LRUCache
  - HashMap
  - LinkedList
  - 코딩테스트
created: 2026-04-02
modified: 2026-04-02
---

# LRU Cache 구현

LRU(Least Recently Used) Cache: 용량이 가득 찼을 때 **가장 오래 사용되지 않은 데이터를 제거**하는 캐시.

## 핵심 자료구조

**HashMap + Doubly Linked List** 조합

- HashMap: O(1) 조회
- Doubly Linked List: O(1) 삽입/삭제, 순서 관리

```
head ↔ [최신] ↔ [중간] ↔ [오래된] ↔ tail
         ↑                    ↑
     최근 사용             LRU (제거 대상)
```

### 왜 PriorityQueue가 아닌가?

PriorityQueue는 최솟값 추출은 O(log N)이지만, **중간 요소 삭제가 O(N)**. 캐시 get은 자주 호출되는데 매번 O(N)이면 성능 부적합.

Doubly Linked List는 노드 참조만 있으면 삭제/이동이 O(1).

## 구현

```java
import java.util.HashMap;

public class LRUCache {

    private class Node {
        String key;    // 삭제 시 map에서 찾기 위해 key도 저장!
        int value;
        Node prev;
        Node next;

        Node(String key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    private int capacity;
    private HashMap<String, Node> map = new HashMap<>();
    // 더미 head/tail → 경계 처리 if문 없이 깔끔
    private Node head = new Node("", 0);
    private Node tail = new Node("", 0);

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    // ===== 핵심 헬퍼 2개 =====

    // 리스트에서 노드 제거
    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    // 리스트 맨 앞(head 다음)에 추가 = 최근 사용
    private void addToHead(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }

    // ===== API =====

    public int get(String key) {
        if (!map.containsKey(key)) {
            return -1;
        }

        Node node = map.get(key);
        // 최근 사용으로 갱신: 제거 → 맨 앞에 추가
        removeNode(node);
        addToHead(node);
        return node.value;
    }

    public void put(String key, int value) {
        // 1. 이미 있는 키 → 값 갱신 + 맨 앞으로
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.value = value;
            removeNode(node);
            addToHead(node);
            return;
        }

        // 2. 용량 초과 → 가장 오래된(tail.prev) 제거
        if (map.size() >= capacity) {
            Node lru = tail.prev;
            removeNode(lru);
            map.remove(lru.key);  // Node에 key 저장한 이유!
        }

        // 3. 새 노드 추가
        Node newNode = new Node(key, value);
        addToHead(newNode);
        map.put(key, newNode);
    }

    public static void main(String[] args) {
        LRUCache cache = new LRUCache(3);

        cache.put("a", 1);
        cache.put("b", 2);
        cache.put("c", 3);
        // 리스트: c ↔ b ↔ a

        System.out.println(cache.get("a")); // 1 (a가 맨 앞으로)
        // 리스트: a ↔ c ↔ b

        cache.put("d", 4); // 용량 초과 → b 제거 (가장 오래됨)
        // 리스트: d ↔ a ↔ c

        System.out.println(cache.get("b")); // -1 (제거됨)
        System.out.println(cache.get("c")); // 3
    }
}
```

## 설계 포인트

### 더미 head/tail을 쓰는 이유

```
더미 없이: 매번 head == null, tail == null 체크 필요
더미 사용: head.next가 진짜 첫 번째, tail.prev가 진짜 마지막
         → 삽입/삭제 코드가 일관적
```

### Node에 key를 저장하는 이유

용량 초과 시 `tail.prev`를 제거하는데, HashMap에서도 해당 항목을 삭제하려면 key를 알아야 함. Node에 key가 없으면 HashMap에서 찾을 방법이 없음.

### 헬퍼 2개로 모든 동작 조합

| 동작 | 조합 |
|------|------|
| get (조회 + 갱신) | removeNode + addToHead |
| put (기존 키 갱신) | removeNode + addToHead |
| put (새 키 + 용량 초과) | removeNode(tail.prev) + addToHead |

## 시간 복잡도

- get: **O(1)** — HashMap 조회 + LinkedList 이동
- put: **O(1)** — HashMap 저장 + LinkedList 추가/삭제

## 면접 포인트

- **"왜 HashMap + LinkedList?"** → get O(1) + 순서 관리 O(1) 동시 만족
- **"왜 PriorityQueue 안 쓰나요?"** → 중간 요소 갱신이 O(N)
- **"더미 노드 왜 쓰나요?"** → 경계 처리 단순화
- **"Node에 key 왜 저장?"** → LRU 제거 시 map에서도 삭제하기 위해
