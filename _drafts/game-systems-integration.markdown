---
title:  "Game systems integration"
layout: "post"
date:   2015-05-25 10:10:00
---
When I started writing my own game engines I found a fair amount of literate about various game systems such as rendering, physics, AI, networking etc. But very little was written about how to tie these things together. With this post I humbly hope to improve this situation just tiny notch.

I have struggled with systems integration in my games for years. For example, how should animation, physics, and rendering communicate? Making systems work together isn't the hard part. Making systems work together in an elegant decoupled fashion, however, has turned out to be quite tricky.

If you are a game developer you have probably heard about [*entity/component systems*](http://en.wikipedia.org/wiki/Entity_component_system) or ECS for short. It is an architectural pattern for organising and defining entities, behaviors, and systems in a game world.

While I have always liked the design philosophy behind ECS, I have always found it difficult to get the implementation right. It is easy to get working but difficult to make elegant. Especially in strongly typed languages like C++ which I'm using.

However, for my current game I wrote yet another implementation and for the first time, it feels like I have finally nailed it!

<p class="photo">
  <img src="/assets/images/nailed-it-moon.jpg"><br>
  Flawless
</p>

Okay, my solution of course isn't perfect. But I am very happy that I finally have an implementation that I like. Let me tell you about it. Gather round, kids!

Before I start, be warned! This is just my own personal implementation/interpretation of ECS so it is probably different from what you'll find elsewhere.

## Components, not objects

A classic approach in game code is to do something like this:

{% highlight cpp %}
void update(double timeDelta) {
  for(entity : entities) {
    entity->update(timeDelta);
  }
}
{% endhighlight %}

The idea is that an entity *knows* what it is and how it should behave so it is able to update itself. In my experience such designs tend to fail unless you're making very simple games. Two predominant problems that often arise are the dreaded [diamond problem](http://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem) and unpredictable execution order.

With ECS you try to think of everything in the world as *components*.

* Components are simple data structures with no behavior.
* Components are updated by corresponding *systems*.
* Entities are nothing more than a list of components.

For example, an entity has a `GravityComponent` and all gravity components are updated by `GravitySystem`.

The difference between this and the classic approach is subtle but important. Instead of entities *knowing what they are* and *how they behave*, entities only know which components they are comprised of.

It might help to think of it like this: In the real world an apple doesn't fall because it somehow knows it should. It falls because its atoms (a component) are affected by gravity (a system).

<!--
This kind of thinking is a serious departure from traditional object oriented paradigmes where you often "model the world with objects". And for me at least, that has turned out to be a great thing! If you like object oriented programming (OOP) I would not expect you to agree with me on that point. A couple of years ago I wouldn't have agreed myself. For me personally it was a great eye-opener when I learned to question the dogmas of OOP. If curious for more about this go [here](http://gamesfromwithin.com/data-oriented-design) or [here](http://gamesfromwithin.com/the-always-evolving-coding-style) (yes, I like Noel's writings very much).
-->

<p class="photo">
  <img src="/assets/images/notebook.jpg" style="width: 500px"><br>
  My appropriately named notebook
</p>

## Systems

Each component type corresponds to a certain system. For example, a gravity components would be created, destroyed, and updated through the gravity system. In my implementation, a typical system interface looks like this:

{% highlight cpp %}
typedef uint16_t GravityHandle;

namespace GravitySystem {
  GravityHandle create();
  void destroy(GravityHandle handle);
  void update(double timeDelta);
  float getSomeValue(GravityHandle handle);
}
{% endhighlight %}

Most systems in my game are just namespaces with good old simple functions. I only convert a system to a class if I need several instances of it, and most often I don't.

Also note how the `GravityHandle` is just an integer, not a pointer. Integers have several advantages over pointers. Here's a few important ones:

* The system can reorder or relocate its components internally without corrupting the handle. This is very important for [data oriented design](http://gamesfromwithin.com/data-oriented-design).
* It makes ownership, allocation, and deallocation explicit and clear. With a pointer it would not immediately be clear how each component should be freed.

Some systems use the handle as an index into its components. Other convert each handle to a component index through a lookup table. The important thing is that this design enables each system to decide what it think is best for its specific task. For more about handles check out the awesome [Molecular Musings blog](https://molecularmusings.wordpress.com/2013/05/17/adventures-in-data-oriented-design-part-3b-internal-references/).

The `update()` method will usually update all components associated with its system. This has a few advantages over the classic `entity->update()` approach mentioned above:

* Explicit execution order: You can freely choose the update order of each component type.
* Performance: Due to [CPU caches](http://en.wikipedia.org/wiki/CPU_cache) computers are much faster at doing the same thing over and over.

Note the above are just system guidelines. Some systems have several different `update()` methods and some manages several types of components.

## Entities as integers, not objects

Game entities are typically represented by objects. I used to do it like that too. But at some point it dawned on me: If an entity's behavior 100% determined by the components of which is comprised, why then have an `Entity` class at all?

In classic game engines, entities are object instances. In my design they represented by an 16bit integer.



Instead of representing an entity as an object instance, I use a plain old 16 bit integer. Creating a

Entites bind stuff together. It is just a number! Theoretically they are not necessary - they are just for bookkeeping.

## Example

Drawing


Bonus thoughts: no templates!


Curious? Checkout the book [Game Engine Architecture](http://www.gameenginebook.com/).
