---
layout: post
title:  "Acceleration Techniques"
date:   2020-05-10 00:00:00 -0800
tags: [ray-tracing, cs636-raytracer, optimization]
visible: 1
---
## Building the Bounding Volume Hierarchy

I recursively build the BVH tree inside of the Mesh object, BVH nodes are defined as:
{% highlight c++ %}
struct BVHNode {
    BoundingBox bbox;
    std::vector<Face> faces;
    std::unique_ptr<BVHNode> left;
    std::unique_ptr<BVHNode> right;
};
{% endhighlight %}

The strategy is to sort a collection of faces by their centroid (alternating sorts on the x, y, and z axis as we recurse down), and then divide the sorted collection in half. One half is passed down to the current node's `right` child, and the other half to the `left`. We then calculate the bounding box of the children nodes and recurse down to the left and the right. The recursion is controlled by two tunable parameters `triangleThreshold` - the maxumum number of triangles we want in a node, and `maxDepth` - the maximum number of times we can recurse down and create new nodes, this overrides triangleThreshold to prevent a runaway recursion scenario. These are  are set on a per mesh basis in the scene description Hypothetically allowing us to tune each mesh [for example](https://gitlab.com/TaylorEllington/cs636-advanced-rendering-techniques/-/blob/2796b0107b6372da89e95dedb70bfd95bec000e5/assets/models-and-spheres.json). There are two functions controlling this recursive creation behavior:

{% highlight c++ %}
BuildBVH(size_t triangleThreshold, size_t maxDepth)
RecursiveBuildBVH(size_t triangleThreshold, size_t maxDepth, size_t currentDepth, BVHNode* nodePtr)
{% endhighlight %}

The first, has the added responsibility of calculating the entire mesh's bounding box, it also sets up the initial depth counter, this can be found in `mesh.cpp` on line 169.

{% highlight c++ %}
void Mesh::BuildBVH(size_t triangleThreshold, size_t maxDepth) {

    std::sort(faces.begin(), faces.end(), CentroidX());
    size_t faceDiv = faces.size() / 2;
    headNode.bbox = BoundingBox(faces);

    //distribute the faces among sub nodes
    headNode.left = std::unique_ptr<BVHNode>(new BVHNode);
    headNode.left->faces = std::vector<Face>(faces.begin(), faces.begin() + faceDiv);
    headNode.left->bbox = BoundingBox(headNode.left->faces);

    headNode.right = std::unique_ptr<BVHNode>(new BVHNode);
    headNode.right->faces = std::vector<Face>(faces.begin() + faceDiv, faces.end());
    headNode.right->bbox = BoundingBox(headNode.right->faces);

    RecursiveBuildBVH(triangleThreshold, maxDepth, 1, headNode.left.get());
    RecursiveBuildBVH(triangleThreshold, maxDepth, 1, headNode.right.get());
}

void Mesh::RecursiveBuildBVH(size_t triangleThreshold, size_t maxDepth, size_t currentDepth, BVHNode* nodePtr){
    if (currentDepth > maxDepth || nodePtr->faces.size() < triangleThreshold) {
        return;
    }

    size_t sortIndex = currentDepth % 3;

    switch (sortIndex)
    {
    case 0:
        std::sort(nodePtr->faces.begin(), nodePtr->faces.end(), CentroidX());
        break;
    case 1:
        std::sort(nodePtr->faces.begin(), nodePtr->faces.end(), CentroidY());
        break;
    case 2:
        std::sort(nodePtr->faces.begin(), nodePtr->faces.end(), CentroidZ());
        break;
    default:
        std::cout << "Mesh - Recursive Build BVH - This should never happen" << std::endl;
        break;
    }

    size_t faceDiv = nodePtr->faces.size() / 2;

    nodePtr->left = std::unique_ptr<BVHNode>(new BVHNode);
    nodePtr->left->faces = std::vector<Face>(nodePtr->faces.begin(), nodePtr->faces.begin() + faceDiv);
    nodePtr->left->bbox = BoundingBox(nodePtr->left->faces);

    nodePtr->right = std::unique_ptr<BVHNode>(new BVHNode);
    nodePtr->right->faces = std::vector<Face>(nodePtr->faces.begin() + faceDiv, nodePtr->faces.end());
    nodePtr->right->bbox = BoundingBox(nodePtr->right->faces);

    RecursiveBuildBVH(triangleThreshold, maxDepth, currentDepth + 1, nodePtr->right.get());
    RecursiveBuildBVH(triangleThreshold, maxDepth, currentDepth + 1, nodePtr->left.get());
}
{% endhighlight %}

