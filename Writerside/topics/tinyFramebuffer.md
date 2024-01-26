# tinyFramebuffer

This is something I want to write since a long time. A better clone of tinyPTC.
Perhaps I'll start coding it if I add an entry here ?

All I want is a pointer to an array of pixel, and whatever I write in it is displayed on screen.
It really needs 3 functions:
- open
- update
- close

Everything else is optional. But I want to write a software renderer of course.

## Getting started

I created a "Desktop application" in Visual Studio. It creates some kind of default project with resources, menu, dialog.
Meh. Perhaps i should start with an empty project.

## Getting started, take 2

- Starting Visual Studio 2022, Empty C++ project, tinyFramebuffer.
- Add Class, tinyFramebuffer.cpp & .h.
- Project, properties (all configurations), linker, system, subsystem, change from console to windows.

### WIN32_LEAN_AND_MEAN

i do need windows.h (i think) but the smaller, the better.

WIN32_LEAN_AND_MEAN is a popular #define, but [according to Raymond Chen](https://devblogs.microsoft.com/oldnewthing/20091130-00/?p=15863), it doesn't have much use anymore.
It doesn't seems to hurt to define it anyway.

From Windows.h:
```c
#ifndef WIN32_LEAN_AND_MEAN
#include <cderr.h>
#include <dde.h>
#include <ddeml.h>
#include <dlgs.h>
#ifndef _MAC
#include <lzexpand.h>
#include <mmsystem.h>
#include <nb30.h>
#include <rpc.h>
#endif
#include <shellapi.h>
#ifndef _MAC
#include <winperf.h>
#include <winsock.h>
#endif
#ifndef NOCRYPT
#include <wincrypt.h>
#include <winefs.h>
#include <winscard.h>
#endif

#ifndef NOGDI
#ifndef _MAC
#include <winspool.h>
#ifdef INC_OLE1
#include <ole.h>
#else
#include <ole2.h>
#endif /* !INC_OLE1 */
#endif /* !MAC */
#include <commdlg.h>
#endif /* !NOGDI */
#endif /* WIN32_LEAN_AND_MEAN */
```

Ho well, we'll see later if it was a good idea.

### A basic "do nothing" winmain

O the joy of WinMain. I don't know why it's so complicated. 

![winmain.png](winmain.png)

Yes, yes, thank you Copilot. So much negativity. 

![positivity.png](positivity.png)

Right, wishful thinking from Copilot. We'll see how it goes.

```C++
// tinyFramebuffer.h
#pragma once

#define WIN32_LEAN_AND_MEAN

#include <windows.h>
#include <ddraw.h>
```

```C++
// tinyFramebuffer.cpp
#include "tinyFramebuffer.h"

int APIENTRY wWinMain(_In_ HINSTANCE hInstance,
    _In_opt_ HINSTANCE hPrevInstance,
    _In_ LPWSTR    lpCmdLine,
    _In_ int       nCmdShow)
{
    UNREFERENCED_PARAMETER(hPrevInstance);
    UNREFERENCED_PARAMETER(lpCmdLine);

    return 0;
}
```

Right, it compiles, do nothing, exit, without error. That's a good start.
And yes, the elephant in the room : #include <ddraw.h>. I guess i'll have to.

Unless I switch to "modern" Direct2D. (then that would be d2d1.h instead of ddraw.h)

## Sorting out the mess


~~I want all the windows stuff in tinyFramebuffer and the app in some kind of application.cpp.~~
~~What make it slightly difficult is that the winmain is in tinyFramebuffer, not in application.~~

~~- So at tinyFramebuffer startup we need to call our application Startup,~~
~~- at tinyFramebuffer update we need to call our application Update,~~
~~- and at tinyFramebuffer shutdown we need to call our application Shutdown.~~

~~I code the tinyFramebuffer thing, and the "application" is whatever it wants to be.~~
Or perhaps tinyFramebuffer should be a proper library.

I haven't the faintest idea how to make a windows library.
And more importantly, i don't know if can do what i need to do with a library.

I'll have to try later but for now let's make something that i know it can work.


