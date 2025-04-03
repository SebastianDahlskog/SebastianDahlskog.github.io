---
layout: post
title:  "Clustered Rendering"
categories: jekyll update
author: Sebastian Dahlskog
thumbnail: "/img/clustered.png"
---

During the last semester at The Game Assembly we had to plan and perform an eight-week project of our choosing.  
I chose to implement clustered rendering, 
an optimization technique for reducing the number of lights evaluated per pixel.

I took a lot of inspiration from Emil Persson's [Practical Clustered Shading](https://www.humus.name/Articles/PracticalClusteredShading.pdf)  
As well as David Hu's [https://github.com/DaveH355/clustered-shading](https://github.com/DaveH355/clustered-shading)

## Clustered Rendering Overview

Clustered rendering (aka clustered shading) is about partitioning the view frustum into "clusters" (think of it like AABBs) that are each assigned any lights that intersect their volume. The lighting shader can then look up the only the lights that were assigned to the cluster that contains the pixel being shaded, thereby reducing the amount of considered lights per pixel.

So, every frame, the steps are
1. Assign lights to every cluster their bounding spheres overlap
2. Copy data to GPU
3. Consume data in shader

## Light assignment on CPU!

The bulk of the work performed in clustered rendering is assigning lights to clusters. 
I do this by brute-force testing each light against every cluster in the view frustum (sphere vs AABB). 
In a scene with many lights and thousands of clusters this is quite a performance hit.

```cpp
for (unsigned clusterIndex = 0; clusterIndex < aClusterGroup->Size(); ++clusterIndex)
{
    short pointLightCount = 0;
    short spotLightCount = 0;

    ClusterData& cluster = aClusterGroup->GetClusterData(clusterIndex);
    cluster.lightIndexOffset = lightIndexOffset;

    const AABBF& cellBoundingBox = aClusterGroup->GetClusterBoundingBox(clusterIndex);
    for (unsigned lightIndex = 0; lightIndex < aScene->lightData.pointLights.size(); ++lightIndex)
    {
        const PointLight& pLight = aScene->lightData.pointLights[lightIndex];
        if (!IntersectionFrustumSphere(aScene->clusterFrustum, SphereF(pLight.GetPosition(), pLight.GetRange())))
            continue;
        Vec4f pos(pLight.GetPosition(), 1.f);

        pos = pos * aScene->cameraData.views[aScene->clusterCamera]; // Convert to view space
        SphereF rangeSphere(&pos.x, pLight.GetRange());
        if (IntersectionAABBSphere(cellBoundingBox, rangeSphere))
        {
            ++lightIndexOffset;
            ++pointLightCount;
            aClusterGroup->AddLightIndex(aThreadIndex, lightIndex);
        }
    }

    // Omitting spotlights for the sake of brevity!!!

    cluster.lightCount = CombineLightCounts(pointLightCount, spotLightCount);
}
```

My implementation performs light assignment on the CPU, which honestly isn't a very good idea in regards to performance. 
Multi threading was a must if I wanted to run it in realtime with hundreds of lights.
fortunately implementing multithreaded light assignment was extremely simple since all 
elements can be computed independantly.

### Wait, how is the view frustum divided???
Width and height are divided in a uniform grid. However, depth is divided exponentially.  
For any depth-slice $slice$ the depth where that slice starts is given by

$$Z=Near_z(\frac{Far_z}{Near_z})\frac{slice}{numslices}$$

## Copying cluster data to the GPU

After assigning lights to clusters, the data must be made available to the GPU.  
I store

## Pixel Shader light-lookup

When shading the scene, each pixel looks up which cluster it belongs to:

```hlsl

```

## Result

Unfortunately I was unable to completely finish the project. In many ways

## Takeaways

### CPU Light culling = SLOW 

Should have made light assignment run as a compute shader for less performance impact. 

As far as I'm aware only reason to choose CPU light assignment over the compute shader alternative would be if you need to support older hardware that doesn't implement a graphics API that supports compute shaders. Obviously my student-project rendering system 
