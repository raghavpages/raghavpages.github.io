---
layout: post
title: "Using iterators with arrays"
permalink: using-iterators-with-arrays
date: 2016-12-06 23:33:39
comments: true
description: "Using iterators with arrays"
keywords: ""
categories:

tags:

---
When dealing with objects that are stored as a collection using different data structures, popular languages provide a way to process each element of the collection by abstracting the underlying implementation of the data structure. In `C++` and `Java` they are called `iterators`. With the introduction of `Map`, `Set` and other data structures in `ES6`, `iterators` have found their way into `Javascript` as well. It has standard methods that can be used for any type of collection without knowledge of how the elements are stored.

In the `ES2015` specification, the Array object provides 3 methods that returns an object following the iterator pattern similar to that found in `C++` and `Java`. One of them just lets us access the value of each element in the array, `Array.prototype.values`. This means it can used directly on a variable defined as an `Array`.
{% highlight javascript %}
var arr = ['h', 'e', 'l', 'l', 'o'];
var iterator = arr.values();

console.log(iterator.next().value); // h
console.log(iterator.next().value); // e
console.log(iterator.next().value); // l
console.log(iterator.next().value); // l
console.log(iterator.next().value); // o
{% endhighlight %}
The collection (`Array` in this case) has a method that returns an iterator object which has a method called `next()` that advances by one index in the collection each time it is called. The value at each index can be accessed using the property `value`. Also, there is another property that lets us know whether we've reached till the end of the collection, `done`. So, after the all the `console.log` statements in above example, if done is called,
{% highlight javascript %}
console.log(it.next().done); // true
{% endhighlight %}

We can write a `function` that provides this behavior. We just need to pass the array as an argument. We can call this function `createIterator`.
{% highlight javascript %}
function createIterator(arr) {
    var i = 0;
    return {
        next: function next() {
            var returnVal;
            if (i < arr.length) {
                return {
                    value: arr[i++],
                    done: false
                };
            }
            return {
                done: true
            };
        }
    }
}

var iterator = createIterator(arr);
{% endhighlight %}
The function needs to maintain state of the index which is available in the returned object after it is initialized and is incremented each time `next` is called. When the end of the array is reached, it just has to return one `Boolean` `done` property with value `true`.

The other methods specified under `Array` that returns an iterator object are `entries` and `keys`. The method `entries` return key/value pairs of each element when the `value` property is accessed while `keys` return just the indices.
{% highlight javascript %}
var itEntries = arr.entries();
var itKeys = arr.keys();

console.log(itEntries.next().value); // [0, 'h']
console.log(itKeys.next().value); // 0
{% endhighlight %}
In `node.js` until version 7, the current version at the time the article was being written, `Array.prototype.values` is not supported.

However, `Arrays` are **iterable**. This means it defines its own iteration behavior such that its values can be looped over using `for..of`. This is again part of `ES2015` that provides a way to loop over **iterable** objects. This means that the `Array` has a property called `Symbol.iterator` as one of its keys that implements an `iterator`. It is a built-in `Symbol` exposed post `ES5`. So, even though `Array.prototype.values` is not supported, we can use the `createIterator` described above or use the built-in `Symbol.iterator` to make use the concept of iterators.
{% highlight javascript %}
var arr = ['h', 'e', 'l', 'l', 'o'];

var iterator = arr[Symbol.iterator]();
console.log(iterator.next()); // {value: 'h', done: false}
{% endhighlight %}
There are other Javascript built-in types as well other than arrays that have default iteration behavior such as `Map`, `Set`, `String` to name a few. Even they have `Symbol.iterator` property and methods that return an iterator object.

`Java` implements collections under the collection framework part of Java utils, `java.util.Collection`. In `C++`, they are called **Containers** and are part of the standard template library.

To learn more about `Symbol`, use this [reference](https://hacks.mozilla.org/2015/06/es6-in-depth-symbols/).
