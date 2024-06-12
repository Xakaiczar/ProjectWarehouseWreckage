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

IMAGE - PWW 2.1
> (_a_ < n < 360 - _a_, where _a_ is the angular tilt)

The idea here is that a prop must rotate by an angle of _`a`_ before it starts falling with no hope of return. Inversely, this means that any prop with an `x` or `y` rotation of 45 degrees or lower must still be stood up, to some degree (no pun intended...).

This is then mirrored on the other side, as an angle of _`-a`_ will cause it to fall the other way. Unreal doesn't seem to like negative angles very much (as far as I can tell), so I'm instead subtracting it from 360 degrees, which is fundamentally the same thing.

I decided on a default angular tilt value of 45 degrees for both axes:

IMAGE - PWW 2.2

As you can see from the figure above and the image below, this number was not arbitrary; it falls exactly between perfectly upright and perfectly fallen:

IMAGE - PWW 2.3

Maybe 45 degrees is a bit too large for some props, but it gets the job done; for now, that's all that matters. We can fine-tune it on a case-by-case basis later.

To determine if a prop has fallen, we need to get these boundaries. I wrote this function called `GetTiltBounds`:

IMAGE - PWW 2.4

This function takes in one parameter - `Angle`, which we'll get to later - and returns two values: a `MinTilt` and a `MaxTilt`. While not very well named, these return values are the upper and lower bounds for the prop's rotation. In other words, the prop must be between `MinTilt` and `MaxTilt` to be considered "fallen".

The graph looks more complex than it is. In C++ it would look like this:
```C++
float a = ToppleTilt % Angle;
float b = (360 - ToppleTilt) % Angle;

float MinTilt = min(a, b);
float MaxTilt = max(a, b);
```
> _Where `ToppleTilt` is the variable angular tilt the prop would need to fall to be considered "fallen", mentioned earlier and defaulted to `45`._

It starts by getting the remainder of the `ToppleTilt` divided by the `Angle`, on the off chance the `ToppleTilt` causes over-rotation. In this case, if `ToppleTilt = 405`, it would still return `45`. It then also does the same for `-ToppleTilt`, as mentioned before.

The smallest of the two values is passed to `MinTilt`, and the largest is passed to `MaxTilt`.

These values get passed into a much more convoluted function:

IMAGE - PWW 2.5

If the prop's rotation on the X OR Y axis happens to exceed the range of (45 < n < 315), then it will be flagged as "fallen" in the main blueprint.

IMG OF FUNC

In an ideal world, this method would've worked for every prop in the game. But sadly, not all props work the same.

The pallet prop was imported lying flat (as one would expect a pallet to be in a real-world context), but this didn't quite work, and I had no idea how to reimport it without the original fbx asset...

It took me an embarrassingly long time to figure out, but this is effectively what it looks like:

BARREL IN PALLET

Now relate that to the graph from above (here it is again, for your reference):

IMAGE OF FALLING RADIUS OVER PREV IMG

Looking at it, the pallet (or any perpendicular object, for that matter), seems to be the _complete opposite_ of the upright barrel: when the barrel should be upright, the pallet should be be down, and vice versa. This means that the fall arc of a perpendicular object is actually the _inverse_ of a regular object.

DIAGRAM OF THAT IDEA

So if you use a NOT gate to invert the fall check, it should invert the theoretical segment:

IMAGES SHOWING PALLETS LEANING AND STOOD

And, lo and behold, the inverse segment works!

IMAGE OF INVERSE WORKING

This method still doesn't seem to work on negative values however. That still needs fixing, at some point...

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
