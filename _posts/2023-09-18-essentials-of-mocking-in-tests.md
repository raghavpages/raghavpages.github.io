---
layout: post
title: "Essentials of mocking in tests"
date: 2023-09-18 09:23:00
comments: true
description: "Essentials of mocking in tests"
keywords: "mocks, unit tests, code coverage, implementation, call count, arguments"
categories:

tags:

---
It is definitely satisfying to see the code you've written is implementing something working the way you imagined it. However, when you are ready to commit it and create a pull request for those changes you realize it might break the CI because of your code not being covered by unit tests. So, you go about writing unit tests. It is easy enough if your code doesn't have any dependencies on other libraries, you can change your inputs and get predictable outputs for it. But, when there are dependencies, it is not always the case to get predictable outputs. That's when writing unit tests might stumble you in the development process. The problem becomes more clear when your dependency is a HTTP request making an API call or a function that does cryptography. Consider the following example:
{% highlight javascript %}
const getFuelPrices = async () => {
    const response = await fetch("https://api.fuelprices.com/all");
    return response.json();
}
{% endhighlight %}
You can write a unit test by calling this function in your test and asserting for whatever response it gives back.
{% highlight javascript %}
test('Check fuel price response', async () => {
    const fuelPrices = await getFuelPrices();
    assert.equal(fuelPrices, '{"gasoline":"$5.3","diesel":"$3.1"}');
});
{% endhighlight %}
Firstly, this is a very naive approach because the prices keep changing. One way is to have placeholder for the prices in the assertion. However, when testing this particular function individually, it is not necessary to involve an actual API call. The tests would run slowly and if there is a connection problem with the API, it would throw off the assertions. There is benefit in testing the actual API as well but, those kind of tests belong to a separate category which is not in scope of this discussion.

Alternatively, an approach that has gained wide recognition is to mock the original API call such that the actual API is never called. Instead the actual API call's reference is replaced by a function that mocks how the actual function would have responded without sending the request over the network. This is how it can be done:
{% highlight javascript %}
fetch = async () => {
    return {
        json: async () => '{"gasoline":"$5.3","diesel":"$3.1"}'
    }
}
test('Check fuel price response', async () => {
    const fuelPrices = await getFuelPrices();
    assert.equal(fuelPrices, '{"gasoline":"$5.3","diesel":"$3.1"}');
});
{% endhighlight %}
In Javascript, since functions are first class objects, it is possible to change the reference of fetch function to another function for setting up the mock like above. Since tests run separately, it won't affect `fetch`'s original reference in actual runtime. But, it is a common pattern to restore the mock once the test has run:
{% highlight javascript %}
let originalFetch = fetch;
fetch = async () => {
    return {
        json: async () => '{"gasoline":"$5.3","diesel":"$3.1"}'
    }
}
test('Check fuel price response', async () => {
    const fuelPrices = await getFuelPrices();
    assert.equal(fuelPrices, '{"gasoline":"$5.3","diesel":"$3.1"}');
    fetch = originalFetch;
});
{% endhighlight %}

