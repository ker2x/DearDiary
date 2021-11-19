# Dear Diary,

## My security oriented Diary

- i found a bug in sudo while studying CVE-2021-3156. accepted and patched. https://github.com/sudo-project/sudo/issues/95
- i'm studying/explaining Malware from PMA Labs using IDA and Hopper
- i tried the sstic challenge (finished 12th on the 1st flag, couldn't find the others)
- on iatus to code stuff and learn more languages
- back with more malware

### 2021/03/30

- Dear Diary, today i'm starting a diary
- Dear Diary, i started you.
- I'm watching youtube video about Fuzzing & Buffer Overflow : https://www.youtube.com/watch?v=FCIfWTAtPr0
  - the video is too beginner oriented, i'll give a try to part 4 (finding the offset) anyway.
      Part 5 is about EIP. 
  - Not really useful to me (a priori), but the videos are very short so it's cool.
- I'm planning to give a try to sstic challenge 2021, it start this weekend : https://www.sstic.org/2021/news/
- i did some ARM64 disassembly stuff this morning while drinking my first coffee of the day, using hopper disasm. 
Turn out it's much easier when the source code isn't written in swift(c)(r)(tm)
  
- CVE-2021-3156 is scary. Congratz Qualys for finding it.
  - https://www.kb.cert.org/vuls/id/794544
  - Apple’s Big Sur is also vulnerable, as well as cisco, netapp, juniper, ...
  - Also, https://www.youtube.com/watch?v=2_ZaNBl6qNo
    - i like this channel, i subbed some times ago, and the discord dudes are cools too
- mysql suck :[
  - why is it so bad ?
- should i buy Hopper disasm ?
- i want to go home. i'm going home.
- i'm home
  
#### Exploring CVE-2021-3156 @ home

This is what i understood : 
- You can use multiple line in argument by escaping with \
- Sudo ignore the character following \
- what if \ is the last character ? it ignores \0 (NULL) and read stuff it shouldn't read because the null terminator is ignored.

```
Sudo before 1.9.5p2 contains an off-by-one error that can result in a heap-based buffer overflow, 
which allows privilege escalation to root via "sudoedit -s" 
and a command-line argument that ends with a single backslash character.
```

Let's find out.

- The commit fixing the bug is here https://github.com/sudo-project/sudo/commit/1f8638577d0c80a4ff864a2aad80a0d95488e9a8
- and here https://github.com/sudo-project/sudo/commit/b301b46b79c6e2a76d530fa36d05992e74952ee8
- and ... here ? https://github.com/sudo-project/sudo/commit/c4d384082fdbc8406cf19e08d05db4cded920a55
- This was also submitted by Qualys the same day, let's assume it's part of it https://github.com/sudo-project/sudo/commit/c0eecf85c8b0920a9398920d5f5dae0ee2804b46

Well... it's not as simple as "_we forgot that having a backlash as last character could happens_". No no no.
It's an error with flags and stuff, because sudo DO check for this. (i'll have to check for sure but i assume it does)

Also, sudo and sudoedit are the same binary. sudoedit is just a symlink to sudo, 
and sudo is checking its own name to set some flags here and there in order to behave as "sudo" or as "sudoedit".
Well, that's what i understood anyway. i'll check, of course.

My immediate thought was : what if it's called neither "sudo" nor "sudoedit" ? huh ? huh ? _aren't i smart_ ?
Well... i'm not the smartest one. 

This is the patch published 3 days after the CVE fix : https://github.com/sudo-project/sudo/commit/19d5845f8b6ae429a597d53c7f8201514537b590
```The program name may now only be "sudo" or "sudoedit".```

Anyway... code stuff !

The old code :
```c
    /* Pass progname to plugin so it can call initprogname() */
    progname = getprogname();
    sudo_settings[ARG_PROGNAME].value = progname;

    /* First, check to see if we were invoked as "sudoedit". */
    proglen = strlen(progname);
    if (proglen > 4 && strcmp(progname + proglen - 4, "edit") == 0) {
	progname = "sudoedit";
	mode = MODE_EDIT;
	sudo_settings[ARG_SUDOEDIT].value = "true";
    }
```

This is honestly a weird way to check if it's invoked as "sudoedit". It could technically be "anythinglongerthan4edit".
Why ? dunno ! Probably very _legacy_.

The new check is as easy as :
```c
    /* The plugin API includes the program name (either sudo or sudoedit). */
    progname = getprogname();
    sudo_settings[ARG_PROGNAME].value = progname;

    /* First, check to see if we were invoked as "sudoedit". */
    if (strcmp(progname, "sudoedit") == 0) {
	mode = MODE_EDIT;
	sudo_settings[ARG_SUDOEDIT].value = "true";
	valid_flags = EDIT_VALID_FLAGS;
    }
```

While we're here we can clearly see one bigass fix : 
```valid_flags = EDIT_VALID_FLAGS;```

Well... i know it is because youtube told me it was about flags. :]

The new code is still weird, imho.
It says : _The plugin API includes the program name (either sudo or sudoedit)._
But this is not what the code is doing. it checks if it's "_sudoedit_" or "_anything else_". 

Anyway... i'm on my windows, i have WSL and ubuntu installed.

```bash
sudoedit --version
sudoedit: Only one of the -e, -h, -i, -K, -l, -s, -v or -V options may be specified
usage: sudoedit [-AknS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p prompt] [-T timeout] [-u user] file ...
```

huh ? okay then ...

```bash
sudoedit -V
sudoedit: Only one of the -e, -h, -i, -K, -l, -s, -v or -V options may be specified
usage: sudoedit [-AknS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p prompt] [-T timeout] [-u user] file ...
```

C'mooon... really ?

how about this ?
```bash
sudo -V
Sudo version 1.8.31
Sudoers policy plugin version 1.8.31
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.31
```

I tried on my mac M1. "_sudo -V_" works. and _sudoedit_ is "_command not found_".

