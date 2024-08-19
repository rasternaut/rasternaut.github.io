---
layout: post
title:  "Refractions"
date:   2020-06-08 00:00:00 -0800
tags: [refraction, cs636-raytracer]
visible: 1
---



## Adding Transmission and Refraction Rays

The first step was to add the necessary components to the material object so that we could model the new properties. We needed to add the Transmission constant, as well as the refraction Index per material. We also added a text field that picks up a comment from the json scene files to aid in debugging. The final version of  the `Material` class is:


{% highlight c++ %}
class Material {

public:
    glm::vec3 mSpecularColor;
    glm::vec3 mDiffuseColor;
    glm::vec3 mTransmissionColor;
    float mAmbient = 0;
    float mSpecular = 0;
    float mDiffuse = 0;
    float mReflection = 0;
    float mShinyness = 0;
    float mTransmission = 0;
    float mRefractionIndex = 0;
    std::string comment;
};
{% endhighlight %}

Next we had to modify our hall shading. We did this in several places. The first was to modify our shadow code to move from a binary "on/off" for lights, to one that allowed some light to pass through transmissive objects. This implementation is limited, as we don't account for an ordering of objects where a shadow ray hits a transparent object, and continues on to a further away non-transmissive object. This only accounts for the nearest intersection of the light ray. These changes are on line 223 of `Raytracer.cpp`.

{% highlight c++ %}
for (LightDesc light : settings.lights) {
    Intersectable* objectPtr = nullptr;
    glm::vec3 intNormal;
    float intDistance;
    Pixel intPix;

    // Account for Shadows
    glm::vec3 shadowCheckPoint = intersectionPoint + (intersectionNorm * 1e-4);
    float transmissionVal = 1;
    float distToLight = glm::distance(light.position, shadowCheckPoint);

    if (ShootRay(BuildRay(shadowCheckPoint, light.position), objectPtr, intNormal, intDistance, intPix)
                                                                            && intDistance < distToLight) {
        transmissionVal = objectPtr->getMaterial().mTransmission;

        // if the object isnt transmissive, skip this light. using this val to account for epsilons
        if (transmissionVal < 0.0001) {
            continue;
        }
    }

    //apply shading, as subject to any objects in the way of the light source
    dif += HallDiffuse(mat.mDiffuse, mat.mDiffuseColor, light, intersectionNorm, intersectionPoint)
             * transmissionVal;
    spec += HallSpecular(mat.mSpecular, mat.mShinyness, mat.mSpecularColor, light, intersectionNorm,
        intersectionPoint, settings.camera) * transmissionVal;
}
{% endhighlight %}




Then further down, on line 217 we account for light passing through an object via transmission. Then calculate the next level of hall shading based on where that ray lands. Finally we add the transmissive component to the total value for the pixel.

{% highlight c++ %}
//calcuate transmission
    glm::vec3 tra = { 0.0, 0.0, 0.0 };

    Ray exitRay;
    if (mat.mTransmission > 0 && ComputeInternalReflections(object, intersectionPoint,
                                                viewRay, intersectionNorm, exitRay)) {
        //hall shading
        Intersectable* objPtr;  glm::vec3 intNorm;
        float dist;  Pixel px;

        if (ShootRay(exitRay, objPtr, intNorm, dist, px)) {
            glm::vec3 transmissionPoint = exitRay.mOrigin + exitRay.mNormRayVector * dist;
            HallShading(tra, objPtr, objPtr->getMaterial(), transmissionPoint, intNorm,
                     exitRay.mOrigin, recursionDepth + 1, amb + spec + dif);
        }
    }
    tra = tra * mat.mTransmission * mat.mTransmissionColor;

// sum it all up
pixel = dif + spec + amb + ref + tra;
{% endhighlight %}

Much of the work goes on inside `ComputeInternalReflections`. Admittedly this function is a mess. There was an attempt at a recursive version of this discussed below, however issues occurred. Instead we built this to handle one level of internal reflection if necessary, and that seems to have worked or our scenes.


