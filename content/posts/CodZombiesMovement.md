---
title: 'Quake inspired Movement with Cod Zombies adjustments'
date: 2025-03-26T15:31:02+01:00
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

{{< video src="../../../Portfolio/bluredShowcaesBlurred.mp4" autoplay="false" loop="true" width="800" height="450" >}}  
<!--more-->

## Purpose
For a game project aiming to mimic CoD Zombies, we needed a character controller and fluid movement. The initial system was built around Quake 2 movement due to its plentiful documentation and fluidity. However, some quirks of this system necessitated additional modifications to achieve a fully functional in-game implementation.

This movement was implemented on a PhysX object. However, since this is a kinematic object, motion is fully controlled by the movement controller and later validated by PhysX.
## Quake movement
Creating movement that feels good and intuitive by following in Quake’s footsteps is not a difficult task. This movement system has been analyzed countless times, and the documentation detailing what makes it work is well established. With these resources and the IdTech 2 engine as a reference, the initial setup was completed within the first day.

## MW3 adjustments
While this system feels great to use, it is not entirely suited for our CoD Zombies-inspired gameplay. To address this, we studied the movement mechanics of Call of Duty: Modern Warfare 3. This version was chosen due to internal close communities considering it one of the best-feeling movement systems in the series. Additionally, MW3 has a custom client where movement can be tested in sandbox environments.

### Bunny hopping

{{< video src="../../../Portfolio/bhop.mp4" autoplay="false" loop="true" width="800" height="450" >}}  

One of the core features of Quake's IdTech movement, and the engines inspired by it, is bunny hopping. This mechanic relies on the difference in length between the desired velocity, current velocity, and desired direction to gain additional speed when changing direction mid-air. This allows for a continuous increase in speed while airborne, avoiding the effects of ground friction.

To maintain this effect, players must jump on the first frame upon landing to prevent triggering ground friction. While this mechanic is fun and feels great to perform, it poses a problem in a round-based CoD Zombies-style game. Effective kiting of zombies is a vital gameplay element, requiring players to use their surroundings to maintain distance without being caught off guard. If players can continuously gain speed and evade threats effortlessly, the challenge of the game is diminished.

A potential solution is adding air friction. While this might seem like a clear fix, it significantly reduces the smoothness of movement and interferes with another key feature: long jumping.

### Long Jumping

{{< video src="../../../Portfolio/longjumpBlurred.mp4" autoplay="false" loop="true" width="800" height="450" >}}  

Long jumping involves executing a single extended jump by strafing just before and during takeoff. This technique is crucial in high-level CoD Zombies play for escaping tight situations quickly.

### The fix
How can we balance these mechanics? One solution is to enforce brief ground contact to apply ground friction. This was implemented by introducing a jump stamina system. Each consecutive jump increases an internal counter, effectively tiring out the player if they jump too frequently. This mechanic introduces a delay before the player can jump again, preventing excessive bunny hopping while preserving movement fluidity.

### Polishes
As most developers know, restricting player movement often leads to frustration. To counteract this, we needed to create the illusion of grounded movement even while enforcing the jump delay. This was achieved by adjusting the player's view model to maintain motion, scaling with the jump stamina to make the wait feel more natural.

Additionally, input buffering was implemented, allowing players to jump smoothly without needing to precisely time their landings or wait for stamina recovery. This was key to maintaining the fluid feel of the movement system.

## Areas of improvement
Many areas for improvement lie in the surrounding systems rather than the movement itself. More refined sound design to reflect player actions and clearer animations to enhance fluidity would significantly improve the experience. However, thses were not my areas of responsibility.

Within the movement system itself, most improvements would involve fine-tuning variables to better replicate the reference movement mechanics.