I'm starting up a linux cloud instance.

```bash
sudo -V
Sudo version 1.8.21p2
+ a hundred lines of stuff
```

```bash
sudoedit -V
sudoedit: Only one of the -e, -h, -i, -K, -l, -s, -v or -V options may be specified
usage: sudoedit [-AknS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p prompt] [-T timeout] [-u user] file ...
```

nice ! i found a boring bug ! :o)

- it should not happen. the usage() clearly say that -v and -V are valid. 
- There are some other potential issues still happening with sudoedit. could it lead to a security issue ? dunno.

So let's check parse_args.c, again.

```c
    /* Is someone trying something funny? */
    if (argc <= 0)
	usage();
```

No, i'm not. I'm checking all calls to usage() from parse_args.

```
		case 'V':
		    if (mode && mode != MODE_VERSION)
			usage_excl();
		    mode = MODE_VERSION;
		    valid_flags = 0;
		    break;
		default:
		    usage();
```

mmm ?
Let's try something.

```
# sudo -z
sudo: invalid option -- 'z'
usage: sudo -h | -K | -k | -V
usage: sudo -v [-AknS] [-g group] [-h host] [-p prompt] [-u user]
usage: sudo -l [-AknS] [-g group] [-h host] [-p prompt] [-U user] [-u user] [command]
usage: sudo [-AbEHknPS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p prompt] [-T timeout] [-u user] [VAR=value] [-i|-s] [<command>]
usage: sudo -e [-AknS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p prompt] [-T timeout] [-u user] file ...

sudoedit -z
sudoedit: invalid option -- 'z'
usage: sudoedit [-AknS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p prompt] [-T timeout] [-u user] file ...
```

ho so there are two different usage() for sudo and sudoedit.

```
/*
 * Tell which options are mutually exclusive and exit.
 */
static void
usage_excl(void)
{
    debug_decl(usage_excl, SUDO_DEBUG_ARGS);

    sudo_warnx("%s",
	U_("Only one of the -e, -h, -i, -K, -l, -s, -v or -V options may be specified"));
    usage();
}
```

Okay ... actually so the warning i get isn't from usage() but from usage_excl()

Let's check the 1.8.21p2 source code just in case

The source i got from apt source shows : 
```
case 'V':
    if (mode && mode != MODE_VERSION)
        usage_excl(1);
    mode = MODE_VERSION;
    valid_flags = 0;
    break;
default:
    usage(1);
```

```
/*
 * Tell which options are mutually exclusive and exit.
 */
static void
usage_excl(int fatal)
{
    debug_decl(usage_excl, SUDO_DEBUG_ARGS)

    sudo_warnx(U_("Only one of the -e, -h, -i, -K, -l, -s, -v or -V options may be specified"));
    usage(fatal);
}
```

It's pretty much the same i guess ?

The source also show that the main bug that issued the CVE is fixed :
```
    /* First, check to see if we were invoked as "sudoedit". */
    proglen = strlen(progname);
    if (proglen > 4 && strcmp(progname + proglen - 4, "edit") == 0) {
        progname = "sudoedit";
        mode = MODE_EDIT;
        sudo_settings[ARG_SUDOEDIT].value = "true";
        valid_flags = EDIT_VALID_FLAGS;
    }
```

The flag is here. mode is set and different than MODE_VERSION so it indeed show usage_excl().
But why ?

After some time thinking about it, it's obvious :

```
		case 'V':
		    if (mode && mode != MODE_VERSION)
			usage_excl();
		    mode = MODE_VERSION;
		    valid_flags = 0;
		    break;
```

But why does it call usage_excl() here ?

why is mode set but it's clearly not MODE_VERSION ?

Because sudoedit set ```mode = MODE_EDIT;```

I'll submit a bug report and see how it goes.

First day, first bug. yay \o/

Bug report : https://github.com/sudo-project/sudo/issues/95

---

- That was one big messy daily report. I wanted to explore a CVE and found a (minor) but instead.
- That's even better, isn't it ?
- Zzzz
- Or not, my bug was (partially :( )) fixed and closed already.
- and i found another problem (pretty much the same bug actually) when calling sudoedit -h
- I think i understand why there is no sudoedit on mac. it shouldn't exist in the first place, imho.
- When you read this diary you cleary see that i wrote stuff that were clearly incorrect. i'm not removing it. 
  it accurately describe my thought process and i'm not always right on first try.
- Zzz ?

---

### 2021/03/31

- Dear diary, it's morning already. I'm not working today. More CVE ?
- It turn out that Macos also have (had) the sudoedit problem. 
  - sudoedit does not exist by default on mac but the code is still here anyway
  - just create a sudoedit symlink ```ln -s /usr/bin/sudo sudoedit```
  - it does not require any privilege to be created
  - the sudoedit code is still in the sudo binary
  
```bash
% ln -s /usr/bin/sudo sudoedit                   
% ./sudoedit -V
sudoedit: Only one of the -e, -h, -i, -K, -l, -s, -v or -V options may be specified
usage: sudoedit [-AknS] [-C num] [-D directory] [-g group] [-h host] [-p prompt] [-R directory] [-T timeout] [-u user] file ...
% rm sudoedit
```

<!-- img src="./img/cyberexperience.jpg" alt="drawing" width="200"/ -->

- My bugreport has been closed with a patch that check if progname is "sudoedit"
- They also added a seperate getopt config for sudoedit

```
case 'V':
    if (mode && mode != MODE_VERSION) {
    if (strcmp(progname, "sudoedit") != 0)
        usage_excl();
    }
```

```
static const char sudo_short_opts[] = "+Aa:BbC:c:D:Eeg:Hh::iKklnPp:R:r:SsT:t:U:u:Vv";
static const char edit_short_opts[] = "+Aa:BC:c:D:g:h::knp:R:r:ST:t:u:V";
+ more stuff to handle the new getopt config
```

