---
layout: post
title:  "Generating a semantic version string for an IAR project in CMake"
date:   2022-12-29 11:30:00 +0100
categories: iar cmake linker version
---

Create the `version.h.in` for your IAR project:

{% highlight c %}
#pragma once

#define VERSION_MAJOR @versioning_VERSION_MAJOR@
#define VERSION_MINOR @versioning_VERSION_MINOR@
#define VERSION_PATCH @versioning_VERSION_PATCH@

#define STR_MSG(msg) #msg
#define STR(m) STR_MSG(m)
#define VERSIONFY(MAJOR, MINOR, PATCH, SEP) MAJOR STR(SEP) MINOR STR(SEP) PATCH

#define VERSION VERSIONFY(STR(VERSION_MAJOR), STR(VERSION_MINOR), STR(VERSION_PATCH))
{% endhighlight %}

The `versioning_VERSION_*` symbols are taken from the CMake configuration. Example for `CMakeLists.txt`:

{% highlight cmake %}
cmake_minimum_required(VERSION 3.25)

project(versioning VERSION 3.2.1)

add_executable(versioned)

target_sources(versioned main.c)

target_compile_options(versioned PRIVATE --cpu=cortex-m7)

target_link_options(versioned PRIVATE --semihosting --config config.icf)

configure_file(version.h.in version.h)
{% endhighlight %}
Example for `main.c`:

{% highlight c %}
#include <stdio.h>
#include "version.h"

void main()
{
    printf(VERSION);
}
{% endhighlight %}

It is even possible to use the CMake-provided project version in a dynamically configured linker configuration (`config.icf.in`):

{% highlight icf %}
/* all the other linker configuration stuff... */
define symbol APP_VERSION_MAJOR @versioning_VERSION_MAJOR@
define symbol APP_VERSION_MINOR @versioning_VERSION_MINOR@
define symbol APP_VERSION_PATCH @versioning_VERSION_PATCH@

if (APP_VERSION_MAJOR > 3)
{
   /* perform special flash partitioning */
}
{% endhighlight %}

Then add the following line to the `CMakeLists.txt`:

{% highlight cmake %}
configure_file(config.icf.in config.icf)
{% endhighlight %}
