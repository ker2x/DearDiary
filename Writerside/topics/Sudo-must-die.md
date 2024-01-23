# Sudo must die

Yes, [sudo](https://github.com/sudo-project/sudo/tree/main) must die.
It's a dirty mess and a security nightmare.

- The source code is poorly documented.
- It's (at least) two programs in one : sudo, sudoedit.
- Flags, flags everywhere.

## Flags

````C
/*
 * Various modes sudo can be in (based on arguments) in hex
 */
#define MODE_RUN		    0x00000001U
#define MODE_EDIT		    0x00000002U
#define MODE_VALIDATE		0x00000004U
#define MODE_INVALIDATE		0x00000008U
#define MODE_KILL		    0x00000010U
#define MODE_VERSION		0x00000020U
#define MODE_HELP		    0x00000040U
#define MODE_LIST		    0x00000080U
#define MODE_CHECK		    0x00000100U
#define MODE_MASK		    0x0000ffffU

/* Mode flags */
/* XXX - prune this */
#define MODE_BACKGROUND		    0x00010000U
#define MODE_SHELL		        0x00020000U
#define MODE_LOGIN_SHELL	    0x00040000U
#define MODE_IMPLIED_SHELL	    0x00080000U
#define MODE_RESET_HOME		    0x00100000U
#define MODE_PRESERVE_GROUPS	0x00200000U
#define MODE_PRESERVE_ENV	    0x00400000U
#define MODE_NONINTERACTIVE	    0x00800000U
#define MODE_LONG_LIST		    0x01000000U

/* Indexes into sudo_settings[] args, must match parse_args.c. */
#define ARG_BSDAUTH_TYPE	 0
#define ARG_LOGIN_CLASS		 1
#define ARG_PRESERVE_ENVIRONMENT 2
#define ARG_RUNAS_GROUP		 3
#define ARG_SET_HOME		 4
#define ARG_USER_SHELL		 5
#define ARG_LOGIN_SHELL		 6
#define ARG_IGNORE_TICKET	 7
#define ARG_UPDATE_TICKET	 8
#define ARG_PROMPT		     9
#define ARG_SELINUX_ROLE	10
#define ARG_SELINUX_TYPE	11
#define ARG_RUNAS_USER		12
#define ARG_PROGNAME		13
#define ARG_IMPLIED_SHELL	14
#define ARG_PRESERVE_GROUPS	15
#define ARG_NONINTERACTIVE	16
#define ARG_SUDOEDIT		17
#define ARG_CLOSEFROM		18
#define ARG_NET_ADDRS		19
#define ARG_MAX_GROUPS		20
#define ARG_PLUGIN_DIR		21
#define ARG_REMOTE_HOST		22
#define ARG_TIMEOUT		    23
#define ARG_CHROOT		    24
#define ARG_CWD			    25
#define ARG_ASKPASS		    26
#define ARG_INTERCEPT_SETID	27
#define ARG_INTERCEPT_PTRACE	28
#define ARG_APPARMOR_PROFILE	29

/*
 * Flags for tgetpass()
 */
#define TGP_NOECHO	    0x00U		/* turn echo off reading pw (default) */
#define TGP_ECHO	    0x01U		/* leave echo on when reading passwd */
#define TGP_STDIN	    0x02U		/* read from stdin, not /dev/tty */
#define TGP_ASKPASS	    0x04U		/* read from askpass helper program */
#define TGP_MASK	    0x08U		/* mask user input when reading */
#define TGP_NOECHO_TRY	0x10U		/* turn off echo if possible */
#define TGP_BELL	    0x20U		/* bell on password prompt */
````

"*Prune this*", well, no kidding.
__Prune it, burn it, bury it, and never speak of it again.__

But that's clearly not enough flags, we need more flags ! Moooaaar !

```C
#define RUN_VALID_FLAGS	(MODE_ASKPASS|MODE_PRESERVE_ENV|MODE_RESET_HOME|MODE_IMPLIED_SHELL|MODE_LOGIN_SHELL|MODE_NONINTERACTIVE|MODE_IGNORE_TICKET|MODE_UPDATE_TICKET|MODE_PRESERVE_GROUPS|MODE_SHELL|MODE_RUN|MODE_POLICY_INTERCEPTED)
#define EDIT_VALID_FLAGS	(MODE_ASKPASS|MODE_NONINTERACTIVE|MODE_IGNORE_TICKET|MODE_UPDATE_TICKET|MODE_EDIT)
#define LIST_VALID_FLAGS	(MODE_ASKPASS|MODE_NONINTERACTIVE|MODE_IGNORE_TICKET|MODE_UPDATE_TICKET|MODE_LIST|MODE_CHECK)
#define VALIDATE_VALID_FLAGS	(MODE_ASKPASS|MODE_NONINTERACTIVE|MODE_IGNORE_TICKET|MODE_UPDATE_TICKET|MODE_VALIDATE)
#define INVALIDATE_VALID_FLAGS	(MODE_ASKPASS|MODE_NONINTERACTIVE|MODE_IGNORE_TICKET|MODE_UPDATE_TICKET|MODE_INVALIDATE)

#define DEFAULT_VALID_FLAGS	(MODE_BACKGROUND|MODE_PRESERVE_ENV|MODE_RESET_HOME|MODE_LOGIN_SHELL|MODE_NONINTERACTIVE|MODE_PRESERVE_GROUPS|MODE_SHELL)
#define EDIT_VALID_FLAGS	MODE_NONINTERACTIVE
#define LIST_VALID_FLAGS	(MODE_NONINTERACTIVE|MODE_LONG_LIST)
#define VALIDATE_VALID_FLAGS	MODE_NONINTERACTIVE
```

Luckily there is a macro to help is deal with all those flags :

```C
#define SET_FLAG(s, n) \
    if (strncmp(s, info[i], sizeof(s) - 1) == 0) { \
	switch (sudo_strtobool(info[i] + sizeof(s) - 1)) { \
	    case true: \
		SET(details->flags, n); \
		break; \
	    case false: \
		CLR(details->flags, n); \
		break; \
	    default: \
		sudo_debug_printf(SUDO_DEBUG_ERROR, \
		    "invalid boolean value for %s", info[i]); \
		break; \
	} \
	break; \
    }
```

There is more flappy flags than a flag factory. 
(Ok the ones below are legit, if it wasn't for all the #if)

```C
/*
 * Directory open flags for use with openat(2).
 * Use O_SEARCH/O_PATH and/or O_DIRECTORY where possible.
 */
#if defined(O_SEARCH)
# if defined(O_DIRECTORY)
#  define DIR_OPEN_FLAGS	(O_SEARCH|O_DIRECTORY)
# else
#  define DIR_OPEN_FLAGS	(O_SEARCH)
# endif
#elif defined(O_PATH)
# if defined(O_DIRECTORY)
#  define DIR_OPEN_FLAGS	(O_PATH|O_DIRECTORY)
# else
#  define DIR_OPEN_FLAGS	(O_PATH)
# endif
#elif defined(O_DIRECTORY)
# define DIR_OPEN_FLAGS		(O_RDONLY|O_DIRECTORY)
#else
# define DIR_OPEN_FLAGS		(O_RDONLY|O_NONBLOCK)
#endif
```

I'm not done yet. But I need some beer.