- It's not my code, so i can't say that my code has been added to billions of devices.
But it's because of my report, and this Diary, that the code has been patched.
it still feels good :]

- Perhaps i should take it easy since i want to do a difficult challenge starting in a few days.
Non non Biyori, then some simple asm later.
- I'm going to play with "Practical Malware Analysis" binaries. 
  It's easy stuff but it's exactly what i want for today. 
  I'm going back to work tomorrow and it will be a PITA day.

---

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

![](img/idapma01.png)

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

![](img/ida_proxi.png)

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
  
---

#### SSTIC 2021

The challenge will be published in less than 2h. 

I just hope it won't be something i know nothing about, i'll be happy if i get one flag.
Please don't be an iPhone app or something that start with crypto.
The rules probably won't allow me to publish what i'm doing so i'll have to start a private diary and publish it after the deadline.

My bet is that it will be an IOT device.

- I was checking the github repo you're reading : **33 Clones, 18 Unique cloners <- wtf ?**

The challenge is up.

TL;DR : i need to learn the USB protocol.

...

i found the first flag <3

---

### 2021/04/03

- Dear Diary, while asleep i got some idea about how to solve the 2nd flag.
- and here i am struggling with too many potential attack vector...
- i reversed most of the _flag2_ code, i can't find a flaw yet.
- 4PM and i still can't find the 2nd flag. Also, i did some shopping to take a break.
- nobody found the 2 flag yet.

--- 

### Brain melttown 

- 2021/04/04 : Dear diary, i still can't find the flaw i need in the sstic challenge. i RE'd everything, twice. My brain is melting
- 2021/04/05 : Dear diary, my brain need a break.
- 2021/04/06 : Dear diary, i'm making a Neural network based game using GMS2 during my break (if you can call that a break)

--- 

### 2021/04/07 

Dear diary, i'll go back to my usual easier stuff. 
The sstic challenge can wait a little bit.

I found 2 interesting links : 
- https://github.com/thalium/icebox : Icebox is a Virtual Machine Introspection solution that enable you to stealthily trace and debug any process (kernel or user).
- https://github.com/umanovskis/baremetal-arm : This repository contains a tutorial ebook concerning programming a bare-metal ARM system. More specifically it deals with a ARMv7A version of the ARM Versatile Express platform, emulated on a regular PC through QEMU.

---

### 2021/04/11

Dear Diary, i started learning TypeScript, it easy.

### 2021/04/21

Dear Diary, i'm also learning Rust, it's not as easy as typescript (obviously).

I gave up on the SSTIC challenge, i still have a long way to go i guess.
The difficulty gap between the 1st and 2nd flag is hardcore. I hope the solution to the 2nd flag isn't some obvious trap i failed to notice.
(probably is. i think i'll be upset when i'll read the solution)

I should go back trying to find bug in stuff. 

~~I haven't read the PostgreSQL source code for a long time.
Guess i'll just do that. Easy, peaceful, no stress.~~ I'm an idiot.

--- 

### 2021/05/12

Dear Diary, i'm learning rust. it's very time-consuming, sorry.

---

### 2021/11/09

Dear Diary, long time no see. I learned Rust and Swift, and a bit of kotlin.

Also, i'm really trying hard to switch to Ghidra, but everytime i use it remind me of how good IDA is. Still, IDA is so damn expensive... of course it's time to nenew my IDA licence and i can't really spare the expense. 

About the SSTIC challenged. I finished 12th on the first flag but couldn't find the others. imho, the 2nd flag was too far away. I was on the right path. I fully reverse engineered the binary, found the exploit (a poorly designed struct related to username, if i remember correctly), the exploit would have helped to poke the windows kernel and get a shell.

Anyway, i'm busy analyzing a malware in my small spare time. An oldschool one corrupting the MBR. Good old time.

### uselessdisk.exe

I'm not done with it but here a summary of the important part :

```c
void k_writeToDiskShutdown(void)
{

k_createDataToWriteToDisk(&lpBuffer,&MBR_MALWARE,480);
fhandle = CreateFileA("\\\\.\\PHYSICALDRIVE0",0xc0000000,FILESHARE_CHANGE_MODIFY,0, 3,0,0);

if (fhandle == -1) {
    fhandle = 0x401159; // <- ???
    return;
  }
  DeviceIoControl(device,FSCTL_LOCK_VOLUME,0,0,0,0,&byteReturned, 0);
  WriteFile(device,&lpBuffer,512,&nbOfByteWritten,0);
  DeviceIoControl(device,0x9001c,0,0,0,0,&byteReturned,0);
  CloseHandle(device); // :D
  WinExec("shutdown -r -t 0",0);
  ExitProcess(-1); // but... why ? :)
}
```

---

### 2021/11/10 : Exploring emotet

* SHA256 : 878d5137e0c9a072c83c596b4e80f2aa52a8580ef214e5ba0d59daa5036a92f8
* Probably the scariest trojan of the current days. Let's explore it. I using ghidra again.
* According to ghidra, the only import is ```KERNEL32.DLL::WTSGetActiveConsoleSessionId```
* I wonder what it can possibly be with so little and i'll have to find out.
* The obvious step for now is to find out how it load other functions to be able to do anything.

* There isn't that much function and a quick overview found this stuff, i renamed the functions with my own naming convention.
* I have no idea what it's doing. I'll have to (posibly) patch the function signature too.
* There is also a lot of repetitive call to the same function pointer
* Then i'll have to trace back the references to the function pointers
* Here is how it looks for now


