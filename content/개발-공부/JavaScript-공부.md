---
title: JavaScript 공부
tags: [개발, JavaScript]
---

# JavaScript 공부

## 핵심 개념
- `fetch()` — API 호출
- `async/await` — 비동기 처리
- DOM 조작 — `querySelector`, `addEventListener`

## 유용한 패턴
```js
const getData = async (url) => {
  const res = await fetch(url);
  return res.json();
};
```

## 관련 노트
- [[HTML CSS 기초]]
- [[API 연동과 자동화]]
