---
layout: post
title:  "Number of leading zeros"
date:   2023-08-02 00:00:00 +0200
categories: bit hack
---

## Simple loop

TODO

{% highlight cpp %}
int nlz32_basic(uint32_t k)
{
    int shift = 0;
    while(((k >> (31 -shift)) == 0) && shift < 32)
    {
        ++shift;
    }

    return shift;
}
{% endhighlight %}

## Division loop

TODO

{% highlight cpp %}
int nlz32_div2(uint64_t x) {
    uint64_t y;
    int64_t n, c;
    n = 64;
    c = 32;
    do {
        y = x >> c;
        if (y != 0) {
            n = n - c;
            x = y;
        }
        c = c >> 1;
    } while (c != 0);
    return n - x;
}
{% endhighlight %}

## Masking 

TODO

{% highlight cpp %}
int pop(uint32_t x) {
    uint32_t n;
    n = (x >> 1) & 033333333333;
    x = x - n;
    n = (n >> 1) & 033333333333;
    x = x - n;
    x = (x + (x >> 3)) & 030707070707;
    return x % 63;
}

int nlz2(uint32_t x) {
    x = x | (x >> 1);
    x = x | (x >> 2);
    x = x | (x >> 4);
    x = x | (x >> 8);
    x = x | (x >>16);
    return pop(~x);
}
{% endhighlight %}

## Float magic

The last function use tricky uint to float conversion to obtain and use exponent part of float representation. 

{% highlight cpp %}
int nlz32(uint32_t k)
{
    union
    {
        uint32_t    i;
        float       f;
    }
    int n;

    k = k & ~(k >> 1);
    f = (float)k;
    n = 158 - (i >> 23);
    return (n & 31) + (n >> 6);
}
{% endhighlight %}

To fully understand above code let's recall how exactly float represenatation works.

To store float32 we dedicate **1 bit** for sign, **8 bits** for storing exponent and **23 bits** are used for mantissa.

* **Sign** is just 1bit flag, if it's set to 1 - value is negative.
* **Exponent** is stored on 8bits as unsigned int. To calculate final value of exponent we substract 127 from it. So i.e. bits set to 0b01111111 give value of 0. Bits set to 0b10000000 give value of 1. Etc.
* Finally **matissa** is stored on last 23 bits, also there is single invisible leading bit set to 1 at the beginning. After leading bit, fraction part of the number beginns. So leftmost bit of the mantissa set to 1 gives 1/2, next one gives 1/4 etc. Effectively we can store in mantissa value between [1.0, 2.0).

The final value of whole float32 can be calculated with equation: 

$$(-1)^{sign} * 2^{exp} * mantisa$$

Now lets step through the code line by line.

``union{ ... }``

For those without much knowledge of C++, with union we just declare multiple variables that occupy the same piece of memory. Here we can access the same 32 bits as uint32 and float.

``k = k & ~(k >> 1)`` 

Lets skip this one for now.

``f = (float)k``

Here conversion from uint32 to float32 takes place. Because mantissa is only able to encode values between $$[1.0, 2.0)$$, to encode integer value between $$[2^{n},2^{n+1})$$ we must store $$n$$ as exponent part.

$$[2^{n},2^{n+1}) = 2^{n} * [2^{0},2^{1}) = 2^{n} * [1.0,2.0)$$  

(Taking substracton into account it should be exactly $$n + 127$$) 

After that we need to store remaining $$[1.0, 2.0)$$ inside mantissa part.

``n = 158 - (i >> 23)``

First, let's rewrite this line as ``n = 31 - ((i >> 23) - 127);``. On the right side we extract exponent bits of the float representation (we do not care about leftover sign bit because we are working with uint32 as input). Then we substitute $$127$$ to derive real value of $$exp$$ and substract the result from 31. 

