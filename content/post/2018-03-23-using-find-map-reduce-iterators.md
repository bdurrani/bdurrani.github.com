---
title: "Using find/map/reduce with javascript iterators"
date: 2018-03-23
draft: true
---

I come from the .NET world, where I'm used to using LINQ for massaging
collections of data in a clear functional way. 
ES6 added lots of useful functions to [Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
that lets me write code like

```js
[1,2,3,4].filter(x => x ===3).map(x => x *2);

```

I like this style because it avoids deep nested loops and makes the 
intention of the code very clear.

To my surprise, I learnt that I can't write code like this for other
data structures like [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map).
because these expose their data as [iterators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#The_iterator_protocol).

Iterators are a whole other beast introduced in ES6, and the only built-in way
to use filter/map with them is to convert then to Arrays.

```js
let map = new Map([['foo', 3], ['bar', 2], ['baz', 4]])
const reducer = (accumulator, currentValue) => accumulator + currentValue;
const result = Array.from(map)
  .map(x => `key: ${x[0]} value: ${x[1]}`)
  .reduce(reducer);

console.log(result)
```

There are serveral libraries like [wu](https://github.com/fitzgen/wu.js/) that 
adds support for these operations and lots more. 
Instead of just adding another library, I figured I would see how `wu`
worked and see if I can create something for just the parts I need.

Luckily `wu` is a pretty well organized library and while my
javascript foo is not that strong, 
I was able to break down the basic functionality.
