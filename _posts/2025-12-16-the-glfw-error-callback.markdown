---
layout: post
title:  "The GLFW error callback"
date: 2025-12-16 12:00:00 +0100
categories: support-tips
tags: glfw
author: noggin_bops
commentIssueId: 4
---
As promised, in this post I'll be talking about the GLFW error callback and why you might want to set your own.

As GLFW is a C API it does not rely on exceptions to relay errors, instead it uses an error callback that is called when an error happens. An important fact about glfw errors is that as long as [`glfwInit()`](https://www.glfw.org/docs/3.3/group__init.html#ga317aac130a235ab08c6db0834907d85e) succeeded no error is fatal, things might not work correctly but they will not bring down the process or thread.

Here is an exceprt from the [GLFW documentation](https://www.glfw.org/docs/3.3/intro_guide.html#error_handling):
> **Reported errors are never fatal.** As long as GLFW was successfully initialized, it will remain initialized and in a safe state until terminated regardless of how many errors occur. If an error occurs during initialization that causes glfwInit to fail, any part of the library that was initialized will be safely terminated.

But looking at the [default OpenTK error callback function](https://github.com/opentk/opentk/blob/eab65e5c34abec4673b4672256e0e6c86018e3ad/src/OpenTK.Windowing.Desktop/GLFWProvider.cs#L27-L30) we can see the following:

{% highlight cs %}
private static void DefaultErrorCallback(ErrorCode errorCode, string description)
{
    throw new GLFWException($"{description} (this is thrown from OpenTKs default GLFW error handler, if you find this exception inconvenient set your own error callback using GLFWProvider.SetErrorCallback)", errorCode);
}
{% endhighlight %}

It simply throws a `GLFWException` for every error. But if GLFW errors are non-fatal why throw an exception? Visibility. Most GLFW errors mean that the user is doing something with the API that they are not meant to do, and most of the time it makes a lot of sense to surface these errors to the user as fast as possible.

This is however not as true on Wayland. As it is impossible to implement some features of the GLFW api on Wayland (e.g. [`glfwGetWindowPos`](https://www.glfw.org/docs/latest/group__window.html#ga73cb526c000876fd8ddf571570fdb634)) GLFW will emit a [`ErrorCode.FeatureUnavailable`](https://opentk.net/api/OpenTK.Windowing.GraphicsLibraryFramework.ErrorCode.html) error when using these features. This means that suddenly programs that would usually not cause any errors on other platforms could cause a whole plethora of error callbacks on Wayland.

This is one reason why a custom error handler could be useful. Both for logging purposes as it gives the application sufficient info to do the logging of these errors themselves and because Wayland is the way it is and causes GLFW to report errors where it would normally not.

"So how do I set my own error callback?" I hear you ask. It's very simple, use [`GLFWProvider.SetErrorCallback`](https://opentk.net/api/OpenTK.Windowing.Desktop.GLFWProvider.html#OpenTK_Windowing_Desktop_GLFWProvider_SetErrorCallback_OpenTK_Windowing_GraphicsLibraryFramework_GLFWCallbacks_ErrorCallback_):
{% highlight cs %}
// Define the custom error callback function.
private static void MyErrorCallback(ErrorCode errorCode, string description)
{
    if (errorCode == ErrorCode.FeatureUnavailable)
    {
        Console.WriteLine($"GLFW feature unavailable: {description}");
    }
    else
    {
        throw new GLFWException($"{description} (this is thrown from OpenTKs default GLFW error handler, if you find this exception inconvenient set your own error callback using GLFWProvider.SetErrorCallback)", errorCode);
    }
}

// Set the error callback function.
GLFWProvider.SetErrorCallback(MyErrorCallback);
{% endhighlight %}

There is also a more complicated reason to set your own callback related to p/invoke and exceptions, but that is the topic of the [next post]({% post_url 2025-12-17-exceptions-and-pinvoke %}) so it will have to wait.
