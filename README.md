# Dear Diary,

## My security oriented Diary

### 2021/03/30

- Dear Diary, today i'm starting a diary
- Dear Diary, i started you.
- I'm watching youtube video about Fuzzing & Buffer Overflow : https://www.youtube.com/watch?v=FCIfWTAtPr0
  - the video is too beginner oriented, i'll give a try to part 4 (finding the offset) anyway.
      Part 5 is about EIP. 
  - Not really useful not me (a priori), but the videos are very short so it's cool.
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
- i want to go home
- i'm going home
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
  it accuratly describe my thought process and i'm not always right on first try.
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

Isn't it much better ? initenv (whatever that is) create envp. now it feels right.

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

Next, all the following jnz goes to exist so i'll skip it :

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

