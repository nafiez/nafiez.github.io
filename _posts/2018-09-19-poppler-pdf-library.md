---
layout: post
title:  "Poppler (PDF) Library - Memory Corruption"
date:   2018-09-19 03:39:03 +0700
categories: [security, corruption]
---

Vulnerability Details
---------------------
In 2015, I found a memory corruption in a different type of PDF reader / viewer. The issue found from fuzzing the Poppler library.

Further digging the issue found that "ExtGState" failed to process and perform checking for valid blend mode. Thus, resulting to 
crash (can trigger Denial-of-Service).  

Issue was tested on: Ubuntu 14.04.3 LTS (trusty)
PDF viewer / reader: ```diffpdf, evince, gpdftext, pdf-presenter-console, xpdf, zathura_viewer, qpdfview```

Crash information from various type of PDF viewer:

1. Crash on Zathura PDF Reader
```
#0  0xb7fdccb0 in ?? ()
#1  0xb733133a in malloc_printerr (action=<optimized out>, str=0xb741fa6e "malloc(): memory corruption", ptr=0xb567f538) at malloc.c:4996
#2  0xb7332ee9 in _int_malloc (av=av@entry=0xb5600010, bytes=bytes@entry=0x20) at malloc.c:3447
#3  0xb7334708 in __GI___libc_malloc (bytes=0x20) at malloc.c:2891
#4  0xb51cf8b7 in operator new(unsigned int) () from /usr/lib/i386-linux-gnu/libstdc++.so.6
#5  0xb53af68c in GooString::formatv(char const*, char*) () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#6  0xb5309d44 in error(ErrorCategory, long long, char const*, ...) () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#7  0xb53127c0 in ExponentialFunction::ExponentialFunction(Object*, Dict*) () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#8  0xb5317c46 in Function::parse(Object*, std::set<int, std::less<int>, std::allocator<int> >*) () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#9  0xb5317552 in StitchingFunction::StitchingFunction(Object*, Dict*, std::set<int, std::less<int>, std::allocator<int> >*) () from /usr/lib/i386-linux-gnu/libpoppler.so.44
```
2. Crash on Evince PDF Reader
```
#0  0xb77d7cb0 in ?? ()
#1  0xb6aa533a in malloc_printerr (action=<optimized out>, str=0xb6b98058 "free(): invalid next size (normal)", ptr=0xb4246008) at malloc.c:4996
#2  0xb6aa5fad in _int_free (av=0xb4200010, p=<optimized out>, have_lock=0x0) at malloc.c:3840
#3  0xb629482f in operator delete(void*) () from /usr/lib/i386-linux-gnu/libstdc++.so.6
#4  0xb0a0ccc4 in ExponentialFunction::~ExponentialFunction() () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#5  0xb0a0ceb7 in StitchingFunction::~StitchingFunction() () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#6  0xb0a0cf2c in StitchingFunction::~StitchingFunction() () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#7  0xb0a13b7a in Function::parse(Object*, std::set<int, std::less<int>, std::allocator<int> >*) () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#8  0xb0a13cf9 in Function::parse(Object*) () from /usr/lib/i386-linux-gnu/libpoppler.so.44
#9  0xb0a3e9d5 in GfxAxialShading::parse(Dict*, OutputDev*) () from /usr/lib/i386-linux-gnu/libpoppler.so.44
```
Fixed
-----
https://cgit.freedesktop.org/poppler/poppler/commit/?id=b3425dd3261679958cd56c0f71995c15d2124433
