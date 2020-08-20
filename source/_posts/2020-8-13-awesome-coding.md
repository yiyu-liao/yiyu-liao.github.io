
---
layout: post
title:  "Awesome Coding for js"
date:   2020-07-23
tags: js
---

1. Given an unordered array of integers and a value sum, return true if any two items may be added such that they equal the value of sum . Otherwise, return false.

```js

const findSum = (arr, val) => {
  let searchValues = new Set();
  searchValues.add(val - arr[0]);
  for (let i = 1, length = arr.length; i < length; i++) {
    let searchVal = val - arr[i];
    if (searchValues.has(arr[i])) {
      return true;
    } else {
      searchValues.add(searchVal);
    }
  };
  return false;
};

```

### awesome way

```js
const findSum = (arr, sum) => arr.some(set => n => set.has(n) || !set.add(sum - n)(new Set);
```