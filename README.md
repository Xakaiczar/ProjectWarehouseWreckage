# Project "Warehouse Wreckage"
_Developed with Unreal Engine 5_

**DISCLAIMER**: This project is based heavily on "Warehouse Wreckage", the first module of "Unreal Engine 5 C++ Developer: Learn C++ & Make Video Games" by GamedevTV. Most of the code will have been written and designed by me, and it may differ significantly as I diverge from the lecturer to solve the problem myself. However, all work here is inspired by - and heavily derivative of - work that is not my own, therefore this project is not my own and cannot be distributed. This project is for assessment by prospective employers only.

# About the Project
"Warehouse Wreckage" is the name of the first module in the course "Unreal Engine 5 C++ Developer: Learn C++ & Make Video Games" by GamedevTV. The premise is simple: knock over every object in the room with a limited supply of ammunition. However, the course does not implement any successful end condition; the player can only lose. I added a win condition myself, properly telegraphed that information to the player, and - to challenge myself - I did it all using Blueprints.

# Demonstration
video / screenshots

# Development
## The Original Project
The original project is very simple. Aimed at beginners who are learning the engine for the first time, it covered the basics well. The lectures briefly run down:
- Importing assets and using the provided assets
- The physics-based properties attached to an object in the world
- Using Unreal's Blueprints to reference and modify objects in the game

Once all that's out the way, we're set to work on the core gameplay.

First, I made a projectile with a simple sphere in a silver colour. This sphere was converted into a Blueprint Class and used as the projectile.

![Image PWW 1.1 - Blueprint Class for projectile](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%201.1.png)

To fire a projectile, one first needs to be spawned into the world. Using a reference to the projectile blueprint made in the previous step, an instance of that class is created in the Main level blueprint and spawned in the world at the player's location using SpawnActor.

PROJECTILE CODE

The projectile is then fired. It was given an impulse, which is the force required to fire it from the player. This impulse is determined by finding the forward vector of the projectile, then multiplying that by a value that - in hindsight - should've been a variable. I'll fix that later.

![Image PWW 1.3 - Blueprint Code of projectile launch code](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%201.3.png)

A rotational vector is applied to aim the projectile in the right direction, and another is used to determine where it needs to be fired from. TALK ABOUT CODE.

I THINK I NEED AN IMAGE HERE

The ammo is then tracked. TALK ABOUT CODE?

AMMO TRACK CODE

The level is reloaded when out of ammo. TALK ABOUT CODE?

OUT OF AMMO CODE

I also added in a manual reload for testing purposes.

![Image PWW 1.7 - Blueprint Code of reload code](https://github.com/Xakaiczar/Portfolio/blob/main/images/PWW/PWW%20-%201.7.png)

And there we have it! A game about a person stood in a floating abandoned warehouse throwing metal bowling balls at whatever random garbage happens to be around them!

## My Contribution
Now, that doesn't really sounds like a _game_, does it? It's more of a sandbox than anything else, but there's no goal, no win state, and the fail state is completely silent and sudden.

I wanted my contribution to make this demo into an actual game. To do that, I'd need to fix the issues above.

### Falling Over
First of all, I wanted the game to actually be able to track which props had fallen and which were still upright, then return true if all of them had been knocked over; this would make for a good win condition. If I was using Unity, I would slap a box collider on the bottom of the prop and call it a day. But this time, since I'm in a new engine, I thought I'd try to solve it a new way.

That new way looks like this:

IMAGE OF FALLING RADIUS

What's happening when an object is falling over? Well, mathematically, it's rotating around a certain plane by a certain angle. In this case, the X and Y axes tilt the prop, while the Z just kinda... _spins_ it...

Using this, we can actually determine whether a prop has fallen over or not by its angular tilt! If it surpasses a maximum tilt, then it has fallen over, otherwise it is still stood upright. I decided on a default maximum tilt value of 45 degrees for both axes:

IMAGE OF FALLING RADIUS WITH NUMBERS

As you can see from the figure above, this number was not arbitrary; it falls exactly between perfectly upright and perfectly fallen (or perfectly perpendicular, if you will). For a prop to still be stood up at that angle, it would have to be truly magical...

GIF OF MAGIC HAPPENING BC OF COURSE IT DOES

Maybe 45 degrees was a bit too large... But it gets the job done, and for now, that's all that matters. We can fine-tune it later.

The bounds of each prop are then fetched, creating two theoretical 45 degree angles, one tipping forwards from 0 degrees and one tipping backwards from 360 degrees.

IMG OF FUNC

If its rotation on the X OR Y axis happens to be in the range of:
> (0 + _t_ < n < 360 - _t_, where _t_ is the angular tilt)

Or (45 < n < 315) in this case, then it will be flagged as "fallen" in the main blueprint.

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
