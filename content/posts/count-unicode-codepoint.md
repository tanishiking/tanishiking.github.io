---
title: 'Count unicode characters in JavaScript'
description: 'There are several ways to count strings in JS in Unicode Code Point units instead of 16-bit units, but in the end, I wonder which method works in most environments (including legacy browsers like IE11) as of May 2017.'
publishDate: 2017-05-29
author: 'Rikito Taniguchi'
tags:
  - javascript
---


There are several ways to count strings in JS in Unicode Code Point units instead of 16-bit units, but in the end, I wonder which method works in most environments (including legacy browsers like IE11) as of May 2017. I've done some research, and I'll summarize it for you.

## Unicode Code Points
In Unicode, all characters are assigned an ID (code point) (`0` to `0x10FFFF`). The code point is written as `U+hexdecimal`.

In UTF-16, code points from `U+0000` to `U+FFFF` are represented by a single 16-bit code unit, and code points after `U+10000` (e.g., the code point for `𩸽` (Arabesque Sea in Japanese)) are represented by `surrogate pairs` as described below because they cannot be represented by 16 bits alone.

## Surrogate Pairs
In UTF-16, some characters (corresponding to code points after `U+10000`) are represented by 32 bits (16 bits x 2) in order to represent code points that cannot be represented by 16 bits alone. These characters (or expressions?) are called `surrogate pairs`.

### Surrogate Code Point
The range of 16-bit values that make up a surrogate pair is from `U+D800` to `U+DFFF`, and these are called `surrogate code points` (no characters are assigned to these surrogate code points).

In addition

- `U+D800 ~ U+DBFF` are called `upper surrogate`.
- `U+DC00 ~ U+DFFF` are called `lower surrogates`.

A surrogate pair is represented by the combination of the upper and lower surrogates.

## `.length` property
JavaScript's `str.length` does not return the length of the string in Unicode code points, but in UTF-16 code units.

For example, "𩸽" is a surrogate pair, so the length will be 2.

```javascript
"𩸽".length; // 2
```


We want to count surrogate pairs such as "𩸽" as 1.

## How to check the length of each code point

### for of
ES2015 added `for ... of...` and it can be used to repeat code point by code point.

```javascript
for (let c of '𩸽定食') console.log(c)
// 𩸽
// 定
// 食
```

But currently (May 2017) unsupported by IE11 https://kangax.github.io/compat-table/es6/#test-for..of_loops

### Spread operator
The split assignment also seems to be code point aware. If you use the spread operator, you can quickly transform a string into an array by code points.

```javascript
[...'𩸽定食']
// ['𩸽', '定', '食']
```

But currently (May 2017) unsupported by IE11 https://kangax.github.io/compat-table/es6/#test-spread(…)operator

### RegExp unicode flag
Starting with ES2015, the unicode flag has been introduced, which treats patterns as a sequence of code points.

```javascript
'𩸽定食'.match(/./ug);
// ['𩸽', '定', '食']
```

Again, IE11 doesn't support it https://kangax.github.io/compat-table/es6/#test-RegExp_y_and_u_flags

### Work hard with for statements.
Loop through the string in 16-bit units, counting surrogate pairs as 1.


```javascript
function stringLength(str) {
  let count = 0;
  for (let i = 0; i < str.length; i++) {
    count++;
    // obtain the i-th 16-bit
    const code = str.charCodeAt(i);
    if (0xD800 <= code && code <= 0xDBFF) {
      // if the i-th 16bit is an upper surrogate
      // skip the next 16 bits (lower surrogate)
      i++;
    }
  }
  return count;
}
```

Obviously it works with IE11 🎉

### Regular Expressions (Unicode Sequence)
Idea is to convert surrogate pairs to non-surrogate pair characters first and then take length

```javascript
function stringLength(str) {
  // Replace surrogate pair with _
  return str.replace(/[\uD800-\uDBFF][\uDC00-\uDFFF]/g, '_').length;
}

stringLength('𩸽定食'); // 3
```


If you want to convert it to an array

```javascript
function stringToArray(str) {
  return str.match(/[\uD800-\uDBFF][\uDC00-\uDFFF]|[^\uD800-\uDFFF]/g) || [];
}

stringToArray('𩸽定食');
// ['𩸽', '定', '食']
```

## Conclusion
That's all, maybe we should drop IE11 support.
