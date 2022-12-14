# Source code 
https://gitlab.com/pac85/GameKernel/

# Screenshots and features

<img align="left" src="https://cdn.discordapp.com/attachments/177443107355230209/784192703013453824/Screenshot_20201203_235429.png" width="450" />
<img src="https://cdn.discordapp.com/attachments/177443107355230209/784192707710812190/Screenshot_20201203_235555.png" width="450" />
<img align="left" src="https://cdn.discordapp.com/attachments/177443107355230209/784192707160571944/Screenshot_20201203_235454.png" width="450" />
<img src="https://cdn.discordapp.com/attachments/177443107355230209/784192711796195368/Screenshot_20201203_235630.png" width="450" />

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

<img align="left" src="https://cdn.discordapp.com/attachments/177443107355230209/784197372611395614/Screenshot_20201204_001614.png" width="450" />
<img src="https://cdn.discordapp.com/attachments/177443107355230209/784197373924212776/Screenshot_20201204_001626.png" width="450" />

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

My renderer uses tiled light culling. The screen is divided in 16x16 tiles and
min-max depth is calculated for each using parallel reduction. After that lights
are assigned to tiles.

A compute shader is dispatched to calculate diffuse and specular irradiance from
the buffers of the previous pass, a workgroup runs for each tile.

<img align="left" src="https://cdn.discordapp.com/attachments/177443107355230209/784197381851578378/Screenshot_20201204_001724.png" width="450" />
<img src="https://cdn.discordapp.com/attachments/177443107355230209/784197383558004767/Screenshot_20201204_001737.png" width="450" />

By removing fallof tile culling becomes obvious

<img src="https://github.com/pac85/pac85.github.io/raw/main/images/nofallof.png" width="450" />

(Some details are noticable due to normal maps)

## Third pass

During this pass the scene is rendered again and material parameters (and textures) are read to calculate radiance. During this pass the depth buffer is not cleared and depth test is set to EQUAL to avoid overdraw. Those two buffers are produced:

- One with SSR contribution not accounted for
- One with SSR contribution

<img align="left" src="https://cdn.discordapp.com/attachments/177443107355230209/784197387916279828/Screenshot_20201204_001816.png" width="450" />
<img src="https://cdn.discordapp.com/attachments/177443107355230209/784197389900054528/Screenshot_20201204_001828.png" width="450" />

The first render target is used in the next frame for SSR.
 
## Final pass

At this point tone mapping is applied using the classic Uncharted 2 function.
This pass applies a function to the hdr image to compress it down to LDR and
tries to mimic the response of film. Gamma correction is also applied.

![](https://cdn.discordapp.com/attachments/177443107355230209/784197402675511326/Screenshot_20201204_001909.png)

At this point UI is rendered on top and the final image is produced.

# Audio

My engine also includes a simple spatial audio renderer that I made from
scratch. The actual audio output is handled by a rust library calledrodio. 
I implemented the following features:

- Binaural audio 
Using the overlap save method to perform the convlution between the HRTF and the
audio signal.
- Reverberation using a Schroeder filter
- Inverse square fallof

The audio rendered is made to be simple, modular and extensible.

# First efforts 

I started working on my own engine after being let down by how much "magic"
happened while using engines such as unreal and unity. I started studying opengl
and the basics of 3D computer graphics from various online resources such as
[this](https://web.archive.org/web/20150225192601/http://www.arcsynthesis.org/gltut/)
At some point I started extending the Hello triangle example to load and display
a scene loaded from a .obj file. I then implementare deferred shading and shadow
mapping and got the following results:

<img align="left" src="https://cdn.discordapp.com/attachments/177443107355230209/784204414458003487/Screenshot_20201204_004714.png" width="450" />
<img src="https://cdn.discordapp.com/attachments/177443107355230209/784204414080122920/Screenshot_20201204_004646.png" width="450" />

I attempted to port this engine to vulkan however the code was not well
structured so I decided to start from scratch, this time using Rust.
