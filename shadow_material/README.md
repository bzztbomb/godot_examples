We are building an app that let's you prototype mobile AR apps on your phone or tablet.  Shadows improve mixing virtual objects with the camera feed immensely.
We currently use plane detection to figure out what geometry to cast shadows on.  Once we know where to cast shadows, we
need to actually cast them.  The easiest way to accomplish this is a transparent material that receives shadows.
Like the [ShadowMaterial](https://threejs.org/docs/#api/materials/ShadowMaterial) in ThreeJS.
Godot doesn't provide this out of the box, so we need to figure out how to do it.

We need to get the shadow strength for a pixel and if it is above a certain threshold, we
render the pixel otherwise we skip it.  Godot provides a [ShaderMaterial](http://docs.godotengine.org/en/latest/tutorials/shading/shader_materials.html)
which allows one to write their own shaders.  This seems like the way to go!  Looking at the
[shading language doc](http://docs.godotengine.org/en/3.0/tutorials/shading/shading_language.html)
it looks like we want to write a "light" processor which checks the shadow and sets the alpha accordingly.

Alas, the light processor can not modify the alpha of the pixel.  Let's change this!  To do that,
we need to learn a bit more about Godot's [rendering setup](https://godotengine.org/article/godot-3-renderer-design-explained).
The key sentence in this document for us is:
"Shaders are, then, translated to native language (real GLSL) and fitted inside the engine's main shader."
We need to modify this main shader to allow us to modify the alpha in the light processor.  Digging
around a bit, we can find it is named ["scene.glsl"](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shaders/scene.glsl)
for the GLES3 backend.   This is an [uber shader](https://www.gamedev.net/forums/topic/659145-what-is-a-uber-shader/) which is compiled into different shaders based on the
material flags.  Looking around, we see a [LIGHT_SHADER_CODE](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shaders/scene.glsl#L943) here,
that looks like where our light processor code will be injected.

Poking around the engine source code a bit, you can find out where this happens, two spots in shader_gles3.cpp.
* First it looks for LIGHT_SHADER_CODE in the shader. [Here](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shader_gles3.cpp#L632)
* Then it inserts the light code from the material. [and here](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shader_gles3.cpp#L384).

In scene.glsl, the LIGHT_SHADER_CODE is contained in a glsl function named "light_compute".  We need to be able to modify the final alpha here somehow.  Looking at
how this function is called, it seems easiest to pass the current alpha in as a in/out parameter.
The steps to accomplish this are: modifying light compute, modifying callers to [light compute](https://github.com/bzztbomb/godot/commit/f68d1ebb4b00db71772cb4a83c9ee1cf4b7168f9).

Recompile the engine and start up a project.

After doing this, we still don't seem to be able to modify the alpha via the ALPHA variable the way you can in the
fragment shader.  It turns out we need to add it to [servers/visual/shader_types.cpp](https://github.com/bzztbomb/godot/blob/shadow_material/servers/visual/shader_types.cpp#L146) so that the
Godot shader compiler will rename ALPHA to alpha in the GLSL code.  This is also used for the shader editor
in the editing environment.

After all of this, we can finally write our custom ShaderMaterial that looks like this:

```
shader_type spatial;
// These flags affect the #defines that are used in the GLES3 scene uber shader.
render_mode blend_mix,cull_back;
uniform vec4 albedo : hint_color;

void fragment() {
	ALBEDO = vec3(0.0, 0.0, 0.0);
	ALPHA = 0.5; // Make sure Godot sends this through the transparent rendering path.
}

void light() {
	DIFFUSE_LIGHT = DIFFUSE_LIGHT;
	ALPHA = clamp(1.0 - length(ATTENUATION), 0.0, 1.0) * 0.75;
}
```

