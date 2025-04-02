---
title: 'Quake inspired Movement with Cod Zombies adjustments'
date: 2025-03-26T15:31:00+02:00
draft: false
hidden: false
externalURL: false
showDate: false
showModDate: false
showReadingTime: false
showTags: false
showPagination: true
invertPagination: true
showToC: true
openToC: false
showComments: false
showHeadingAnchors: true
---
| Type          | Description |
| -----------   | ----------- |
| **Engine**    | Penugine    |
| **Timeline**  | ~1 week 50% |

<!--more-->

## Purpose
For a game project with the goal of mimicing CoD Zombies we needed a character controller and fluid movement. For this purpose inital system was build around the quake 2 movement for it's plenitufl documentation and fluidity of movement. However some quirks of this system made new systems neccisary to fully reach a working in-game implementation.

This movement was implemented on a PhysX object. However this is a kinematic object so the motion is fully controlled by the movement controller and later on validated by PhysX.
## Quake movement
Getting movemnt that feels good and intuitive to use by following in quakes footsteps is not a hard task. With this movement system being analyzed countless times, the documentation of what makes it click is mostly set in stone. With all of this and the IdTech 2 engine to refer to this was set up within the first day.

## MW3 adjustments
This system, though it feels well to use, is not fully suited for our CoD zombies purpose. To fix this, studying of Call of Duty: Modern Warfare 3's movemnt had to be done. This version was choosen due to internal cloes communities finding this to be some of the best feeling movemnt in the series. It also has a custom client where the movement can be tested in sandbox enviourments. 

### Bunny hopping
One of the core features in the Quake IdTech movement, and the engines inspired by it is the mechanic of bunny hopping. This mechanic is based in the differance in leght of a wished velocity, current velocity, and wished direction is used to gaint more speed when the movement is turing towards a new direction. This allows for continuous increase of speed while in the air and not being effected by ground friction. 

To keep this effect going, one jumps on the first frame the player lands to avoid triggering the ground friction. This effect is indeed very fun to use and feels quite good to preform, so why is it a problem. In a round based CoD Zombies inspired game, kiting zombies effectily is vital. Using your sorroundings to be able to keep a fair distance while not being caught offguard is a key gameplay element. So if you can just fly away at incredible speeds this becomes a bit redundant.

So one solution would be to add air friction. Sounds like a clear fix. One problem with this is that the movement instantly stars feeling as smooth by this, and one other key feature yet to be mentioned gets destroyed by this. That would be long jumping

### Long Jumping
Long jumping is the act of almost performing a bunny hop but only once. This allows for one long fast jump by strafing just before and in the air when you jump. A core mechanic in higher levels of CoD Zombies play to get out of thight spots. 

### The fix
What can be done to fix this then? Well, forcing the players to stay on the ground for a bit to force the application of ground friction would be a way. To achieve that, one implementation is to introduce jump stamina. Exactly that was done. After jumping, an internal counter goes up for how many times you jump in a row, effectivly tiring out the player if jumping too much in a row. This is then used to delaying the player from jumping again. 

### Polishes
Most developers know, limiting the player from doing something does not feel too well. To solve this we need to give the player the illusion of still landing while being grounded. This implementation achieved this by using the view model of the player to keep moving, scaling with the jump stamina. Making it feel more intuitive with the wait.

While also handling input buffering to be able to jump without having to time the landing or waiting for the stamina was key to the fluid feeling of the movement syste.

## Areas of improvement
A lot of the areas of improvment land on the systems around the movemnt. More solid sounds to acompany what you are currently doing, or clearer animations to make it feel more fluid would be on the top of the list. However these were not my areas of implementation.

In the system itself, most the improvemnt lay in tweaking of variables to mimic my referances more properly.