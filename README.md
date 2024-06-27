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
    - [Prop Vision](#prop-vision)
    - [The Player](#the-player)
    - [GUI](#gui)
    - [Main](#the-main-level)
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

![Image PWW 2.01 - Angular Tilt Graph](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.01.png)
> (_a_ < n < 360 - _a_, where _a_ is the angular tilt)

The idea here is that a prop must rotate by an angle of _`a`_ before it starts falling with no hope of return. Inversely, this means that any prop with an `x` or `y` rotation of _`a`_ degrees or lower must still be stood up, to some degree (no pun intended...).

This is then mirrored on the other side, as an angle of _`-a`_ will cause it to fall the other way. Unreal doesn't seem to like negative angles very much (as far as I can tell), so I'm instead subtracting it from 360 degrees, which is fundamentally the same thing.

I decided on a default angular tilt value of 45 degrees for both axes:

![Image PWW 2.02 - Angular Tilt Graph Filled](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.02.png)

As you can see from the figure above and the image below, this number was not arbitrary; it falls exactly between perfectly upright and perfectly fallen:

![Image PWW 2.03 - Angular Tilt Barrel](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.03.png)

Maybe 45 degrees is a bit too large for some props, but it gets the job done; for now, that's all that matters. We can fine-tune it on a case-by-case basis later.

To determine if a prop has fallen, we need to get these boundaries. I wrote this function called `GetTiltBounds`:

![Image PWW 2.04 - Get Tilt Bounds](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.04.png)

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

![Image PWW 2.05 - Has Tilted Too Far](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.05.png)

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

![Image PWW 2.06 - Barrel Pallet](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.06.png)

Now relate that to the graph from earlier (here it is again, for your reference):

![Image PWW 2.07 - Barrel Pallet with Graph](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.07.png)

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

![Image PWW 2.08 - Gelt Tilt Angle](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.08.png)

This then gets fed into `GetTiltBounds`, where it... _just works_...

I was actually surprised when it worked, and I'm _still_ surprised it works, because I'm honestly not quite sure _how_ or _why_ it works! My best guess: the modulus I implemented in `GetTiltBounds` - the one that stops the angles from overflowing outside 360 degrees - might be dividing the area up for me.

Weirdly though, it only works as intended at 360 degrees and 180 degrees. If you set it to 120 degrees (i.e. the circle is split into 3), it still only gives 2 areas instead of 3, as does an angle of 90 degrees (a circle split into 4).

Weirder still, while there are only 2 areas, they each start and end _exactly_ where you'd expect; the bounds around 0 are completely correct, as are the start bound around the second division and the end bound around the final division. For example, if we assume a `ToppleTilt` of `10` and an angle of `120`, the bounds around 0 will be correct (`-10 < n < 10`), but the second division (`110 < n < 130`) and third division (`230 < n < 250`) are replaced by a single bound (`110 < n < 250`).

So those values are still _used_ properly, it's just that all but the first fuse into one _giga_-bound!

Maybe I'm just too ill to understand this at the moment. I think I'll need to give it a look-over another time...

### Falling Through
But what if, hypothetically, something went flying out the window? Or, I don't know, _through the floor_...?

If that were to happen, they could end up flying off into the sunset, spinning forever. This could result in an unwinnable game state, as the flying objects constantly rotate between fallen and upright. No bueno.

I set some upper and lower bounds in the main blueprint to combat this, passing them into `HasFallenOutOfBounds` in `BP_Prop`:

![Image PWW 2.09 - Has Fallen Out Of Bounds](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.09.png)

As always, the mess of cables makes this code harder to follow than it is. In short, if the `x`, `y`, or `z` components of the prop's `Transform` location are outside their respective high or low bounds (i.e. higher than the high bound on that axis, or lower than the low bound on that axis), then the function returns `true`.

This was added to `HasFallen`, a function that combines both `HasFallenOutOfBounds` and `HasTiltedTooFar`.

![Image PWW 2.10 - Has Fallen](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.10.png)

Now, once a prop falls out of range of those bounds, it counts as fallen.

The issue now is, once they fall out of range, they _keep_ falling.

While they do despawn eventually, it felt a bit buggy. It didn't stop the player from completing the game, but it didn't feel like a good user experience; if the player was unaware of the bug, how were they to know which prop had suddenly despawned?

To fix this, I simply teleport any object that falls out of range to a point that's _just_ out of range:

![Image PWW 2.11 - Keep Props In Bounds](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.11.png)

This function - `KeepPropsInBounds` - iterates through all the props and sets the location of any out-of-bounds props to `100` units outside the low bounds in all 3 directions. That prop is then held there indefinitely.

And with that all props are present and accounted for!

### Prop Vision
I realised while playing that some props aren't obviously upright. For example, the hose looks pretty similar both upright and upside down, and sometimes an upright object can be hidden in a pile of fallen objects.

I wrote a function in `Main` that helps the player identify all the props that are still upright:

![Image PWW 2.12 - Highlight Standing Props](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.12.png)

This function iterates through all the props, identifies whether they're upright or not using the `IsFallen` function in `BP_Prop`, then changes the material based on the result.

The two materials it alternates between are the `HighlightedProp` material and the `DefaultMaterial`. `HighlightedProp` is a custom material I made that turns the props red and slightly transparent, emitting a slight glow to help the upright prop stand apart from the rest of the world. The `DefaultMaterial` is the original material of the prop, with each `BP_Prop` storing their default material in a variable at the start of the game:

![Image PWW 2.13 - BP_Prop Begin Play](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.13.png)

With that, the upright props all glow as intended!

However, as it is, the upright props would always be highlighted, which takes some of the fun out of the game. The highlight is supposed to be a _hint_, not a solution, so the material change should be temporary.

I tagged `HighlightStandingProps` to the end of a new function, `SetPropMaterials`:

![Image PWW 2.14 - Set Prop Materials](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.14.png)

At the end of the function, it swaps between the `HighlightStandingProps` function and another function that iterates through all the props and reverts them back to their `DefaultMaterial`.

A new variable - `HighlightActive` - stores whether or not prop vision is active, and its value is passed into the branch; if it's true, `HighlightStandingProps` is called, otherwise the props are returned to normal.

To make sure they don't stay highlighted forever, the function starts by updating a `Counter` variable, which ticks upwards every frame. Once that counter reaches `3` (or, in other words, when 3 seconds have elapsed), `HighlightActive` is automatically set to `false`.

Finally, to kick off the whole procedure, we have the triggering code in the `Main` blueprint:

![Image PWW 2.15 - RMB in Main Blueprint](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.15.png)

When the right mouse button is pressed, the `Counter` is reset and the highlight function is activated.

There we have it! The game now swaps textures out for a few seconds to give the player a hint before resetting. It also has the added bonus of dynamic material changing; as props move from upright to fallen to upright again, their materials reflect the state they're in.

In future, it may be a good idea to create some conditions on this ability. For example, knocking over 5 props might unlock a single use of this ability, or perhaps the player gets unlimited uses once they have less than 10 ammo remaining. Maybe we'll see more on this in the future!

### The Player
You may have noticed that I have a blueprint for the player, while the original course doesn't include one, instead opting to add their code to the `Main` level blueprint instead.

I wanted to factor out the player code from the level code. This was mostly just to keep my code clean, but it would also mean less repeated code if I ever wanted to add more levels in the future.

However, to refactor the code into my player, I'd need to have a player in the first place...

I went the easy route: I decided to import the "First Person Example Level" assets through the content window.

I moved the `SpawnProjectile` code over and refactored it into a new function, `FireProjectile`:

![Image PWW 2.16 - Fire Projectile](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.16.png)

As you can see, I made some changes to the original (see [above](#the-original-project)). First of all, I removed `Launch` from `SpawnProjectile`, so each function does only what it says on the tin: the first spawns the projectile into the world, the second launches it from the player.

Secondly, I updated `DecreaseAmmo`:

![Image PWW 2.17 - Decrease Ammo](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.17.png)

I edited it slightly, so the `ammo` could not drop below `0`.

Now, on left click, `FireProjectile` is called instead:

![Image PWW 2.18 - Player LMB Click](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.18.png)

This concluded the refactor, now the player blueprint has total control over the firing code.

### GUI
The last thing it really needed to feel like a _game_ was a HUD.

There's quite a bit of information that needs to be shared with the player so they can make informed decisions.

But first, we need some reference to the player, which will be cached as soon as the game starts:

![Image PWW 2.19 - BP_HUD Construct](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.19.png)
> [!NOTE]
> _It may be better to call this function every time - rather than calling once and caching - in case the reference ever becomes stale or null._

After a reference to the player is stored in `Character`, the "Game Over" screen is hidden. Obviously, we don't want a chance of that showing up until the game has actually finished!

Using the `Character` cached before, the HUD checks the player's ammo data to display the amount of ammo the player started with:

![Image PWW 2.20 - Get Max Ammo](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.20.png)

And how much they have left:

![Image PWW 2.21 - Get Ammo Remaining](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.21.png)

This information allows the player to manage their ammo carefully and avoid defeat. But that's only half the information they need; they also need a prop count, so they know if they are close to winning.

Finding the total number of props is straightforward:

![Image PWW 2.22 - Get Total Number of Props](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.22.png)

Finding the number of remaining props, maybe less so:

![Image PWW 2.23 - Get Props Remaining](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.23.png)

The logic is pretty simple, but there's a lot of repeated code that needs refactoring.

`GetPropsRemaining` starts with a branch: if the player has won the game, then the HUD will say there are no props remaining. This is because the HUD can appear glitchy if an object is still flying through the air, switching rapidly between 0 and 1. While it makes no functional difference, I felt that it made the game logic seem indecisive even after the player has won, which would likely lead to an ambivalent player experience.

If the player has not won, the function will iterate through all the props in the game, incrementing `nProps` for each upright prop.

Once the loop is complete, the value of `nProps` is returned, then displayed on the HUD as the number of props remaining upright.

These functions give the player direct feedback about how close they are to reaching or failing their primary objective, a critical piece of information for a satisfying experience.

I also added a sidebar to help the player with controls; while some of it is intuitive (e.g. WASD to walk, or LMB to shoot), there may be functionality they otherwise wouldn't know about (e.g. RMB for "Prop Vision"). Either way, it's better to be safe than sorry!

I didn't want to clutter the screen however, so I made a simple show / hide function for the controls menu:

![Image PWW 2.24 - Toggle Controls Menu](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.24.png)

Calling `ToggleControlsMenu` inverts the boolean `IsTooltipOpen`, which then toggles between two paths: one which displays the full controls menu, and one which displays an abridged version.

Then, for the finishing touches, I added a variable game over screen:

![Image PWW 2.25 - Set End Screen](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.25.png)

Using the same screen, the HUD displays a different message depending on whether the player won or lost.

With that, the player has all the information they need. Hopefully this will reduce frustration, resulting in a better player experience.

In the future, I may add a crosshair, or some indication of where the player is aiming. However, in my experience, not quite knowing was part of the fun! Otherwise, it didn't feel particularly challenging. That said, it may end up as an optional feature. After all, it's always nice to have the option!

### The Main Level
While I've talked through a lot of my code, I haven't shown much of the `Main` level blueprint, which ties it all together.

I've honestly been avoiding this one, as it's quite messy and could do with a _lot_ of refactoring in the future. I feel like this was the default place I put things as I was learning blueprint, which was likely a poor decision design-wise.

When the game starts, the level tells the HUD to initialise:

![Image PWW 2.26 - Main Level Begin Play](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.26.png)

This calls `SetupHUD`:

![Image PWW 2.27 - Setup HUD](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.27.png)

The function starts by creating the HUD and adding it to the viewport. The `HighBounds` and `LowBounds` are then passed in for `GetPropsRemaining`. It probably shouldn't work like this, but it does for now.

Speaking of the HUD, the level blueprint also binds the `ToggleControlsMenu` function from `BP_HUD` to the `Tab` key:

![Image PWW 2.28 - Main Level Tab Menu](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.28.png)

The `R` key is also bound to a quick reload:

![Image PWW 2.29 - Main Level R Reload](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.29.png)

The function `ReloadLevel` runs this code:

![Image PWW 2.30 - Reload Level](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.30.png)

Which just opens the current level again.

All that's left to talk about is `Tick`...

![Image PWW 2.31 - Main Level Tick Full](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.31.png)

Oh boy...

Let's start at the beginning:

![Image PWW 2.32 - Main Level Tick p1](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.32.png)

Straight away, the level checks to see if the game is over (whether the player has won or lost is irrelevant right now). If the game is over, then the level stops performing functions on `Tick`. Otherwise, it carries on to the prop functions.

These have all been highlighted in previous sections. `SetPropMaterials` comes from [Prop Vision](#prop-vision) above, while `KeepPropsInBounds` was first described in [Falling Through](#falling-through).

`HaveAllPropsFallen` hasn't been shown before, but may be... _familiar_...

![Image PWW 2.33 - Have All Props Fallen](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.33.png)

You may be wondering: "How many times do you need to iterate through the same list of props and check if they've fallen?"

I ask myself the same question every time I open a function and see _another_ for-each loop...

As contradictory as it may sound, I actually think it can be _harder_ to visualise code in blueprint. Or, at the very least, it's _different_.

In future, I'd like as few for-each loops as possible. But it is what it is for now.

Tangent aside, you will see in the screenshot above that the return node has an output. If _any_ prop returns as not fallen, the function returns early with a value of `false`. If the loop concludes without any upright props found, then it concludes that all props must have fallen, returning `true`.

This value then gets passed into the branch:

![Image PWW 2.34 - Main Level Tick p2](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.34.png)

If all the props have fallen over then `GameOver` is set to `true`, thus stopping `Tick` from going any further in future calls. The victory message is shown on the HUD, then there's a short delay before the level is reloaded. In future, this may load the next level instead.

So, what happens if any of the props are still upright?

![Image PWW 2.35 - Main Level Tick p3](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.35.png)

Then we need to check if the _player_ has fallen off the map and reload the level if so.

The graph for `HasPlayerFallen` is pretty simple:

![Image PWW 2.36 - Has Player Fallen](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.36.png)

Like with props, it simply checks to see if the `BP_FirstPersonCharacter` is within the high and low bounds. It mostly just prevents players from doing a Tommen Baratheon and jumping to their doom.

In hindsight, I probably should've let them continue from where they left off. That sounds like a better (and less frustrating!) player experience. Or maybe - on certain maps - that's part of the challenge!

Either way, it's something to think about as I make new levels.

So, now we've established the player is in the room and there are props left to knock over. What happens next?

![Image PWW 2.37 - Main Level Tick p4](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%202.37.png)

Finally! The end of this if / else nightmare we're trapped in!

First, it checks to see if the player still has ammo; if they don't, that could be the end of the game!

But sometimes, a player may use their last projectile to knock over the final prop, which I would still want to be a victory. So the level waits a few seconds before making this decision.

After that time has elapsed, the current game state is evaluated. The truth table for the `NOR` gate looks something like this:
> - If the player does not have ammo remaining and the game is not already over, the player has lost (true) - the "Game Over" screen will show
> - If the player does not have ammo remaining but the game is already over, the player has either won with their last projectile or ran out _after_ the game ended (false) - the "Congratulations!" screen has already appeared
> - If the player has ammo remaining and the game is not over, play continues as normal (false) - the player is still playing the game and hasn't triggered either condition
> - If the player has ammo remaining and the game is over, the player has already won (false) - the "Congratulations!" screen has already appeared

If this evaluates to `true`, then it runs the same 4 functions as when the player wins: `GameOver` is set to `true`, `ShowPlayerEndScreen` is called with a short delay before reloading. The only real difference here is that `HasWon` (a parameter on `ShowPlayerEndScreen`) is set to `false`.

I normally would've condensed that series of functions into one for the sake of cleanliness and avoiding repetition, but it's impossible to put `Delay` inside other functions. In future builds, I may try to avoid `Delay` entirely, instead opting for something like the `Counter` that tracks the "Prop Vision" countdown.

Needless to say, it could do with some work. A _lot_ of work. But that's the `Main` level blueprint in its entirety.

### Conclusion
I think this went okay! The code is a bit messy, but that's to be expected when trying something new. I can already see plenty of areas I want to fix and improvements I want to make, which is more important to me. While I'm a perfectionist at heart, I'm learning to appreciate the process, the small iterations that _lead_ to perfection (or close enough!).

But _most_ importantly, I've taken a project that was barely a tech demo, and turned it into a full - albeit small - game!

With the addition of a proper win condition and a HUD that clearly communicates the state of the game with the player, it actually _feels_ like a game. Plus, the addition of "Prop Vision" gives the player a little something extra to help them in their quest.

It still has a lot of room for improvement though! More levels would obviously be great, as well as more features like ammo pickups or limited "Prop Vision". But even small quality of life tweaks, like optional crosshairs and a respawn on `HasPlayerFallen` instead of a reload, would go a long way. Maybe even some music or sound effects (once I know how to do that!)

I think it's fair to say I'll be revisiting this project when I'm a little better trained. Or maybe tomorrow! Even just by doing this in the first place, then explaining what I did and why, I feel like I'm already a little better than before.

I really enjoyed working on this. I hope you've enjoyed playing and reading along!

# Summary
Thank you for reading! If you want to see more of my work, please check out my [portfolio](https://github.com/Xakaiczar/Portfolio), and if you have any questions or opportunities, please don't hesitate to drop me an [email](xaqatkins@virginmedia.com) or message me via [LinkedIn](https://www.linkedin.com/in/xaqatkins/)!