```c
void k_DLL_loadfunction?(void)

{
  undefined4 uVar1;
  undefined4 local_8;
  
  k_DLL_loadfunction2?(0x4a604ebc,&local_8);
  uVar1 = local_8;
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(0x21,0x54b7e774,&DAT_0040c040);
  uVar1 = (*_k_DLL_FP3)(0,uVar1);
  (*_k_DLL_FP1)(uVar1);
  k_DLL_loadfunction2?(0x4a604ebc,&local_8);
  uVar1 = local_8;
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(1,0x3c505b91,&DAT_0040c0c8);
  uVar1 = (*_k_DLL_FP3)(0,uVar1);
  (*_k_DLL_FP1)(uVar1);
  k_DLL_loadfunction2?(0x4a604ebc,&local_8);
  uVar1 = local_8;
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(2,0x10577008,&DAT_0040c214);
  uVar1 = (*_k_DLL_FP3)(0,uVar1);
  (*_k_DLL_FP1)(uVar1);
  k_DLL_loadfunction2?(0x4a604ebc,&local_8);
  uVar1 = local_8;
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(1,0x7194b56b,&DAT_0040c0c4);
  uVar1 = (*_k_DLL_FP3)(0,uVar1);
  (*_k_DLL_FP1)(uVar1);
  k_DLL_loadfunction2?(0x4a604ebc,&local_8);
  uVar1 = local_8;
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(1,0x20edec96,&DAT_0040c0cc);
  uVar1 = (*_k_DLL_FP3)(0,uVar1);
  (*_k_DLL_FP1)(uVar1);
  k_DLL_loadfunction2?(0x4a604ebc,&local_8);
  uVar1 = local_8;
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(2,0x620cb38e,&DAT_0040c21c);
  uVar1 = (*_k_DLL_FP3)(0,uVar1);
  (*_k_DLL_FP1)(uVar1);
  k_DLL_loadfunction2?(0x4a604ebc,&local_8);
  uVar1 = local_8;
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(0xe,0x5a7185ae,&DAT_0040c230);
  uVar1 = (*_k_DLL_FP3)(0,uVar1);
  (*_k_DLL_FP1)(uVar1);
  k_DLL_loadfunction2?(0x4a604ebc,&local_8);
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(3,0x73ee0ad8,&DAT_0040c224);
  uVar1 = (*_k_DLL_FP3)(0,local_8);
  (*_k_DLL_FP1)(uVar1);
  k_DLL_afterLoadFunction?();
  return;
}

```

* Tons of "CALL dword ptr [k_DLL_FP1/2/3]" and there isn't a single write directly refering to the FP's addresses
	* It must be part of a struct or an array
	* AND their addresses are : 0040c1e8, 0040c17c, 0040c1a8, they're quite close to eachothers
	* AND there is a lot of 4 bytes data in there. possibly a huge list of (function?) pointer
	* if i scroll up a little bit i find the address "0040c040", an XREF find me a PUSH to this address.
	* and it lead directly to "k_DLL_loadfunction3?(0x21,0x54b7e774,&DAT_0040c040);"
	* Heh :)
	* If we look at all the &DAT_ in k_DLL_loadfunction3, the address are not that far to each other
	* And not far from the FP too.
	* I'll bet on an array of struct for now and take a closer look at k_DLL_loadfunction3?
	* (Btw, the 2nd argument may be a hash)


#### k_DLL_loadfunction3

* Duuuuh ! Of course there is a loop, of course the "suspected hash" is XOR'd 
* Time for celebration

```c

void __cdecl k_DLL_loadfunction3?(uint loop,uint hash?,void *FP?)

{
  char cVar1;
  int iVar2;
  int iVar3;
  int iVar4;
  int iVar5;
  uint uVar6;
  int in_ECX;
  uint uVar7;
  int in_EDX;
  char *pcVar8;
  uint uVar9;
  uint i;
  
  iVar5 = *(int *)(in_ECX + 0x3c) + in_ECX;
  uVar6 = *(int *)(iVar5 + 0x78) + in_ECX;
  iVar2 = *(int *)(uVar6 + 0x1c);
  iVar3 = *(int *)(uVar6 + 0x20);
  iVar4 = *(int *)(uVar6 + 0x24);
  uVar9 = 0;
  if (*(int *)(uVar6 + 0x18) != 0) {
    do {
      pcVar8 = (char *)(*(int *)(iVar3 + in_ECX + uVar9 * 4) + in_ECX);
      uVar7 = 0;
      cVar1 = *pcVar8;
      while (cVar1 != '\0') {
        pcVar8 = pcVar8 + 1;
        uVar7 = uVar7 * 0x1003f + (int)cVar1;
        cVar1 = *pcVar8;
      }
      i = 0;
      if (loop != 0) {
        do {
          if (*(uint *)(in_EDX + i * 4) == (uVar7 ^ hash?)) {
            uVar7 = *(int *)(iVar2 + in_ECX + (uint)*(ushort *)(iVar4 + in_ECX + uVar9 * 2) * 4) +
                    in_ECX;
            if ((uVar6 <= uVar7) && (uVar7 < *(int *)(iVar5 + 0x7c) + uVar6)) {
              uVar7 = FUN_00401a20();
            }
            *(uint *)((int)FP? + i * 4) = uVar7;
            break;
          }
          i = i + 1;
        } while (i < loop);
      }
      uVar9 = uVar9 + 1;
    } while (uVar9 < *(uint *)(uVar6 + 0x18));
  }
  return;
}
```

#### Back to k_DLL_loadfunction?

* The call to function2 is always the same : k_DLL_loadfunction2?(0x4a604ebc,&local_8);```
* It's not loading something as my naming imply. it's probably doing a system call.
* it is now named "k_DLL_systemCall?"
* We have a repetition of this pattern

```c
  k_DLL_systemCall?(0x4a604ebc,&local_8);
  uVar1 = local_8;
  (*_k_DLL_FP2)(local_8);
  k_DLL_loadfunction3?(0x21,0x54b7e774,&DAT_0040c040);
  uVar1 = (*_k_DLL_FP3)(0,uVar1);
  (*(code *)k_DLL_FP1)(uVar1);
