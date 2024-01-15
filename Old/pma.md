(extracted from the main diary)

#### Playing with PMA Labs

- Let's start with Lab01-01.exe.
- i'm even using _IDE Free 70_ instead of my licensed version.
- According to "_detect it easy_" it's a 32bits PE executable, unpacked, compiled with MSVC 6.0
- Opening it in IDA with default option
- Only the EntryPoint is exported, it import kernel32 and msvcrt
- Some usefull strings

```
WARNING_THIS_WILL_DESTROY_YOUR_MACHINE
C:\Windows\System32\Kernel32.dll
Lab01-01.dll
Kernel32.
C:\windows\system32\kerne132.dll <- it's one three two, not L 3 2
kernel32.dll
kerne132.dll <- it's one three two, not L 3 2
```

- it's easy to guess what's going to happens if you execute it. it will destroy/replace your kernel32.dll
- Opening the Entry Point in IDA. Nothing unusual, it appears to be some standard init.
- Before exiting, it call a sub, which should be our WinMain and the var pushed before the call should be our usual args.

```
...
.text:004018EA                 call    ds:__p___initenv
.text:004018F0                 mov     ecx, [ebp+var_20]
.text:004018F3                 mov     [eax], ecx
.text:004018F5                 push    [ebp+var_20]
.text:004018F8                 push    [ebp+var_2C]
.text:004018FB                 push    [ebp+var_1C]
.text:004018FE                 call    sub_401440   ;<- this should be our WinMain
.text:00401903                 add     esp, 30h
.text:00401906                 mov     [ebp+var_24], eax
.text:00401909                 push    eax             ; Code
.text:0040190A                 call    ds:exit
```

Let's rename it to WinMain and let IDA do its magic (the stuff we're willing to pay for)

```
...
.text:004018EA                 call    ds:__p___initenv
.text:004018F0                 mov     ecx, [ebp+lpCmdLine]
.text:004018F3                 mov     [eax], ecx
.text:004018F5                 push    [ebp+lpCmdLine] ; lpCmdLine
.text:004018F8                 push    [ebp+hPrevInstance] ; hPrevInstance
.text:004018FB                 push    [ebp+hInstance] ; hInstance
.text:004018FE                 call    WinMain
.text:00401903                 add     esp, 30h
.text:00401906                 mov     [ebp+var_24], eax
.text:00401909                 push    eax             ; Code
.text:0040190A                 call    ds:exit
```

Jumping into WinMain, it feels wrong. Let's call it main instead.

```
...
call    _initterm
call    ds:__p___initenv
mov     ecx, [ebp+envp]
mov     [eax], ecx
push    [ebp+envp]      ; envp
push    [ebp+argv]      ; argv
push    [ebp+argc]      ; argc
call    main
add     esp, 30h
mov     [ebp+var_24], eax
push    eax             ; Code
call    ds:exit
```

Isn't it much better ? _initenv_ (whatever that is) create _envp_. now it feels right.

The first few lines of our main(int argc, const char **argv, const char **envp), commented

```
mov     eax, [esp+argc]
sub     esp, 44h        ; reserve space on stack for local var
cmp     eax, 2          ; argc == 2 ?
push    ebx             ; save our usual stuff
push    ebp
push    esi
push    edi
jnz     loc_401813      ; jump according to the result of cmp
```

Let's explain, because i have time. remove the book-keeping stuff and focus on the user code.
```
mov     eax, [esp+argc]
cmp     eax, 2          ; argc == 2 ?
jnz     loc_401813      ; jump according to the result of cmp
```

- eax = argc. nothing to see here. argc is the number of argument. 1 mean "no argument" (the 1st arg is the program name), 2 mean, of course, 1 argument
- compare eax with 2. So we can guess it's expecting to be called with an argument in command line.
- cmp set the ZF and CF flag according to the result of the comparison.

