# Source code 
https://gitlab.com/pac85/GameKernel/

 # Screenshots and features

![](https://cdn.discordapp.com/attachments/177443107355230209/784192703013453824/Screenshot_20201203_235429.png)

![](https://cdn.discordapp.com/attachments/177443107355230209/784192707710812190/Screenshot_20201203_235555.png)

![](https://cdn.discordapp.com/attachments/177443107355230209/784192707160571944/Screenshot_20201203_235454.png)

![](https://cdn.discordapp.com/attachments/177443107355230209/784192711796195368/Screenshot_20201203_235630.png)

Highlights:

- screen space reflections
- PBR rendering (cook torrance)
- HDR rendering and tone mapping
- shadow mapping 
    - cascaded shadow mapping
    - variable size penumbra 

Screenshots that show the sky contain some artifacts due to SSR raymarching
reading uninitialized data (buffers are not cleared) 

## Technical details 

The engine uses vulkan, has an ECS and is capable of loading scenes either from
a custom format (for which I've written a crude blender exporter plugin) or
gltf.

The renderer is a "deferred lighting" renderer consisting of the following steps:

## First pass

The scene is drawn and depth, normals and motion vectors are written to the
framebuffer

![](https://cdn.discordapp.com/attachments/177443107355230209/784197372611395614/Screenshot_20201204_001614.png)

![](https://cdn.discordapp.com/attachments/177443107355230209/784197373924212776/Screenshot_20201204_001626.png)

Normal maps are applied during this pass.

##  SSR

A Computer shader is dispatched that reads color from the previous frame, 
motion vectors, depth and normals to produce an approximation of reflections.
reflections are computed at half resolution.

![](https://cdn.discordapp.com/attachments/177443107355230209/784197394996396032/Screenshot_20201204_001851.png) 

## Shadow pass

The scenes is rendered multiple times from the lights point of view into an 8k x 8k atlas. The viewport is set appropriately for each light.

![](https://cdn.discordapp.com/attachments/177443107355230209/784197378180644904/Screenshot_20201204_001703.png)

(This image pnly shows part of the buffer as most of it is empty)

## Lighting

A compute shader is dispatched to calculate diffuse and specular irradiance from
the buffers of the previous frame.

![](https://cdn.discordapp.com/attachments/177443107355230209/784197381851578378/Screenshot_20201204_001724.png)

![](https://cdn.discordapp.com/attachments/177443107355230209/784197383558004767/Screenshot_20201204_001737.png)

(Some details are noticable due to normal maps)

## Third pass

During this pass the scene is rendered again and material parameters (and textures) are read to calculate radiance. During this pass the depth buffer is not cleared and depth test is set to EQUAL to avoid overdraw. Those two buffers are produced:

- One with SSR contribution not accounted for: ![](https://cdn.discordapp.com/attachments/177443107355230209/784197387916279828/Screenshot_20201204_001816.png)
- One with SSR contribution:
![](https://cdn.discordapp.com/attachments/177443107355230209/784197389900054528/Screenshot_20201204_001828.png)

The first render target is used in the next frame for SSR.
 
## Final pass

At this point tone mapping is applied using the classic Uncharted 2 function.
This pass applies a function to the hdr image to compress it down to LDR and
tries to mimic the response of film. Gamma correction is also applied.

![](https://cdn.discordapp.com/attachments/177443107355230209/784197402675511326/Screenshot_20201204_001909.png)

At this point UI is rendered on top and the final image is produced.

# First efforts 

I started working on my own engine after being let down by how much "magic"
happened while using engines such as unreal and unity. I started studying opengl
and the basics of 3D computer graphics from various online resources such as
[this](https://web.archive.org/web/20150225192601/http://www.arcsynthesis.org/gltut/)
At some point I started extending the Hello triangle example to load and display
a scene loaded from a .obj file. I then implementare deferred shading and shadow
mapping and got the following results:


![](https://cdn.discordapp.com/attachments/177443107355230209/784204414458003487/Screenshot_20201204_004714.png)

![](https://cdn.discordapp.com/attachments/177443107355230209/784204414080122920/Screenshot_20201204_004646.png)

I attempted to port this engine to vulkan however the code was not well
structured so I decided to start from scratch, this time using Rust.