## Intersecting the BVH

The intersection is structured similarly with a recursive tree traversal, checking the ray against smaller and smaller bounding boxes until a node with no children is reached, then we run a ray intersect against the faces in that node and return up the results of the intersection. This code is on line 225 of `mesh.h`

{% highlight c++ %}
bool Mesh::CheckIntersectionBVHRec(const Ray& ray, float& distance, glm::vec3& normAtIntersection, Pixel& pix,
     BVHNode * node) {

    // sanity null check
    if (node == nullptr) {
        return false;
    }

    //does the ray intersect this node or its children?
    if (!node->bbox.CheckIntersection(ray, distance)){
        return false;
    }

    // if there are no child nodes, this is a leaf, run intersection against
    // the faces
    if (node->left == nullptr && node->right == nullptr) {
        return IntersectCollectionOfFaces(ray, distance, normAtIntersection, pix, node->faces);
    }

    if (node->left == nullptr || node->right == nullptr) {
        std::cout << "Mesh - CheckIntersectionBVHRec - This should never happen" << std::endl;
    }

    //if there are child nodes, recurse down the tree, "return" smallest distance
    float leftDist = std::numeric_limits<float>::max();
    float rightDist = std::numeric_limits<float>::max();

    glm::vec3 leftNorm;
    glm::vec3 rightNorm;

    Pixel leftPix;
    Pixel rightPix;

    bool intersectLeft = CheckIntersectionBVHRec(ray, leftDist, leftNorm, leftPix, node->left.get());
    bool intersectRight = CheckIntersectionBVHRec(ray, rightDist, rightNorm, rightPix, node->right.get());

    // This is obnoxiously verbose, but when I tried to condense everything down into clever little blocks
    // of code I ended up with false negatives, or inverted geometry where the farther pixel was reported
    // over the nearer one. Ill trade elegance for readability and correctness any day.
    if (intersectRight && intersectLeft) {
        if (rightDist < leftDist) {
            distance = rightDist;
            normAtIntersection = rightNorm;
            pix = rightPix;;
        }
        else {
            distance = leftDist;
            normAtIntersection = leftNorm;
            pix = leftPix;
        }
        return true;
    }

    if (intersectRight ) {
        distance = rightDist;
        normAtIntersection = rightNorm;
        pix = rightPix;;
        return true;
    }

    if (intersectLeft ) {
        distance = leftDist;
        normAtIntersection = leftNorm;
        pix = leftPix;
        return true;
    }

    return false;
}
{% endhighlight %}

## Images and The Speedup From Optimization
Remember this scene from assignment 2?

![Super-sampled spheres and meshes before optimization - HW3](/assets/images/HW_3/before-ss-misc-models-and-spheres.png)

And recall its over 8 minute render time? Here is its log, just in case.

{% highlight text %}
Screen - Setting up image as 512 by 512
Sphere - Set up sphere at: [0, 0, -20] with radius: 15
Mesh - Loaded mesh: assets/penny3.smf with 3675 verts and 7344 faces
Mesh - Loaded mesh: assets/teapot.smf with 3644 verts and 6320 faces
Mesh - Loaded mesh: assets/bunny-1000.smf with 502 verts and 1000 faces
Sphere - Set up sphere at: [-2, 5, 3] with radius: 0.5
Sphere - Set up sphere at: [-1, 5, 3] with radius: 0.5
Sphere - Set up sphere at: [0, 5, 3] with radius: 0.5
Sphere - Set up sphere at: [1, 5, 3] with radius: 0.5
Sphere - Set up sphere at: [2, 5, 3] with radius: 0.5
Raytracer - Performing inital setup with:
        Camera at vec4(0.000000, 0.000000, 20.000000, 1.000000)
        Camera Space - X: vec3(1.000000, -0.000000, 0.000000)
        Camera Space - Y: vec3(0.000000, 1.000000, 0.000000)
        Camera Space - Z: vec3(0.000000, 0.000000, -1.000000)

        Dist to view plane 1

        View angle (DEG) 56

        View Plane center at: vec3(0.000000, 0.000000, 19.000000)
        View Plane upper left at: vec3(-0.531709, 0.531709, 19.000000)