| cmp dst, src | ZF | CF |
|--------------|----|----|
| dst = src    | 1  |  0 |
| dst < src	   | 0  | 1  |
| dst > src	   | 0  | 0  | 

So if _argc == 2_ then ZF should be 1

Next is JNZ (Jump Non Zero), also known as JNE (Jump Not Equal)
```
jnz : jumps to the specified location if the Zero Flag (ZF) is cleared (0).
jnz is commonly used to explicitly test for something not being equal to zero whereas jne is commonly found after a cmp instruction.
```

I'll be honest here. i always get confused by JNZ. it jump if ZF = 0. But if you think of it as being "JNE" it's much easier.

Anyway : ```if(argc != 2) { goto loc_401813; }```

```loc_401813:
pop     edi
pop     esi
pop     ebp
xor     eax, eax    ; eax = 0
pop     ebx
add     esp, 44h
retn
```

- it jump directy to the end of our main.
  Therefore, because eax = 0, we get something like : ```if(argc != 2) { return 0; }```
- our "malware" wont work without argument. It's probably a security measure because this is a fake malware for educational purpose.

What's happening if argc = 2 ?
```
mov     eax, [esp+54h+argv]
mov     esi, offset aWarningThisWil ; "WARNING_THIS_WILL_DESTROY_YOUR_MACHINE"
mov     eax, [eax+4]
```

- First, it get a pointer to argv (which contain our arguments from command line)
- next, esi will point to a string, esi is often used for loop
- finally, eax = eax+4. we're in 32bit, a pointer is 4 byte long.
  Basically, eax will now point to argv[1] instead of argv[0]
- we have a string, a loop and argv[1]. easy guess : it will check if the exe will be called like this :

```
lab01.exe WARNING_THIS_WILL_DESTROY_YOUR_MACHINE
```

I'm skipping a bunch of mov, cmp, test, loop, with a final jnz going straight to exit if the comparaison fail.

For the curious :

![](../img/idapma01.png)

Next, all the following jnz goes to _exit_ so i'll skip it :

```
mov     edi, ds:CreateFileA
push    eax             ; hTemplateFile
push    eax             ; dwFlagsAndAttributes
push    3               ; dwCreationDisposition
push    eax             ; lpSecurityAttributes
push    1               ; dwShareMode
push    80000000h       ; dwDesiredAccess
push    offset FileName ; "C:\\Windows\\System32\\Kernel32.dll"
call    edi ; CreateFileA
mov     ebx, ds:CreateFileMappingA
push    0               ; lpName
push    0               ; dwMaximumSizeLow
push    0               ; dwMaximumSizeHigh
push    2               ; flProtect
push    0               ; lpFileMappingAttributes
push    eax             ; hFile
mov     [esp+6Ch+hObject], eax
call    ebx ; CreateFileMappingA
mov     ebp, ds:MapViewOfFile
push    0               ; dwNumberOfBytesToMap
push    0               ; dwFileOffsetLow
push    0               ; dwFileOffsetHigh
push    4               ; dwDesiredAccess
push    eax             ; hFileMappingObject
call    ebp ; MapViewOfFile
push    0               ; hTemplateFile
push    0               ; dwFlagsAndAttributes
push    3               ; dwCreationDisposition
push    0               ; lpSecurityAttributes
push    1               ; dwShareMode
mov     esi, eax
push    10000000h       ; dwDesiredAccess
push    offset ExistingFileName ; "Lab01-01.dll"
mov     [esp+70h+argc], esi
call    edi ; CreateFileA
cmp     eax, 0FFFFFFFFh
mov     [esp+54h+var_4], eax
push    0               ; lpName
jnz     short loc_401503

loc_401503:             ; dwMaximumSizeLow
push    0
push    0               ; dwMaximumSizeHigh
push    4               ; flProtect
push    0               ; lpFileMappingAttributes
push    eax             ; hFile
call    ebx ; CreateFileMappingA
cmp     eax, 0FFFFFFFFh
push    0               ; dwNumberOfBytesToMap
jnz     short loc_40151B

loc_40151B:             ; dwFileOffsetLow
push    0
push    0               ; dwFileOffsetHigh
push    0F001Fh         ; dwDesiredAccess
push    eax             ; hFileMappingObject
call    ebp ; MapViewOfFile
mov     ebp, eax
test    ebp, ebp
mov     [esp+54h+argv], ebp
jnz     short loc_401538
```


