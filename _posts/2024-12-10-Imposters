---
layout: post
title: Imposter Baking and Rendering
---

## Imposter Baking and Rendering (WIP)

Chances are you've heard the term imposters used several times by now. But what exactly are they, and how do they differ from billboards? I don't think there is a strict definition, but imposters are generally more advanced, and attempt to recreate the shading -inputs- of the object they are capturing, such as albedo, normals, and even depth for accurate shadowcasting/receiving, SSAO, etc. Imposters are typically rendered as a camera-facing quad. (However, I will describe another approach below)

So how do imposters work? Fairly simply, you essentially take screenshots from multiple angles, and save them into a texture atlas. At runtime, you use the direction to the camera to select from a screenshot, and display it. However, this can lead to noticable popping as the angle changes, so imposters also allow for blending between multiple frames. This is done by pretending you are rendering a triangulated octahedral, or hemi-octahedral mesh. You figure out which triangle you are looking at, and then how close you are to each of the three corners of the triangle. (Each of which stores a specific image/screenshot) You then blend between those three screenshots based on your location in the triangle. (You could also do this with a quad, but it means 4x texture samples instead of 3, and we like reducing texture samples where possible. You could go even further and use stochastic techniques+noise to do a single sample each frame, and use TAA to smooth the blends, but TAA has it's own issues. If this was combined with some kind of alpha-weighted temporal strategy based on the frame blends, it could be intersting however, but.. Lets not worry about that for now, but if you do anything with this, let me know how you go!)

Anyway, while capturing the object, the captured depth can also be saved to another texture. This can be used in the same way as you'd use a parallax map, to distort the UV coordinates to reproduce the captured object more correctly. Going a step further, you can use this depth/height/parallax map (Same thing really) to output a custom depth value from the pixel shader. This allows the imposter to interact with both shadow casting and receiving in a way that is similar to the full 3D model, and can even allow effects such as Screen Space Ambient Occlusion or contact shadows to affect the imposter.

However, all of the above needs to be done in a fairly specific and consistent way. Which angles do you use for the captures? There are tradeoffs here, as a simple angle scheme such as polar coordiantes (pitch/yaw) results in many captures around the top and bottom of the object, while spreading them out elsewhere, but a uniform distribution is generally better. This is why octahedral and hemi-octahedral layouts are popular, as they give a fairly uniform distribution, and are also fairly cheap to calculate in a shader. 

So to capture the object, you need to use the inverse oct/hemi-oct transform to take a frame index and convert it into a view vector. This then becomes your direction. You need to also set up the correct camera matrices. An ortho matrix should be used here. How big should it be? You could calculate the min/max of the object's AABB in camera space, however this could change based on angles, and we don't want to have to store a separate aspect ratio/min-max for each angle for access by the shader. So a simple approach is to calculate a minimal bounding-sphere for all the vertices of the object. (You can also simply calculate a sphere that encloses the object's AABB, but this may be wasteful) Now the camera can rotate around this AABB, and the orthographic projection matrix's width/height/depth will simply be the diameter of this sphere.

It's fairly straightforward to lay the screenshots out in an atlas. However, depending on your engine, a texture array can be a better choice. This is because there is no constraint on size/resolution/frame-count, and no issues with border filtering or mipmapping. (A traditional atlas needs additional checks to avoid crossing boundaries, and mip-levels start to bleed into eachother) For example you could have a 12x12 atlas with 128x128 frames, but this wouldn't be as clean with an atlas, as 2048 does not neatly divide into 12, so you would end up with some odd pixel offsets.

While capturing your object, you'll want to take note of the sphere position and radius, as you'll need these values in your shader/mesh. Speaking of the mesh, you can simply use a camera-aligned quad, however this can cause unneccessary overdraw. While you can use a cutout mesh instead, since the shape changes with each angle, this limits your options. My preferred solution is to use a convex hull. This is conservative, so will not cut off the mesh, but is otherwise fairly optimal and fast to generate. Of course you want to keep the vertex count low. You can also use a mesh representing the AABB of the object in place of a convex hull.

Depending on your engine, there are a few ways to set up the mesh/transform matrix. For my implementation in Unity, I set the position/scale to be the bounding sphere's size/diameter. The mesh I generate is a unit-cube (range -0.5 to 0.5) but scaled by the AABB size of the mesh, and inversely offset by the sphere. This lines everything up nicely, while keeping the center/scale of the imposter consistent with the bounding sphere for best use of texture space while baking/rendering.

Now for the shader. The only input to my shader is the vertex position, everything else is calculated or comes from textures. For max efficiency, I implement as much as possible in the vertex shader. I save a few divides for the fragment shader to avoid any distortion, as passing pre-divided vertex shader values to the fragment shader does not give correct results.

First, calculate the view vector to the center of the transform, -not- the vertex position. We want all vertices to use the same imposter atlas frames, as we don't want strange interpolations happening between frames we can't control, as this will just cause jumbled Uvs that don't look correct. 

An important detail here is that you should do everything in object space. This means transforming the camera vector (eg worldCameraPos - worldTransformPos) into object space. This will handle any rotation/scale on the object. Now use this object-space camera vector as an input into your octahedral or hemi-octahedral transform, to get the frame index. (You can skip normalizng the above vector, as it does not affect the result)

Next, we need to calculate the UV coordinates of our frame. This is done by doing a ray-plane intersection. Each vertex in our input mesh is essentially a target point for a view vector. We intersect this with a virtual "plane" that matches the angle we captured the imposter from, and the coordinates of that plane intersection give us the UV coordinates. We have to do some of this work in the fragment shader however, otherwise the result will be distorted.a

-- todo: add frame transform stuff --



 You could simply pass this to your frag shader, sample the right frame, and be done. However there are several quality improvements that can be made:
	- Parallax mapping
	- 3-frame blending
	- Pixel Depth Offset

We'll start with parallax mapping. This is similar to a regular parallax shader, though there are some slight differences. Calculate the object-space view vector. This is similar to the above vector used to calculate the atlas/array index, but instead of calculating it to the transform center, we calculate it to the vertex position. We want this to vary across the surface of the object, to give the parallax effect. 
