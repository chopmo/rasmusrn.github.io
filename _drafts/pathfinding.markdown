---
title:  "Pathfinding"
layout: "post"
date:   2015-05-10 10:10:00
---
In my game you control a small society made up of a bunch of block people. But you do not control them directly as in most RTS games like Starcraft or Age of Empires. Instead, you tell your society as a whole to complete certain tasks.

<p class="photo">
  <img src="/assets/images/flowstone-ss.jpg" /><br>
  Admittedly, this doesn't look much like a "society" yet
</p>

Exactly who does what and when is up to the block people themselves. They always try to carry out your commands in the most effective manner possible. This control scheme is also seen in [The Settlers](http://en.wikipedia.org/wiki/The_Settlers) and [Banished](http://www.shiningrocksoftware.com/game/). Awesome games. The idea is to relieve the commander (that's you!) from tedious micro-management to allow for more focus on the grander strategy.

So given this autonomous nature of the block dudes, I needed a pretty efficient pathfinding system. I had implemented pathfinding several times before, but I couldn't quite remember all the details. [This awesome article from Red Blob Games](http://www.redblobgames.com/pathfinding/a-star/introduction.html) got me up to speed. Thanks [Amit](https://twitter.com/redblobgames)!

<p class="photo">
  <img src="/assets/images/coffee-shop-work.jpg" style="width: 500px"><br>
  I like to work from coffee shops in the evening. Change of scenery is good for the mind.
</p>

I decided to use the popular [A* algorithm](http://en.wikipedia.org/wiki/A*_search_algorithm) with a grid map representation. The game world is 3D, but as far as the pathfinding is concerned, a 2D grid will do just fine and it's way simpler to manage.

A* does a lot of book-keeping. While searching the map it is continously storing information about each potential path. This information is stored in various specific data containers. The overall performance of the algorithm is to a large extend determined by the design and implementation of these containers.

This was the first time I had to actually implement a hash map and a priority queue. It was very interesting to learn how a map/hash/dictionary/table actually works. If you like to know more, I recommend [this lesson](http://c.learncodethehardway.org/book/ex37.html) by Zed Shaw. The priority queue I built as a binary heap inspired by [this](http://stackoverflow.com/questions/17009056/how-to-implement-ologn-decrease-key-operation-for-min-heap-based-priority-queu).

I managed to implement the data structures in a [data-oriented](http://gamesfromwithin.com/data-oriented-design) way. No dynamic memory allocation, no pointers, and tailored to my exact use-case. On the downside, this means my structures aren't reuseable in other contexts but that's OK.

Sometimes test driven development can be hard to apply to game development. But here, it shined beautifully! Each data structure got its own unit test and when all passed I continued to write a unit test for the A* implementation itself. Here's an example from the A* unit test:

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

  MapFieldCoors origin = { 3, 0 };
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

I wish I could show you an amusing video of the block people following paths in the world, but unfortunately I haven't yet integrated the pathfinding system and the steering system. Hopefully, I'll be able to post a video soon so stay tuned :-)