Thank you IDA for knowing the win32 api <3

- Hey... did you know that writing this take forever ?
- i'm supposed to be on a break.
- Enough for now. All the calls above speak for themselves. go read MSDN to know more :)


---

### 2021/03/32

- Dear diary, this day doesn't exist (citation needed)

---

### 2021/04/01

- Dear Diary, new COVID lockdown, again ... meh... :[
- Anyway, back to the previous exercise.

- _MapViewOfFile_ : https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile
- _CreateFileMappingA_ : https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfilemappinga
- _CreateFileA_ : https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea

About CreateFileA dwDesiredAccess :

|  dwDesiredAccess  |   Mask     |
|-------------------|------------|
|  GENERIC_READ     | 0x80000000 |
|  GENERIC_WRITE    | 0x40000000 |
|  GENERIC_EXECUTE  | 0x20000000 |
|  GENERIC_ALL      | 0x10000000 |

- _CreateFileA_ return a file handle (in eax, return value are always in eax),
- then _CreateFileMappingA_ is passing this eax as _hFile_ argument and return a file handle (HANDLE) in eax
- which is used as the _hFileMappingObject_ argument.
- finally, _CreateFileMappingA_ return a LPVOID
    - If the function succeeds, the return value is the starting address of the mapped view.
    - If the function fails, the return value is NULL. To get extended error information, call GetLastError.
- TL;DR : _"C:\\Windows\\System32\\Kernel32.dll"_ is mapped in memory

- _Lab01-01.dll_ is created with CreateFileA, follow by CreateFileMappingA & MapViewOfFile. same as above.

I'm not at home with my IDA, VM, Windows, etc... so that's it for now. i'll have to go home to read the rest of the code.

---

Dear Diary, i'm home again.

I'm reading the next piece of code and it's going to get a little bit messy.
The reason is that it call a bunch of sub_xxxxxx and i'll also need to rename some local var.
It's not difficult to guess what's going to happens, but since i'm writing this diary i'll push myself to reverse every part of it.

The list of local vars :
```
var_44= dword ptr -44h
var_40= dword ptr -40h
var_3C= dword ptr -3Ch
var_38= dword ptr -38h
var_34= dword ptr -34h
var_30= dword ptr -30h
var_2C= dword ptr -2Ch
var_28= dword ptr -28h
var_24= dword ptr -24h
var_20= dword ptr -20h
var_1C= dword ptr -1Ch
var_18= dword ptr -18h
var_14= dword ptr -14h
var_10= dword ptr -10h
var_C= dword ptr -0Ch
hObject= dword ptr -8
var_4= dword ptr -4
argc= dword ptr  4
argv= dword ptr  8
envp= dword ptr  0Ch
```

With a quick look at the code, i have no way to really what the vars means so i'll have to reverse every sub one by one first.
Luckily, we have only 7 sub_, which is very very low.

After some browsing each of them, this is a small one. i already renamed it :

```
call_controlfp proc near
push    30000h          ; Mask
push    10000h          ; NewValue
call    _controlfp
pop     ecx
pop     ecx
retn
call_controlfp endp
```

But it's only called by the entry point, which isn't user code so we can safely ignore it.
```_controlfp : Gets and sets the floating-point control word.```

Another one, renamed it, also called by the entry point :

```
return_zero proc near
xor     eax, eax
retn
return_zero endp
```
We're down to 5 sub.
3 of them do not do any external call to dll's function.
2 of them call kernel32.dll (& msvcrt ?).

- sub_4010A0 (renamed to : read_file)
    - called by : sub_4011E0 (which i renamed to search_file)
    - system calls : CreateFileA, CreateFileMappingA, MapViewOfFile, IsBadReadPtr, UnmapViewOfFile, CloseHandle, _stricmp

- sub_4011E0 (renamed to : search_file)
    - called by : main, sub_4011E0 (yes, it's calling itself)
    - system calls : FindFirstFileA, malloc, _stricmp, FindClose, FindNextFileA,

- For now i'll just rename "sub_4011E0" to "search_file" and "sub_4010A0" to "read_file".
  Considering how it also call "Create_File" it's probably not just "reading" files, but for now i'll keep it as is.
  It's just a random guess according to the calls it's doing.
- So, "main" call "search file" which is calling "read_file". I wish i could do graphs, but github doesn't support it.
- "read_file" also call "sub_401040"
- "sub_401040" only call "401000"

I guess it's time for a graph, i'll use the IDA's proximity browser.

![](../img/ida_proxi.png)

Not bad, huh ? So, what is sub_401000 ? there is a loop in it, but here is the first part before the loop.

```
mov     edx, [esp+arg_4] ; what is arg_4 ?
xor     eax, eax        ; eax = 0
xor     ecx, ecx        ; ecx = 0
push    ebx
mov     ax, [edx+14h]
mov     cx, [edx+6]
push    esi
xor     esi, esi        ; esi = 0
test    ecx, ecx
push    edi
lea     eax, [eax+edx+18h]
jle     short loc_401039
```

Let's remove some stuff for now :

```
mov     edx, [esp+arg_4] ; what is arg_4 ?
xor     ecx, ecx         ; ecx = 0
mov     cx, [edx+6]      ; <- what's at edx+6 ?
test    ecx, ecx         ; if (ecx == 0)
jle     short loc_401039 ; then jump to loc_401039
```

- Haaa what a pain ! "cx" is the the lower 16bit of the 32bit ecx register.
- Basically it seems to test if whatever is at edx+6 == 0
- "whatever is at edx+6" is defined by esp+arg_4.
- So, again, what is arg_4 ?

Dear Diary, i'm going to eat before i forget to, and read some manga. (Beware of the villainess)

See you tomorrow. i'm forcing myself to take a break.

---

### 2021/04/02

- Dear Diary, i just tested IDA Free 70 on my mac M1 : it works.
- The part of the code where i stopped yesterday still upset me.
- why +6 ? why 16 bits ? it's important to give up and keep going, perhaps i'll know why later.
  Perhaps it does not matter and it's just the compiler doing compiler stuff.
  But the purpose of the Diary is to go in depth. Or not... it's just my notepad, i do wtf i want to.
- For some reason i can't extract the PMA Labs on my M1, it says the password is incorrect.
- it seems to be a known problem. i installed "keka" and the extraction works.
- TL;DR : i can keep on reversing this exe on my mac now
- To be honnest, this is the kind of code where a decompiler would be helpful.
- And i have one, hopper disassembler. does it help ?

```
int sub_401000(int arg0, int arg1) {
    var_4 = arg0;
    stack[-4] = ebx;
    eax = *(int16_t *)(arg1 + 0x14);
    ecx = *(int16_t *)(arg1 + 0x6);
    stack[-8] = esi;
    esp = esp - 0xc;
    stack[-12] = edi;
    eax = eax + arg1 + 0x18;
    if (ecx <= 0x0) goto loc_401039;

loc_40101d:
    esi = 0x0;
    edi = var_4;
    goto loc_401021;

loc_401021:
    edx = *(eax + 0xc);
    if ((edi < edx) || (edi >= *(eax + 0x8) + edx)) goto loc_401031;

.l1:
    return eax;

loc_401031:
    esi = esi + 0x1;
    eax = eax + 0x28;
    if (esi < ecx) goto loc_401021;

loc_401039:
    eax = 0x0;
    return eax;
}
```

Meh ...

By the way, if we go back near the end of the the main :

```
push    offset NewFileName ; "C:\\windows\\system32\\kerne132.dll"
push    offset ExistingFileName ; "Lab01-01.dll"
call    ds:CopyFileA
```

Just saying.

Anyway... i have a little bit of dilema here. is it my diary or does it become some kind of tutorial ?
If it was just me, i would have happily just ignored this annoying sub. it doesn't seems to matter that much.
I can always come back to it later, or not. does it even matter ?

i really want to go check that dll instead. And drink coffee too.

---

#### Lab01.dll

- It import kernel32 (createProcess, sleep, createmutex, ...), msvcrt, and ws2_32. nice :]
- Isn't it much more fun ? ws2 is winsock <3 internet stuff \o/ https://docs.microsoft.com/en-us/windows/win32/winsock/windows-sockets-start-page-2
- it export the EntryPoint only, what least that's what IDA says, and hopper agrees.
- strings are : "hello", "sleep", "SADFHUHF", "172.26.152.13"

A quick look at the main function tell me that :
- It create a mutex named SADFHUHF
- it open a socket, send "hello"
- wait to receive something
    - "sleep" : call _Sleep_
    - "exec" : call _CreateProcessA_
    - "q" : call _CloseHandle_ and exit
- loop back to hello

I don't think i need to do any intensive reverse engineering here. it's a 5mn job.

---

### Lab01_02.exe

Opening in IDA :

```
The imports segment seems to be destroyed. This MAY mean that
the file was packed or otherwise modified in order to make it
more difficult to analyze. If you want to see the imports
segment in the original form, please reload it with the
'make imports section' checkbox cleared.
```

- Nice <3
- Only function found, the entry point.
- Import : LoadLibraryA, GetProcAddress, VirtualProtect, VirtualAlloc, VirtualFree, ExitProcess, CreateServiceA, exit, InternetOpenA
- The imports are super suspicious of course. the broken PE too.
- Conclusion : it's probably UPX packed.
- Strings : Kernel32.dll, Advapi32.dll, msvcrt.dll, wininet.dll
- I guess i have to explain a little bit what's happening here ?

- It's very easy, it's trying to load dll dynamically  :
    - LoadLibraryA :
        - https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibrarya
        - Loads the specified module into the address space of the calling process. The specified module may cause other modules to be loaded.
        - HMODULE LoadLibraryA( LPCSTR lpLibFileName );
        - lpLibFileName : The name of the module. This can be either a library module (a .dll file) or an executable module (an .exe file). The name specified is the file name of the module and is not related to the name stored in the library module itself, as specified by the LIBRARY keyword in the module-definition (.def) file.
    - GetProcAddress :
        - https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress
        - Retrieves the address of an exported function or variable from the specified dynamic-link library (DLL).
        - FARPROC GetProcAddress( HMODULE hModule, LPCSTR  lpProcName );
        - hModule : A handle to the DLL module that contains the function or variable.
        - lpProcName : The function or variable name, or the function's ordinal value.
        - If the function succeeds, the return value is the address of the exported function or variable.
        - If the function fails, the return value is NULL.

Easy ? Right ? Instead of importing a function from a dll, you load it at runtime.

- About the broken PE now :
    - IDA -> View -> Open Subviews -> Segments
        - UPX0
        - UPX1
        - UPX2
        - .idata
        - UPX2
    - Toldyaso, it's UPX packed. There is an IDA plugin. Named "Universal Unpacker" or something but i need my licensed IDA verion.
    - And it's on my computer at home. My mac run IDA Free 70. Pooh.
    - i don't want to bother trying to unpack it on my mac when in can do it at home. i'll do something else instead.
