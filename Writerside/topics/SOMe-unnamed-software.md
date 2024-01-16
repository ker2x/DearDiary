# SOMe unnamed software

There is "this software" that I very much enjoy in its free version.
I don't feel the need to "unlock" the full version, but i like to understand what i'm using, right ?
And the software is kind enough to have a LicenceManager.exe (not the real name) that may or may be not obfuscated.

Reinstall Ghidra, add some clickety-click, and here we go.

> I'll obfuscate the name of the software, 
> i'm not doing anything illegal here but i don't want to provide some kind of "how to crack this software" guide.
> (or if i do, i won't tell you which software it is)

## The LicenceManager.exe

Loading the exe in ghidra, recursively import DLLs and so on, you know the drill. (if you don't, you're reading the wrong page)

### The search for strings

When in doubt, search for strings.

Lots of strings, lots of error messages about invalid licence, invalid key, invalid whatever.
There are a few messages as well about not being able to contact to a server. Slightly more interesting but nothing unexpected.

### The search for imported DLLs

- advapi32.dll : 
  - [https://www.geoffchappell.com/studies/windows/win32/advapi32/api/](https://www.geoffchappell.com/studies/windows/win32/advapi32/api/)
  - This DLL support security and registry calls.
  - The LicenceManager is using registry calls. (Open, Enum, Query, Create, Delete) + LookupAccountNameW & ConvertSidToStringSidA 
- kernel32.dll, user32.dll, ole32.dll, oleaut32.dll
- shell32.dll : CommandLineToArgvW, no big deal.
- wtsapi32.dll
  - it's a dll that deal with Remote Desktop, call me crazy but that's enough to hook me up (I'm that desperate enough to use it as an excuse to reverse engineer.)
  - WTSFreeMemory, WTSQuerySessionInformationW. It could have been worse.
  - [https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/](https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/)
  - [https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/nf-wtsapi32-wtsquerysessioninformationw](https://learn.microsoft.com/en-us/windows/win32/api/wtsapi32/nf-wtsapi32-wtsquerysessioninformationw)

#### WTSQuerySessionInformationW

> Retrieves session information for the specified session on the specified Remote Desktop Session Host (RD Session Host) server. It can be used to query session information on local and remote RD Session Host servers.
> 
> ```c
> BOOL WTSQuerySessionInformationW(
>   [in]  HANDLE         hServer,
>   [in]  DWORD          SessionId,
>   [in]  WTS_INFO_CLASS WTSInfoClass,
>   [out] LPWSTR         *ppBuffer,
>   [out] DWORD          *pBytesReturned
> );
> ```

How it's used in the LicenceManager.exe :

```C++                 
    iVar3 = WTSQuerySessionInformationW(0,0xffffffff,5,&local_70,&local_74);
```

Awwww... It's not even using the *SessionId*. It's just querying the current session.

5 is *WTSUserName*, so it's querying the username of the current session.

Disappointing.

### Holdonaminit !

The strings search report a lot of stuff about internet, server, proxy, but where is the DLL import ?
You know, winsock, wininet, stuff like that.

Should we bother to look for it ?

Perhaps. 

Later.



