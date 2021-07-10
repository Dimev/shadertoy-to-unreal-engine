### Warning: Not done yet!


# shadertoy-to-ue
Guide for converting shadertoy shaders to Unreal Engine materials

This assumes you want to port a 3D shader to materials, that renders something like clouds, an atmosphere, or other fog-like effects

Knowledge required:
 - you can work with ue materials
 - you have a general idea of how the thing you want to port works

### Chapters
 * What are shaders?
 * How (Most) Shadertoy shaders work
 * Shaders in UE
 * Porting shaders
 * Maximum depth 
 * Functions *How to use functions in node*
 * Distance field shadows
 * When to port *(Caveats of porting + translating to nodes)*

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

However the shader works, all code that does the actual rendering is called in the mainImage function in shadertoy, which is then run for every pixel

*TODO SHADERTOY INPUTS*

*note: This function might be hidden inside the mainImage function*

### Example
We'll be using [this atmosphere shader](https://www.shadertoy.com/view/wlBXWK) as an example to port (It's my own atmosphere shader)
It's also available [here, as a SHADERed project](https://github.com/Dimev/atmosphere-shader) (SHADERed is similar to shadertoy, but needs to be installed to work)

For ease of use, I've put the version we'll use here in the file [atmosphere.glsl](https://github.com/Dimev/shadertoy-to-unreal-engine/blob/main/atmosphere.glsl)

This example was specifically made to be easy to port to unreal
 - the main functionality is in one function
 - the input and output are made to work easily with unreal inputs and outputs

# Shaders in UE
There's a few ways to do shaders in ue
 - add them via USF/USH files
 - translate them to material nodes
 - the custom expression in materials

The easiest option of these is the custom expression node
This has a few parameters when you select it and look at the details pane
 - Code: the code for this node
 - Output type: which is what the node outputs 
 - Description: what the node will be named in the node graph
 - Inputs: this is a list of inputs the node takes in, and you can access from the code inside the node

Under the hood, custom nodes get converted into a function, meaning you sadly can't define your own functions inside the custom node without some workarounds (in chapter Functions, which also goes about )

If you want to know what the final shader is unreal generates for your material, you can see it under window -> view shader source *CHECK IF THIS IS CORRECT*

Now, we can write arbitrary code into the Code field for the node
*right?*

We can, but it's the worst text editor available.
I suggest instead making a seperate file somewhere else, with the .hlsl extention, and edit it with your favorite text editor (I use vscode, and it seems to do automatic syntax highlighting for this file type, which helps you see what's going on)

Then, once you're done, you can copy-paste the code from there into the Code field of the custom node.

# Porting
Now we can start porting the shader
But, what kind of material do we need?

Post process materials are the most versatile, as they run after most of the rendering is done, and have direct control of the color of a pixel
However, we can also use a translucent materials with the AlphaComposite blend mode.
But, what does AlphaComposite do?
It works like this: `pixel_color = material_color.xyz + pixel_color * material_color.w` under the hood, meaning we have control of how much we add to the current color, and how much we keep of the original.

If you are rendering some form of geometry from the shader that can't be represented with a mesh, it's possible to use a shader with alpha scissors and pixel depth offset *CHECK IF THE NAMING HERE IS CORRECT*

But, what mesh do we use to render this?
An inverted cube works good enough for this. It needs to be inverted (meaning faces point inward) to keep working when the camera moves inside

We can also disable depth testing, which will render our material over every object in the scene.
This allows us to have the shader determine how interaction with the scene work, which most shadertoy shaders need

The shader we're going to port is an atmosphere effect, which works well as an AlphaComposite effect, as it adds a color on top of the scene, and partly masks things behind it

### The shader
Now, let's look around the example in [atmosphere.glsl](https://github.com/Dimev/shadertoy-to-unreal-engine/blob/main/atmosphere.glsl)

In mainImage, we can find a few variables, `camera_vector`, `camera_position`, and `light direction`
Camera vector is the direction from the camera to the pixel being rendered
Luckily, ue has this built in: ![image](https://user-images.githubusercontent.com/49782454/125177734-8aa52480-e1de-11eb-888d-cac63c036628.png)
There's also camera position, which is the camera position relative to the atmosphere
This isn't directly available in ue, but we have the camera position in world space: ![image](https://user-images.githubusercontent.com/49782454/125177756-bf18e080-e1de-11eb-8f16-eab80408253d.png)
and the object position ![image](https://user-images.githubusercontent.com/49782454/125177769-dbb51880-e1de-11eb-9c20-feb8c0c3e562.png)
If we subtract the object position from the camera position, we get the relative position of the camera

Then there's light direction
There's no equivalent of this in unreal sadly, so we have a few options
 - hardcode it in the shader
 - pass it to the shader as a parameter
 - use some other node instead of this

Depending on what it does, you can either hardcode it or pass it as a parameter
However, this is the light direction of the atmosphere. It would be nice if we could easily change that, so how about using the object orientation? ![image](https://user-images.githubusercontent.com/49782454/125177821-3e0e1900-e1df-11eb-9db2-3db33c30a73d.png)

That way we can easily change the light direction by rotating our mesh!

# Maximum depth
In ue4, use this bit of code to get the correct distance from a pixel to the camera inside a translucent material
```hlsl
float3 Forward = mul(float3(0.00000000,0.00000000,1.00000000), ResolvedView.ViewToTranslatedWorld);
float DeviceZ = LookupDeviceZ(ScreenAlignedPosition(GetScreenPosition(Parameters))).r;

float Depth = DeviceZ * View.InvDeviceZToWorldZTransform[0] + View.InvDeviceZToWorldZTransform[1] + 1.0f / (DeviceZ * View.InvDeviceZToWorldZTransform[2] - (View.InvDeviceZToWorldZTransform[3] + 0.00000001));

return Depth / abs(dot(Forward, Parameters.CameraVector));
```

