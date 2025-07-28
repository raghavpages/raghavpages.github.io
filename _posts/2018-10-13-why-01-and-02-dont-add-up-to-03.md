---
layout: post
title: "Why 0.1 and 0.2 don't add up to 0.3?"
date: 2018-10-13 11:09:34
comments: true
description: "Why 0.1 and 0.2 don't add up to 0.3?"
keywords: "math, floating-point, numbers, arthmetic"
categories:

tags:

---
It is famously talked about in JavaScript books and the community that the `number` type is not very reliable and the example most commonly used to support such a conclusion is `0.1 + 0.2 !== 0.3`. How could a programming language that is arguably the most widely used, error out on something so obvious and intuitive? The reason is beyond the capabilities and limitations of this language before we start thinking "have they really screwed up on this one too?". You may think, "at least, my money is safe because this language is mostly used only in the browser". Don't mean to make you feel insecure about your money but, this language has already made huge inroads on the server-side. Also, there are many applications now doing computations on the browser and this problem is not just limited to one language. Let's look at this example in `C++`, a language used in most JavaScript Engine implementations,
{% highlight c++ %}
int main() {
    double a = 0.1;
    double b = 0.2;
    double c = 0.3;
    /*Printing up to 56 decimal places*/
    printf("a is %0.56f\n", a);
    printf("b is %0.56f\n", b);
    printf("Sum is %0.56f\n", a + b);
    printf("c is %0.56f\n", c);
    return 0;
}
{% endhighlight %}
JavaScript has only one `number` type which uses 64-bit double precision floating-point. The `printf` statements in the above code is expanding out the number to as many decimal places possible. This will give visibility to how the number is actually stored. This is the output:
{% highlight shell %}
a is 0.10000000000000000555111512312578270211815834045410156250
b is 0.20000000000000001110223024625156540423631668090820312500
Sum is 0.30000000000000004440892098500626161694526672363281250000
c is 0.29999999999999998889776975374843459576368331909179687500
{% endhighlight %}
It is the same result in JavaScript.
{% highlight javascript %}
console.log(0.1.toFixed(56));
console.log(0.2.toFixed(56));
console.log((0.1 + 0.2).toFixed(56));
console.log(0.3.toFixed(56));
console.log(0.1 + 0.2 === 0.3);
{% endhighlight %}
Output:
{% highlight shell %}
0.10000000000000000555111512312578270211815834045410156250
0.20000000000000001110223024625156540423631668090820312500
0.30000000000000004440892098500626161694526672363281250000
0.29999999999999998889776975374843459576368331909179687500
false
{% endhighlight %}
What? What are all those digits doing in the end? How did they appear? They are the ones messing up the results. It is even misrepresenting `0.3` as `0.2999...`. The reason is computers store numbers in binary. When it comes to storing integers, it has no trouble. Decimal `0` and `1` is just `0` and `1` in binary as well. Bigger numbers need more binary digits or bits. Decimal `2` is `10`, `3` is `11`, `4` is `100` etc. It uses the positions of the `0`s and `1`s in the bit to get the decimal representation. Since `1` is the maximum single digit in binary, higher its bit position, higher is its value. Its contribution to the number in decimal is higher. Also, more the number of `1`s, the number's value is also more. For example, in `10`, `1` is at the position `1` from right assuming positions are counted from `0`. That is why it represents decimal `2`. Its value is calculated as `1*2**1 + 0*2**0`. The bit's value is multiplied with a power of `2` of the position in which it is in and these results at every position are added together to get the decimal value. This is a standard formula to convert numbers from one system to the other. So, it is able to represent every integer and the number of integers it can represent is limited by the number of bit positions allowed by the computer.

So, what is the problem with floating-point numbers? `0.1` in decimal is `1/10` or `10^-1`. So, what would `0.1` be in binary? Since, everything is in powers of two in binary, and positions after the decimal have negative powers, `0.1` would be `2**-1`. And, if we convert it by computing the power, it is `1/2` or `0.5` in decimal. **Not** `0.1`. Similarly `0.01` would be `2**-2` or `1/4` or `0.25`. And, `0.001` would be `2**-3` or `1/8` or `0.125`. Now, it is getting closer to `0.1`. `0.0001` is `0.0625`. It has become smaller but, to accurately represent `0.1`, we seem to need more bits. Lets try `0.00011`. It is `0.09375`. This is much closer to `0.1`. But not exactly `0.1` yet. So, it is not so straightforward representing floating-point numbers on a computer. Computers don't even understand the decimal point. We need a more formal representation such as the IEEE 754 standard which is used in almost all computers now. It has bits allocated to indicate just the position of the decimal point. It uses scientific notation. This standard has more bits (52 bits) to represent `0.1` more precisely. But, not enough to represent it accurately. `0.1` is `1/10` or `1/1010` in binary. If we do the long division, we get `0.0001100110011...`. So, `0011` keeps repeating forever. Since, only finite number of bits are available, it cannot store the number accurately. 

It uses this fully stored format in computing the sum with other floating-point numbers, some of which cannot be represented accurately themselves. So, the problem is not just with the `0.1 + 0.2 !== 0.3`. There are other examples as well, `0.1 + 0.7 !== 0.8`, `0.8 + 0.9 !== 1.7`, `0.3 + 0.6 !== 0.9`, `0.21 + 0.7 !== 0.91` which is in no way an exhaustive list because there are way too many of them to exhaust so easily. It is like, `2/3 + 1/3 = 1`, but in decimal it is, `0.666... + 0.333... = 0.999...`. It will always be closer to `1` but never equal to `1`.

In order to safely deal with money, the numbers are operated on as integers and then the end result is divided by `100`. Or, we can check the equality this way: `(0.1 + 0.2).toFixed(2) === 0.3.toFixed(2)`. In financial computation, the result is almost always expected to be upto two places after the decimal point. In scientific computation, the equality is sometimes replaced by `(a - b) <` &epsilon;. Where, &epsilon; is the smallest number that can be tolerated by the application. There is a branch of Mathematics called Numerical Analysis that deals with the inaccuracies of floating-point numbers to minimize errors in more complex scientific and mathematical applications.

Hope this gives a better picture of the challenges that engineers and computer scientists go through in keeping our money safe and researchers happy in a system where reliance on a computer has become almost impossible to avoid.