---
layout: post
title:  "The OpenTK matrix guide"
date: 2026-01-02 15:00:00 +0100
categories: support-tips
tags: matrix glsl math
author: noggin_bops
commentIssueId: 8
excerpt: The ultimate guide to matrix majorness and multiplication conventions, and how to use matrices in OpenTK.
---

TL;DR: [How to get consistency when using OpenTK](#how-to-get-consistency-when-using-opentk)

There is a lot of confusion about matrices in OpenTK and how they relate to glsl. In this post I aim to clear up this confusion and show that it actually isn't that complicated.

There is often two distinct properties of matrices that are often conflated but have surprisingly little in common; *element order* and *multiplication convention*.

1. Do not remove this line (it will not be displayed)
{:toc}

## Matrix element order

The element order of the matrix tells us in which order the elements of the matrix is stored. Because a matrix is a grid of numbers we have many ways of storing this in memory. There are two popular ways to store matrices; row-major order and column-major order. There are other orders you could use but they are not nearly as popular as row- or column-major order.


Row-major order stores the rows of the matrix one after the other, like this:

<div class="light-dark-svg equation-svg" style="--svg-custom-height: 4lh;">
    {% include post-2026-01-02/row-layout.svg %}
</div>

Which would look like this when put into memory:

<div class="light-dark-svg equation-svg" style="--svg-custom-width: 100%; --svg-custom-height: 2lh;">
    {% include post-2026-01-02/row-linear-layout.svg %}
</div>

And in code we can write it like this:

{% highlight cs %}
public struct Matrix4
{
    public Vector4 Row0;
    public Vector4 Row1;
    public Vector4 Row2;
    public Vector4 Row3;
}
{% endhighlight %}

Similarly, in column-major order we store the columns one after the order, like this:

<div class="light-dark-svg equation-svg" style="--svg-custom-height: 5lh;">
    {% include post-2026-01-02/column-layout.svg %}
</div>

Which would look like this when put into memory:

<div class="light-dark-svg equation-svg" style="--svg-custom-width: 100%; --svg-custom-height: 2lh;">
    {% include post-2026-01-02/column-linear-layout.svg %}
</div>

And in code we can write it like this:

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

Imagine receiving a `float[16]` representing a matrix, there is nothing to tell us which matrix position each element has, we have to decide on what convention we will follow and stick to that. If someone else follows a different convention we need to convert our representation into their convention before passing the matrix to them.

But, what happens if we read a row-major order matrix as if it was a column-major order? Look at the above order illustrations and see if you can figure it out. The answer is quite simply that the rows of the row-order matrix will be read as the columns of a column-major order matrix. This is the same thing as transposing the matrix. Which means that converting our row-major order into a column-major order is the same as transposing the elements of the row-major order matrix. The same is true for the other direction too.

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

As we can see, these two alternatives produce identical results but the matrices are not equal. So to know how to multiply our vectors with matrices we need to know if we are supposed to multiply our vector from the right or from the left. In math notation it is very explicit if a vector is multiplied as a column or row vector, but in code libraries often make all vectors the same type. As an example in OpenTK say we have a `Vector4 v` variable, OpenTK will allow both `v * M` (multiplying from the left as a row vector) and `M * v` (multiplying from the right as a column vector) which can be confusing.

An important matrix multiplication fact is that `vM = Mᵀvᵀ`, that is if a matrix expects a row vector from the left we can transpose the matrix and multiply with the column vector from the right. In code this would look something like this:
{% highlight cs %}
Matrix4 M = ...;
Vector4 v = ...;

// This will always be true (with reservations for float precision).
Debug.Assert(v * M == Matrix4.Transpose(M) * v);
{% endhighlight %}

## Why do these get mixed up?

There are many reasons that these two properties of matrices get conflated and confused, but I suspect the biggest reason is that you can flip both conventions by transposing the matrix, either by accident or on purpose.

Transposing a matrix could be done because it will be sent to a function expecting a matrix of opposite *majorness* or it could be transposed to change the *multiplication convention* from left-to-right to rigth-to-left or vice versa.

Another confusion that exists is about GLSL and OpenGL. Historically, OpenGL used column-major order matrices with right-to-left multiplication convention, this is however, no longer true in modern versions of OpenGL. Modern OpenGL is almost completely *majorness*[^1] and *multiplication convention*[^2] agnostic, as we will see in the next section.

[^1]: Sending matrices as vertex attributes can unfortunately only be defined in [column-major order](#glsl-vertex-attributes).
[^2]: Matrix type constructors take column-vector arguments, there is no constructor that constructs a matrix from rows.



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

In GLSL, if you've passed your matrices correctly, the same left-to-right multiplication convention applies. So vectors to the left of the matrices and operations happen left-to-right.

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

This is the one API in OpenGL that is not majorness agnostic, because matrix vertex attributes are defined by four separate `vec4` vertex attributes where each `vec4` becomes a column of the matrix. It's impossible to define a `vec4` attribute where the individual floats have a stride, which means it is impossible to have row-major order matrices as inputs to vertex shaders. That doesn't mean that it's impossible to use them in OpenTK, it just means that the matrix you receive in GLSL is going to be inverted.

{% highlight glsl %}
// Small instancing shader that uses vertex attribute model matrices using glVertexAttribDivisor to do instancing

in vec3 v;
in mat4 instance_M_T; // transpose of the model matrix due to vertex attribute limitation

uniform mat4 VP; // view * projection

void main()
{
    mat4 instance_M = transpose(instance_M_T); // Transpose the transposed matrix
    gl_Position = vec4(v, 1.0) * instance_M * VP;
}
{% endhighlight %}

Personally I prefer the [SSBO approach](#glsl-ubossbo) for instancing as it is easier, much more flexible, and generally should not be any slower than vertex attributes[^3].

[^3]: [This article](https://www.yosoygames.com.ar/wp/2018/03/vertex-formats-part-2-fetch-vs-pull/) compares SSBOs vs vertex attributes for per-vertex attributes. Similar results for per-instance data but to a much smaller degree seem reasonable to expect.

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

As we can see there is a strong correlation between majorness and multiplication convention, why is that? Well, it turns out that when the matrix is in row-major order format it's actually faster to multiply the vector from the left of the matrix as all components of the result vector can be calculated in parallel. So it's pretty rare to find a library where majorness and multiplication convention isn't correlated, but it doesn't mean they couldn't exist. I mean if you create your own matrices in OpenTK you could create right-to-left multiplication convention matrices using OpenTKs `Matrix4` struct.