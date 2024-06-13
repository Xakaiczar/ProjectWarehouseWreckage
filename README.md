# Project "Warehouse Wreckage"
_Developed with Unreal Engine 5_

> [!WARNING]
> **DISCLAIMER**: This project is based heavily on "Warehouse Wreckage", the first module of _Unreal Engine 5 C++ Developer: Learn C++ & Make Video Games_ by GameDevTV. Most of the code will have been written and designed by me, and it may differ significantly as I diverge from the lecturer to solve the problem myself. However, all work here is inspired by - and heavily derivative of - work that is not my own, therefore this project is not my own and cannot be distributed. This project is for assessment by prospective employers only.

## Contents
- [About the Project](#about-the-project)
- [Demonstration](#demonstration)
- [Development](#development)
  - [The Original Project](#the-original-project)
  - [My Contribution](#my-contribution)
    - [Falling Over](#falling-over)
    - [Falling Through](#falling-through)
    - [The Player](#the-player)
    - [GUI](#gui)
    - [Conclusion](#conclusion)
- [Summary](#summary)

# About the Project
"Warehouse Wreckage" is the name of the first module in the course _Unreal Engine 5 C++ Developer: Learn C++ & Make Video Games_ by GameDevTV ([Udemy](https://www.udemy.com/course/unrealcourse/) / [GameDevTV](https://www.gamedev.tv/courses/unreal-5-0-c-developer-learn-c-and-make-video-games)). The premise is simple: knock over every object in the room with a limited supply of ammunition. However, the course does not implement any successful end condition; the player can only lose. I added a win condition myself, properly telegraphed that information to the player, and - to challenge myself - I did it all using Blueprints.

# Demonstration
A video will be uploaded soon!

# Development
## The Original Project
The original project is very simple. Aimed at beginners who are learning the engine for the first time, it covered the basics well. The lectures briefly run down:
- Importing assets and using the provided assets
- The physics-based properties attached to an object in the world
- Using Unreal's Blueprints to reference and modify objects in the game

Once all that's out the way, we're set to work on the core gameplay.

First, I made a projectile with a simple sphere in a silver colour. This sphere was converted into a Blueprint Class and used as the projectile.

![Image PWW 1.1 - Blueprint Class for Projectile](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%201.1.png)

Before we can fire a projectile, one needs to be spawned into the world. Using a reference to the projectile blueprint made in the previous step, an instance of that class is created in the `BP_FirstPersonCharacter` blueprint and spawned in the world just ahead of the player's location using `SpawnActor` (100 units in the forward direction).

![Image PWW 1.2 - Blueprint function for SpawnProjectile](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%201.2.png)
>[!NOTE]
>_Pictured above is a function called `SpawnProjectile`. In the current build however, `SpawnProjectile` no longer calls `Launch`. Both of these functions - as well as `DecreaseAmmo` - are instead sequentially called in `FireProjectile`. I will update images and descriptions when I have time, but it works fundamentally the same, it was just a simple refactor._

The projectile is then fired using the `Launch` function. It is given an impulse, which is the force over a time period required to produce momentum, causing the projectile to fly away from the player. This impulse is determined by finding the forward vector of the projectile, then multiplying that by a value that - in hindsight - should've been a variable. I'll fix that later.

![Image PWW 1.3 - Blueprint function for Launch](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%201.3.png)

This set of nodes is then added to the `BP_FirstPersonCharacter`'s event graph, triggering when the left mouse button is clicked.

![Image PWW 1.4 - Blueprint function for lmb click](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%201.4.png)

As you can see, there are two other nodes here too: one that decreases the ammo when a projectile is fired (aptly named `DecreaseAmmo`), and one that tracks the ammo remaining. The latter stops projectiles from being fired when there are none left.

`DecreaseAmmo` is pretty straightforward too. It looks a bit ugly in BP:

![Image PWW 1.5 - Blueprint function for DecreaseAmmo](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%201.5.png)

But in code this would essentially amount to:
```c++
ammo -= 1;
```
> [!NOTE]
> _I have since changed this, adding a `max` node before the `set`. The `max` compares `Ammo` to `0` and returns the lowest, meaning the player can never have negative ammo._

In the `Main` level blueprint, the game is reloaded when the player is out of ammo (more on that graph later!). I also added in a manual reload for testing purposes.

And there we have it! A game about a person stood in a floating abandoned warehouse throwing metal bowling balls at whatever random garbage happens to be around them!

## My Contribution
Now, that doesn't really sounds like a _game_, does it? It's more of a sandbox than anything else, but there's no goal, no win state, and the fail state is completely silent and sudden.

I wanted my contribution to make this demo into an actual game. To do that, I'd need to fix the issues above.

### Falling Over
First of all, I wanted the game to actually be able to track which props had fallen and which were still upright, then return true if all of them had been knocked over; this would make for a good win condition. If I was using Unity, I would probably slap a box collider on the bottom of the prop and call it a day. But this time, since I'm in a new engine, I thought I'd try to solve it a new way.

What's happening when an object is falling over? Well, mathematically, it's rotating around a certain plane by a certain angle until it becomes unbalanced and gravity does the rest of the work. In this case, the X and Y axes tilt the prop, while the Z just kinda... _spins_ it...

Using this, we can actually determine whether a prop has fallen over or not by its angular tilt!

![Image PWW 2.1 - Angular Tilt Graph](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.1.png)
> (_a_ < n < 360 - _a_, where _a_ is the angular tilt)

The idea here is that a prop must rotate by an angle of _`a`_ before it starts falling with no hope of return. Inversely, this means that any prop with an `x` or `y` rotation of _`a`_ degrees or lower must still be stood up, to some degree (no pun intended...).

This is then mirrored on the other side, as an angle of _`-a`_ will cause it to fall the other way. Unreal doesn't seem to like negative angles very much (as far as I can tell), so I'm instead subtracting it from 360 degrees, which is fundamentally the same thing.

I decided on a default angular tilt value of 45 degrees for both axes:

![Image PWW 2.2 - Angular Tilt Graph Filled](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.2.png)

As you can see from the figure above and the image below, this number was not arbitrary; it falls exactly between perfectly upright and perfectly fallen:

![Image PWW 2.3 - Angular Tilt Barrel](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.3.png)

Maybe 45 degrees is a bit too large for some props, but it gets the job done; for now, that's all that matters. We can fine-tune it on a case-by-case basis later.

To determine if a prop has fallen, we need to get these boundaries. I wrote this function called `GetTiltBounds`:

![Image PWW 2.4 - Get Tilt Bounds](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.4.png)

This function takes in one parameter - `Angle`, which we'll get to later, but defaults to `360` - and returns two values: a `MinTilt` and a `MaxTilt`. While not very well named, these return values are the upper and lower bounds for the prop's rotation. In other words, the prop must be between `MinTilt` and `MaxTilt` to be considered "fallen".

The graph looks more complex than it is. In C++ it would look like this:
```C++
float a = ToppleTilt % Angle;
float b = (360 - ToppleTilt) % Angle;

float MinTilt = min(a, b);
float MaxTilt = max(a, b);
```
> _Where `ToppleTilt` is the variable angular tilt the prop would need to fall to be considered "fallen", mentioned earlier and defaulted to `45`._

It starts by getting the remainder of the `ToppleTilt` divided by the `Angle`, on the off chance the `ToppleTilt` is impossible due to over-rotation. In this case, if `ToppleTilt = 405`, it would still return `45`. It then also does the same for `-ToppleTilt`, as mentioned before.

The smallest of the two values is passed to `MinTilt`, and the largest is passed to `MaxTilt`.

These values get passed into a much more convoluted function:

![Image PWW 2.5 - Has Tilted Too Far](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.5.png)

The function above - `HasTiltedTooFar` - calls `GetTiltAngle` and passes the result to `GetTiltBounds` (more on that later!). The `MinTilt` and `MaxTilt` are then compared with the rotation of the prop; if the prop's rotation on the `x` _or_ `y` axis falls outside the safe range and into the fallen range of `MinTilt < n < MaxTilt`, then it will be flagged as "fallen" in the `Main` blueprint.

To make it readable, I will once again condense down a complex-looking graph into code. In C++ it would look like this:
```C++
float x = abs(Rotation.X);
float y = abs(Rotation.Y);

bool HasFallenOnX = MinTilt < x && x < MaxTilt;
bool HasFallenOnY = MinTilt < y && y < MaxTilt;

bool HasFallen = HasFallenOnX || HasFallenOnY;
```

Now, I'm sure you're wondering: "What about `InvertTilt` and the XOR? What about that mysterious `Angle` you keep referring to?"

In an ideal world, the method above would've worked for every prop in the game. But sadly, not all props work the same.

For example, the pallet prop was imported lying flat (as one would expect a pallet to be in a real-world context). But this didn't quite work in this context, as this meant that its default state was along the floor, while standing it up would actually trigger the prop as fallen.

It took me an embarrassingly long time to figure out, but this is effectively what is happening and how I visualised it:

![Image PWW 2.6 - Barrel Pallet](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.6.png)

Now relate that to the graph from earlier (here it is again, for your reference):

![Image PWW 2.7 - Barrel Pallet with Graph](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.7.png)

Frustratingly, an object lying parallel with the floor appears to be the _complete opposite_ of their upright, perpendicular counterparts: when they are within the safe tilt bounds, they are actually _closer_ to the floor than when they should be "falling".

My first instinct was just to rotate the prop visual in the editor; this is a simple trick I use in Unity all the time. But being new to the engine, I couldn't figure for the life of me how to do it, and I had no idea how to reimport it without the original fbx asset...

So, once again, I had to do it the _hard way_...

I went through a few mathematical options, trying to adjust the bounds by the angular offset and the like, when I stumbled upon a startlingly simple solution.

If you look again at the diagram above, you'll probably notice the same thing I did: if you rotated that whole barrel-pallet abomination 90 degrees in either direction, it would not only make the barrel fall, but the pallet stand. In other words, the fall arc of a parallel object is actually the _inverse_ of a perpendicular object!

So for _some_ objects, I needed to ensure that the unsafe bounds (`MinTilt < n < MaxTilt`) became the _safe_ bounds, and vice versa.

Again, I'm sure this can be done mathematically, but logically works too!

I first tried to NOT the original output under certain conditions, but after drawing up a little truth table, I realised what I wanted was a XOR gate. In plain English that would equate to:
> - If the object is not outside the bounds and is not parallel to the floor by default, it's not fallen on the floor (false) - this describes a perpendicular object stood upright
> - If the object is not outside the bounds and is parallel to the floor by default, it's fallen on the floor (true) - this describes a parallel object lying flat
> - If the object is outside the bounds and is not parallel to the floor by default, it's fallen on the floor (true) - this describes a perpendicular object lying flat
> - If the object is outside the bounds and is parallel to the floor by default, it's not fallen on the floor (false) - this describes a parallel object stood upright

And, lo and behold, it works! _Kinda_...

Actually, it highlighted a _different_ issue I had.

If the pallet fell on its bottom side it correctly flagged as having fallen, but if it fell on its _top_ side it was still considered upright.

This was actually true of _all_ props, I just never successfully made a barrel flip onto its top before, for obvious skill-issue-based reasons. Also, some props _needed_ to be upside-down, in case their sides were too narrow to support them (e.g. the hose).

This is where that fabled `Angle` comes in to save the day!

If you're a smarter person than me, you may have noticed that all my calculations were based off a 360 degree arc around the object, while all my diagrams were based off a 180 degree arc. In other words, I was holding two different ideas of how this would work simultaneously, then somehow got confused when I realised they contradicted each other...

Clearly, some objects were basically the same upside-down, so they needed to have _two_ safe zones: one around 0 degrees, and one around 180 degrees. I made `GetTiltAngle` to represent this:

![Image PWW 2.8 - Gelt Tilt Angle](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.8.png)

This then gets fed into `GetTiltBounds`, where it... _just works_...

I was actually surprised when it worked, and I'm _still_ surprised it works, because I'm honestly not quite sure _how_ or _why_ it works! My best guess: the modulus I implemented in `GetTiltBounds` - the one that stops the angles from overflowing outside 360 degrees - might be dividing the area up for me.

Weirdly though, it only works as intended at 360 degrees and 180 degrees. If you set it to 120 degrees (i.e. the circle is split into 3), it still only gives 2 areas instead of 3, as does an angle of 90 degrees (a circle split into 4).

Weirder still, while there are only 2 areas, they each start and end _exactly_ where you'd expect; the bounds around 0 are completely correct, as are the start bound around the second division and the end bound around the final division. For example, if we assume a `ToppleTilt` of `10` and an angle of `120`, the bounds around 0 will be correct (`-10 < n < 10`), but the second division (`110 < n < 130`) and third division (`230 < n < 250`) are replaced by a single bound (`110 < n < 250`).

So those values are still _used_ properly, it's just that all but the first fuse into one _giga_-bound!

Maybe I'm just too ill to understand this at the moment. I think I'll need to give it a look-over another time...

### Falling Through
But what if, hypothetically, something went flying out the window...?

IMAGE OF THAT

Or, I don't know, through the floor...?

IMAGE OF THAT TOO

If that were to happen, they could end up flying off into the sunset, spinning forever. This could result in an unwinnable game state, as the flying objects constantly rotate between fallen and upright. No bueno.

I set some upper and lower bounds in the main blueprint to combat this.

IMG OF BP

Then, once a prop falls out of range of those bounds, it counts as fallen.

IMG OF BP

They do despawn at some point...

IMG OF NUMBER GOING DOWN?

But it doesn't affect the completion rate, so that's good.

### The Player
I had to extract the projectile method from the main blueprint into the player in order to fetch the variables for the UI. However, to do that, I'd need to have a player in the first place...

I went the easy route: I decided to import the First Person Example Level assets through the content window.

I moved the SpawnProjectile code over, refactored it, and exposed the ammo for the UI:

IMG OF THAT

Last, but by no means least, I prevented players from doing a Tommen Baratheon...

GIF IF I CAN FIND ONE OR JUST GAME FOOTAGE

I just kept the players within the same box as the props by resetting the level after they leave it.

IMG OF BP

In hindsight, I probably should've let them continue from where they left off. Maybe in the next update...

### GUI
The last thing it really needed to feel like a _game_ was a HUD.

IMAGE OF HUD

I not only included an ammo count so the player could manage their ammo carefully and avoid defeat, but also a prop count so they would know if they were close to winning. These numbers are variable - as you'd expect - and connect directly to the player.

IMAGE OF BP

Then, for the finishing touches, I added a win screen:

WIN SCREEN IMG

And a game over screen:

GO SCREEN IMG

### Conclusion
With the addition of a proper win condition and a HUD that clearly communicates the state of the game with the player, it actually feels like a game. It still has a lot of room for improvement though! Maybe I'll revisit this project when I'm a little better trained.

# Summary
Thank you for reading! If you want to see more of my work, please check out my [portfolio](https://github.com/Xakaiczar/Portfolio), and if you have any questions or opportunities, please don't hesitate to drop me an [email](xaqatkins@virginmedia.com) or message me via [LinkedIn](https://www.linkedin.com/in/xaqatkins/)!
