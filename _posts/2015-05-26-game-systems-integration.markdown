---
title:  "Game systems integration"
layout: "post"
date:   2015-05-26 14:30:00
private: true
---
In the [last post](/pathfinding.html) I talked about pathfinding. In this post I want to talk about how I integrated pathfinding and other systems into the rest of the game.

I have always found it difficult to integrate game systems such as rendering, physics, AI, networking, etc. It is fairly easy to make things work. However, it can be very tricky to make systems work together in a way that is elegant, decoupled, and flexible. I think I have more than 25 scrapped designs/architectures implemented in various languages lying around. However, over the last couple of months I have finally managed to come up with a solution I like. It feels great.

<p class="photo">
  <img src="/assets/images/startrek-yes-meme.jpg"><br>
  Yesssss
</p>

## My implementation

Put shortly my solution is a [*data oriented*](http://gamesfromwithin.com/data-oriented-design) implementation of the [*entity/component system*](http://en.wikipedia.org/wiki/Entity_component_system) (ECS) architectural pattern.

The ultra short introduction to ECS is that world objects are modelled as *entities* and behavior is modelled as *components*. Components are simple data structures that are updated by certain *systems*. For example, if you attach a `GravityComponent` to a game entity the `GravitySystem` will apply gravity to that entity.

If you want a certain behavior you must create the corresponding component. For example, if you want to add air drag in my game, you would write:

{% highlight cpp %}
DragHandle dragHandle = DragSystem::create(rigidBodyHandle);
{% endhighlight %}

`DragSystem::update()` will then process each drag component. It will use `rigidBodyHandle` to apply drag force to the corresponding rigid body. You stop the behavior by calling `DragSystem::destroy(dragHandle)`.

In my ECS implementation an entity is nothing more than a unique integer: `EntityHandle`. There is no `Entity` class. To apply behavior to an entity you create the associated component as described above and link it to an entity like this:

{% highlight cpp %}
EntityHandle entity = EntityManager::create();

RigidBodyHandle rigidBody = Physics::createRigidBody();
ComponentManager::link(entity, ComponentType::RigidBody, rigidBody);

DragHandle drag = DragSystem::create(rigidBody);
ComponentManager::link(entity, ComponentType::Drag, drag);
{% endhighlight %}

If you are wondering why I'm using *handles* (plain old integers) instead of pointers, check out the awesome [Molecular Musings blog](https://molecularmusings.wordpress.com/2013/05/17/adventures-in-data-oriented-design-part-3b-internal-references/).

Notice how the `Physics` and the `Drag` systems don't know anything about entities or component managers. Some higher level systems, such as AI, might *want* to know about entites but it isn't required. The only requirement is that systems allow for components to be created and destroyed.

<p class="photo">
  <img src="/assets/images/notebook.jpg" style="width: 400px"><br>
  My appropriately named notebook
</p>

The above code might seem cumbersome but I think the loose coupling more than justifies the added verbosity. To make the code easier to work with I made a convenience layer called `Database`. Using `Database` the example presented above looks like this:

{% highlight cpp %}
EntityHandle entity = Database::create();
Database::createRigidBody(entity);
Database::createDrag(entity);
{% endhighlight %}

With this architecture I can make an entity just by creating and linking its constituent components. Technically, there is no such thing as a coherent entity. The image of an entity that appears on screen is just a manifestation of the structured chaos of components being created, destroyed, and updated by their corresponding systems. Isn't that beautiful? Forgive me for this far-fetched association but I think that it is a bit like how nature works. In the real world an apple doesn't fall because it somehow knows that apples ought to fall. It falls because it has mass (a component) and mass is affected by gravity (a system). Maybe that is part of the reason why this architecture works so well in games.

<p class="photo">
  <img src="/assets/images/no-spoon.jpg" style="width: 650px"><br>
  There is no spoon
</p>

## Example

I have covered the basic idea about the integration design. We can now look at some specific systems and how they are coupled. Below is a simplified high level overview of some of the game's systems. Notice the directions of the arrows. An arrow from A to B means A has a one-way dependency on B.

<p class="photo">
  <svg width="950" height="200" style="background: rgb(51, 84, 136)">
    <defs>
      <marker id="arrow-marker" markerWidth="8" markerHeight="8" refx="1" refy="2" orient="auto">
        <path d="M0,0 L0,4 L4,2" fill="white" />
      </marker>
    </defs>
    <text fill="white" font-size="20" font-family="sans-serif" font-style="normal">
      <tspan x="20" y="65">AI</tspan>
      <tspan dx="50" dy="0">Pathfinding</tspan>
      <tspan dx="50" dy="0">Steering</tspan>
      <tspan dx="50" dy="0">Physics</tspan>
      <tspan dx="50" dy="0">Interpolation</tspan>

      <tspan x="392" y="144">Direction</tspan>
      <tspan dx="50" dy="0">Animation</tspan>

      <tspan x="670" y="102">RenderFeed</tspan>
      <tspan dx="50" dy="0">Rendering</tspan>
    </text>

    <g stroke="white" stroke-width="5" fill="none" marker-end="url(#arrow-marker)">
      <g transform="translate(50,58)">
        <path d="M0,0 L23,0" />
        <path d="M155,0 L178,0" />
        <path d="M285,0 L308,0" />
        <path d="M446,0 L423,0" />
      </g>

      <path d="M477,138 L500,138" />

      <path d="M660,90 L630,68" />
      <path d="M660,100 L630,128" />

      <path d="M794,95 L817,95" />

      <path d="M430,120 L420,90" />
    </g>

    Sorry, your browser does not support inline SVG.
  </svg><br>
  Systems overview
</p>

Most systems have only a single dependency. A few systems have no dependencies at all. I like that because it is usually a sign of good decoupling. It also makes it easier to think in terms of input/output which is a central part of [data oriented programming](http://gamesfromwithin.com/data-oriented-design) (which is awesome).

Here's a summary of each system's responsibilities (with respect to the chart above):

* AI: High level decision making. Uses the pathfinding system to create and configure pathfinding components.
* Pathfinding: Uses A* to recalculate paths with certain intervals. Checks waypoints and uses steering system to adjust current direction on steering components.
* Steering: Converts each component's direction into force and torque that is applied to the corresponding rigid body component in the physics system.
* Physics: Updates position, velocity, and spin based on applied force and torque. Also performs collision detection/resolution. Updated at a [fixed interval](http://gafferongames.com/game-physics/fix-your-timestep/).
* Interpolation: This system updates at a [smaller interval](http://gafferongames.com/game-physics/fix-your-timestep/) and generates interpolated positions, velocities, and orientations based on recent values retrieved from the physics system. This enables smooth rendering even though physics only run at ~33fps.
* Direction: Uses velocity from the physics engine to determine which animation the animation system should play.
* Animation: Advances animations by transforming a 4x4 matrix for each bone in each animation component. Also responsible for blending between animations.
* RenderFeed: Feeds the rendering system with appropriate transformation and pose data using input from the interpolation system and the animation system.
* Rendering: Draws everything on screen using OpenGL.

As a more detailed example let's examine how the steering system applies force to a rigid body. I have added a lot of comments to make the code understandable.

{% highlight cpp %}
namespace SteeringSystem {
  void update() {
    // getCount() returns number of steering components
    for(uint16_t i=0; i<getCount(); ++i) {
      // A dynamic driver is a component that drives a rigid body
      // by applying force and/or torque.
      DynamicDriver driver = physicsSystem.getDynamicDriver(dynamicDriverHandles[i]);

      // Body has pointers to the body's position, velocity, and spin.
      Body body = physicsSystem.getBody(driver.bodyHandle);

      // Calculates the difference between where the body is and
      // where it should go.
      Vector3 positionDifference = targets[i] - (*body.position);

      // If the difference is too small we won't bother applying forces.
      // Also, without this check we might get a division by zero error
      // from the call to normalize().
      if(positionDifference.calcSquaredLength() > tolerance) {
        Vector3 direction = Vector3::normalize(positionDifference);
        (*driver.force) += direction;
      }

      // Do the equivalent for orientation/torque
      // [...]
    }
  }
}
{% endhighlight %}

## Up next

All things considered, I am very happy with this design. It feels like a strong foundation for the upcoming systems and features. Speaking of which, my next major task will be to implement AI. I think I'll go for the [behavior tree](http://www.gamasutra.com/blogs/ChrisSimpson/20140717/221339/Behavior_trees_for_AI_How_they_work.php) modelling technique although that [will not be trivial](https://web.archive.org/web/20140217141804/http://www.altdevblogaday.com/2011/04/24/data-oriented-streams-spring-behavior-trees/) to implement in a data oriented fashion. We'll see. Thanks for stopping by. I hope you enjoyed reading.

<p class="photo">
  <img src="/assets/images/game-ss1.jpg" style="width: 500px"><br>
  Just imagine these little guys cutting down trees. I can't wait!
</p>
