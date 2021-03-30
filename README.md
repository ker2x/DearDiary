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

Old version. nice.

```bash
env -i 'AA=a\' 'B=b\' 'C=c\' 'D=d\' 'E=e\' 'F=f' sudoedit -s '1234567890123456789012\'
usage: sudoedit [-AknS] [-r role] [-t type] [-C num] [-g group] [-h host] [-p prompt] [-T timeout] [-u user] file ...
k
```

i assume this is some kind of WSL weirdness. I tried on my mac M1. "_sudo -V_" works. and _sudoedit_ is "_command not found_".

I'm starting up a linux cloud instance.




---

### 2021/03/31

- this is tomorrow, not today. nothing here.

---

### 2021/03/32

- this day doesn't exist (citation needed)

