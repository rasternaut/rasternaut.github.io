---
layout: post
title:  "Reflection"
date:   2020-05-31 00:00:00 -0800
categories: reflection
visible: 1
---

## Changing to Hall Shading
To adapt Phong to Hall, we made a series of  new methods, and kept everything self contained.there are several sub methods that keep certain parts isolated for reuse. we also added the necessary terms to the scene descriptions to support diffuse and specular color and the reflection constants. New code startsg on line 68 in Raytracer.cpp

{% highlight c++ %}
glm::vec3 HallDiffuse(const float diffuse, const glm::vec3 diffuseColor, const LightDesc& light,
      glm::vec3 normal, glm::vec3 intersectionPoint) {

    glm::vec3 color = { 0.0, 0.0, 0.0 };


    glm::vec3 toLight = glm::normalize(light.position - intersectionPoint);
    float diffuseTheta = std::max(glm::dot(normal, toLight), 0.0f);

    color.r += std::min(light.color.r * diffuseColor.r * diffuseTheta, 1.0f);
    color.g += std::min(light.color.g * diffuseColor.g * diffuseTheta, 1.0f);
    color.b += std::min(light.color.b * diffuseColor.b * diffuseTheta, 1.0f);

    color *= diffuse;

    return color;
}

glm::vec3 HallSpecular(const float specular, const float shinyness, const glm::vec3 specularColor,
     const LightDesc& light, glm::vec3 normal, glm::vec3 intersectionPoint, const Camera& camera) {

    glm::vec3 color = { 0.0, 0.0, 0.0 };

    glm::vec3 toLight = glm::normalize(light.position - intersectionPoint);
    glm::vec3 toEye = glm::normalize(glm::vec3(camera.mPosition) - intersectionPoint);

    glm::vec3 half = glm::normalize(toLight + toEye);

    float specularTheta = glm::dot(half, normal);

    float specularThetaToTheN = std::pow(specularTheta, shinyness);

    color.r += std::min(light.color.r * specularColor.r * specularThetaToTheN, 1.0f);
    color.g += std::min(light.color.g * specularColor.g * specularThetaToTheN, 1.0f);
    color.b += std::min(light.color.b * specularColor.b * specularThetaToTheN, 1.0f);


    color *= specular;
    return color;
{% endhighlight %}
 These methods are called from within `HallShading`

{% highlight c++ %}
 void RayTracer::HallShading(glm::vec3& pixel, const Material& mat, const glm::vec3& intersectionPoint,
     const glm::vec3& intersectionNorm, const Camera& camera, const std::vector<LightDesc>& lights,
     const float& globalAmbient) {

    //reset pixel to black, working with a "blank" canvas
    pixel = glm::vec3(0.0, 0.0, 0.0);

    //now calculate ambient
    glm::vec3 amb = mat.mAmbient * globalAmbient * mat.mSpecularColor;

    //Calcuate reflection first
    glm::vec3 ref = { 0.0,0.0,0.0 };
    glm::vec3 toPrevious = intersectionPoint - glm::vec3(camera.mPosition);
    glm::vec3 reflectionVec = toPrevious - ((2 * glm::dot(toPrevious, intersectionNorm)) * intersectionNorm);
    glm::vec3 reflectionOrigin = intersectionPoint + (intersectionNorm * 1e-4);


    Intersectable* objectPtr = nullptr;
    glm::vec3 intNormal;
    float intDistance;
    Pixel intPix;

    if (ShootRay(BuildRay(reflectionOrigin, reflectionVec), sceneObjects, objectPtr, intNormal,
         intDistance, intPix)) {

        glm::vec3 intPoint = reflectionOrigin + reflectionVec * intDistance;
        ref = HallReflection(objectPtr->getMaterial(), intPoint, intNormal, reflectionOrigin,
             camera, lights, globalAmbient, amb, 0);
    }
    ref = ref * mat.mReflection * mat.mSpecularColor;

    //finally specular, diffuse
    glm::vec3 dif = { 0.0, 0.0, 0.0 };
    glm::vec3 spec = { 0.0, 0.0, 0.0 };

    for (LightDesc light : lights) {
        Intersectable* objectPtr = nullptr;
        glm::vec3 intNormal;
        float intDistance;
        Pixel intPix;

        // Account for Shadows
        glm::vec3 shadowCheckPoint = intersectionPoint + (intersectionNorm * 1e-4);
        float distToLight = glm::distance(light.position, shadowCheckPoint);
        if (ShootRay(BuildRay(shadowCheckPoint, light.position), sceneObjects, objectPtr, intNormal,
             intDistance, intPix) && intDistance < distToLight) {
            continue;
        }

        dif += HallDiffuse(mat.mDiffuse, mat.mDiffuseColor, light, intersectionNorm, intersectionPoint);
        spec += HallSpecular(mat.mSpecular, mat.mShinyness, mat.mSpecularColor, light, intersectionNorm,
            intersectionPoint, camera);
    }

    pixel = dif + spec + amb + ref;
    //pixel = ref;
}
{% endhighlight %}


## Adding Reflections

Reflection needed to be recursively callable, and it needed to be able to perform full shading for each bounce of the reflected ray. So it has basicaly the same contents as the basic Hall shading method. From line 105 in raytracer.cpp.


{% highlight c++ %}
glm::vec3 RayTracer::HallReflection(const Material& mat, const glm::vec3& intersectionPoint,
     const glm::vec3& intersectionNorm, const glm::vec3& previousIntersection,
     const Camera& camera, const std::vector<LightDesc>& lights, const float& globalAmbient,
     glm::vec3 previousAmbient, int depth) {

    if (depth >= 20) {
        return previousAmbient;
    }

    glm::vec3 pixel = glm::vec3(0.0, 0.0, 0.0);

    // Calculate ambient
    glm::vec3 amb = mat.mAmbient * globalAmbient * mat.mSpecularColor;

    // Calcuate reflection
    glm::vec3 ref = glm::vec3(0.0, 0.0, 0.0);
    glm::vec3 toPrevious = intersectionPoint - previousIntersection;
    glm::vec3 reflectionVec = toPrevious - ((2 * glm::dot(toPrevious, intersectionNorm))
                               * intersectionNorm);
    glm::vec3 reflectionOrigin = intersectionPoint + (intersectionNorm * 1e-4);


    Intersectable* objectPtr = nullptr;
    glm::vec3 intNormal;
    float intDistance;
    Pixel intPix;

    if (ShootRay(BuildRay(reflectionOrigin, reflectionVec), sceneObjects, objectPtr, intNormal,
        intDistance, intPix)) {

        glm::vec3 intPoint = reflectionOrigin + reflectionVec * intDistance;
        if (objectPtr->getMaterial().mShinyness == 0) {
            return objectPtr->getMaterial().mDiffuseColor;
        }

        ref = HallReflection(objectPtr->getMaterial(), intPoint, intNormal, intersectionPoint, camera,
             lights, globalAmbient, amb, depth + 1);


    }

    ref = ref * mat.mReflection * mat.mSpecularColor;
    //finally specular, diffuse

    glm::vec3 dif = { 0.0, 0.0, 0.0 };
    glm::vec3 spec = { 0.0, 0.0, 0.0 };

    for (LightDesc light : lights) {
        Intersectable* objectPtr = nullptr;
        glm::vec3 intNormal;
        float intDistance;
        Pixel intPix;

        // Account for Shadows
        glm::vec3 shadowCheckPoint = intersectionPoint + (intersectionNorm * 1e-4);
        float distToLight = glm::distance(light.position, shadowCheckPoint);
        if (ShootRay(BuildRay(shadowCheckPoint, light.position), sceneObjects, objectPtr, intNormal,
            intDistance, intPix) && intDistance < distToLight) {

            continue;
        }

        dif += HallDiffuse(mat.mDiffuse, mat.mDiffuseColor, light, intersectionNorm, intersectionPoint);
        spec += HallSpecular(mat.mSpecular, mat.mShinyness, mat.mSpecularColor, light, intersectionNorm,
                      intersectionPoint, camera);
    }

    pixel = dif + spec + amb + ref;

    return pixel;
}
{% endhighlight %}


## Images

These images were generated as the reflection system was implemented. I started with a single sphere, and a few meshes to reflect.
![Simple Reflection Scene](/assets/images/HW_6/basic-reflection.png)

Then made sure that the reflections worked on meshes
![Mesh Reflection Scene](/assets/images/HW_6/blobby-reflection.png)

Finally creating a scene with a floor, 3 meshes (dragon, teapot, penny3), and two spheres. The centerpoint of the scene is a reflective sphere that reveals objects that are behind the camera via reflection.
![ComplexReflection Scene](/assets/images/HW_6/complex-reflection.png)

Unfortunately this project step was plauged by difficult to diagnose bugs. There are a few issues with the reflections. For one, in the reflection of the floor on the sphere, there is a square shadow that should not be there. second there is a border around the reflected objects that seems to transmit the color of something behind it, see the purple ring around the central sphere. My gut feeling is that this is most likely a issue with my reflection vectors, and I am simply not accounting for a few edge cases, however by the deadline I was not able to resolve all of these issues. Most notably however, it seems my multi-reflection inst working correctly.