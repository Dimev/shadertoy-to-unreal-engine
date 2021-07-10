### Warning: Not done yet!


# shadertoy-to-ue
Guide for converting shadertoy shaders to Unreal Engine materials

This assumes you want to port a 3D shader to materials, that renders something like clouds, or a special shape

### Chapters
 * What are shaders?
 * How (Most) Shadertoy shaders work
 * Shaders in UE
 * Porting shaders
 * Maximum depth
 * Functions

# What are shaders?
To put it simply, shaders are small programs that run on the GPU, and can affect what the final image looks like.
Shadertoy runs a shader for each pixel on the screen (This is known as the fragment shader).

Unreal instead uses Materials, which uses a visual way to make shaders
Under the hood this gets converted to a shader.
Note that unreal (and all other game engines) also have a vertex shader.

The vertex shader moves the vertices that make up a mesh, which allows you to change what the final mesh looks like
This is handy for adding detail, but it's also essential for actual rendering, but we'll skip this in the tutorial, as unreal handles the vertex shader already

### Shader languages
There are several shader languages (programming languages for shaders)
Shadertoy uses GLSL, while unreal allows programming custom shaders in HLSL, as well as the material system (which gets converted to a shader under the hood)

# How (Most) Shadertoy shaders work
Most 3D shaders on shadertoy work via some form of raytracing or raymarching.
This traces a line from where the camera is in the scene through a pixel on the screen, and then for this line, the shader checks if it hits any objects (or other things) and determines the color based on that.

If you want a good intro to raytracing, you can read [Ray tracing in one weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html) by Peter Shirly

If the shader makes use of raytracing or raymarching, there's usually a function called render or similar, that takes in the ray origin (camera position, usually also called ro, ray_start, origin) and the ray direction (camera vector, usually also called rd, dir, or ray_dir).

*note: This function might be hidden inside the mainImage function*

### Example
We'll be using [this atmosphere shader](https://www.shadertoy.com/view/wlBXWK) as an example to port (It's my own atmosphere shader)
It's also available [here, as a SHADERed project](https://github.com/Dimev/atmosphere-shader) (SHADERed is similar to shadertoy, but needs to be installed to work)

For ease of use, I've put the version we'll use here in a file

# Maximum depth
In ue4, use this bit of code to get the correct distance from a pixel to the camera inside a translucent material
```hlsl
float3 Forward = mul(float3(0.00000000,0.00000000,1.00000000), ResolvedView.ViewToTranslatedWorld);
float DeviceZ = LookupDeviceZ(ScreenAlignedPosition(GetScreenPosition(Parameters))).r;

float Depth = DeviceZ * View.InvDeviceZToWorldZTransform[0] + View.InvDeviceZToWorldZTransform[1] + 1.0f / (DeviceZ * View.InvDeviceZToWorldZTransform[2] - (View.InvDeviceZToWorldZTransform[3] + 0.00000001));

return Depth / abs(dot(Forward, Parameters.CameraVector));
```

