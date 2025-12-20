---
layout: post
title:  "Dealing with exceptions in callbacks"
date: 2025-12-20 17:00:00 +0100
categories: support-tips
tags: glfw pinvoke
author: noggin_bops
commentIssueId: 6
---
In the [last post]({% link _posts/2025-12-17-exceptions-and-pinvoke.markdown %}) we left off saying that OpenTK does deal with re-throwing exceptions to some degree. In this post we will explore what this means.

In the last post we concluded that throwing exceptions in callbacks called from native code is a Bad Idea™ and that code should make sure to make this not occur. I also suggested that any exception thrown in the callback could be captured and then later rethrow the exception when you've returned from the native code.

And this is in-fact something OpenTK does, just not for the GLFW error callback. On all events that are part of `NativeWindow` have this exception capturing and rethrowing behavior. We can see this by looking at [the code for one of these callbacks](https://github.com/opentk/opentk/blob/eab65e5c34abec4673b4672256e0e6c86018e3ad/src/OpenTK.Windowing.Desktop/NativeWindow.cs#L1220-L1230):
{% highlight cs %}
private unsafe void WindowPosCallback(Window* window, int x, int y)
{
    try
    {
        OnMove(new WindowPositionEventArgs(x, y));
    }
    catch (Exception e)
    {
        _callbackExceptions.Enqueue(ExceptionDispatchInfo.Capture(e));
    }
}
{% endhighlight %}

The key API we are using here is [`ExceptionDispatchInfo.Capture`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo.capture?view=net-10.0) to get an [`ExceptionDispatchInfo`](https://learn.microsoft.com/en-us/dotnet/api/system.exception?view=net-10.0) which will allow us to rethrow this exception with the original source location and stack trace that the exception was thrown from. Adding this dispatch info to a list will then allow us to re-throw these exceptions when we've returned from the native code. 

The thing that makes this feasible for the `NativeWindow` events but not for the GLFW error callback is that any function in GLFW is allowed to call the GLFW error callback. This means that we would have to add a rethrow check after every single GLFW function called. This would then add extra overhead for a case that should normally never happen. The `NativeWindow` event callbacks, however, only happen in response to a call to `glfwPollEvents` and similar functions (called [`ProcessWindowEvents`](https://opentk.net/api/OpenTK.Windowing.Desktop.NativeWindow.html#OpenTK_Windowing_Desktop_NativeWindow_ProcessWindowEvents_System_Boolean_) in `NativeWindow`). So, for these callbacks there is only a handful of functions where we need to check if we need to re-throw exceptions.

There is a complication however. Because GLFW is not able to receive any of the exception  information from OpenTK or C# GLFW will have no idea that an exception has been thrown and will continue event processing as normal, potentially calling other callbacks before finally returning back to the C# code that started the event processing, and only then will the exception be called. This makes this a very complicated scenario to deal with properly as some callbacks might be called in a "broken" state.

So what should you do? The simple solution is to simply avoid uncaught exceptions in callback functions entirely. That way you avoid all of these subtleties.