```

* One of them have to a LoadLibrary() or something close to it.
* it would be way to inconveniant if it wasn't the case (but we never know with malware, i may be deep into a rabbit hole instead)
* i'll take a small break, rename stuff, explore some more.
* And it's getting late anyway.


The function calling k_DLL_loadfunction is this one, looks familiar ? i hope it does or you haven't been paying attention

```c

/* WARNING: Globals starting with '_' overlap smaller symbols at the same address */

undefined4 k_DLL_beforeLoad?(void)

{
  undefined4 uVar1;
  undefined4 uVar2;
  int iVar3;
  int iVar4;
  int iVar5;
  undefined local_364 [520];
  undefined local_15c [128];
  undefined local_dc [128];
  undefined4 local_5c [11];
  undefined4 local_30;
  undefined4 local_18;
  undefined4 local_14;
  undefined4 local_8;
  
  (*_DAT_0040c1d4)(0,local_364,0x104);
  uVar1 = FUN_004019e0();
  k_DLL_systemCall?(0x4dbac13f,&local_8);
  uVar2 = local_8;
  (*_DAT_0040c200)(local_15c,0x40,local_8,uVar1);
  uVar2 = (*(code *)k_DLL_FP3)(0,uVar2);
  (*(code *)k_DLL_FP1)(uVar2);
  k_DLL_systemCall?(0x4dbac13f,&local_8);
  (*_DAT_0040c200)(local_dc,0x40,local_8,uVar1);
  uVar2 = (*(code *)k_DLL_FP3)(0,local_8);
  (*(code *)k_DLL_FP1)(uVar2);
  iVar3 = (*_DAT_0040c1b4)(0,1,0,local_15c);
  if (iVar3 != 0) {
    iVar4 = (*_DAT_0040c1cc)(0,1,local_dc);
    if (iVar4 == 0) {
      (*_DAT_0040c120)(iVar3);
    }
    else {
      iVar5 = (*_DAT_0040c158)();
      if (iVar5 == 0xb7) {
        (*_DAT_0040c128)(iVar3);
        (*_DAT_0040c120)(iVar3);
        (*_DAT_0040c120)(iVar4);
        k_DLL_loadfunction?();
        return 1;
      }
      (*k_DLL_FP4)(local_5c,0,0x44);
      local_5c[0] = 0x44;
      local_30 = 0x80;
      iVar5 = (*_DAT_0040c118)(local_364,0,0,0,0,0,0,0,local_5c,&local_18);
      if (iVar5 != 0) {
        (*_DAT_0040c18c)(iVar3,0xffffffff);
        (*_DAT_0040c120)(local_18);
        (*_DAT_0040c120)(local_14);
        (*_DAT_0040c120)(iVar3);
        (*_DAT_0040c120)(iVar4);
        return 1;
      }
    }
  }
  return 0;
}
```

---

### 2021/11/13

- got my hand surgery yesterday, it went well
  - both my brain and hand are out of service
  - it hurt just a little bit, from time to time, and the opioïd medication is effective

- The press is talking about BotenaGO malware, targeting NAS, Router & IoT.
  - look like it use a lot of different exploit
  - eg : (from https://www.bleepingcomputer.com/news/security/botenago-botnet-targets-millions-of-iot-devices-with-33-exploits/)
    - BotenaGo incorporates 33 exploits for a variety of routers, modems, and NAS devices, with some notable examples given below:
      - CVE-2015-2051, CVE-2020-9377, CVE-2016-11021: D-Link routers
      - CVE-2016-1555, CVE-2017-6077, CVE-2016-6277, CVE-2017-6334: Netgear devices
      - CVE-2019-19824: Realtek SDK based routers
      - CVE-2017-18368, CVE-2020-9054: Zyxel routers and NAS devices
      - CVE-2020-10987: Tenda products
      - CVE-2014-2321: ZTE modems
      - CVE-2020-8958: Guangzhou 1GE ONU
      
i'm still exploring emotet. i resolved some FP by guessing their parameters.
Eg:
```
filehandle = (*_CreateFileA?)(fileNameString,0x40000000,0,0,2,FILE_ATTRIBUTE_NORMAL,0);
```

--- 

### 2021/11/14

Dear diary, my left hand is still pretty much unusable for daily activity (cooking, putting on socks, ...)
BUT i can type on keyboard much more easily now. Anyway ...

I successfully compiled ghidra from source using IntelliJ IDEA Ultimate. 
Thank god i won't have to use the recommended Eclipse IDE. 

* On a M1 Powered Mac.
* I'm renting a Mac Mini M1 at scaleway so i don't have to use my macbook Air M1 for SRE.
* I just finished running the UnitTest, it took 35mn.
* All tests related to debugger failed. I'm not surprised. M1 isn't supported, yet.
* this failed is a bit more surprising but pretty much harmless :
```
Caused by: net.sf.sevenzipjbinding.SevenZipNativeInitializationException: Error loading SevenZipJBinding native library into JVM: Can't find suited platform for os.arch=aarch64, os.name=Mac... Available list of platforms: Linux-amd64, Linux-i386, Mac-x86_64, Windows-amd64, Windows-x86 [You may also try different SevenZipJBinding initialization methods 'net.sf.sevenzipjbinding.SevenZip.init*()' in order to solve this problem] 
```
* there is no M1 native for 7zip. Meh... i don't care.
* Now i can use the latest and greatest Ghidra and perhaps patch some bugs too ?

--- 

### 2021/11/15

Lots of post-surgery paperwork today. 

I hope i'll have some post-paperwork time to play with emotet.

#### to fastcall or not fastcall ?

```
                             undefined entry()
             undefined         AL:1           <RETURN>
             undefined4        HASH:85f2c43   fp?
             undefined4        HASH:3f3805b   hash
             undefined4        HASH:3fcdfb3   loop
                             entry                                           XREF[2]:     Entry Point(*), 004000f0(*)  
        00409ee0 56              PUSH       ESI
        00409ee1 68 f0 c1        PUSH       first_imported_FP
                 40 00
        00409ee6 68 6c 64        PUSH       0x3966646c
                 66 39
        00409eeb 6a 09           PUSH       0x9
        00409eed b9 14 20        MOV        ECX,0xd22e2014
                 2e d2
        00409ef2 e8 e9 7c        CALL       k_get_something_from_TIB                         undefined4 k_get_something_from_
                 ff ff
        00409ef7 ba f0 11        MOV        EDX,DAT_004011f0                                 = A7h
                 40 00
        00409efc 8b c8           MOV        ECX,EAX
        00409efe e8 0d 7c        CALL       k_DLL_importWithHash                             undefined k_DLL_importWithHash(u
                 ff ff
```

The decompiled code was : 
```c
  fp? = &first_imported_FP;
  hash = 0x3966646c;
  loop = 9;
  k_get_something_from_TIB();
  k_DLL_importWithHash(loop,hash,fp?);
```

But with that, 0xd22e2014 vanished into nothingness.

Clearly, it's there for a reason and ECX have to be an argument, right ?
Luckily, there is a call convention for that kind of call.

```
__fastcall
Argument-passing order :
The first two DWORD or smaller arguments that are found in the argument list from left to right are passed in ECX and EDX registers; 
all other arguments are passed on the stack from right to left.
```

And it become :

```c
  fp? = &first_imported_FP;
  hash = 0x3966646c;
  loop = 9;
  k_get_something_from_TIB(0xd22e2014);
  k_DLL_importWithHash(loop,hash,fp?);
```

Yet, it doesn't return anything, which is unlikely.
And the 2nd call is : 

```c
        00409efc 8b c8           MOV        ECX,EAX
        00409efe e8 0d 7c        CALL       k_DLL_importWithHash                             undefined k_DLL_importWithHash(u
                 ff ff
```

EAX could be the returned value of k_get_something_from_TIB and passed to k_DLL_importWithHash

Another fast call ? Mmmmm

I don't know... it look like it but i don't like the output.

```c

void entry(void)

{
  undefined4 uVar1;
  int iVar2;
  void *loop;
  
  loop = (void *)0x9;
  uVar1 = k_get_something_from_TIB(0xd22e2014);
  k_DLL_importWithHash(uVar1,&DAT_004011f0,loop);
  loop = (void *)0x48;
  uVar1 = k_get_something_from_TIB(0x8f7ee672);
  k_DLL_importWithHash(uVar1,&DAT_004010d0,loop);
  uVar1 = (*(code *)k_DLL_FP3)(0,0x8000000);
  iVar2 = (*DAT_0040c10c)(uVar1);
  if (iVar2 != 0) {
    (*k_DLL_FP4)(iVar2,0,0x8000000);
    uVar1 = (*(code *)k_DLL_FP3)(0,iVar2);
    (*(code *)k_DLL_FP1)(uVar1);
    k_DLL_beforeLoad?();
  }
  (*k_quit?)(0);
  return;
}

```

It's probably wrong.

At some point you gotta stop the guesswork, or even stop reading the decompiled output,
and focus on the real code : ASM.

That's why i really like IDA : Assembly first.

---

Oops, paperwork. BBL.

Note for later : https://docs.microsoft.com/en-us/cpp/cpp/thiscall?view=msvc-170

---

### 2021/11/16

* More post-surgery stuff, i'll finally get to see what's hiding behind all the bandage.
* I did some googling about the TIB and stuff, it's not easy. And more specifically about the PEB.
* The thing, i still don't really get how it get to load DLL without using a single string ("kernel32.dll")
* I understand that the code is using hash to find the function it's looking for, but still...
* When wikipedia'ing about the PEB i found this : 
  * Ldr	
  * A pointer to a PEB_LDR_DATA structure providing information about loaded modules	
  * Contains the base address of kernel32 and ntdll.
* Aaaand there it is.
* It's safe to assume that emotet will browse all loaded dll, which is guaranteed to have at least kernel32.dll
  * safe to assume, because there are a lot of loops in the code
  * I can guess it will check every method in every dll
  * compare it with a hash
  * save the address of the function that match the hash
  * This is the stuff i named with "FP"
* i'll have "to go deeper" and analyse "k_get_something_from_TIB" to confirm my intuition

PS : i'm still digging my rabbit hole, everything you see above is based on a lot of assumption and intuition coming from experience

It would be embarrassing if i were completely off-track but that wouldn't be the first time
and that's part of the process 

Not for later, this look super important as i've seen this somewhere into the code :
From wikipedia's PEB : 

```
SessionId	
The session ID of the Terminal Services session that the process is part of	
The NtCreateUserProcess() system call initializes this by calling the kernel's internal MmGetSessionId() function.
```

---

Back from the clinic, my hand is officially "OK" <3

---

So...

```
puVar4 = *(int *)(*(int *)(in_FS_OFFSET + 0x30) + 0xc) + 0xc;
```

Some good info : 
* https://rstforums.com/forum/topic/107778-anatomy-of-the-process-environment-block-peb/
* https://0xevilc0de.com/locating-dll-name-from-the-process-environment-block-peb/
* https://www.aldeid.com/wiki/PEB-Process-Environment-Block


* (in_FS_OFFSET + 0x30) is the PEB
* PEB + 0xC = PEB_LDR_DATA
* PEB_LDR_DATA + 0xC = InLoadOrderModuleList : _LIST_ENTRY

See : https://docs.microsoft.com/fr-fr/windows/win32/api/winternl/ns-winternl-peb_ldr_data?redirectedfrom=MSDN

```
typedef struct _LDR_DATA_TABLE_ENTRY {
    PVOID Reserved1[2];
    LIST_ENTRY InMemoryOrderLinks;
    PVOID Reserved2[2];
    PVOID DllBase;
    PVOID EntryPoint;
    PVOID Reserved3;
    UNICODE_STRING FullDllName;
    BYTE Reserved4[8];
    PVOID Reserved5[3];
    union {
        ULONG CheckSum;
        PVOID Reserved6;
    };
    ULONG TimeDateStamp;
} LDR_DATA_TABLE_ENTRY, *PLDR_DATA_TABLE_ENTRY;
```

```
typedef struct _PEB_LDR_DATA {
  BYTE       Reserved1[8];
  PVOID      Reserved2[3];
  LIST_ENTRY InMemoryOrderModuleList;
} PEB_LDR_DATA, *PPEB_LDR_DATA;
```
```
typedef struct _LIST_ENTRY {
struct _LIST_ENTRY *Flink;
struct _LIST_ENTRY *Blink;
} LIST_ENTRY, *PLIST_ENTRY, *RESTRICTED_POINTER PRLIST_ENTRY;
```

Some more infodump : 

```
struct _PEB {
    0x000 BYTE InheritedAddressSpace;
    0x001 BYTE ReadImageFileExecOptions;
    0x002 BYTE BeingDebugged;
    0x003 BYTE SpareBool;
    0x004 void* Mutant;
    0x008 void* ImageBaseAddress;
    0x00c _PEB_LDR_DATA* Ldr;
    0x010 _RTL_USER_PROCESS_PARAMETERS* ProcessParameters;
    0x014 void* SubSystemData;
    0x018 void* ProcessHeap;
    0x01c _RTL_CRITICAL_SECTION* FastPebLock;
    0x020 void* FastPebLockRoutine;
    0x024 void* FastPebUnlockRoutine;
    0x028 DWORD EnvironmentUpdateCount;
    0x02c void* KernelCallbackTable;
    0x030 DWORD SystemReserved[1];
    0x034 DWORD ExecuteOptions:2; // bit offset: 34, len=2
    0x034 DWORD SpareBits:30; // bit offset: 34, len=30
    0x038 _PEB_FREE_BLOCK* FreeList;
    0x03c DWORD TlsExpansionCounter;
    0x040 void* TlsBitmap;
    0x044 DWORD TlsBitmapBits[2];
    0x04c void* ReadOnlySharedMemoryBase;
    0x050 void* ReadOnlySharedMemoryHeap;
    0x054 void** ReadOnlyStaticServerData;
    0x058 void* AnsiCodePageData;
    0x05c void* OemCodePageData;
    0x060 void* UnicodeCaseTableData;
    0x064 DWORD NumberOfProcessors;
    0x068 DWORD NtGlobalFlag;
    0x070 _LARGE_INTEGER CriticalSectionTimeout;
    0x078 DWORD HeapSegmentReserve;
    0x07c DWORD HeapSegmentCommit;
    0x080 DWORD HeapDeCommitTotalFreeThreshold;
    0x084 DWORD HeapDeCommitFreeBlockThreshold;
    0x088 DWORD NumberOfHeaps;
    0x08c DWORD MaximumNumberOfHeaps;
    0x090 void** ProcessHeaps;
    0x094 void* GdiSharedHandleTable;
    0x098 void* ProcessStarterHelper;
    0x09c DWORD GdiDCAttributeList;
    0x0a0 void* LoaderLock;
    0x0a4 DWORD OSMajorVersion;
    0x0a8 DWORD OSMinorVersion;
    0x0ac WORD OSBuildNumber;
    0x0ae WORD OSCSDVersion;
    0x0b0 DWORD OSPlatformId;
    0x0b4 DWORD ImageSubsystem;
    0x0b8 DWORD ImageSubsystemMajorVersion;
    0x0bc DWORD ImageSubsystemMinorVersion;
    0x0c0 DWORD ImageProcessAffinityMask;
    0x0c4 DWORD GdiHandleBuffer[34];
    0x14c void (*PostProcessInitRoutine)();
    0x150 void* TlsExpansionBitmap;
    0x154 DWORD TlsExpansionBitmapBits[32];
    0x1d4 DWORD SessionId;
    0x1d8 _ULARGE_INTEGER AppCompatFlags;
    0x1e0 _ULARGE_INTEGER AppCompatFlagsUser;
    0x1e8 void* pShimData;
    0x1ec void* AppCompatInfo;
    0x1f0 _UNICODE_STRING CSDVersion;
    0x1f8 void* ActivationContextData;
    0x1fc void* ProcessAssemblyStorageMap;
    0x200 void* SystemDefaultActivationContextData;
    0x204 void* SystemAssemblyStorageMap;
    0x208 DWORD MinimumStackCommit;
);
```

```
typedef struct _PEB_LDR_DATA
{
    0x00    ULONG         Length;                            /* Size of structure, used by ntdll.dll as structure version ID */
    0x04    BOOLEAN       Initialized;                       /* If set, loader data section for current process is initialized */
    0x08    PVOID         SsHandle;
    0x0c    LIST_ENTRY    InLoadOrderModuleList;             /* Pointer to LDR_DATA_TABLE_ENTRY structure. Previous and next module in load order */
    0x14    LIST_ENTRY    InMemoryOrderModuleList;           /* Pointer to LDR_DATA_TABLE_ENTRY structure. Previous and next module in memory placement order */
    0x1c    LIST_ENTRY    InInitializationOrderModuleList;   /* Pointer to LDR_DATA_TABLE_ENTRY structure. Previous and next module in initialization order */
} PEB_LDR_DATA,*PPEB_LDR_DATA; // +0x24

typedef struct _LDR_DATA_TABLE_ENTRY
{
    LIST_ENTRY InLoadOrderLinks; /* 0x00 */
    LIST_ENTRY InMemoryOrderLinks; /* 0x08 */
    LIST_ENTRY InInitializationOrderLinks; /* 0x10 */
    PVOID DllBase; /* 0x18 */
    PVOID EntryPoint;
    ULONG SizeOfImage;
    UNICODE_STRING FullDllName; /* 0x24 */
    UNICODE_STRING BaseDllName; /* 0x28 */
    ULONG Flags;
    WORD LoadCount;
    WORD TlsIndex;
    union
    {
         LIST_ENTRY HashLinks;
         struct
         {
              PVOID SectionPointer;
              ULONG CheckSum;
         };
    };
    union
    {
         ULONG TimeDateStamp;
         PVOID LoadedImports;
    };
    _ACTIVATION_CONTEXT * EntryPointActivationContext;
    PVOID PatchInformation;
    LIST_ENTRY ForwarderLinks;
    LIST_ENTRY ServiceTagLinks;
    LIST_ENTRY StaticLinks;
} LDR_DATA_TABLE_ENTRY, *PLDR_DATA_TABLE_ENTRY;
```

Luckily, Ghidra knows wth is a LIST_ENTRY, but it doesn't know about PEB and LDR :(

My typing and naming doesn't seem to be correct yet, but we're getting somewhere :

```

undefined4 __fastcall k_get_something_from_TIB(DWORD param_1)

{
  DWORD uVar1;
  DWORD DVar1;
  LIST_ENTRY32 *InLoadOrderModuleList;
  char *baseDLLName;
  DWORD *in_FS_OFFSET;
  DWORD *LDR_DATA_TABLE_ENTRY;
  ushort j;
  
                    /* FS:0x30 = PEB
                       PEB + 0xC = PEB_LDR_DATA
                       PEB_LDR_DATA + 0xC = InLoadOrderModuleList
                       InLoadOrderModuleList is a LIST_ENTRY */
  InLoadOrderModuleList = (LIST_ENTRY32 *)(*(int *)(in_FS_OFFSET[0xc] + 0xc) + 0xc);
  LDR_DATA_TABLE_ENTRY = (DWORD *)InLoadOrderModuleList->Flink;
  while( true ) {
    if ((LIST_ENTRY32 *)LDR_DATA_TABLE_ENTRY == InLoadOrderModuleList) {
      return 0;
    }
                    /* +0x30 -> BaseDllName */
    baseDLLName = ((LIST_ENTRY32 *)((int)LDR_DATA_TABLE_ENTRY + 0x30))->Flink;
    DVar1 = 0;
    j = *(ushort *)baseDLLName;
    while (j != 0) {
      uVar1 = (DWORD)j;
      if ((ushort)(j - 0x41) < 0x1a) {
        uVar1 = uVar1 + 0x20;
      }
      baseDLLName = (char *)((int)baseDLLName + 2);
      DVar1 = DVar1 * 0x1003f + uVar1;
      j = *(ushort *)baseDLLName;
    }
    if (DVar1 == param_1) break;
    LDR_DATA_TABLE_ENTRY = (DWORD *)*(undefined **)LDR_DATA_TABLE_ENTRY;
  }
  return ((LIST_ENTRY32 *)((int)LDR_DATA_TABLE_ENTRY + 0x18))->Flink;
}
```

i found a good video : https://www.youtube.com/watch?v=Tk3RWuqzvII

The video appears to describe what our function is doing. Nice.

We can go back for now and safely assume it loaded kernel32.dll

---

### 2021/11/17

TL;DR : i sorted out the mess i made the past 2 days.

```
                    /* first import */
  dllBase_kernel32? = k_get_something_from_TIB(0xd22e2014);
  k_DLL_importWithHash(dllBase_kernel32?,&DAT_004011f0);
                    /* second import */
  dllBase_kernel32? = k_get_something_from_TIB(0x8f7ee672);
  k_DLL_importWithHash(dllBase_kernel32?,&DAT_004010d0);
  ```

Now it make sense, right ?

Also, i created the LDR_DATA_TABLE_ENTRY in ghidra, sorted out some more mess too.

```

void * __fastcall k_get_something_from_TIB(DWORD hash)

{

void * __fastcall k_getBaseDllFromHash(DWORD hash)

{
  WORD lowerLetter;
  DWORD hashedName;
  LIST_ENTRY *InLoadOrderModuleList;
  char *baseDLLName;
  DWORD *in_FS_OFFSET;
  _LDR_DATA_TABLE_ENTRY *list;
  WORD letter;
  
                    /* FS:0x30 = PEB
                       PEB + 0xC = PEB_LDR_DATA
                       PEB_LDR_DATA + 0xC = InLoadOrderModuleList
                       InLoadOrderModuleList is a LIST_ENTRY */
  InLoadOrderModuleList = (LIST_ENTRY *)(*(int *)(in_FS_OFFSET[0xc] + 0xc) + 0xc);
  list = (_LDR_DATA_TABLE_ENTRY *)InLoadOrderModuleList->Flink;
  while( true ) {
    if (list == (_LDR_DATA_TABLE_ENTRY *)InLoadOrderModuleList) {
      return (void *)0x0;
    }
                    /* +0x30 -> BaseDllName */
    baseDLLName = (char *)(list->BaseDllName).Buffer;
    hashedName = 0;
    letter = *(WORD *)baseDLLName;
    while (letter != 0) {
      _lowerLetter = (uint)letter;
                    /* to lower case */
      if ((ushort)(letter - 0x41) < 0x1a) {
        _lowerLetter = _lowerLetter + 0x20;
      }
      baseDLLName = (char *)((int)baseDLLName + 2);
      hashedName = hashedName * 0x1003f + _lowerLetter;
      letter = *(WORD *)baseDLLName;
    }
    if (hashedName == hash) break;
    list = (_LDR_DATA_TABLE_ENTRY *)(list->InLoadOrderLinks).Flink;
  }
  return list->DllBase;
}


```

i still have some weird stuff but : 
* it loop over each entry of the table.
* if the (lowercased) hashed name == hash then return a pointer to baseDll
* return 0 if it couldn't find it

a mini-victory self-five for me ! yay !

---

### 2021/11/18

Dear Diary,

i checked some video about github autopilot. it looks seriously awesome.

I'm on the waiting list...

### 2021/11/19

i really, really, want my access to autopilot :(