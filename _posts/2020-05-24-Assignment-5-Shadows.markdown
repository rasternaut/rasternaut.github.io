---
layout: post
title:  "Shadows"
date:   2020-05-24 00:00:00 -0800
tags: [lighting,  cs636-raytracer, raytracing]
visible: 1
---

This is the first of three changes that add more of the "fun" part of raytracing - simulating light. Here we improve on the standard Phong implementation that dates back to HW 2, by adding shadows.

Our approach to shadows is simple, on a ray/object intersection, shoot a new "shadow" ray to each of the light sources in the scene. If the ray strikes anything on the way to the light, then the light is obscured and it does not factor into the lighting calculation.

## Adding Shadows

Due to some forsight and attention during the design phase, much of the code necessary for shooting a new kind of ray, shading, and getting shadows working had been designed in such a way that the primary change needed to make this all work was less than 10 lines of code. The change can be found in `raytracer.cpp` on line 81.

{% highlight c++ %}
//phong shade diffuse and specular per each light
for (auto& const light : lights) {

    Intersectable* objectPtr = nullptr;
    glm::vec3 intNormal;
    float intDistance;
    Pixel intPix;

    // Shadows
    glm::vec3 shadowCheckPoint = intersectionPoint + (intersectNorm);
    float distToLight = glm::distance(light.position, shadowCheckPoint);
    if (ShootRay(BuildRay(shadowCheckPoint, light.position), sceneObjects, objectPtr, intNormal,
        intDistance, intPix) && intDistance < distToLight) {
        continue;
    }
{% endhighlight %}

Since we built `ShootRay` as a stand-alone function, we can reuse the same logic block that shoots our primary ray. True, we get back information that we don't use here, the intersection normal, and the color at the intersection, but we can simply ignore it.

## Images
The [first scene](https://gitlab.com/TaylorEllington/cs636-advanced-rendering-techniques/-/blob/0688b48249fbe072d4ebcdbac1f3e8d398c88648/assets/single-shadow-scene.json) we created was simple, just a light, a floor, a sphere at the origin,  and an object. This was the minimum needed to make sure that the feature was working.

![Simple Shadow Scene](/assets/images/HW_5/single-shadow.png)

Render time - 1956ms

Next, we made a [scene](https://gitlab.com/TaylorEllington/cs636-advanced-rendering-techniques/-/blob/0688b48249fbe072d4ebcdbac1f3e8d398c88648/assets/complex-shadow-scene.json) that met the requirements for the assignment. Multiple meshes, spheres and lights. One of the meshes in this scene is a hollow ring. Positioned so that when viewed straight on it appears to be a solid, only when looking at the shadow does its true shape become apparent.

![Complex Shadow Scene](/assets/images/HW_5/complex-shadow.png)

Render time - 2944ms

Finally, we wanted to create something a little artsy. We first tried to generate something resembling a Warhol-esque image on a "canvas" instead of just shadows on the floor. Sadly, we couldn't quite get there, but one of the [final attempts](https://gitlab.com/TaylorEllington/cs636-advanced-rendering-techniques/-/blob/0688b48249fbe072d4ebcdbac1f3e8d398c88648/assets/shadow-puppet-scene.json) still looks pretty good! ( I had to modify the mesh file, the file contains a flat plane that the dragon is standing on. I removed this to get a good full silhouette)

![Shadow Puppets Scene](/assets/images/HW_5/shadow-puppets.png)

Render time - 6001ms
