---
layout: post
title: "Calling a function multiple times"
permalink: calling-a-function-multiple-times-declaratively-in-javascript
date: 2016-06-15 22:44:40
comments: true
description: "Calling a function multiple times declaratively in Javascript"
keywords: "declarative, function, javascript"
categories:

tags:

---
Just recently I watched the WWDC keynote. In that they introduced a new app to teach Swift programming language. The lessons taught programming through a game. In one of the lessons, the objective was to move the game character forward and grab a gem. The character had to move forward 3 times before collecting the gem. So, it was called 3 times.
{% highlight swift %}
moveForward();
moveForward();
moveForward();
{% endhighlight %}
As programmers we ask ourselves how can we avoid this duplication or code repetition. The most obvious answer is using loops which is imperative.
{% highlight javascript %}
for (var i = 0; i < 3; i++) {
  moveForward();
}
{% endhighlight %}
If there are other functions that need this behavior we are going to have many such functions with for loops around them. This can be generalized by writing a function.
{% highlight javascript %}
function times(func, number) {
  for (var i = 0; i < number; i++) {
    func();
  }
}
{% endhighlight %}
Functions being first class objects in Javascript allow them to be passed as arguments to a function and then called inside. We just need to pass the function the needs repetition and how many times it needs to be repeated.
{% highlight javascript %}
times(moveForward, 3);
{% endhighlight %}
It is looking more declarative now. We can go one step further and attach this as a method of the function as functions are objects in Javascript.
{% highlight javascript %}
moveForward.times = function(num) {
  for (var i = 0; i < num; i++) {
    this.call();
  }
}
{% endhighlight %}
Here we are using `this` to refer to the function which behaves as an object here and using it's call method available in `Function.prototype.call`. Now, we just need to treat the function as an object and call it with its newly attached method.
{% highlight javascript %}
moveForward.times(3);
{% endhighlight %}
To make it available across all functions we can attach this method to the prototype of Javascript's `Function`.
{% highlight javascript %}
Function.prototype.times = Function.prototype.times || function(num) { // check to see if this method was already attached
  for (var i = 0; i < num; i++) {
    this.call();
  }
}
{% endhighlight %}
If we needed to pass arguments to our original function, we can rewrite the implementation to incorporate variable arguments as the arguments passed is available as an `Object` called `arguments` inside the function. Then, we can use a common trick to convert the arguments to an array so that it can be passed to the `apply` method of the `Function` which is another method similar to `call` but accepts arguments as an array.
{% highlight javascript %}
Function.prototype.times = Function.prototype.times || function() { // check to see if this method was already attached
  var args = Array.prototype.slice.call(arguments);
  var length = args.length;
  var num = length && Number.isInteger(args[length - 1]) && args[length - 1] || 0; // Treat the last argument as the count. If no arguments the number of repetitions is zero
  for (var i = 0; i < num; i++) {
    this.apply(this, args.pop()); // Except the last argument, the rest of them are actual arguments to the function.
  }
}
{% endhighlight %}
Here is how we can call it if we have a function that accepted an argument,
{% highlight javascript %}
function print(message) {
  console.log(message);
}
print.times('Hello', 3);
{% endhighlight %}
'Hello' will be printed 3 times.

This was just an overview of the power of functions in Javascript which can be used to provide a declarative approach to calling a function multiple times making the invocation generic, reusable and readable.
