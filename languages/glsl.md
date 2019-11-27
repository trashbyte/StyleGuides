# Style

## File naming

Files should be named in `snake_case` and have the following extensions:

| Shader type | Extension |
| ----------- | :-------: |
| Vertex      | `.vert`   |
| Fragment    | `.frag`   |
| Compute     | `.comp`   |

## Code naming style

Use `snake_case` for variable, uniform, and function names. Use `TitleCase` for struct type names.

Order inputs, outputs, and uniforms at the top of the file as follows: stage inputs, stage outputs, input attachments, texture samplers, push constants, uniform structs.

```glsl
layout(location = 0) in vec3 position;
layout(location = 1) in vec2 uv;

layout(location = 0) out vec3 uv_out;

layout (input_attachment_index = 0, binding = 0) uniform subpassInput albedo_buffer;

layout (set = 0, binding = 1) uniform sampler2D brdf_lookup;

layout(push_constant) uniform Constants {
    mat4 view;
    mat4 proj;
} constants;

layout(set = 1, binding = 0) uniform InstanceData {
    mat4 world;
} instance;

void main() {
    uv_out = uv;
    gl_Position = constants.proj * constants.view * instance.world * vec4(position, 1.0);
}
```

## Version number

Do not include a version number in shader files. The build system handles adding these automatically

```glsl
#version 450
// don't do this
```

## Comments

Comment as much as possible, especially in places that wouldn't make immediate sense at first glance.

```glsl
// equirectangular UVs from normal
vec2 uv = vec2(atan(N.z, N.x), acos(N.y));
uv /= vec2(2 * PI, PI);
```

# Optimization

## Vertex shader usage

Try to implement as much shader logic as possible in the vertex shader (if it can be interpolated), since it will be executed far less times than doing it in the fragment shader.

```glsl
layout (location = 0) in vec2 uv;

layout(location = 0) out vec2 out_uv;

void main() {
	// UVs are already interpolated anyway, so you might as well do transformations in the vertex shader.
	out_uv = vec2( uv.x, -abs(uv.y - 0.5) + 0.5 );
}
```

Additionally, if texture coordinates are modified in the fragment shader, they require a dynamic texture lookup, which is slower than a regular texture lookup. This is another reason to pre-compute UVs in the vertex shader.

## Uniforms and constants

Whenever a value is constant across an entire shader invocation, pass it as a uniform instead of calculating it for every vertex/fragment. Upload uniforms with push constants when possible. Even better, when a value is constant for *all* shader invocations, declare it as a `const` in the shader. No need to pass it as a uniform if it never changes.

## Specifiers

Be as explicit as possible with specifiers. Not only does this allow the compiler to make more optimizations, it also makes the code easier to understand. This is especially important for output/reference parameters. To make things more explicit, suffix output parameters with `_out`.

```glsl
// ambiguous:
void point_light(vec3 light_pos, vec3 light_color, vec3 N, vec3 V, vec3 diffuse, vec3 specular)

// easier to figure out how parameters are used:
void point_light(const in vec3 light_pos, const in vec3 light_color, const in vec3 N, const in vec3 V, inout vec3 diffuse_out, inout vec3 specular_out)
```

## Swizzling

Swizzles are very compact to write, and essentially free on hardware. Use them whenever possible.

```glsl
// bad
vec3 flipped = vec3(original.z, original.y, original.x);

// much better
vec3 flipped = original.zyx;
```

## MAD operations

MAD is short for "multiply then add". Most modern GPUs have special hardware for this which allow a multiply and an add in a single cycle. Prefer multiplying to dividing, and put multiplies before adds whenever it's easy to do so.

```glsl
// division can't be turned into a MAD operation. the same thing could be accomplished with `* 0.5`.
vec3 result = (value / 2.0) + 0.5;

// the compiler might not optimize this into a MAD operation
result = 0.5 * (1.0 + variable);
```

## Lerp

Use the `mix()` function for lerping values, instead of doing a lerp manually (`(a*(x-1)+(b*x))`) or writing your own function. The built-in `mix()` is specially optimized and will run faster than a manual lerp operation (possibly single-cycle).

```glsl
// manual lerp: slow and hard to read
result = (color_0 * (1.0 - alpha)) + (color_1 * alpha);

// custom function: easier to read but still not optimized
result = lerp(color_0, color_1, alpha);

// mix() is fast and easy to read
result = mix(color_0, color_1, alpha);
```

## Dot product

Despite the complexity of the calculation, dot products are fast on hardware (possibly single-cycle). Use `dot()` as a replacement for multiply-and-sum operations.

```glsl
// calculating luminance:
float luma = dot(diffuse, LUMA_CONST);

// vector sums like these can be optimized...
float sum = some_vec.x + some_vec.y + some_vec.z + some_vec.w;

// ...like so:
const vec4 ALL_ONES = vec4(1.0);
float sum = dot(some_vec, ALL_ONES); // multiplying by 1 does nothing, so this is just a fast 4-component add
```

## Branching instructions

Avoid using branching instructions when possible, since they can limit the ability to perform computations in parallel on many cores. Don't use branching solely to avoid a complex code path, because branch prediction often results in both paths being processed anyway. Always use constant limits in for loops whenever possible, since it allows for loop-unrolling optimizations that can't be performed with dynamic limits.