Let's do a simple check - ``uint32 = 1`` will be stored inside ``float32`` as $$[f = (-1)^{0}*2^{0}*1.0]$$ so we have $$exp = 0$$ and exponent bits store value 127 as unsigned integer. Then $$31 - (127 - 127) = 31$$ and this is correct result because we have 31 leading zeros with ``uint32 = 1``. This will go like that until we reach $$exp = 31$$ for values between $$[2^{31},2^{32})$$ which will yield $$31 - (158 - 127) = 0$$ as a function result. 

Unfortunatelly there are some corner cases here which we need to handle. Sometimes we can reach $$exp = 159$$ due to conversion rounding. Sometimes we can reach next $$exp$$ value for lower numbers due to the same reason. Also, for ``uint32 = 0`` there is a problem because it will store all zeros inside 8 bits of exponent area and we will end up with ``n = -1`` which will eventually return 30 from the function and this is obviously wrong. That's why we need ``k = k & ~(k >> 1)`` line.

``k = k & ~(k >> 1)``

What we are doing here effectively is making this value a little lower but keeping most significant active bit intact. For the uint32 values that are very close to the top range of $$[2^{n},2^{n+1})$$ we can obtain value rounded to $$2^{n+1}$$ after conversion to float. This will happen exactly when we have more than 24 consecutive active bits stored inside initial uint32 argument. The value of 24 is strictly connected to the lenght of matissa. We can store only 23 bits inside mantissa and 1 additional leading bit that is always set to 1 so effectively we can store 24 bits. If next 25th bit is set to 0 we are fine because float value will be rounded down - next bits are trimmed despite of their value. If next 25th bit is set to 1 the whole value is at least half way to the next value that can be stored inside 24 bits and final value will be rounded up. That will give us next $$exp$$ value which will provide incorrect result. By doing ``k = k & ~(k >> 1)`` we will clear all the bits that are present at the right side of the most significant active bit. This is enough to keep us safe from rounding errors.
 
``(n & 31) + (n >> 6)``


## Benchmarks

**CPU:** AMD Ryzen 5 3600  
**System:** Windows 10  
**Compiler:** MSVC 19.36.32537  

{% highlight cpp %}
int main()
{
    srand(time(0));

    // Generate data
    const int nnn = 1000000;
    std::vector<uint32_t> randoms(nnn);
    for(int i = 0; i < nnn; ++i)
    {
        randoms[i] = (uint32_t)rand();
    }

    // Div2
    {
        auto start = high_resolution_clock::now();
        for(uint32_t& v : randoms)
        {
            nlz32_div2(v);
        }
        auto stop = high_resolution_clock::now();
        auto duration = duration_cast<microseconds>(stop - start);
        cout << "t div2=" << duration.count() << " microseconds." << endl;
    }

    // Basic
    {
        auto start = high_resolution_clock::now();
        for(uint32_t& v : randoms)
        {
            nlz32_basic(v);
        }
        auto stop = high_resolution_clock::now();
        auto duration = duration_cast<microseconds>(stop - start);
        cout << "t basic=" << duration.count() << " microseconds." << endl;
    }

    // Masking
    {
        auto start = high_resolution_clock::now();
        for(uint32_t& v : randoms)
        {
            nlz32_masking(v);
        }
        auto stop = high_resolution_clock::now();
        auto duration = duration_cast<microseconds>(stop - start);
        cout << "t masking=" << duration.count() << " microseconds." << endl;
    }

    // Float
    {
        auto start = high_resolution_clock::now();
        for(uint32_t& v : randoms)
        {
            nlz32_float(v);
        }
        auto stop = high_resolution_clock::now();
        auto duration = duration_cast<microseconds>(stop - start);
        cout << "t float=" << duration.count() << " microseconds." << endl;
    }
}
{% endhighlight %}

| Algorithm   | Average     | Min      | Max      |
| ----------- | ----------- |----------|----------|
| Loop        |             |          |          |
| Loop div    |             |          |          |
| Masking     |             |          |          |
| Float       |             |          |          |

### Links
[IEEE-754 Floating Point Converter](https://www.h-schmidt.net/FloatConverter/IEEE754.html)  
[Bit Twiddling Hacks by Sean Eron Anderson](https://graphics.stanford.edu/~seander/bithacks.html)  
 