---
layout: post
title: "The coming together of Map, Filter and Reduce as higher order functions"
date: 2019-02-03 00:49:56
comments: true
description: "The coming together of Map, Filter and Reduce as higher-order functions"
keywords: "map, filter, reduce, functions, higher-order"
categories:

tags:

---
The way we deal with organizing data at a huge enterprise is by categorizing them in tables split into specific columns designated for a particular piece of data. For instance, data about purchases made by their customers may be organized in a table as follows:

|Product ID|Name|Manufacturer|Price|Quantity|Customer ID|Date|
|:---:|:----:|:---:|:---:|:---:|:---:|:---:|
|454353|GPS|Garmin|59.99|1|56456456|12-01-2018|
|904905|Smartphone|Apple|899.99|1|98292340|11-04-2018|
|902343|Camera|Canon|299.99|1|23423423|12-20-2018|
|234233|Smartphone|Google|799.99|1|34324213|11-18-2018|
|123123|Smartphone|Apple|999.99|1|21312321|10-05-2018|

It is an important requirement to store this data and keep it organized. But, a more practical requirement is to derive some meaning out this vast rows of data. It could be, how many smartphones Apple sold in the fourth quarter? or, what was the base price they sold at? So, from a table of so many rows and columns, we need to derive this information from specific rows and columns and combine them into one result. This can be done by selecting, filtering and aggregating, features that are available in spreadsheets or by scripting queries offered by databases which also stores this data.
{% highlight sql %}
select min(price) from purchases where manufacturer = "Apple"
{% endhighlight %}
This can also be accomplished within the realms of functional programming through the concepts of *Map*, *Filter* and *Reduce*. This could very well be accomplished imperatively but, the approach seems cumbersome. Assuming the data is available as an `Array`,
{% highlight javascript %}
{
    "purchases": [
        {
            "productId": 454353,
            "name": "GPS",
            "manufacturer": "Garmin",
            "price": "59.99",
            "quantity": 1,
            "customerId": 56456456,
            "date": "12-01-2018"
        },
        ...
        {
            "productId": 123123,
            "name": "Smartphone",
            "manufacturer": "Apple",
            "price": "999.99",
            "quantity": 1,
            "customerId": 21312321,
            "date": "10-05-2018"
        }
    ]
}
{% endhighlight %}
we can find the minimum price that Apple sold at,
{% highlight javascript %}
for (let i = 0; i < purchases.length - 1; i++) {
    if (purchases[i].manufacturer === 'Apple') {
        let min = purchases[i].price;
        if (purchases[i+1].price < min) {
            min = purchases[i+1].price;
        }
    }
}
return min;
{% endhighlight %}
Now, let's discuss *Map*, *Reduce* and *Filter* individually, so that we can apply them to achieve the same result.

#### Map
In JavaScript, the language of choice in the article, *Map* is a method of the `Array` object. It's syntax is specified as:
{% highlight javascript %}
var newArray = arr.map(function callback(currentValue[, index[, array]]) {
    // Return element for newArray
}[, thisArg])
{% endhighlight %}
It creates a new array using the results returned from the `callback` function which operates on the original array `arr`. In our example, it can be used to generate a new array containing only the prices.
{% highlight javascript %}
purchases.map((currentVal) => currentVal.price) // ['59.99', ..., '999.99']
{% endhighlight %}
This is how it can be implemented:
{% highlight javascript %}
function map(array, callback) {
    var result = [];
    for (let i = 0; i < array.length; i++) {
        result.push(callback(array[i], i, array));
    }
    return result;
}
{% endhighlight %}
Here, it is implemented as a standalone function which accepts the array as one of its arguments and a function passed as an argument that should accept 3 parameters, the current value, the index and the array itself. It stores the results of the callback function when called, in a new array thereby, keeping the contents of the original array undisturbed.

#### Filter
Filter's syntax is specified as:
{% highlight javascript %}
var newArray = arr.filter(callback(element[, index[, array]])[, thisArg])
{% endhighlight %}
Again it creates a new array but, they will contain only items which pass the test of the callback function. It should return a `Boolean`. In our example, it can be used to filter only those purchases who manufacturer is Apple.
{% highlight javascript %}
purchases.filter((element) => element.manufacturer === 'Apple'); //[{"productId": 904905, "manufacturer": "Apple", ....}, {"productId": 123123, "manufacturer": "Apple",....}]
{% endhighlight %}
This is how it can be implemented:
{% highlight javascript %}
function filter(array, callback) {
    var result = [];
    for (let i = 0; i < array.length; i++) {
        if (callback(array[i], i, array)) {
            result.push(array[i]);
        }
    }
    return result;
}
{% endhighlight %}
This time it only stores results in a new array only if the callback function after operating on each element, returns `true`.

#### Reduce
Reduce's syntax is specified as:
{% highlight javascript %}
var newArray = arr.reduce(callback(accumulator, currentValue, currentIndex, array)[, initialValue])
{% endhighlight %}
The callback function is a reducer function that operates on each individual element of the array but, produced a single value result. The reducer's returned value is assigned to the accumulator which is remembered for the next iteration and is updated each time we operate on the array and when we reach the end, it becomes the final returned value. In our example, it can be used to find the minimum price of the purchased items,
{% highlight javascript %}
purchases.reduce((accumulator, currentValue) => Math.min(accumulator.price, currentValue.price), Number.MAX_VALUE); // 599.99
{% endhighlight %}
This is how it can be implemented:
{% highlight javascript %}
function reduce(array, callback, initialValue) {
    var result = initialValue || array[0];
    for (let i = 0; i < array.length; i++) {
        result = callback(result, arr[i], i, array);
    }
    return result;
}
{% endhighlight %}
The result that we end up returning was actually the accumulator in every iteration that the reducer function gets called.

They are higher-order functions because each of them accept a function as an argument or parameter because functions are first class objects in JavaScript. If we put together all the three, we can determine the base price at which Apple sold.
{% highlight javascript %}
purchases.filter((element) => element.manufacturer === 'Apple')
.map((currentVal) => currentVal.price)
.reduce((accumulator, currentValue) => Math.min(accumulator, currentValue)) // 899.99
{% endhighlight %}
It is less verbose than the imperative method we saw earlier and clearly much easier to understand and similar in approach to queries offered by databases. We could argue that the imperative method is better in terms of time because there is only one `for` loop giving it linear time complexity. Even though the *Map/Filter/Reduce* approach interally ends up looping multiple times, we can make a case that they are not executed with one inside the other and still continues to have linear time complexity because they are executed one after the other. And, it is less verbose offering better maintanability, avoiding writing completely different implementations for deriving some other meaningful metrics that we might be interested in.