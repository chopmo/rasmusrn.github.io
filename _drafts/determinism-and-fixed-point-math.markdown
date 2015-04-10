---
title:  "Determinism and fixed point math"
layout: "post"
date:   2015-05-10 10:10:00
---
In the game I'm currently working on you control one character a bit like in the Diablo games. Last week I realized the game would be so much better if you'd control a group of characters instead of just one. I decided I had to give it a try.

For technical reasons, this meant I'd have to use the socalled *lockstep* network synchronization technique. Most RTS games uses it because it lets you network hundreds, if not thousands of entities. It's awesome, actually. Normally, the game server simulation the game and send updates to clients. But with the lockstep technique, each client simulate the entire game themself so the server doesn't have to send state updates.

But it comes at price: your game state simulation must be 100% deterministic down to the very last bit. Even the tiniest differences would accumulate over time and cause problems. It's not as easy as it may sound.

Most simulations rely on floating point math. And even though floating point math is [standardized](http://en.wikipedia.org/wiki/IEEE_floating_point) it turns out to be [rather difficult](http://gafferongames.com/networking-for-game-programmers/floating-point-determinism/) to get deterministic behavior across operation systems, compilers, and CPU architectures. And even if you do manage to get basic arithmetic working properly, there's also trancendental functions (`sin()`, `cos()`, etc.).

I deemed it a battle not worth fighting. Instead, I learned that there's another way to do real number arithmetic using only integer CPU operations. This is great because integer operations are deterministic out of the box across compilers, OSs, CPUs. Determinisn and real number support - that's exactly what I was looking for! Yay!

The technique is called fixed point math and despite its limitations it's very cool. Historically, fixed point math have been very popular in games and graphics application because of its high performance. But since CPUs during the mid 90s got dedicated floating point units, its less popular today.

Normally, we implicitly regard integers as a number of ones. For example, 5 represent 5 ones, 5*1=5. Stick with me, this'll make sense in a second. In fixed point math, you instead regard integers as a number of some fraction. You can choose any fraction you like (well almost).

For example, let's say we chose 1/2 aka a half as our fraction. The integer 5 would then represent 5 halves aka 2.5 instead of 5 ones, 5*(1/2)=2.5.

In my library (creatively named Fixie) I chose the fraction 1/1024. That means the number 1 is represented by the integer 1024 and the number 0.5 is represented by the integer 512:

{% highlight cpp %}
1024 * (1/1024) = 1

512 * (1/1024) = 0.5
{% endhighlight %}

If this makes your head spin a little, you're not alone. But fortunately, we can hide the complexity in classes and functions. For example using C++'s custom operator this is how Fixie is used:

{% highlight cpp %}
Fixie::Num a = 1;
Fixie::Num b = 2;
Fixie::Num c = a/b;
printf("%f\n", (float)c);
{% endhighlight %}

This will print the correct results even though no floating point math was used. And the same goes for addition, subtraction, and multiplication. As the time of this writing I have implemented `sqrt()`, `floor()`, `sin()`, `cos()`, and `acos()`.

As an example of how this `Num` class works, here you can see how addition is implemented:

{% highlight cpp %}
class Num {
public:
  Num& operator+=(const Num &rhs) {
    raw += rhs.raw;
    return *this;
  }
};
{% endhighlight %}

`raw` is the base integer number, in my case the number of 1/1024ths. `Num x = 2` would yield an instance where raw is set to `2048`.

Division is an example of an operation that is more involved but still not too complex and fairly performant:

{% highlight cpp %}
class Num {
public:
  Num& operator/=(const Num &rhs) {
    assert(rhs.raw != 0);
    const int32_t resultNegative = ((raw ^ rhs.raw) & 0x80000000) >> 31;
    const int32_t sign = resultNegative*-2+1;
    int64_t temp = static_cast<int64_t>(raw) << fractionBits;
    temp += rhs.raw/2*sign;
    raw = temp / rhs.raw;
    return *this;
  }
}
{% endhighlight %}

The bitwise operations here are primarily to ensure "correct" rounding, e.g. -2/3=-1 (and not 0 which is the default for integer division).

I used all these basic primitives to implement vectors, matrices, and quaternions. So all constructs are ultimately based on deterministic integer operations while still allowing real numbers. Maybe its just me but I think it's pretty neat.

However, it is important to note that fixed point math has limitations. They generally support fewer significant digits that floating point so you'll get lower precision and smaller numeric ranges. That's not a problem for my game though, it might be for yours.

For this reason I'll probably only use Fixie in parts of the game where I need determinisn, that is the "state simulation", e.g. physics, AI, etc. Rendering and animation is not required to be synchronized down the last bit, so here I'll stick with good old floating point (`float`, `double`, etc.).

Full source of is available on GitHub [here](https://github.com/rasmusrn/fixie).

Learning about fixed point math and implementing Fixie have been great fun. Next I need to actually use it with my state simulation logic. When the state simulation is fully deterministic, I should be able to synchronize thousands of entities across the network. It will be glorious!

If you want to know more about fixed point math I recommend these great articles:

1. [Fixed-Point Numbers and LUTs](http://www.coranac.com/tonc/text/fixed.htm)
2. [Fixed Point Arithmetic and Tricks](http://x86asm.net/articles/fixed-point-arithmetic-and-tricks/)
3. [Doing It Fast](http://gameprogrammer.com/4-fixed.html)
