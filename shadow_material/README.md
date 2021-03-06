Using  Godot&#39;s GLES3 Shader System in Augmented Reality Apps

_Note: This is our first blog post for 3D app developers and, as such, it represents a bit of a moment for Torch. Over the last year we&#39;ve published more than forty blog posts meant to instruct and inspire designers and share our view of the fast evolving 3D market. After our launch in September, though, we started getting asked more frequently not just how to design mobile AR apps, but how to build them. And it turns out building a complex, multi-user, interactive mobile AR application gave us some useful experience, experience we plan to share with you regularly. So keep an eye on the Torch blog not just for design how-to&#39;s and app reviews and so forth, but for posts that explain how we built Torch, useful tools and tricks we&#39;ve come across, and what we think is cool in the world of 3D app dev. - Josh_

At Torch we are building [an app](https://www.torch.app/) that allows anyone to design, prototype, test, and soon deploy mobile AR apps on their phone or tablet (currently iOS/ARKit only). We use Godot, an open source game engine, to handle the majority of the spatial work. Godot is great and we are glad we chose it. Among the many reasons we like - it&#39;s lightweight, it&#39;s free to use, the Godot community is a pleasure to work with -- the fact we can modify it to fit out needs is pretty high up there. After all, we are building one of the first, and certainly among the most complex, multi-user, multi-scene, interactive augmented reality apps around. We are bound to find things we need to do that Godot doesn&#39;t support. And since Godot is open source, we can just add the features we need. Simple!

![Example of shadow](shadow1.jpg)

Making Rendered Objects More Believable with Shadows

Torch, at its most basic, is a tool that allows a designer to place objects in a 3D space superimposed on the camera feed, all without code. It&#39;s a WYSIWYG editor for 3D. To make the virtual objects feel more believable next to real-world objects, we need to render shadows. It turned out to be a bit more involved than we initially thought, so after we figured out how to do it, we decided we&#39;d share the solution in case other Godot users run into a similar problem.

We currently use [ARKit&#39;s](https://developer.apple.com/documentation/arkit)[plane detection](https://developer.apple.com/documentation/arkit/building_your_first_ar_experience?language=objc) to figure out what geometry to cast shadows on. Once we know where to cast shadows, we need to actually cast them. The easiest way to accomplish this is a transparent material that receives shadows, like the [ShadowMaterial](https://threejs.org/docs/#api/materials/ShadowMaterial) in [ThreeJS](https://threejs.org/). Godot doesn&#39;t provide this out of the box, so we need to figure out how to do it.

For every pixel on a detected plane we need to find out if it is shadowed by a virtual object. If it is, we render that pixel with a shadow. If not we skip it. Godot provides a [ShaderMaterial](http://docs.godotengine.org/en/latest/tutorials/shading/shader_materials.html) which allows developers to write their own shaders. This seems like the way to go!

Looking at the [shading language doc](http://docs.godotengine.org/en/3.0/tutorials/shading/shading_language.html), it appears that there are three spots to hook custom code into: &quot;vertex processor&quot; which computes the vertex position, &quot;fragment processor&quot; which computes the albedo and alpha of a fragment, and &quot;light processor&quot; which computes the lighting for a fragment. So we want to write a &quot;light processor&quot; which checks the shadow and sets the alpha accordingly.

Alas, the light processor cannot modify the alpha of the pixel. Let&#39;s change this! To do that, we need to learn a bit more about Godot&#39;s [rendering setup](https://godotengine.org/article/godot-3-renderer-design-explained). The key sentence in this document for us is: &quot;Shaders are, then, translated to native language (real GLSL) and fitted inside the engine&#39;s main shader.&quot; We need to modify the engine&#39;s main shader to allow us to modify the alpha in the light processor. Digging around a bit, we find the main shader, and it is named [&quot;scene.glsl&quot;](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shaders/scene.glsl) for the GLES3 backend. This is an [uber shader](https://www.gamedev.net/forums/topic/659145-what-is-a-uber-shader/) which is basically a large shader with conditional blocks of code which are turned on and off to produce different shaders with shared code. Looking through this large shader, we see a [LIGHT\_SHADER\_CODE](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shaders/scene.glsl#L943) tag here. That looks like where our light processor code will be injected.

Now we know where the code is injected, but what actually puts our light processor into the scene shader? Poking around the engine source code a bit, you can find this happens in two spots in shader\_gles3.cpp. First, Godot looks for LIGHT\_SHADER\_CODE in the scene shader [here](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shader_gles3.cpp#L632), then it inserts the light code from the material (SpatialMaterial or ShaderMaterial) [here](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shader_gles3.cpp#L384).

Having just established an understanding about how all the pieces involved fit together, up next we simply need to make the changes needed to modify alpha in the light processor.  In the scene.glsl shader, the light processor (via the LIGHT\_SHADER\_CODE tag) is contained in a GLSL function named &quot;[light\_compute](https://github.com/bzztbomb/godot/blob/shadow_material/drivers/gles3/shaders/scene.glsl#L931)&quot;. We need to be able to modify the final alpha in this function somehow. Looking at how this function is called, it seems the easiest way is to pass the current alpha in as an _in/out_ parameter. The steps to accomplish this are:

1. Modifying light compute,
2. Modifying callers to light compute. You can see the code for these modifications on my Github repo [here](https://github.com/bzztbomb/godot/commit/f68d1ebb4b00db71772cb4a83c9ee1cf4b7168f9).

After doing this, we still don&#39;t seem to be able to modify the alpha via the ALPHA variable the way you can in the fragment shader. It turns out we need to add it to [servers/visual/shader\_types.cpp](https://github.com/bzztbomb/godot/blob/shadow_material/servers/visual/shader_types.cpp#L146) so that the Godot shader compiler will rename &quot;ALPHA&quot; to &quot;alpha&quot; in the GLSL code. This is also used for the shader editor in the editing environment.

After all of this, we can finally write our custom ShaderMaterial that looks like this:

shader\_type spatial;

// These flags affect the #defines that are used in the GLES3 scene uber shader. blend\_mix causes the material to get shadows
render\_mode blend\_mix,cull\_back;

void fragment() {
    ALBEDO = vec3(0.0, 0.0, 0.0); // Black shadows
    ALPHA = 0.5; // Make sure Godot sends this through the transparent rendering path.
}

void light() {

         // Take the attenuation (shadow strength) and use it to manipulate the alpha

         // 0.75 is there to let some of the camera feed through.
    ALPHA = clamp(1.0 - length(ATTENUATION), 0.0, 1.0) \* 0.75;
}

I have a Godot project [here](https://github.com/bzztbomb/godot_examples/tree/master/shadow_material) that already has this shader setup in a simple scene.

![Example of shadows](shadow2.jpg)

What have we learned after making this modification?

- Godot splits the fragment shader into the fragment and light pieces, and the light piece is run per light.  Pretty nice design!
- The main shader that Godot uses is an uber shader with lots of #defines which change how the shader is compiled.
- SpatialMaterial and ShaderMaterial flags/options mostly map to #defines in scene.glsl
- Adding variables to a Godot shader involves modifying shader\_types so the shader compiler knows about the variable.

Thanks for reading, I hope this has helped increase your understanding of the Godot rendering system a bit! If you want to see how Godot renders shadows in an AR context checkout [Torch](https://torch.app/), our mobile AR prototyping tool. Developers are one of our fastest growing user groups. We&#39;ve been told this is because they are using it to communicate concepts with designers and others on their team instead of using things like drawings and hand gestures but hopefully we are inspiring others to try their hand at building mobile AR applications.

If you are interested in learning more about how to develop mobile AR applications or have questions about my post, join us in our slack channel.
