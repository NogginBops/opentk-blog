---
layout: post
title:  "The OpenTK matrix guide"
date: 2026-02-17 12:00:00 +0100
categories: support-tips
tags: matrix glsl math
author: noggin_bops
commentIssueId: 12
excerpt: The ultimate guide to matrix majorness and multiplication conventions, and how to use matrices in OpenTK.
---

TL;DR: [How to get consistency when using OpenTK](#how-to-get-consistency-when-using-opentk)

There is a lot of confusion about matrices in OpenTK and how they relate to GLSL. In this post I aim to clear up this confusion and show that it isn't as complex as many believe.

Matrices have two distinct properties: *element order* and *multiplication convention* which are often conflated but have surprisingly little in common.

1. Do not remove this line (it will not be displayed)
{:toc}

## Matrix element order

A matrix is a 2D grid of numbers which doesn't have an "obvious" way of being stored in memory, which is linear.
There are many ways to make a 1D array of a 2D grid, but for matrices there are two popular ways; *row-major* order and *column-major* order.

*Row-major* order stores the rows of the matrix one after the other, like this:

<div class="light-dark-svg equation-svg" style="--svg-custom-height: 4lh;">
    {% include post-2026-01-02/row-layout.svg %}
</div>

Which would look like this when put into memory:

<div class="light-dark-svg equation-svg" style="--svg-custom-width: 100%; --svg-custom-height: 2lh;">
    {% include post-2026-01-02/row-linear-layout.svg %}
</div>

In code we can write it like this:

{% highlight cs %}
public struct Matrix4
{
    public Vector4 Row0;
    public Vector4 Row1;
    public Vector4 Row2;
    public Vector4 Row3;
}
{% endhighlight %}

Similarly, in *column-major* order we store the columns one after the order, like this:

<div class="light-dark-svg equation-svg" style="--svg-custom-height: 5lh;">
    {% include post-2026-01-02/column-layout.svg %}
</div>

Which would look like this when put into memory:

<div class="light-dark-svg equation-svg" style="--svg-custom-width: 100%; --svg-custom-height: 2lh;">
    {% include post-2026-01-02/column-linear-layout.svg %}
</div>

In code we can write it like this:

{% highlight cs %}
public struct Matrix4
{
    public Vector4 Column0;
    public Vector4 Column1;
    public Vector4 Column2;
    public Vector4 Column3;
}
{% endhighlight %}

Knowing the *majorness* of a matrix is important when sending matrices between libraries as both sides need to agree which float goes into which matrix position.

This is especially important when communicating between programming languages where the compiler is unable to do type-checking between programming languages.
Imagine receiving a `float[16]` representing a matrix, there is nothing to tell us which matrix position each element has, we have to decide on what matrix order we will use and stick to that.
If someone else uses a different matrix order we need to convert our representation into their matrix order before passing the array to them.

What happens if we read a *row-major* order matrix as if it was a *column-major* order? 
In other words, what if we send a matrix that is in *column-major* order to a function that will read the matrix in *row-major* order?
Look back at the illustrations above and see if you can figure it out.

The answer is that the rows of the *row-major* order matrix will be read as the columns of a *column-major* order matrix.
What this means is that column 0 of the matrix is now going to be considered row 0 of the matrix.
The mathematical name for swapping the rows and columns of a matrix is called the "transpose" of the matrix.
So, sending *column-major* order matrix into a function that expects a *row-major* order matrix means that the function will read a transposed version of the matrix.
Which means that for us to correctly call the function we need to convert our *row-major* order matrix into a *column-major* order matrix which we can do by transposing the matrix.
The same is true for the other direction too.

## Multiplication convention

The second property of matrices relevant is the *multiplication convention* which in simplified terms tells us if you are expected to multiply a vector from the left or from the right to get the correct result. As an example lets look at a simple 4x4 translation matrix. We have two ways of defining a matrix that will result in a translation when multiplied by a vector. The first is to create a translation matrix where the translation is placed in the last column of the matrix:
<div class="light-dark-svg equation-svg" style="--svg-custom-height: 4lh;">
    {% include post-2026-01-02/column-translation.svg %}
</div>
To translate a vector using this matrix we need to multiply the vector from the right as follows:
<div class="light-dark-svg equation-svg" style="--svg-custom-height: 4lh;">
    {% include post-2026-01-02/column-multiplication.svg %}
</div>
As we can see, this results in a vector that has been translated by the values in the matrix. But there is actually another way we could have constructed our translation matrix. We could instead put the translation in the last row of the matrix and multiplied it with a row-vector from the left, like this:
<div class="light-dark-svg equation-svg" style="--svg-custom-width: 100%; --svg-custom-height: 4lh;">
    {% include post-2026-01-02/row-multiplication.svg %}
</div>

These two alternatives produce identical results but the matrices are not equal. So to know how to multiply our vectors with matrices we need to know if we are supposed to multiply our vector from the right or from the left. In math notation it's very explicit if a vector is multiplied as a column or row vector, but in programming languages libraries often make all vectors the same type and you don't see the column or row shape of the vector written out as you would in math notation.
As an example, in OpenTK if we have a `Vector4 v` variable, OpenTK will allow both `v * M` (multiplying from the left as a row vector) and `M * v` (multiplying from the right as a column vector) which can be confusing.

An important matrix multiplication fact is that `vM = Mᵀvᵀ`, that is if a matrix expects a row vector from the left we can transpose the matrix and multiply with the column vector from the right. In code this would look something like this:
{% highlight cs %}
Matrix4 M = ...;
Vector4 v = ...;

// This will always be true (with reservations for float precision).
Debug.Assert(v * M == Matrix4.Transpose(M) * v);
{% endhighlight %}

## Why do these get mixed up?

There are many people coming to the OpenTK [discord server](https://discord.gg/eFXkNsrB3V) asking why nothing renders or why all of their transforms are acting weird.
The answer is almost always that the user has got either the *majorness* or the *multiplication convention* wrong, either from not knowing what these things are or because of a conflation between the two.

There are many reasons that these two properties of matrices get conflated and confused, but I suspect the biggest reason is that you can flip both conventions by transposing the matrix, either by accident or on purpose.

The reason to transpose a matrix could be because it will be sent to a function expecting a matrix of opposite *majorness* it could also be transposed to change the *multiplication convention* from left-to-right to rigth-to-left or vice versa. The transpose is also a useful mathematical operation that can be used in some equations without any direct connection to *majorness* or *multiplication convention*.

Another confusion that exists is about GLSL and OpenGL. Historically, OpenGL used column-major order matrices with right-to-left multiplication convention, this is however, no longer true in modern versions of OpenGL. Modern OpenGL is almost completely *majorness*[^1] and *multiplication convention*[^2] agnostic, as we will see in the next section.

[^1]: Sending matrices as vertex attributes can unfortunately only be defined in [column-major order](#glsl-vertex-attributes).
[^2]: Matrix type constructors take [column-vector arguments](#glsl-tbn-matrix-for-normal-mapping), there is no constructor that constructs a matrix from rows.

## How to get consistency when using OpenTK

OpenTK uses row-major order and left-to-right multiplication order. This means that in C# code vectors are multiplied from the left of matrices. The following sections will explore how to get consistent matrix multiplication order in both C# and GLSL.

### C#

In C# OpenTK uses row-major order matrices with left-to-right multiplication order which means that operations happen in a left-to-right order. The left-most matrix will be applied first. In this example we have a simple model view projection setup where we send the matrix as a uniform to some shader:

{% highlight cs %}
Matrix4 M = Matrix4.CreateScale(2); // model
Matrix4 V = Matrix4.CreateTranslation(0, 0, -2).Inverted(); // view
Matrix4 P = Matrix4.CreatePerspectiveFieldOfView(90f/180f * MathF.PI, Width / Height, 0.1f, 100f); // projection 

Matrix4 MVP = M * V * P; // left to right multiplication

// Pass transpose=true to tell OpenGL we are sending a row-major order matrix
GL.UniformMatrix4(0, true, ref MVP);
{% endhighlight %}

Interesting to note here is the `GL.UniformMatrix4` call where we send `true` in the `transpose` argument, this is because OpenGL by default reads matrices in a column-major order and transposing converts our row-major order matrix into column-major order. Personally I've renamed this argument in my head to something like `is_row_major` as transpose to me as an operation that relates to multiplication convention and has nothing to do with majorness.

### GLSL Uniforms

In GLSL, when you pass your matrices as above, the same left-to-right multiplication convention applies. So vectors to the left of the matrices and operations happen left-to-right.

{% highlight glsl %}
// Small shader that transforms vertex positions using a model, view, and projection matrix.

in vec3 v;

uniform mat4 M; // model
uniform mat4 VP; // view * projection

void main() {
    // Using OpenTKs left-to-right matrix multiplication convention.
    gl_Position = vec4(v, 1.0) * M * VP;
}
{% endhighlight %}

### GLSL UBO/SSBO

When using Uniform Buffer Objects (UBOs) or Shader Storage Buffer Objects (SSBOs) we can use the `row_major` layout qualifier to tell OpenGL that our matrices have a row-major order.

{% highlight glsl %}
// Small instancing shader that uses per-instance model matrices uploaded through an SSBO.

in vec3 v;

// Tell OpenGL that the buffer contains row-major order matrices.
layout(row_major, binding = 0) readonly buffer InstanceTransforms {
    mat4 instance_M[];
}

uniform mat4 VP; // view * projection

void main() {
    // Using OpenTKs left-to-right matrix multiplication convention.
    gl_Position = vec4(v, 1.0) * instance_M[gl_InstanceID] * VP;
}
{% endhighlight %}

### GLSL Vertex Attributes

This is the one API in OpenGL that is not *majorness* agnostic. This is because matrix vertex attributes are defined by four separate `vec4` vertex attributes where each `vec4` becomes a column of the matrix. It's impossible to define a `vec4` attribute where the individual floats have a stride, which means it's impossible to have row-major order matrices as inputs to vertex shaders. That doesn't mean that it's impossible to use them in OpenTK, it just means that the matrix you receive in GLSL is going to be inverted.

{% highlight glsl %}
// Small instancing shader that uses vertex attribute model matrices 
// using glVertexAttribDivisor to do instancing

in vec3 v;
in mat4 instance_M_T; // transpose of the model matrix due to vertex attribute limitation

uniform mat4 VP; // view * projection

void main()
{
    mat4 instance_M = transpose(instance_M_T); // Transpose the transposed matrix
    gl_Position = vec4(v, 1.0) * instance_M * VP;
}
{% endhighlight %}

Personally I prefer the [SSBO approach](#glsl-ubossbo) for instancing as it's easier, much more flexible, and generally should not be any slower than vertex attributes[^3].

[^3]: [This article](https://www.yosoygames.com.ar/wp/2018/03/vertex-formats-part-2-fetch-vs-pull/) compares SSBOs vs vertex attributes for per-vertex attributes. It would be reasonable to expect similar results for per-instance data but with a smaller performance difference.

### GLSL TBN matrix for normal mapping

When implementing normal mapping it's typical to construct a Tangent Bitangent Normal (TBN) matrix.
Matrix constructors in GLSL are the only part of GLSL that is not *multiplication convention* agnostic.
Matrix multiplication comes from where matrix elements are placed, and matrix constructors in GLSL take column vectors as input to construct the matrix.
So using the matrix constructor to make the TBN matrix like this, `mat3(fTangent, fBitangent, fNormal)`, will create a matrix with right-to-left multiplication convention.
The easy solution is to just transpose the constructed matrix `transpose(mat3(fTangent, fBitangent, fNormal))` which restores the left-to-right multiplication convention.

{% highlight glsl %}
// Part of a fragment shader implementing normal mapping 
// showing how matrix constructors construct matrices using column vectors.

in vec3 fNormal;
in vec4 fTangentW; // xyz: Tangent, w: Bitangent sign
in vec2 fUV;

uniform sampler2D texNormal;

out vec3 oNormal;

void main()
{
    vec3 fTangent = fTangentW.xyz;
    vec3 fBitangent = cross(fNormal, fTangent) * fTangentW.w;

    // The input vectors are treated as columns,
    // creating a right-to-left multiplication convention matrix.
    mat3 TBN = transpose(mat3(fTangent, fBitangent, fNormal));

    vec3 tangentSpaceNormal = texture(texNormal, fUV) * 2.0 - 1.0;
    
    oNormal = tangentSpaceNormal * TBN;
}
{% endhighlight %}

And if you are worried that the extra transpose is going to cause lower performance, I can tell you that the [compiler will figure it out and optimize away the transpose](https://godbolt.org/#z:OYLghAFBqd5TKALEBjA9gEwKYFFMCWALugE4A0BIEAZgQDbYB2AhgLbYgDkAjF%2BTXRMiAZVQtGIHgBYBQogFUAztgAKAD24AGfgCsp5eiyagk9JfXIrGqIgSHVmmAMLp6AVzZMQADllOAGQImbAA5TwAjbFIQACYtcgAHdCVieyZXDy9fWWTUuyEgkPC2KJj4q2wbAqYRIhZSIkzPbz9K6vS6hqIisMjouISlesbm7Lbh7t6SssGASit0d1JUTi4AenWAagAVJGwt5iJSAE8t5OCiLeNMLZHgbCvE0nQ6RmvSA5DsHFuSLYwbESDAORCQBCUh3U7ESkgApFoAIKbLYAWh25yUAH0AGy4tG4LaqEQAWRYwQRiMpw1I7lsWwAkkxEu4iJS4QB2ABClK2fK2NHo6BYRAAzFsmGQ2BI4aKeUj%2BQKhSLpHdjA9hFiIgB3WXyxGKwXCsVbTAilg8PXsjkAEStSMpACUAOp1Wm2ZY/LnuGg0aKy5xMlls0WE72%2B6JaLYgLafYAQojRCDuLRze1Uh0AVi5TE8YM%2BLEwSggPHIW1L5bTmbtSIAbugCLdnKTyUxk5dxTado3o1sRAA1LE2iGJEWoJB7AuYBk2tMO7m8/lGkXiyWkaX0Laym1bcN%2B0haOHZw/VgB0a436cNypN9RMRy3op3e8jR65J5tp7vGqIWu1p/UE4AC8ryXG9xQiYh1QfbcAReJRiwvCQy2/I45i2AAqXcfX3E93yPT9UM1HVT11OV2QVfkXwPN8P1PM16h4R8dzYdx6FocD1FFCAiKIMtIN4sskPoBZsIjGjjwI%2BjzR4NNyPnGtES4BZ6G4TN%2BG8LgdHIdBuAACQCEQAi2JQlhWA44ViUU%2BHIIhtGUhYAGs4gATlPFyOVFHFRS0DkfB4FytBc0VYkMbhpH4NgQGkaRTw5DkeFiaQeB8HytBxTMvNkTTtN0rh%2BCUEAEjsrTlPIOBYBQDAcHwYgyEoag3mYdg1hswRhDECROBkORhGUNRNFK8h9FCowTBAMwLHabBbHSRwmBcNwWhAFzM3IQJgj6UoBhSpIUjSIQxm8Va9vydJpn6GJdusGaai6UYluyE6btmoR7p6TaZh2nwrBGJpHuOtbJkaC7tqunwFlM5ZVm4aljjpK4g1Za19WvY1VylGV5INMDjVVXi/1AvllxNBiLXTTk7Wxp1XXhj1PkwaiAyRkMwxwyNezjBMkxTOT9XZbNczYfNsELYsKwlqtFPrHtmzJYJ22ETtu1uGMByHEcxwnJApxnOcqQXSjifAiVMc3WDqLwujhKJpV0bVe9hGYsTcNoqSCZIwCQOxtGVy2AToKd2DUHgxCzZQwOiHQrDLbds8Pf/Mj%2BaNl3X0ks8yaY2DWPY2P08/TOUNIYwlDybAOPRrieMj/ioMdvjTfXCQ5hbinbUpMrVK4dTyBy/g8oMoyTLM1Ytys2J%2BBKnQW/IfZCwGCAVPCyKQEzTNT1iWIfA5TN/JxFzgukbycV7%2BydO4Aqits%2ByZ%2Bc6QtDi1LpFWnwcX8oLpFG7hRQ0s%2B8snm%2B5VEAQCqugIEIIKBUAgICYEjAYikGACwLEsQsQ8FFAIBgiZSCFQgBEM%2BkFWCnG4DZAhDQTgAHkIi6FusQ/ggIODCHIUwegJwz44AiO4YAzgJDmFoeQHA0oTCSCGoQT4s1azYEKkNbA6gZqslavwS4VQz70AIBEYupxXA4DPscAgUVeD8AkaQCIKRsA2mwII4AqjxqlQWIKFgwAlD9gINgbU5DEjMD4e1UQ4hJA9W8f1DQZ99CljGqYcwlhVEREKpABY6BEg1CkaiSEmB1CJTRKgLYwAaBpPiExVEqJUBKFRGwLAVQATYjxDiNEhTilHFOFsOWrZ%2B5GNII2SR8BIZVFunNCATgjoGA2sUS6Bg8gHQyADUZ%2B0aig1mKWF6d0/oDPmd016tQ/qzJ2r9boyztkg0%2BiM2SixobdSXt3X%2BQ08pbERCSHcjoADiBpYinjQVsCAtUSCkDHtZOYADbELDnjgGIi9yDOViBydysQeBaC0NvIKHItCZjfmFLgEVyBRWhQkPu598pWCvlPMqFVkBoDAbA6IDVoGkogSABBSCUFoIwfQLBOC8FDVIUQgx5B2UUKoTQzl9CjhMJYWw7AHCuE8KkTZAR6phHaVET0iRUjtIyLkYmPhSiu7aSiRok4Wi1jaV0fomyRiTEqHMZY6xoBbECCMI45xrj3GeM5d4zqfjZABJUEEoaI1DDqgmhEwwaiYkgviYk7gyTTS5IyVknJiUtD5NqSUspm4im4nxAUopqJ6lnCacEFp0Q2k4GDV0joDg%2BkLV2UMraczTrjN2WMmZBywYGAWZ0JZkyVmlvWVMJtNbgb/SyN4eZGze07UhiPTgsQzk92xVcm5dzHlbGea895hBPnfKnX86eALRZAuoE5EA1kN6rWfpmT%2BUK/AuS/qii5uUL54uKjfM5E9T6XPvQSmeRjUgOGkEAA%3D%3D). So, you don't have to worry. You can get multiplication convention consistency with no performance penalty.

## What do other libraries do?

Other libraries than OpenTK have different conventions, here is a non-exchaustive list of libraries and their majorness and multiplication convention:

|-------|---------|-------------------------|
|Library|Majorness|Multiplication convention|
|-------|---------|-------------------------|
| [`OpenTK`](https://opentk.net) | Row-major order | `v * M`   |
| [`System.Numerics`](https://learn.microsoft.com/en-us/dotnet/api/system.numerics) | Row-major order | `v * M`   |
| [GLM](https://github.com/g-truc/glm) | Column-major order | `M * v`   |
| [DirectXMath](https://github.com/microsoft/DirectXMath) | Row-major order | `v * M` |
| [Unity](https://unity.com/) | Column-major order | `M * v` |
| [Unreal](https://www.unrealengine.com) | Row-major order | `v * M` |
| [Godot](https://godotengine.org/) | Column-major order | `M * v` |
|-------|---------|-------------------------|

As we can see there is a strong correlation between majorness and multiplication convention, why is that? Well, it turns out that when the matrix is in row-major order format it's actually faster to multiply the vector from the left of the matrix as all components of the result vector can be calculated in parallel. So it's pretty rare to find a library where majorness and multiplication convention isn't correlated, but it doesn't mean they couldn't exist. If you create your own matrices in OpenTK you could create right-to-left multiplication convention matrices using OpenTKs `Matrix4` struct.