Raytracer - Supersampling enabled, computations will be done on an 1024 x 1024image
Raytracer - Starting raytrace of scene..........DONE
Raytracer - Full Screen color norm pass
Raytracer - Supersampling image down to 512 x 512
Raytracer - Tracing took           513715ms
Raytracer - image proccessing took 17ms
Raytracer - Total time: 513732ms
Screen - Writing 262144 pixels to 512 by 512 file: ss-misc-models-and-spheres.png
{% endhighlight %}

That 513732m time is really unwieldy, especially as we add more steps that will be casting rays for shadows, reflection, and other features. After implementing bounding volumes (boxes) and a BVH optimization approach we now have the same image

![Super-sampled spheres and meshes after optimization - HW3](/assets/images/HW_3/after-ss-misc-models-and-spheres.png)

but with a much smaller render time! Its down to 394ms! (note that the log now also prints out the two opposite verts that define a mesh's bounding box)

{% highlight text %}
Screen - Setting up image as 512 by 512
Sphere - Set up sphere at: [0, 0, -20] with radius: 15
Mesh - Loaded mesh: assets/penny3.smf with 3675 verts and 7344 faces
         bounded at min [-5.12625, -2.2605, -1.54025] max [-1.07975 1.7915 1.75375]
Mesh - Loaded mesh: assets/teapot.smf with 3644 verts and 6320 faces
         bounded at min [-2.50771, -5.11259, -2] max [2.04652 0.176809 2]
Mesh - Loaded mesh: assets/bunny-1000.smf with 502 verts and 1000 faces
         bounded at min [2.53791, -1.48794, -0.968468] max [5.5162 0.871868 0.968468]
Sphere - Set up sphere at: [-2, 5, 3] with radius: 0.5
Sphere - Set up sphere at: [-1, 5, 3] with radius: 0.5
Sphere - Set up sphere at: [0, 5, 3] with radius: 0.5
Sphere - Set up sphere at: [1, 5, 3] with radius: 0.5
Sphere - Set up sphere at: [2, 5, 3] with radius: 0.5
Raytracer - Performing inital setup with:
        Camera at vec4(0.000000, 0.000000, 20.000000, 1.000000)
        Camera Space - X: vec3(1.000000, -0.000000, 0.000000)
        Camera Space - Y: vec3(0.000000, 1.000000, 0.000000)
        Camera Space - Z: vec3(0.000000, 0.000000, -1.000000)

        Dist to view plane 1

        View angle (DEG) 56

        View Plane center at: vec3(0.000000, 0.000000, 19.000000)
        View Plane upper left at: vec3(-0.531709, 0.531709, 19.000000)

Raytracer - Supersampling enabled, computations will be done on an 1024 x 1024image
Raytracer - Starting raytrace of scene..........DONE
Raytracer - Full Screen color norm pass
Raytracer - Supersampling image down to 512 x 512
Raytracer - Tracing took           377ms
Raytracer - image proccessing took 17ms
Raytracer - Total time: 394ms
Screen - Writing 262144 pixels to 512 by 512 file: ss-misc-models-and-spheres.png
{% endhighlight %}

That improves the run time by over 1000%, so thats a good thing in my book.

Lets look at a scene that had fewer triangles. While we don't need to replicate the build logs for a second scene, we can discuss the other sphere and mesh scene from the previous homework

![Super-sampled spheres and meshes before optimization - HW3](/assets/images/HW_3/before-ss-solar-spheres-and-supertoroid.png)

In HW2, before the optimizations, this scene took 13240ms. Now with the BVH in play we can cut that down to 479ms. Just to verify, heres the scene rendered with the optimizations

![Super-sampled spheres and meshes after optimization - HW3](/assets/images/HW_3/after-ss-solar-spheres-and-supertoroid.png)

Lastly we can look at a single mesh scene. Since our BVH changes only impacts mesh objects we will see the biggest changes here.  Two lights, one mesh of 50761 faces.

![Super-sampled dragon isolation scene](/assets/images/HW_3/before-ss-dragon.png)

Heres the log:
{% highlight text %}
Screen - Setting up image as 512 by 512
Mesh - Loaded mesh: assets/dragon.smf with 25418 verts and 50761 faces
Raytracer - Performing inital setup with:
        Camera at vec4(0.000000, 0.000000, 20.000000, 1.000000)
        Camera Space - X: vec3(1.000000, -0.000000, 0.000000)
        Camera Space - Y: vec3(0.000000, 1.000000, 0.000000)
        Camera Space - Z: vec3(0.000000, 0.000000, -1.000000)

        Dist to view plane 1

        View angle (DEG) 56

        View Plane center at: vec3(0.000000, 0.000000, 19.000000)
        View Plane upper left at: vec3(-0.531709, 0.531709, 19.000000)

Raytracer - Supersampling enabled, computations will be done on an 1024 x 1024image
Raytracer - Starting raytrace of scene..........DONE
Raytracer - Full Screen color norm pass
Raytracer - Supersampling image down to 512 x 512
Raytracer - Tracing took           26042355ms
Raytracer - image proccessing took 16ms
Raytracer - Total time: 26042371ms
Screen - Writing 262144 pixels to 512 by 512 file: ss-dragon.png
{% endhighlight %}


And now after the improvements we get the same scene

![Super-sampled dragon isolation scene](/assets/images/HW_3/after-ss-dragon.png)

The log shows a significant speed up.

{% highlight text %}
Screen - Setting up image as 512 by 512
Mesh - Loaded mesh: assets/dragon.smf with 25418 verts and 50761 faces
         bounded at min [-11.2707, -5, -11.2707] max [11.2707 11 11.2707]
Raytracer - Performing inital setup with:
        Camera at vec4(0.000000, 0.000000, 20.000000, 1.000000)
        Camera Space - X: vec3(1.000000, -0.000000, 0.000000)
        Camera Space - Y: vec3(0.000000, 1.000000, 0.000000)
        Camera Space - Z: vec3(0.000000, 0.000000, -1.000000)

        Dist to view plane 1

        View angle (DEG) 56

        View Plane center at: vec3(0.000000, 0.000000, 19.000000)
        View Plane upper left at: vec3(-0.531709, 0.531709, 19.000000)

Raytracer - Supersampling enabled, computations will be done on an 1024 x 1024image
Raytracer - Starting raytrace of scene..........DONE
Raytracer - Full Screen color norm pass
Raytracer - Supersampling image down to 512 x 512
Raytracer - Tracing took           614ms
Raytracer - image proccessing took 16ms
Raytracer - Total time: 630ms
Screen - Writing 262144 pixels to 512 by 512 file: ss-dragon.png
{% endhighlight %}


It is apparent that the speedup is directly related to the number of ray-triangle intersections we can avoid, the bounding boxes around progressively smaller clusters of faces enable us to fully skip entire chunks of the model.

## Work in Progress Observations

The first thing I did was to wrap the polymesh objects in bounding boxes determined by the extreme x, y, and z values of the mesh's individual verticies. This saw a massive improvement in render time, On the scene shown above, bounding boxes alone shifted total render time from 513,732ms to  28,889ms, thats an order of magnitude! This was on a scene that had several non-polymesh objects and near 15,000 total triangles. Since this bounding box change only impacts polymeshes and we still have to test one box per polymesh per pixel and n many spheres per pixel, the benefit of this speedup will scale with total scene complexity, ratio of spheres to triangles, and total number of triangles.

The next step was a clean up, I had tons of places where I passed around `glm::vec3` pairs of `origin` and `normRayVector`, so I collapsed these into a `Ray` class. I did something similar with the Phong shading values, packing these into a `Material` class.

Once this was done I added the BVH node and the recursive methods to generate and traverse them as listed above. Once this was working I was able to observe the exceptional speed up, even beyond what bounding boxes by themselves could do.



## Discussion of BVH vs Octree
for my own future reference: https://computergraphics.stackexchange.com/questions/7828/difference-between-bvh-and-octree-k-d-trees