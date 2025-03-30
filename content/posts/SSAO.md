---
title: 'Screen Space Ambient Occlusion'
date: 2025-03-26T15:31:50+01:00
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


| Type          | Description   |
| -----------   | -----------   |
| **Engine**    | Penugine      |
| **Timeline**  | ~1-2 weeks 50%|


{{% figure class="alignright" src="/ssaoEffect.png#floatleft"  %}}
<!--more-->
Screen Space Ambient Occlusion, more commonly known as SSAO, is a post-processing technique used to approximate how occluded a pixel in a 3D scene would be. This is done to adjust the world’s ambient light accordingly, mimicking the effect of the pixel's occlusion. This is performed as a screen space pass, taking in geometry positions and normals, hence the name Screen Space Ambient Occlusion.

# Implementation

The purpose of SSAO is to determine how occluded a pixel is. To do this, we need some knowledge about the rest of the 3D scene. In our case, we take the Z-Buffer from the camera's perspective. In this case, we save the perspective Z value in a texture before our SSAO pass. This corresponds to the depth.

When determining a pixel's occlusion, some precomputed random directions are used as an offset from the pixel's world position, which is taken from the geometry buffer. This new position is projected into view-space to get its z-depth. With the found value, a comparison is made to the original z-depth of our pixel to determine if it's closer to the camera. If it is, that pixel is occluded by that position, contributing to the occlusion factor. This is then repeated a handful of times to get an estimate of the amount of occlusion the pixel would have


{{% figure class="alignright" src="/SSAOIllustation.png#floatleft"  caption="Figure showing occluded projected possitions in a sphere"%}}

However, a pixel cannot be occluded by a position behind its own surface. To improve the efficiency of a sample, instead of using a sphere, we ensure the position is aligned with the pixel's surface normal (see figure below).

To achieve this, we take the surface normal of the pixel and create a valid TBN matrix for its rotation space. A TBN matrix (Tangent, Bitangent, Normal) is used to map the random sample directions into the correct orientation in world space based on the pixel's normal vector. This process is very uniform, so noise may be used to introduce variability, and used to rotate the TBN matrix into another valid one, resulting in less uniformity by having varied matrices.

{{% figure class="alignright" src="/ssao_hemisphere.png#floatleft"  caption="Figure showing hemisphere"%}}



# Results

Following, a comparison image with and without the effect can be seen. The results act in a way to adding relation to each scene object and a level of depth to their surrounding backgrounds. In the comparison image, the effect of SSAO adds realistic shadows and depth to the scene, making the objects feel more grounded in the environment.

{{% figure class="alignright" src="/slideSSAO.gif#floatleft"  %}}

