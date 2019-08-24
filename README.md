# Rewriting Godot's light mapper

Joan Fons Sanchez (@JFonS)

Mentored by: Juan Linietsky (@reduz) and Bastiaan Olij (@mux213)

Project source: https://github.com/JFonS/godot/tree/lightmapper

## Project description
Light maps are a really easy way to improve performance on staticlly lit scenes. Instead of computing the amount of light that reaches a certain surface every frame for every light source, we precompute all this information and store it in a single texture. This means having lots of lights no longer creates a performance hit on the rendering pipeline, since all we need to do is sample a single texture and we get the amount of light coming from all light sources.

Godot's 3.1 light mapper is based on the same approach used in [global illumination](https://docs.godotengine.org/en/3.1/tutorials/3d/gi_probes.html). That means the whole scene is subdivided in a regular grid of voxels and, for each of these voxels, we compute the amount of light reaching it. This allows us to have some great results for computing real time illumination, but the discretization of the scene has various downsides (i.e. some walls may be thinner than the size of a voxel, therefore they can leak light through them). 

The main goal of my GSoC project is to completely rewrite the light mapper in Godot and, instead of a voxel approach, use ray tracing to compute the scene lighting instead.  This will hopefully give better looking light maps and will reduce the amount of artifacts such as self occlusions or light leaks.

## What has been done

### UV coordinate caching
During the first weeks of coding I added caching to UV2 generation. The process of generating light map texture coordinates takes a while, and it was being triggered on every scene reimport. By adding a simple cache to it, we made it so that light map texture coordinates are only computed when there's an actual change to the geometry of the mesh.


### Direct light mapping 
With that simple task out of the way my main focus went to getting the direct illumiation pass done. That involved generating the actual light map texture and, for every light inside the BakedLightmap node, compute the amount of light reaching every texel.


### Embree integration
I integrated Embree into the engine's build system and also added a simple ray tracing API for it. The new API allows for defining a set of meshes and perform ray intersection tests amongst them. Currently it uses Embree as the only backend, but should make the process of adding a different backends (GPU based maybe?) easier if we ever decide to go down that route.


### Dirct light occlusion
Using the new raytracing API, adding occlusion for direct lighting was relatively straightforward. I just needed to cast a ray from every spot on the surface towards the light source and check if any geometry was in the way.

![top: direct light without occlusion, bottom: direct light with occlusion testing](https://github.com/JFonS/gsoc2019/raw/master/img/occlusion-diff.png)
(top: direct light without occlusion, bottom: direct light with occlusion testing)

### Basic indirect lighting
Completing that last step meant that we had some really valuable information: which parts of the scene were receiving direct illumination from lights and which parts were in shadow. But this information alone is not enough, light bounces on objects and tends to fill every possible corner of a scene, with varying intensity, of course. So it was time to add indirect lighting to the mix. Using the new ray tracing API we can randomly throw rays from each surface position and, depending on the surface they hit, get the average amount of light reaching that spot. Here we can see the difference between having only direct lighting and having direct and indirect lighting combined:

![top: no indirect lighting, bottom: 2 bounces of indrect light](https://github.com/JFonS/gsoc2019/raw/master/img/indirect-diff.png)
(top: no indirect lighting, bottom: 2 bounces of indrect light)

Finally, I can show the current light mapper results in the Sponza demo:

![This image uses real-time direct lighting (regular Godot light nodes) and a light map for the indirect light bounces.](https://github.com/JFonS/gsoc2019/raw/master/img/sponza.png)
This image uses real-time direct lighting (regular Godot light nodes) and a light map for the indirect light bounces.

## What's left to do
Sadly, the new light mapper is not ina mergeable state yet. The basic structure is quite settled, but there are still lots of things to work on before it can be merged into Godot. There are three aspects that need to be improved: performance, usability and results. 

The performance improvements will likely involve adding muti-thread support and some minor improvements here and there. In order to improve user experience, the previous progress bar display needs to be restored (it was lost during the rewrite). Finally, the light map results should be improved by applying some artifact reduction techniques and optionally an AI based denoiser.