Mocks can offer additional capabilities. In order to see that, we can generalize the way mock is created and set up:
{% highlight javascript %}
const mock = (fn) => {
  let myFn = (...param) => {
    return fn(...param);
  };

  return myFn;
}
{% endhighlight %}
It allows passing a custom function as an argument and then it returns a function that accepts arguments the way the original function would have using the rest parameters and passes that as arguments to the custom function while calling and returning the custom function. The returned function will get executed only when the mock function is called which happens when the test is run. For an async version, the returned function is specified with an async keyword when the passed in custom function returns a promise.
{% highlight javascript %}
const mockAsync = (fn) => {
  let myFn = async (...param) => {
    return fn(...param);
  }

  return myFn;
}
{% endhighlight %}
We can now use this to assign the custom mock function before the test.
{% highlight javascript %}
fetch = mockAsync(async () => ({
        json: async () => '{"gasoline":"$5.3","diesel":"$3.1"}'
    })
  )
{% endhighlight %}
To see and build other capabilities of mocking, lets try with another example. 
{% highlight javascript %}
const getDaysLeft = (futureDate) => {
  const futureTime = ((new Date(futureDate))).getTime();
  return (futureTime - Date.now())/(1000*60*60*24);
}
{% endhighlight %}
{% highlight javascript %}
console.log(getDaysLeft('12/2/2023')); // 74.43248821759259 at the time of writing 
{% endhighlight %}
It is not possible for the current date to be the same every time the test is run. So, we can mock `Date.now` to return a constant date. Another option is, it may be sufficient to see if `Date.now` was called. This is when mocks offer an additional capability of spying. To enable that we can modify the mock implementation:
{% highlight javascript %}
const mock = (fn) => {
  let called = 0;
  let myFn = (...param) => {
    called += 1;
    myFn.called = called;
    return fn(...param);
  };

  return myFn;
}
{% endhighlight %}
Since functions are essentially objects in Javascript, we can add additional properties and it will still have the qualities of a function. So we can add a new property `called` to `myFn` and assign the private variable `called` in the closure to it. When we write the test, we can assign the mock to `Date.now` and pass in a custom function.
{% highlight javascript %}
let originalDateNow = Date.now;
Date.now = mock(() => (new Date('9/18/2023')).getTime());
test('getDaysLeft', () => {
  assert.equal(Math.round(getDaysLeft('12/2/2023')), 74);
  assert.equal(Date.now.called > 0, true);
});
{% endhighlight %}
By being able to assert how many times the function is called, it is also possible to know whether the function was at all. It enough to actually check if the number of times the function is greater than `0` and see if it is `true`.

The last capability we'll be seeing is checking what arguments are getting passed to the mock function. We can add that easily because our returning function accepts dynamic parameters using Javascript's `Rest` parameters.
{% highlight javascript %}
const mock = (fn) => {
  let called = 0;
  let calledWith = [];
  let myFn = (...param) => {
    called += 1;
    calledWith.push(param);
    myFn.called = called;
    myFn.calledWith = calledWith;
    return fn(...param);
  };

  return myFn;
}
{% endhighlight %}
We just introduced one more property `calledWith` to `myFn` and a private variable `calledWith` which is an `Array`. So when the returning function gets called we push the parameters passed to it into the `calledWith` array and then assign it to `calledWith` property of `myFn`. To see this action consider this example:
{% highlight javascript %}
const isSendReminder = (expiryDate) => getDaysLeft(expireDate) < 30;
{% endhighlight %}
The above function would be used to see whether a reminder message or notification needs to be sent when the current date is nearing expiry of something like say credit card. In our test, we can simply mock `getDaysLeft` itself:
{% highlight javascript %}
getDaysLeft = mock(() => 28);
test('isSendReminder', () => {
  assert.equal(isSendReminder('10/14/2023'), true);
  assert.deepEqual(getDaysLeft.calledWith, [['10/14/2023']]);
});
{% endhighlight %}
So the property `calledWith` is an array and each element of it is also an array, an array of parameters passed to the mock each time it is called. It is an array of parameters because a function can be called with multiple arguments. And, `calledWith` is an array because the function being mocked can be called multiple times.

So to summarize these are the basic capabilities of mocking in tests:
* Replace reference of an indirectly called function with a custom function in a test that is testing a function having other dependencies.
* Spy the indirectly called function to see how many times it is called and by extension whether it is called at least once or not.
* Spy the indirectly called function to see whether the actual implementation is passing arguments to it as expected.

Mocks do let us cheat our way out when writing writing unit tests but it saves us from troubleshooting and knowing the intricacies of every indirectly called function from a test that tests the intended function. As we saw, it was pretty straight forward to create a mocker with the capabilities listed above and thereby give us an understanding of how it works on the inside. This is anyway abstracted away from us by unit testing frameworks which offer more elaborate mocking capabilities such as mocking indirect imports and way to clear or restore mocks through before and after hooks in a test file.

With this, hope you have enough power in writing unit tests and boast about high coverage numbers to the leadership in the Software Industry.
