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

We can start with something like this.

```js
class Chain {
	constructor(iterable) {
		this.iterator = Chain.getIterator(iterable);
	} 

	[Symbol.iterator]() {
		return this.iterator;
	}

	* filter(fn = Boolean) {
		for (let x of this) {
			if (fn(x)) {
				yield x;
			}
		}
	}

	/**
	 * Return whether a thing is iterable.
	 */
	static isIterable(thing) {
		return thing && typeof thing[Symbol.iterator] === 'function';
	};

	/**
	 * Get the iterator for the thing or throw an error.
	 * @param thing
	 * @return {*}
	 */
	static getIterator(thing) {
		if (Chain.isIterable(thing)) {
			return thing[Symbol.iterator]();
		}
		throw new TypeError('Not iterable: ' + thing);
	};
}
```
This is a good starting point.
`Chain` acts as a proxy for the iterable and so we can some helper methods to it like `filter`

You can use it like so

```js
for (const item of new Chain([1, 2, 3]).filter(x => x === 1)) {
	console.log(item);
}
```

This will print `1`

I don't want to keep typing `new Chain()` every time, so I'll add a little helper

```js
function chain(iterable) {
	return new Chain(iterable);
}
```

So now I can do this
```js
for (const item of chain([1, 2, 3]).filter(x => x === 1)) {
	console.log(item);
}
```

The problem is we can't chain these together. 
We need the functions to return another instance of `Chain`

Lets modify `Chain` to do that.

```js
class Chain {
	constructor(iterable) {
		this.iterator = Chain.getIterator(iterable);
	}

	[Symbol.iterator]() {
		return this.iterator;
	}

	/**
	 * internal filter
	 * @param fn
	 * @returns {IterableIterator<*>}
	 * @private
	 */
	* _filter(fn) {
		for (let x of this) {
			if (fn(x)) {
				yield x;
			}
		}
	}

	/**
	 * wrapper filter
	 * @param fn
	 * @returns {*}
	 */
	filter(fn = Boolean) {
		return chain(this._filter(fn))
	}

	/**
	 * Return whether a thing is iterable.
	 */
	static isIterable(thing) { 
		// same as before
	};

	/**
	 * Get the iterator for the thing or throw an error.
	 * @param thing
	 * @return {*}
	 */
	static getIterator(thing) {
		// same as before
	};
}
```

What is different?
`filter()` now returns the `_filter()` iterable, wrapped in a `Chain()`.

Now let me add a `map` to `Chain`.

```js
	* _map(fn) {
		for (let x of this) {
			yield fn(x);
		}
	};

	map(fn) {
		return chain(this._map(fn));
	}
```

The set up looks the same as filter.

Now we can do something like this

```js
for (const item of chain([1, 2, 3, 4]).filter(x => x % 2 === 0).map(x => x + 1)) {
	console.log(item);
}
```

This will print `3` and `5`.