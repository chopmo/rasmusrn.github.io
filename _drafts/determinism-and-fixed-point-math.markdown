---
title:  "Determinism and fixed point math"
layout: "post"
date:   2015-05-10 10:10:00
---
In the game I'm currently working on you control one character a bit like the Diablo games. Last week I realized the game would be so much better if you'd control a group of characters instead of just one. The more I thought about it, the more I knew I had to give it a try.

For technical reasons, this meant I'd have to use the socalled *lockstep* network synchronization technique. Most RTS games uses it because it lets you network hundreds, if not thousands of entities. It's awesome, actually. The basic idea is that each client simulates the entire game so the server doesn't have to send state updates.

But it comes at price: your game state simulation must be 100% deterministic down to the very last bit. Even the tiniest differences would accumulate over time and cause problems. It's not as easy as it may sound.

Most simulations rely on floating point math. And even though floating point math is [standardized](http://en.wikipedia.org/wiki/IEEE_floating_point) it turns out to be extremely difficult to get deterministic behavior across operation systems, compilers, and CPU architectures. And even if you do manage to get basic arithmetic working properly, there's also trancendental functions (`sin()`, `cos()`, etc.).

I deemed it a battle not worth fighting. Instead, I learned that there's another way to do real number arithmetic using only integer CPU operations. This is great because integer operations are deterministic out of the box across compilers, OSs, CPUs. Determinisn and real number support - that's exactly what I was looking for! Yay!

The technique is called fixed point math and despite its limitations it's very cool. Back in the day before CPUs got dedicated floating point units, it was used a lot in games and graphics applications.

Normally, we implicitly regard integers as a number of ones. For example, 5 represent 5 ones, 5*1=5. Stick with me, this'll make sense in a second. In fixed point math, you instead regard integers as a number of some fraction. You can choose any fraction you like (well almost).

For example, let's say we chose 1/2 aka a half as our fraction. The number 5 would then represent 5 halves aka 2.5 instead of 5 ones, 5*(1/2)=2.5.

In my library (creatively named `Fixie`) I chose the fraction 1/1024. That means 1024 is 1 and 512 is 0.5 because `1024*(1/1024)=1` and `512*(1/1024)=0.5`. If this makes your head spin a little, you're not alone. But fortunately, we can encapsulate in classes and functions. For example using C++'s custom operator this is how it looks:

    Fixie::Num a = 1;
    Fixie::Num b = 2;
    Fixie::Num c = a/b;
    printf("%f\n", (float)c);

This will actually print the correct result even though no floating point math was used. And the same goes for addition, subtraction, and multiplication. As the time of this writing I even have `sqrt()`, `floor()`, `sin()`, `cos()`, and `acos()`. I used all these basic primitives to also add vectors, matrices, and quaternion. All based on deterministic integer operations while allowing real numbers. Maybe its just me but I think it's pretty neat.

As an example of how this `Num` class works, here you can see how addition is implemented:

    class Num {
    public:
      Num& operator+=(const Num &rhs) {
        raw += rhs.raw;
        return *this;
      }
    };

`raw` is the base number, in my case the number of 1/1024ths. Division is a bit more involved but not too complex and still fairly performant:

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

If you want to know more about fixed point math I recommend these great articles:

1. [Fixed-Point Numbers and LUTs](http://www.coranac.com/tonc/text/fixed.htm)
2. [Fixed Point Arithmetic and Tricks](http://x86asm.net/articles/fixed-point-arithmetic-and-tricks/)
3. [Doing It Fast](http://gameprogrammer.com/4-fixed.html)
