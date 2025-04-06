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

Special thanks goes out to Adam Rohdin for help with shader debugging.

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
For any depth-slice the depth where that slice starts is given by

$$Z=Near_z(\frac{Far_z}{Near_z})^{\frac{slice}{numslices}}$$

## Copying cluster data to the GPU

After assigning lights to clusters, the data must be made available to the GPU. 

I store the cluster data as a texture in the R32G32_UINT format storing
* its offset into a "light-index buffer" in the R-channel
* its point and spotlight counts in the G-channel (first 16 bits for pointlights, last 16 for spotlights)

### Light Indices in a buffer
For every intersection of a light and a cluster, I add that light's index to a structured buffer of uints. Every cluster will store its offset into this buffer so that it can find its lights.

## Pixel Shader light-lookup

When shading the scene, each pixel looks up which cluster it belongs to:

```hlsl
float viewZ = GetViewpos(views[aCamera], aWorldPos).z;
int zSlice = int((log(abs(viewZ) / near) * cLSRes.z) / log(far / near));
float2 uv = WorldToUV(aWorldPos, aCamera);
int4 clusterCoordinate = int4((uv * cLSRes.xy), zSlice, 0);
```

The shader then gets the light buffer offset and the light counts from the cluster by loading from the texture using that coordinate.

```hlsl
uint2 lightInfo = ClusterTexture.Load(float4(clusterCoord)).rg;
uint lightOffset = lightInfo.r;
uint lightCounts = lightInfo.g; // 16 bits for pointlights, and 16 bits for spotlights

uint pointlightCount = lightCounts & 0xFFFF;
uint spotlightCount = lightCounts >> 16;
```

Then it loops through the light indices
and calculates as usual:

```hlsl
float3 pointAndSpotLightSum = 0;
for (int i = 0; i < pointlightCount; ++i)
{
    uint index = lightIndexBuffer.Load(lightOffset + i).clusterLightIndex;
    PointLightData pl = PointLights[index];
    pointAndSpotLightSum += EvaluatePointLight(diffuseColor, specularColor, normal,
        roughness, pl.color_and_intensity.rgb, pl.color_and_intensity.a, pl.position_and_range.a,
        pl.position_and_range.xyz, viewDir, worldPosition.xyz);
}

// Repeat for spotlights
```

## Result

<iframe allowfullscreen src="https://youtube.com/embed/p_ClWIrqhLI">
</iframe>

Unfortunately I was unable to completely finish the project on time. It has a bug where sometimes the light assignment doesn't correctly determine wether a light should be assigned to a cluster, excluding that light from somewhere it should have been included, leading to flickering. Nonetheless I learnt a lot from the project and I plan to keep working on it.

## Takeaways

### CPU Light culling = SLOW 

Pretty far into the project I realized that I **REALLY** should have made light assignment run as a compute shader for less performance impact. 

As far as I'm aware only reason to choose CPU light assignment over the compute shader alternative would be if you need to support older hardware that doesn't implement a graphics API that supports compute shaders.
