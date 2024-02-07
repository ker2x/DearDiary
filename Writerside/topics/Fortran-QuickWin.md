# Fortran QuickWin

With the help of  [https://www.intel.com/content/www/us/en/docs/fortran-compiler/developer-reference-build-windows-applications/15-0/using-quickwin.html](https://www.intel.com/content/www/us/en/docs/fortran-compiler/developer-reference-build-windows-applications/15-0/using-quickwin.html)

The include directory is useful as well, on my system it's located at : `C:\Program Files (x86)\Intel\oneAPI\compiler\2024.0\opt\compiler\include`
and ifqwin.f90 in particular. (same directory)


- Install Intel OneAPI Toolkit, then install Intel HTC Tooolkit for the fortran compiler.
- Run VS2022
- Create a new project and select Fortran QuickWin Application
- In source file, add a new element : Fortran Free-form File

> Oops, I wrote some stuff, but I finally realized that I was doing a win32 app, not a quickwin app. So I deleted everything and started again.

The very minimalist code is :

```
program source1
    use IFQWIN
    implicit none
    
    TYPE (windowconfig) wc
    LOGICAL status

    status = SETWINDOWCONFIG(wc)

end
```

It displays a minimal default window, with a message box to confirm exit.

Something slightly more interesting :

```
program source1
    use IFQWIN

    implicit none
    
    TYPE (windowconfig) wc
    LOGICAL status
    ! Set the x & y pixels to 800X600 and font size to 8x12.
    wc%numxpixels  = 800                ! pixels on x-axis, window width
    wc%numypixels  = 600                ! pixels on y-axis, window height
    wc%numtextcols = -1                 ! -1 requests system default/calculation
    wc%numtextrows = -1
    wc%numcolors   = -1
    wc%title       = " "C
    wc%fontsize    = #0008000C          ! Request 8x12 pixel fonts
    status         = SETWINDOWCONFIG(wc)

    IF(.NOT.status) status = SETWINDOWCONFIG(wc)

end program source1
```

Running it as-is should display a window followed by a message box to confirm exit.

Most of it should be self-explanatory except the fontsize :
- The variable wc%fontsize is given as hexadecimal constant of #0008000C, which is interpreted in two parts:
    - The left side of 0008 (8) specifies the width of the font, in pixels.
    - The right side of 000C (12 in decimal) specifies the height of the font, in pixels.
- The requested font size is matched to the nearest available font size. If the matched size differs from the requested size, the matched size is used to determine the number of columns and rows.


If the requested configuration cannot be set, SETWINDOWCONFIG returns .FALSE. and calculates parameter values that will work and best fit the requested configuration. 
Another call to SETWINDOWCONFIG establishes default working values for the window.
