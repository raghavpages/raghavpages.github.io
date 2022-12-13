---
layout: post
title: "Distilling the essence of async tasks in Javascript"
date: 2016-05-21 15:45:28
comments: true
description: "Distilling the essence of async tasks in Javascript"
keywords: "async, nodejs, javascript"
categories:

tags:

---
One of the things you have deal with when migrating to `node.js` from other server side platforms is code that executes asynchronously. What do I mean by asynchronous? When reading a file or making a request over a network we call a function and after the fetching of the file or completion of request, it is handled in a callback function passed as a parameter to the function. Here is an example:
{% highlight javascript %}
http.get('http://raghavpages.github.io', function (res) {
  console.log('Status: ' + res.statusCode);
}).on('error', function (e) {
  console.log('Error:' + e.message);
});
{% endhighlight %}
This is pretty straight forward for one request. However, if there is a need to make another request because your final result depended on the both the requests, you would have to do something like:
{% highlight javascript %}
http.get('http://raghavpages.github.io', function (res1) {
  http.get('http://raghavpages.github.io/about', function (res2) {
      console.log('Status 1: ' + res1.statusCode);
      console.log('Status 2: ' + res2.statusCode);
    });
}).on('error', function (e) {
  console.log('Error:' + e.message);
});
{% endhighlight %}
The code already seems hard to follow and becomes harder if you throw in your logic for each request. Imagine if you have to make multiple such requests, your code would start resembling a pyramid pointing to the right of your screen or the Hogwarts Stairways.

There are many `async` libraries that help you work with the above mentioned use case. I plan to show how they are implemented and thereby ascertain if it makes sense to use them.

As a example, consider two tasks. They are simple `setTimeout` functions. But, they help simulate time taken to complete a call in a tangible manner.
{% highlight javascript %}
function task1() {
  setTimeout(function () {
    console.log('2000');
  }, 2000);
}

function task2() {
  setTimeout(function () {
    console.log('1000');
  }, 1000);
}
{% endhighlight %}
Now if we were to call both of them and do something after both of them complete, we would have to call one task inside another task and do that something inside the former's task. Example:
{% highlight javascript %}
function task1() {
  setTimeout(function () {
    console.log('2000');
    task2();
  }, 2000);
}

function task2() {
  setTimeout(function () {
    console.log('1000');
    // Do something you want to do
  }, 1000);
}
{% endhighlight %}
This seems less scalable because we may sometimes want `task2` to complete before `task1` or call both of them so that the faster one completes while the slower one is executing and after the slower one is done, do something. The first one alludes to a serial type of calling and second alludes to a parallel type of calling. We can't achieve them by constantly changing the implementation of the original functions.

Let us look at serial calling first and implement it. So, we have two tasks and we want them to be called one after another and do something after the last one is complete. We can write a function that accepts the two tasks as an array and a callback to do something after the tasks are complete. This is the boilerplate of such a function:
{% highlight javascript %}
asyncSeries(tasks, callback);
{% endhighlight %}
So, we may have to iterate over the tasks and pass the upcoming task in the callback of the previous task so that it executes after the previous task is complete. So, we have to rewrite our tasks like this:
{% highlight javascript %}
function task1(callback) {
  setTimeout(function () {
    console.log('2000');
    callback();
  }, 2000);
}

function task2(callback) {
  setTimeout(function () {
    console.log('1000');
    callback();
  }, 1000);
}
{% endhighlight %}
Notice that each task now accepts a `callback` and is called after the asynchronous call is done. Now, we can proceed with the implementation of `asyncSeries`. This is how it looks:
{% highlight javascript %}
function asyncSeries(tasks, callback) {
  var task = tasks.shift();
  if (task) { // check if any tasks are left
    task(function() {
      asyncSeries(tasks, callback); // call main function again in the callback of the task
    });
  } else {
    callback(); // call the final callback after the tasks are complete (base case)
  }  
}
{% endhighlight %}
Now, we can make use of the above function with the above created tasks.
{% highlight javascript %}
var tasks = [task1, task2];

asyncSeries(tasks, function() {
  console.log('done');
});
{% endhighlight %}

Let us see parallel calling. It also has the same boiler plate. Accepts tasks as an array and a callback after the tasks are complete.
{% highlight javascript %}
asyncParallel(tasks, callback);
{% endhighlight %}
In case, of parallel, we would like to call both the tasks without waiting for one of them to complete. But, we still need to keep track of when all the tasks are complete so that we can do something after the tasks are done. Such an implementation would look like:
{% highlight javascript %}
function asyncParallel(tasks, callback) {
  var cnt = 0;
  var total = tasks.length;
  while (tasks.length) {
    var task = tasks.shift();
    task(function() {
      cnt +=1; // keep track when each task is complete
      if (cnt === total) {
        callback(); // when all the tasks are done call the final callback
      }
    });
  }
}
{% endhighlight %}
It is called the same way as `asyncSeries`:
{% highlight javascript %}
asyncParallel(tasks, function() {
  console.log('done');
});
{% endhighlight %}

Hope this gives you a better idea before you jump onto an `asnyc` library next time. This was a rudimentary implementation which tried to cover the concept. Most `async` libraries could be similar and have more features that would pass the results from one task to the other, aggregate all the results and make them available in the final callback, error handling by calling the final callback when one of tasks fail etc. Have a look at them and use which one suits best for your use case.

Congratulations if you have reached until this point in the article. One thing notable about `asyncSeries` is the recursive call which is suboptimal in `ECMAScript 5`. With the maturity of `Javascript` and release of improved versions of `ECMAScript`, using natively supported `promises` would be wiser than choosing to import another library.
