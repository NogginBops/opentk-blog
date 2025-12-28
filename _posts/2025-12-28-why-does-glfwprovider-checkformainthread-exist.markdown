---
layout: post
title:  "Why does `GLFWProvider.CheckForMainThread` exist?"
date: 2025-12-28 23:00:00 +0100
categories: support-tips
tags: glfw pinvoke
author: noggin_bops
commentIssueId: 8
---

In this post we will look at [`GLFWProvider.CheckForMainThread`](https://opentk.net/api/OpenTK.Windowing.Desktop.GLFWProvider.html#OpenTK_Windowing_Desktop_GLFWProvider_CheckForMainThread) and figure out why it exists and what you could use it for.

OpenTK 4 uses GLFW to deal with creating and managing windows. To use GLFW properly `glfwInit` needs to be called before using most of GLFWs functions. To do this OpenTK uses [`GLFWProvider`](https://opentk.net/api/OpenTK.Windowing.Desktop.GLFWProvider.html) which we've seen in two [previous]({% link _posts/2025-12-15-what-is-opentk4-use-wayland.markdown %}) [posts]({% link _posts/2025-12-16-the-glfw-error-callback.markdown %}). This class is responsible for initializing GLFW with an appropriate error handler before initialization (in case any errors happen during initialization) and actually calling `glfwInit`. OpenTK calls `glfwInit` by calling `GLFWProvider.EnsureInitialized()` before using any GLFW functions, most notably in the `NativeWindow` constructor.

If the first window you create is done on a separate thread, then you will get an exception saying `"GLFW can only be called from the main thread!"`. This is because you can only call `glfwInit` from the main thread and `GLFWProvider` does some validation when first initializing GLFW to make sure that it's actually done on the main thread.

Most GLFW functions need to be called on the main thread, but why? Is the main thread special in any way? The answer is, as it typically is, "it depends". MacOS does make the main thread special and assumes that all window and UI function calls will be made on the main thread. This means that trying to create a window on macOS on some other thread other than the main thread will likely crash or create other weird issues. 

On windows the story is slightly different as each thread has its own event queue, and if a window is created on a specific thread the messages that window receives will only be posted to that threads event queue. So, creating windows on different threads will mess with GLFW ability to handle all window events with a single call to `glfwPollEvents`.

On Linux it's a different story. X11 has a single process wide event queue and can deal with most functions being called from different threads as the whole API is asynchronous by design. Wayland is very explicit with event queues and can redirect events from one object to any user-created event queue, so what happens when creating windows on different threads is up to the GLFW internals.

As we can see, only ever using one thread when calling GLFW, and by proxy OpenTK, makes all of these platform dependent differences disappear. So to guarantee cross-platform compatibility GLFW requires most[^most-functions] functions to be call from the main thread.

The next question I want to answer is how `GLFWProvider` knows which thread is the main thread. This seems like it would be easy to figure out, but it turns out to be surprisingly complicated as there is no API in C# for getting the "main" thread. This is because there are many ways to run C# in a way where there might not be a main thread (running C# in a COM component) or the runtime has no good way of knowing the main thread (calling C# from unmanaged code). So how does OpenTK check for the main thread? In a "best effort" manner. [Here is the code](https://github.com/opentk/opentk/blob/eab65e5c34abec4673b4672256e0e6c86018e3ad/src/OpenTK.Windowing.Desktop/GLFWProvider.cs#L85-L96):
{% highlight cs %}
MethodInfo correctEntryMethod = Assembly.GetEntryAssembly()?.EntryPoint;
StackTrace trace = new StackTrace();
StackFrame[] frames = trace.GetFrames();
for (int i = frames.Length - 1; i >= 0; i--)
{
    MethodBase method = frames[i].GetMethod();
    if (correctEntryMethod == method)
    {
        _mainThread = Thread.CurrentThread;
        break;
    }
}
{% endhighlight %}

The basic idea is to get a `MethodInfo` for the `Main` method and then to investigate the call stack of the current thread to see if the `Main` thread is in the call stack of the current thread. If it isn't part of the call stack we assume this isn't the main thread which means that we cannot initialize GLFW on this thread, so we will throw an exception.

This leads us nicely into the answer to the question this blog post started with. Why does `GLFWProvider.CheckForMainThread` exist? It exists because there can be scenarios where OpenTK can't properly detect the main thread (e.g. calling C# from unmanaged code). Without being able to set `GLFWProvider.CheckForMainThread = false` it would be impossible to initialize OpenTK in these scenarios, but with it we allow you to bypass this sanity check, at your own risk. 

[^most-functions]: [Some](https://www.glfw.org/docs/latest/intro.html#thread_safety) functions are fine to call on any thread.