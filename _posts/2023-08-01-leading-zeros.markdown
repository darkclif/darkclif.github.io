---
layout: post
title:  "Number of leading zeros"
date:   2023-08-02 00:00:00 +0200
categories: bit hack
---

<!-- ## Simple loop

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
{% endhighlight %} -->

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

First, let's rewrite this line as ``n = 31 - ((i >> 23) - 127)``. On the right side we extract exponent bits of the float representation (we do not care about leftover sign bit because we are working with uint32 as input). Then we substitute $$127$$ to derive real value of $$exp$$. With this value we obtained information about position of most significant active bit. Then we substract the result from 31 because we care about number of 0's that are on the left side of it. 

Let's do a simple check - ``uint32 = 1`` will be stored inside ``float32`` as $$[f = (-1)^{0}*2^{0}*1.0]$$ so we have $$exp = 0$$ and exponent bits store value 127 as unsigned integer. Then $$31 - (127 - 127) = 31$$ and this is correct result because we have 31 leading zeros with ``uint32 = 1``. This will go like that until we reach $$exp = 31$$ for values between $$[2^{31},2^{32})$$ which will yield $$31 - (158 - 127) = 0$$ as a function result. 

Unfortunatelly there are some corner cases here which we need to handle. Sometimes we can reach $$exp = 159$$ due to conversion rounding. Sometimes we can reach next $$exp$$ value for lower numbers due to the same reason. Also, for ``uint32 = 0`` there is a problem because it will store all zeros inside 8 bits of exponent area and we will end up with ``n = -1`` which will eventually return 30 from the function and this is obviously wrong. That's why we need ``k = k & ~(k >> 1)`` line.

``k = k & ~(k >> 1)``

What we are doing here effectively is making this value a little lower but keeping most significant active bit intact. For the uint32 values that are very close to the top range of $$[2^{n},2^{n+1})$$ we can obtain value rounded to $$2^{n+1}$$ after conversion to float. This will happen exactly when we have more than 24 consecutive active bits stored inside initial uint32 argument. The value of 24 is connected to the lenght of matissa. We can store only 23 bits inside mantissa and 1 additional leading bit that is always set to 1 so effectively we can store 24 bits. If next 25th bit is set to 0 we are fine because float value will be rounded down - next bits are trimmed despite of their value. If next 25th bit is set to 1 the whole value is at least half way to the next value that can be stored inside 24 bits and final value will be rounded up. That will give us next $$exp$$ value which will provide incorrect result. By doing ``k = k & ~(k >> 1)`` we will clear all the bits that are present at the right side of the most significant active bit. This is enough to keep us safe from rounding errors.
 
``(n & 31) + (n >> 6)``

The last line handles the case when we pass ``uint32 = 0`` as function argument. In this situation all **exp** bits are set to 0 and $$n = 158$$. Then ``(n & 31) = 30`` and ``(n >> 6) = 2`` so we end up with 32 as result which is correct. In any other case ``(n >> 6)`` should just return 0 because only case when $$n$$ exceeds 31 is with ``uint32 = 0``.

## Benchmarks

**CPU:** AMD Ryzen 5 3600  
**System:** Windows 10  
**Compiler:** MSVC 19.36.32537  

Below code compares performance of two algorithms. In each test 1 milion random numbers are prepared, then **average**, **minimum** and **maximum** values are gathered over 100 samples.

{% highlight cpp %}
#include <stdio.h>
#include <float.h>
#include <chrono>
#include <vector>
#include <time.h>
using namespace std;
using namespace std::chrono;

/** Masking */
int pop(uint32_t x) {
    uint32_t n;
    n = (x >> 1) & 033333333333;
    x = x - n;
    n = (n >> 1) & 033333333333;
    x = x - n;
    x = (x + (x >> 3)) & 030707070707;
    return x % 63;
}

int nlz32_masking(uint32_t x) {
    x = x | (x >> 1);
    x = x | (x >> 2);
    x = x | (x >> 4);
    x = x | (x >> 8);
    x = x | (x >>16);
    return pop(~x);
}


/** Float magic */
int nlz32_float(uint32_t k)
{
    union
    {
        uint32_t    i;
        float       f;
    };
    int n;

    k = k & ~(k >> 1);
    f = (float)k;
    n = 158 - (i >> 23);
    n = (n & 31) + (n >> 6);
    return n;
}


/**
 *  Measure function time.
*/
long long timeit(std::vector<uint32_t>& randoms, int(*f)(uint32_t))
{
    auto start = high_resolution_clock::now();
    for(uint32_t& v : randoms)
    {
        f(v);
    }
    auto stop = high_resolution_clock::now();
    auto duration = duration_cast<microseconds>(stop - start);
    return duration.count();
}


int main()
{
    srand(time(0));

    uint32_t run_count = 100;
    double time_sum[] = {0.0, 0.0};
    double time_min[] = {DBL_MAX, DBL_MAX};
    double time_max[] = {0.0, 0.0};

    for(uint32_t r = 0; r < run_count; ++r)
    {
        /** Generate data */
        const int count = 1000000;
        std::vector<uint32_t> randoms(count);
        for(int i = 0; i < count; ++i)
        {
            randoms[i] = (uint32_t)rand();
        }

        /** Benchmark */
        double t0 = (double)timeit(randoms, &nlz32_masking);
        double t1 = (double)timeit(randoms, &nlz32_float);

        time_sum[0] += t0;
        time_sum[1] += t1;

        time_min[0] = std::min(time_min[0], t0);
        time_min[1] = std::min(time_min[1], t1);

        time_max[0] = std::max(time_max[0], t0);
        time_max[1] = std::max(time_max[1], t1);
    }

    time_sum[0] = (double)time_sum[0] / (double)(run_count);
    time_sum[1] = (double)time_sum[1] / (double)(run_count);

    printf("avg.time %10s: %10.3f microseconds.\n", "masking", time_sum[0]);
    printf("avg.time %10s: %10.3f microseconds.\n", "float", time_sum[1]);

    printf("min.time %10s: %10.3f microseconds.\n", "masking", time_min[0]);
    printf("min.time %10s: %10.3f microseconds.\n", "float", time_min[1]);

    printf("max.time %10s: %10.3f microseconds.\n", "masking", time_max[0]);
    printf("max.time %10s: %10.3f microseconds.\n", "float", time_max[1]);
}
{% endhighlight %}

| Algorithm   | Average     | Min        | Max        |
| ----------- | ----------- |------------|------------|
| Masking     | 7079.470μs  | 6754.000μs | 9036.000μs |
| Float       | 4556.250μs  | 4313.000μs | 7070.000μs |



### Links
[IEEE-754 Floating Point Converter](https://www.h-schmidt.net/FloatConverter/IEEE754.html)  
[Bit Twiddling Hacks by Sean Eron Anderson](https://graphics.stanford.edu/~seander/bithacks.html)  
 