{% highlight c++ %}
bool ComputeInternalReflections(Intersectable* object, glm::vec3 intersectionPoint,
     glm::vec3 normViewDirection, glm::vec3 intersectionNormal, Ray& exitRay) {

    // "n"s for computing refraction angles
    float inN = 1.0 / object->getMaterial().mRefractionIndex;
    float outN = object->getMaterial().mRefractionIndex / 1.0;

    //calc the refraction ray, in its own scope so we can reuse names

    glm::vec3 exitPoint;
    glm::vec3 exitNormal;

    {
        glm::vec3 transmissionOrg = intersectionPoint - intersectionNormal * 1e-2;
        Ray internalRay;
        internalRay.mOrigin = transmissionOrg;
        internalRay.mNormRayVector = refract(normViewDirection,
                  intersectionNormal, inN);

        float dist;
        glm::vec3 norm;
        Pixel px;
        bool no = object->CheckIntersection(internalRay, dist, norm, px);
        if (no == false) {
            return false;
        }
        exitPoint = transmissionOrg + internalRay.mNormRayVector * dist;
        exitNormal = -(norm);

        exitRay.mOrigin = exitPoint + (norm * 1e-4);
    }

    // so now we have the intersection where our ray exits the object.
    // compute the exit ray we are going to need to handle total
    // internal reflection eventually

    {
        glm::vec3 backVec = glm::normalize(exitPoint - intersectionPoint);
        glm::vec3 ref = refract(backVec, exitNormal, outN);

        glm::bvec3 totalReflectionTest = glm::epsilonEqual(ref, { 0.0f, 0.0f ,0.0f }
                 , glm::epsilon<float>());

        if (totalReflectionTest.x && totalReflectionTest.y & totalReflectionTest.z) {

            float dist;  glm::vec3 norm;
            Pixel px; Ray intRay;

            intRay.mNormRayVector = glm::reflect(backVec, exitNormal);
            intRay.mOrigin = exitPoint;
            object->CheckIntersection(intRay, dist, norm, px);

            exitRay.mOrigin = intRay.mOrigin + intRay.mNormRayVector * dist;
            exitRay.mNormRayVector = glm::refract(-intRay.mNormRayVector, -norm, outN);

            return true;
        }

        exitRay.mNormRayVector = ref;
        }

        exitRay.mNormRayVector = ref;
    }

    return true;
}
{% endhighlight %}


There are two other utility functions written. `clamp` and `refract`. both are called by the above code

{% highlight c++ %}
float clamp(float min, float max, float val) {
    return std::min(max, std::max(min, val));
}

glm::vec3 refract(const glm::vec3& I, const glm::vec3& N, const float& eta)
{

    float cosi = clamp(-1, 1, glm::dot(I, N));
    glm::vec3 n = N;

    if (cosi < 0) {
        cosi = -cosi;
    }
    else {
        n = -N;
    }
    float k = 1 - eta * eta * (1 - cosi * cosi);

    glm::vec3 val = (eta * I) + (eta * cosi - sqrtf(k)) * n;

    if (k < 0) {
        return glm::vec3(0, 0, 0);
    }
    else {
        return val;
    }

}
{% endhighlight %}

The structure and naming of variables in `refract` were inspired by the scratchapixel tutorials on this topic, which I turned to as a resource when I ran into difficulties with the vector math.

## Images
First we demonstrate a sphere
![Simple Transmission Scene](/assets/images/HW_7/basic-refraction.png)

Then a scene where two spheres are visible through a box mesh. You can note a slight distortion of the spheres due to the refractive index of the box being different from the "vacuum" of the rest of the scene.
![Mesh Transmission Scene](/assets/images/HW_7/medium-refraction.png)

finally a more complicated scene where we show that multiple objects are visible through a single transmissible object, even one with reflection. Things to note, the shadows of the teapot and sphere are less prevalent due to the new transmissive shadow handling. The body of the dragon distorts around the edge of the teapot due to the different refractive index and exaggerated geometry. Finally you can see reflections on the orange sphere even through the teapot.
![Complex Transmission Scene](/assets/images/HW_7/complex-refraction.png)


## Work in progress issues

Besides the obvious issues with the spheres, I encountered significant difficulty in dealing with Total Internal Reflection. Initially I tried to create a recursive method to model the bounces of light inside a object, eventually solving for the out ray, unfortunately this approach never fully came together, and I have preserved the implementation here:


{% highlight c++ %}
Ray internalReflection(const glm::vec3& I, const glm::vec3& pt, const glm::vec3& N,
        const float& eta, Intersectable* object, size_t depth) {

    Ray outRay;
    outRay.mOrigin = pt;
    outRay.mNormRayVector = I;

    if (depth > 2) {
        return outRay;
    }

    //check refraction for  total internal reflection
    glm::vec3 ref = refract(I, N, eta);
    glm::vec3 zero = { 0.0f, 0.0f ,0.0f };
    glm::bvec3 totalReflectionTest = glm::epsilonEqual(ref, zero, glm::epsilon<float>());

    if (totalReflectionTest.x && totalReflectionTest.y & totalReflectionTest.z) {
        //if we have total internal reflection, prep the next bounce

        float dist; glm::vec3 norm; Pixel px; Ray intRay;

        intRay.mNormRayVector = glm::reflect(I, N);
        intRay.mOrigin = pt;
        object->CheckIntersection(intRay, dist, norm, px);

        glm::vec3 intPoint = pt + intRay.mNormRayVector * dist;

        // invert norm because its pointing the wrong way
        return internalReflection(intRay.mNormRayVector, intPoint, -norm, eta, object, depth + 1);
    }

    return outRay;
}
{% endhighlight %}

The difficulty with this implementation is best examined through an image it produced. note the "noise" that runs through both transmissive objects.
![Complex Transmission Scene](/assets/images/HW_7/complex-refraction-broken.png)
