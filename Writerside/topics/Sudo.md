# Sudo (CVE-2021-3156)

(Extracted from my main diary)

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
