---
title:  "Pathfinding"
layout: "post"
date:   2015-05-10 12:00:00
private: true
---
Please check out last week's [introduction post](/my-story-so-far.html) if you haven't already.

For the last couple of weeks I've been working on pathfinding. It is a tremendously exciting subject! However, before we dive in I would like to tell you something about how the controls work in my game.

In the game you command a small society of "block people". You don't control each individual directly as you do in most RTS games like Starcraft or Age of Empires. Instead, you give orders to your society as a whole.

<p class="photo">
  <img src="/assets/images/game-ss1.jpg" /><br>
  Admittedly, this doesn't look much like a society yet
</p>

Exactly who does what and when is up to the block people themselves. They try to carry out your high level commands in the most effective manner they can "think" of. This control scheme is also seen in [The Settlers](http://en.wikipedia.org/wiki/The_Settlers) and [Banished](http://www.shiningrocksoftware.com/game/). Great games. The idea is to relieve the commander (that's you!) from tedious micro-management and thereby giving more focus to the grand strategy.

Given this autonomous nature of the block dudes, I needed a pretty efficient pathfinding system. I had implemented pathfinding several times before, but I couldn't quite remember all the details. [This awesome article from Red Blob Games](http://www.redblobgames.com/pathfinding/a-star/introduction.html) got me up to speed. Thanks [Amit](https://twitter.com/redblobgames)!

<p class="photo">
  <img src="/assets/images/coffee-shop-work.jpg" style="width: 500px"><br>
  I like to work from coffee shops in the evening. Change of scenery is good for the mind.
</p>

## Data structures

I decided to use the popular [A* algorithm](http://en.wikipedia.org/wiki/A*_search_algorithm) with a grid map representation. The game world is in 3D, but as far as the pathfinding is concerned, a 2D grid will do just fine and it is way simpler to manage.

The A* algorithm does a lot of book-keeping. While it searches the map it is continously storing information about each potential path. This information is stored in various data containers. The overall performance of the algorithm is, to a large extent, determined by the design and implementation of these containers.

I considered using [C++'s Standard Template Library](http://en.wikipedia.org/wiki/Standard_Template_Library) (STL) for the containers. As you may know, a [lot](http://gamedev.stackexchange.com/questions/268/stl-for-games-yea-or-nay) has been [said](http://simonask.tumblr.com/post/59763277483/why-stl-isnt-great-for-game-development) about STL, especially in game development circles. Personally, I don't like it for two reasons:

* It's too general. They try to do everything and end up being really good at nothing.
* It's too hung up on object oriented paradigms for my taste. I am not big fan of constructors, destructors, RAII, etc.

So instead I decided to write my own containers. This was the first time I had to implement a *hash map* and a *priority queue*. It was very interesting to learn how the ubiquitous map structure works at a fundamental level. If you would like to know more, I recommend [this lesson](http://c.learncodethehardway.org/book/ex37.html) by Zed Shaw. I built the priority queue as a *[binary heap](http://en.wikipedia.org/wiki/Binary_heap)* inspired by [this](http://stackoverflow.com/questions/17009056/how-to-implement-ologn-decrease-key-operation-for-min-heap-based-priority-queu).

As an example, here's the interface for my priority queue:

{% highlight cpp %}
class AStarPriorityQueue {
public:
  void insert(MapFieldIndex field, Num priority);
  void update(MapFieldIndex field, Num priority);
  MapFieldIndex pop();
  bool isEmpty() const;
  void clear();
};
{% endhighlight %}

The map grid is made up of squares. I call them *fields*. Hence, `MapFieldIndex` is a number representing a particular field. When you insert or update a field in the queue, you also specify a *priority*. The method `pop()` returns the field index for the field with the lowest priority in the queue. The A* algorithm uses this behavior to choose a "reasonable" field to examine next rather than a random one. This makes the search much faster in most cases.

I implemented the containers in a [data-oriented](http://gamesfromwithin.com/data-oriented-design) way. That means no dynamic memory allocation, no pointers, and a sensible data layout tailored to my exact use-case. The downside of this is that my structures aren't reusable in other contexts but that's okay.

I later [learned](http://cglab.ca/~morin/misc/arraylayout/) that the *[B-tree](http://en.wikipedia.org/wiki/B-tree)* data structure may be more performant than a binary heap for this use case. B-trees store more nodes at each level. This increases the chance of memory cache re-use which can make a big difference on modern hardware. I will definitely consider this the next time I need to implement a priority queue.

## Unit testing

Test driven development can sometimes can be hard to apply to game development. Here however, this technique works beautifully! I wrote a unit test for each data structure and once they all passed I continued to write a unit test for the A* implementation itself. Here's an example from the A* unit test:

{% highlight cpp %}
void testSmallMap() {
  Map map;
  MapFieldType o = MapFieldType::Grass;
  MapFieldType x = MapFieldType::Rock;
  MapFieldType fields[] = {
    o, o, x, o,
    o, x, o, o, // just look at that quite little map
    o, o, o, o
  };
  map.reset(4, 3, fields); // setting up neighbours, edge costs, etc.

  MapFieldCoors origin = { 3, 0 }; // upper right
  MapFieldCoors destination = { 1, 0 };
  MapSearchResult result;
  aStar(map, origin, destination, result);

  assertTrue(result.success);
  MapFieldCoors waypoints[] = {
    { 3, 0 }, { 3, 1 }, { 2, 2 }, { 1, 2 },
    { 0, 2 }, { 0, 1 }, { 0, 0 }, { 1, 0 }
  };
  assertPath(result, 8, waypoints);
}
{% endhighlight %}

Just look at that! Now I can refactor to my heart's content and still be 99% sure that everything works. Unit testing is awesome. End of discussion.

And at long last for this post's grand finale. A video!

<p class="photo">
  <img src="/assets/images/pathfinding-demo.gif"><br>
  The weird back and forth behavior is due to a known bug.
</p>

## Integration

Integrating pathfinding into the game is also a very interesting subject. How should the block dudes transform a path into movement/rotation? How should changes in the terrain be managed? What if an obstacle appears after the path has been calculated? Etc. Maybe that should be the topic for the next post.

Well, that's it for now. I hope you enjoyed reading. If you did, please consider signing up for the email newsletter in the form below. I am still new at this devlogging thing and very curious to hear what you think. Feel free to drop me an email with any feedback you might have (address below). I would love to hear from you.
