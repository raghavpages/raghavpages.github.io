---
layout: post
title: "Chaining middleware in express routes"
date: 2018-09-16 23:26:43
comments: true
description: "Implementation specifics of how middleware are chained in express"
keywords: "middleware, chaining, express, javascript, node.js"
categories:

tags:

---
Express.js is a highly popular framework for writing server-side code for web applications in node.js. It is a lightweight framework that simplifies `HTTP` request and response handling from the point the server starts and listens, to sending the response. It is popular among those who have just started writing their first node.js application and among those writing their nth application and want to do it in a quick and easy way. Of course, application generators make it easy to bootstrap an application. Still, the pattern it exposes is intuitive to understand and convenient to write that allows reading requests in any format and respond in any format through pluggable libraries that manage these. It is the support for this that makes it a powerful framework to use. It is the middleware pattern exposed through `app.use` as follows:
{% highlight javascript %}
var express = require('express');
var app = express();
app.use(function(req, res, next) {
  next();
});
{% endhighlight %}
It expects a **function** that has arguments `req`, `res`, `next`. The `req` and `res` are for it to pass the request and response objects. The `next` argument is a **function** `callback` that allows it to proceed to the next middleware if there are any. So, in a generated express application, in the index file used by the server startup script, there may be many `app.use` statements to chain these middleware functions. For example, a small part may look like this:
{% highlight javascript %}
app.use(bodyParser.urlencoded());
app.use(cookieParser());
{% endhighlight %}
The first one parses requests in the format `application/x-www-form-urlencoded` and populates the data in `req.body`. The second one parses the cookies in the request headers and populates them in `req.cookies`. Because the pattern propagates the request object across the middleware chain, each middleware is able to append properties to the request object. 

It can also run a middleware type function only for a particular `URL` path. The usage goes like:
{% highlight javascript %}
app.use('/', index);
{% endhighlight %}
The `index` can just be a function with the same signature, that is, having arguments `req`, `res` and `next`. Or, it can be an exported function from another file that uses `express.Router()`. The purpose is to process functions required for requests only for particular `HTTP` methods. For example the index file can look like:
{% highlight javascript %}
var express = require('express');
var router = express.Router();

router.get('/', function(req, res, next) {
  next();
});

module.exports = router;
{% endhighlight %}
So, the above middleware type function will only be called for requests to the base path `/` of method `GET`. It also permits chaining multiple middleware functions because the implementation of same pattern is continued here, allowing functions with arguments `req`, `res` and `next` to be passed. This is what it may look like:
{% highlight javascript %}
router.get('/', middleware1, middleware2);
{% endhighlight %}
Let's think about how to implement such a chaining. Assuming that `req` and `res` is already processed and available and `URL` `path` specific execution is already taken care, we just need to focus on how to achieve the chaining. `router.get` is essentially a **function** whose first argument is of type **string**. The remaining arguments can be one or more because the pattern permits chaining any number of `middleware` functions. Now that the **rest parameter** syntax is available and implemented in the latest JavaScript engines, the function can be defined as,
{% highlight javascript %}
function get(path, ...middleware) {}
{% endhighlight %}
This allows passing indefinite number of arguments and represent them as an `array`. Now, it's about taking all of this and implement the chaining depending on whether, `next` is called in that respective middleware. A rudimentary implementation would look something like:
{% highlight javascript %}
function get(path, ...middleware) {
    var req = {}; var res = {}; var next; var i = 0;

    function next() {
        i += 1;
        if (typeof middleware[i-1] === 'function') {
            middleware[i-1](req, res, next);
        }
    }

    next();
}
{% endhighlight %}
Basically, the calling of `next` in one of the middleware means, the `next` middleware function will be called within it. So, we just increment to the next middleware and call it inside `next` and continue passing `next`. Since, we are also calling the first middleware, `next` needs to be explicitly called within this `function`'s implementation at the end. So, when we do,
{% highlight javascript %}
function middleware1(req, res, next) {
  console.log('Inside middleware 1');
  next();
}

function middleware2(req, res) {
  console.log('Inside middleware 2');
}

get('/', middleware1, middleware2);
{% endhighlight %}
This will be the output:
{% highlight shell %}
Inside middleware 1
Inside middleware 2
{% endhighlight %}
If you still are attached to the `ES5` world, we still have access to the arguments objects and have to apply a common trick revealed by advanced JavaScript programmers who are not magicians because they've revealed their trick. So the implementation in `ES5` would be:
{% highlight javascript %}
function get(path) {
    var req = {}; var res = {}; var next; var i = 0;
    var argsArray = Array.prototype.slice.call(arguments); // the trick
    argsArray.shift();
    var middleware = argsArray;

    function next() {
        i += 1;
        if (typeof middleware[i-1] === 'function') {
            middleware[i-1](req, res, next);
        }
    }

    next();
}
{% endhighlight %}
We can also get rid of maintaining the count of `i` by using the `shift` method of the **Array** object.
{% highlight javascript %}
function get(path, ...middleware) {
    var req = {}; var res = {}; var next; var func;

    function next() {
        func = middleware.shift();
        if (typeof func === 'function') {
            func(req, res, next);
        }
    }

    next();
}
{% endhighlight %}
But, now we are maintaining a temporary `func` variable to store the shifted function.

We now have a fair idea of how we can chain middleware with a simple few lines of code. `Express` also supports passing `err` as the first argument instead of `req`. So there will be four arguments `err`, `req`, `res` and `next` to the function. This is normally used for error handling when there is a short circuit in the chaining. The short circuit can be achieved by passing the `err` object to `next` like `next(err)`. We just need to incorporate the checks for `err` in `next` in our implementation to achieve the short circuit behavior. We can even write asynchronous functions in each middleware and call `next` in the callback. But, being able to chain pluggable middleware functions during the journey of the request makes `express` a scalable, powerful and easy to use framework to start writing and getting out server-side code of `node.js` applications, quickly. 
