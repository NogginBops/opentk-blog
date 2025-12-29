---
layout: post
title:  "Exceptions and P/Invoke"
date: 2025-12-18 00:27:00 +0100
categories: support-tips
tags: glfw pinvoke
author: noggin_bops
commentIssueId: 5
---

In the [last post]({% post_url 2025-12-16-the-glfw-error-callback %}) we discussed the GLFW error callback and I alluded to a more complicated reason to add your own error callback. And that reason is how P/Invoke deals with exceptions.

P/Invoke is used to call native functions from C#, which is a great thing to be able to do. This is how OpenTK is able to use GLFW or use any other native dependency. Here is an example of how it's [used in OpenTK](https://github.com/opentk/opentk/blob/eab65e5c34abec4673b4672256e0e6c86018e3ad/src/OpenTK.Windowing.GraphicsLibraryFramework/GLFWNative.cs#L180-L181) to call GLFW functions:
{% highlight cs %}
[DllImport("glfw3.dll", CallingConvention = Cdecl)]
public static unsafe extern Window* glfwCreateWindow(int width, int height, byte* title, Monitor* monitor, Window* share);
{% endhighlight %}

Here `glfwCreateWindow` is a C function that is defined in the `glfw3.dll` native dependency. This works by telling the JIT to first load `glfw3.dll`, find the `glfwCreateWindow` function, and call it with the specified arguments. But what does this have to do with exceptions?

When C# code throws an exception we need to unwind the stack frames and run any `finally` blocks we pass and allow any `catch` blocks to catch and handle the exception. This is done differently on each platform but it means having some C# specific metadata about the stack and what functions on it have `catch` or `finally` blocks to run if an exception is thrown.
The C# JIT is able to do this, but this becomes very complicated when we mix C# and other languages. For example lets consider the glfw error callback scenario. We have C# code that calls a native function (GLFW function) which then later calls a C# function (the error callback) and that C# function throws an exception.

This is a complicated scenario because the C# JIT doesn't know anything about the native functions. Say for example that the native function we called is a C++ function that has a `try`-`finally` block that should be executed. What should the C# JIT do in this scenario? One option is to just ignore the native code and unwind to the nearest C# exception handler. This is an exceptionally bad idea as we now purposely undermine the guarantee that `finally` makes in C++, meaning some critical code (like releasing a lock or freeing memory) never gets called but the program continues running. This can cause any number of problems. So what is the solution? There is no solution to this problem, it's a [cursed problem](https://www.youtube.com/watch?v=8uE6-vIi1rQ).

So what happens? It depends. On most platforms this is [undefined behaviour](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/exceptions-interoperability) and will probably terminate your program without propagating the exception to any `catch` statements on the other side of the unmanaged code, and no native `finally` blocks will run. With one exception, Windows. 

On Windows there is something called [Structured Exception Handling (SEH)](https://learn.microsoft.com/en-us/cpp/cpp/structured-exception-handling-c-cpp) that allows the C# runtime to actually know about exceptions in native code. This means that we can properly handle and propagate exceptions up through code that also uses SEH. So on Windows it is possible to throw an exception in a callback from native code. At least it was possible... While writing this article I got the unfortunate news that from .NET 9 and later the runtime defaults to [no longer participate in SEH](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/exceptions-interoperability). All versions of NativeAOT also do not participate in SEH.

So what does this mean for our error callback that throws an exception? It means you probably want to override it yourself! Make it log the errors to your own logging system, make it terminate the application by setting some state so that the application will quit later on the main thread, or, more interestingly, capture the exception and rethrow it on the main thread yourself.

Why can't OpenTK do this for me? Well, it actually does to some degree, but that is the topic of the next post, so see you there!

