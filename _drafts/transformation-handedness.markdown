---
title:  "Transformation handedness?"
layout: "post"
date:   2014-11-11 21:17:25
---
For the last couple of days I've wrestled with the question: Do transformations have <a href="http://en.wikipedia.org/wiki/Orientation_(vector_space)">handedness</a>?
Or more specifically: Why do quaternion rotations rotate and object in a specific direction?
What determines that positive angles turns out as clockwise rotations and not counter-clockwise?

Of course, if your rotations go the "wrong" way around, you can just negate your angle, and get on with your life. But I wanted to understand <em>why</em> and <em>how</em> it works.

I discovered quite a few people on the web had asked the same kind of questions. Most were answered with some variation of "Your question does not make sense". I thought about it and suddenly everything made sense (oh, that feeling). Let me tell you about it!

When I try to gain deeper understanding of some concept, it often helps to reduce the problem to its basics. Hopefully, these basics are easier to grasp. When you've established a solid understanding of the basic version of the problem, I then work myself back to the original complex version of the problem.
