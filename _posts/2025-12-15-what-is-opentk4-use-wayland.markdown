---
layout: post
title:  "What is `OPENTK4_USE_WAYLAND` and what does it do?"
date: 2025-12-15 13:00:00 +0100
categories: support-tips
tags: glfw linux
author: noggin_bops
commentIssueId: 3
---

For the first post on this blog I want to explore the obscure `OPENTK4_USE_WAYLAND` environment variable and how it interacts with OpenTK.

OpenTK 4.x uses GLFW for creating and working with windows.
Initially GLFW had no Wayland support and relied on XWayland for Wayland compatibility.

This changed with GLFW 3.3 where you could compile GLFW for either X11 or Wayland, which produced two different `.so` files.
Wayland support in GLFW 3.3 was "half-baked" and many APIs in GLFW are not possible to implement in Wayland due to restrictions that Wayland imposes.
This meant that always defaulting to Wayland in OpenTK would cause a bunch of issues due to GLFW throwing errors on simple operations such as constructing a `NativeWindow`.

Having two different `.so` files meant that the decision of using Wayland or relying on XWayland need to be made when loading the GLFW `.so` file.
To make this decision OpenTK 4 uses `XDG_SESSION_TYPE` to detect that it is running under wayland, and then considers `OPENTK4_USE_WAYLAND` to decide if the wayland version of GLFW should be loaded. This means that in OpenTK `4.8.0+` wayland is opt-in through `OPENTK4_USE_WAYLAND=1`.

GLFW 3.4 changed things up with first party support for Wayland. Now GLFW could compile for both X11 and Wayland support in the same `.so` file.
This introduced a new api in GLFW to select which backend to use. On wayland platforms GLFW would now default to using the Wayland backend and you have to opt-out of using Wayland through the new GLFW api. So in OpenTK `4.9.1` the behaviour of `OPENTK4_USE_WAYLAND` was changed so that `OPENTK4_USE_WAYLAND=0` would opt-out of using wayland when running under wayland. At the same time the [`GLFWProvider.HonorOpenTK4UseWayland`](https://opentk.net/api/OpenTK.Windowing.Desktop.GLFWProvider.html#OpenTK_Windowing_Desktop_GLFWProvider_HonorOpenTK4UseWayland) property was introduced, which allows users to tell OpenTK to ignore the `OPENTK4_USE_WAYLAND` environment variable all together and behave like the environment variable is not set.

This table breaks down the behaviour of `OPENTK4_USE_WAYLAND` for different OpenTK versions:

|---------------------------------------------------------------------------------------------------------------------------------|
| OpenTK version | Not set | `OPENTK4_USE_WAYLAND=0` | `OPENTK4_USE_WAYLAND=1`                                                    |
|---------------------------------------------------------------------------------------------------------------------------------|
| `< 4.8.0`  | Ignored, no Wayland support in GLFW. | Ignored, no Wayland support in GLFW. | Ignored, no Wayland support in GLFW. |
| `>= 4.8.0` | X11 always | X11 always  | Wayland if `XDG_SESSION_TYPE=wayland`                                                   |
| `>= 4.9.1` | X11 or Wayland depending on platform | X11 always | X11 or Wayland depending on platform                           |
|---------------------------------------------------------------------------------------------------------------------------------|

In the [next post]({% post_url 2025-12-16-the-glfw-error-callback %}) we will look at the GLFW error callback, how OpenTKs default error callback has changed through the versions and why you might want to install your own error callback.