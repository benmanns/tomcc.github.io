---
layout: culling-post
title: The Advanced Cave Culling Algorithm™ Part 2 - Traversing the graph
---

> *You can find the first part of the article [here](http://localhost:4000/2014/07/22/visibility.html)*.

Now that the visibility graph is done, it's time to start thinking how to use it to decide what we're going to show on screen, and this is where stuff starts to be harder to explain :)  
Probably there are many ways to use the information we have right now, but they all hinge on the same speed/accuracy trade off: if we were going for the best accuracy, the best method would be of course to **raycast** each corner of each chunk in the frustum.  
The visibility info we computed above still helps a lot with rays, as we can easily find out the faces they enter/exit a chunk from, and then use our canned visibility to continue or stop.  
However, this is kinda not fast- at max render distance we have thousands of chunks loaded in the area around the player (17000 is not uncommon on PC!) and each chunk needs 7 raycasts at most - even if we would get the maximum culling ratio, running 119.000 raycasts per frame is not going to make the game any faster and wouldn't be an optimization at all, so something smarter was needed.  

**Breadth-first searching**

Instead, what I settled on was a simple [breadth-first search](http://en.wikipedia.org/wiki/Breadth-first_search), with some smart hacks.  
If the chunks are stored in a contiguous 3D grid, the access time is O(1) and it's pretty fast in practice; plus, the BFS has a nice additional benefit: it visits the chunks front-to-back naturally, letting the GPU's Hidden Surface Removal powers shine.  

In fact, the three main time-sinks we had on the CPU side in MCPE 0.8 and PC were **frustum culling**, **depth sorting** and **rebuild scheduling**; their total accounted for around 14% of the frame time on both versions. Plus, the scheduling on PC was notoriously bad and unresponsive (holes in the world, anyone?).  
The BFS algorithm solves allowed of them in linear time, and on top of that we get the visibility culling we were looking for!

So now, here's roughly how it works, in a quite condensed way:
* set up a queue of steps, where each step contains the **chunk** we want to visit next and the **face** we came from
* find chunk the camera is inside of and push it on the queue as the first step
* for each chunk, until the queue is empty, 
* the front chunk in the queue is visited, queued for rendering or for rebuilding, and popped; 
* for each of its neighbors, we check if we need to walk there by applying some **filters** to them:  
    1. first, check if we are going back. We never want to go back because if a chunk is only reachable going backwards, it's not going to be visible. So, we only walk through faces opposite to the view vector, ```N·V < 0```. 
    2. this is the key filter! Now we have 3 chunks going around: **A**, the one we came from; **B**, the one we're inside, and **C**, the one we want to visit next. If we can *reach C from A through B* (reading into B's visibility graph), C passes this visibility test!
    3. check if C passes even more filters (more on that later).
    4. frustum cull C. Yes, frustum culling the most expensive operation here, so it's done as the last step.
    5. if all of the above have been passed, queue this neighbor for visiting: it'll become eventually visible!

Phew. That was a lot of text, but you can play around with this other javascript thing to see it in action:

MAGIC JAVASCRIPT THING 2

More filters!
----

As you may have noticed if I actually succeded in explaining the algorithm, it has a flaw that can cause unpredictable under-culling, and thus, framerate drops.  
Indeed, this problem is the technical reason that got us to cut Ravines from Pocket Edition, because they are always a worst case!  
What happens is that there is nothing that prevents the BFS to walk, say, 10 chunks straight and then 7 chunks down to the bottom of a ravine, rendering everything in the process.  
Visually, it's pretty obvious that the path doesn't approximate a raycast in the slightest because it has a huge bend in it, but the current implementation isn't smart enough to know... a ravine or a huge cave then, can cause *all caves* on the other side of it to be visible from certain angles!  

**Count the steps**  
To try and patch this problem, I've pulled in some info from what we know about the gameplay of Minecraft, coming up with heuristics about what should and should not be visible.  
In other words, with our current terrain generation, the "common cave" bends a lot, exists under the sea level, and is fully unlit unless a player visits it; and because *of course* correlation implies causation, we can penalize chunks that have these attributes during the walk, because *of course* they contain caves!  
The idea is to do this by adding a maximum **walkable steps** to the search, so that a branch passing through "bad" chunks will spend more steps and terminate earlier.  
I've settled on using only two heuristics, that still are fairly effective: going down *when under sea level* costs +1 step, and walking through a fully-dark chunk a whopping +3.  

This hackish thing, experimentally, contributes 5% to 15% of the cull ratio depending on the scene, and it manly smooths out huge caves that would pop into visibility if the search wasn't stopped earlier!  
The bad news is that this doesn't do much at all for surface ravines, because their shape allows sunlight to reach the bottom. Oh well.  
I've since solved this problem (in theory) with a way more effective and generic algorithm based on assigning an sub-frustum to each BFS step, but that will probably be for another article :)

Conclusions
----

It works! Here's a nice picture of the panorama on top after the culling is applied:

<img src="/images/cull_after.jpg" width="830" height="509">
> *Notice how much stuff disappeared!*


