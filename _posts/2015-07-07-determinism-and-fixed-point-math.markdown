---
title:  "Determinism with fixed point math"
layout: "post"
date:   2015-07-07 10:00:00
private: true
---
My girlfriend and I felt like shaking up our daily life a little. So we decided to move to *Budapest* for a month. We arrived two weeks ago and so far it has been great. Only thing is that I am not that good with high temperatures. Here is my new best friend:

<p class="photo">
  <img src="/assets/images/fan.jpg" style="width: 400px">
</p>

As we had hoped, the new environment begets new thoughts. It's very interesting how the mind is affected. Perhaps I will write a full post about the experience when I get back home. But enough with that. Today I am going to talk about network synchronization, determinism, and fixed point math.

## Network synchronization

A couple of months ago I got the idea that my game would be so much better if you control a group of characters instead of just one. I decided I had to give it a try.

For technical reasons, this meant I would have to use the socalled *lockstep* network synchronization technique. Most RTS games uses it because it lets you synchronize hundreds (if not thousands) of entities over the internet. It is awesome, actually. With the regular *authoritative server model*, the game server simulates the game and send state updates to clients. This becomes infeasible if you have many entities. However, with the lockstep technique, each client simulates the entire game themself so the server doesn't have to send state updates.

But it comes at a price: your game state simulation must be 100% deterministic down to the very last bit. Even the tiniest differences would accumulate over time and cause problems. Unfortunately, achieving full cross platform determinism is not easy.

## Floating point problems

Most simulations rely on floating point math. And even though floating point math is [standardized](http://en.wikipedia.org/wiki/IEEE_floating_point) it turns out to be [rather difficult](http://gafferongames.com/networking-for-game-programmers/floating-point-determinism/) to get deterministic behavior across operation systems, compilers, and CPU architectures. Even if you do manage to get basic arithmetic working properly, there is also trancendental functions (`sin()`, `cos()`, etc.) to think about.

For example, the result of `sqrt(x)` may vary slightly between game clients. This means that the world state on each client will drift farther away from each other as time passes. Eventually, each player's view of the world would be completely different from the others. A game where each player sees an alternative reality sounds interesting, but that is usually not what you want.

<p class="photo">
  <img src="/assets/images/sad-cat.jpg"><br>
  Always so many problems
</p>

It [seems](http://gafferongames.com/networking-for-game-programmers/floating-point-determinism/) you *can* fix these discrepancies. At least to some extent. However, the solutions look cumbersome and hacky.

## Fixed point math to the rescue

Luckily, I learned that there is another way to do real number arithmetic using only integer CPU operations. This is great because integer operations are deterministic out of the box across compilers, OSs, CPUs. Determinisn *and* real number support - that's exactly what I was looking for! Yay!

<p class="photo">
  <img src="/assets/images/racoon-excellent.jpg"><br>
  Excellent!
</p>

The technique is called *fixed point math* and despite its limitations it's very cool. Historically, fixed point math was very popular in games and graphics applications because of its high performance. But since CPUs during the mid 90s got dedicated floating point units, it is less popular today.

Normally, we implicitly regard integers as a number of ones. For example, 5 represent 5 ones which can be expressed as `5*1=5`. Stick with me, this'll make sense in a second. In fixed point math, you instead regard integers as a number of some fraction. You can choose any fraction you like (well, almost).

For example, let's say we chose `1/2` as our fraction. The integer 5 would then represent 5 halves aka 2.5 instead of 5 ones: `5*(1/2)=2.5`.

In my library (creatively named *Fixie*) I chose the fraction 1/1024. That means the number 1 is represented by the integer 1024 and the number 0.5 is represented by the integer 512:

`1024 * (1/1024) = 1`

`512 * (1/1024) = 0.5`

If this makes your head spin a little you are not alone. But fortunately, we can hide the complexity in classes and functions. As an example of this, here's how Fixie uses C++'s custom operators to hide complexity:

{% highlight cpp %}
// a and b are internally converted to 1024 and 2048 respectively
Fixie::Num a = 1;
Fixie::Num b = 2;

// Fixie::Num's custom divide operator calculates the result and
// stores the result expressed in number of 1/1024s.
// Internally, c will be 512 because 512 * (1/1024) = 0.5.
Fixie::Num c = a/b;

// Prints the expected result of 0.5. Fixie's custom cast operator
// converts the 512 integer to a 0.5 float.
printf("%f\n", (float)c);
{% endhighlight %}

This will print the expected result even though no floating point math was used. And the same goes for addition, subtraction, and multiplication. At the time of this writing I have also implemented `sqrt()`, `floor()`, `sin()`, `cos()`, and `acos()`.

As an example of how `Num`'s custom operators work, here you can see how addition is implemented:

{% highlight cpp %}
class Num {
public:
  Num& operator+=(const Num &rhs) {
    raw += rhs.raw;
    return *this;
  }
};
{% endhighlight %}

`raw` is the internal base integer number, in my case the number of 1/1024s. `Num x = 2` would yield an instance where raw is set to 2048.

Writing the division operator was [challenging and interesting](stackoverflow.com/questions/2422712/c-rounding-integer-division-instead-of-truncating/29533500). Getting the rounding right with both positive and negative numbers was tricky. The code is not easy to understand, but I thought you might appreciate it anyway. Take a look:

{% highlight cpp %}
class Num {
public:
  Num& operator/=(const Num &rhs) {
    assert(rhs.raw != 0);
    const int32_t resultNegative = ((raw ^ rhs.raw) & 0x80000000) >> 31;
    const int32_t sign = resultNegative*-2+1;
    int64_t temp = static_cast<int64_t>(raw) << numPrecision;
    temp += rhs.raw/2*sign;
    raw = static_cast<int32_t>(temp / rhs.raw);
    return *this;
  }
}
{% endhighlight %}

The bitwise operations here are primarily to ensure "correct" rounding, e.g. `-2/3 = -1` (and not 0 which is the default for integer division).

I used the basic `Fixie::Num` datatype as a building block to implement vectors, matrices, and quaternions. I end up with a math library that is ultimately based on deterministic integer operations while still allowing real numbers. Maybe its just me but I think it's pretty neat.

## Usage and limitations

It is important to note that fixed point math has limitations. It generally supports fewer significant digits that floating point so you'll get lower precision and smaller numeric ranges. That's not a problem for my game though, it might be for yours.

Because of these drawbacks I will probably only use Fixie in parts of the game where I need determinisn, that is the "state simulation": physics, AI, etc. Rendering and animation is not required to be synchronized over the network, so here I can stick with good old floating point.

Full source of Fixie is available on GitHub [here](https://github.com/rasmusrn/fixie).

It has fun and educational to implement my own fixed point math library. Now that my game's backend is based on this library I should be able to synchronize thousands of entities across the network. It will be glorious!

If you want to know more about fixed point math I recommend these great articles:

1. [Fixed-Point Numbers and LUTs](http://www.coranac.com/tonc/text/fixed.htm)
2. [Fixed Point Arithmetic and Tricks](http://x86asm.net/articles/fixed-point-arithmetic-and-tricks/)
3. [Doing It Fast](http://gameprogrammer.com/4-fixed.html)

## Up next

I am currently working on two things for the game:

* AI: Deciding and coordinating behavior and high level actions.
* Backend/frontend architecture: Decoupling state simulation logic from client presentation logic.

So that is presumably what the next few posts will be about. If you want to be notified about new posts please follow me on Twitter or subscribe to the email newsletter below.

As always, I hope you enjoyed the read. Thanks for stopping by.
