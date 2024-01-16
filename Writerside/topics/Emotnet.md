# Emotnet

(extracted from main diary)

## 2021/11/10 : Exploring emotet

* SHA256 : 878d5137e0c9a072c83c596b4e80f2aa52a8580ef214e5ba0d59daa5036a92f8
* Probably the scariest trojan of the current days. Let's explore it. I'm using ghidra again.
* According to ghidra, the only import is ```KERNEL32.DLL::WTSGetActiveConsoleSessionId```
* I wonder what it can possibly be with so little, and I'll have to find out.
* The obvious step for now is to find out how it loads other functions to be able to do anything.

* There isn't that much function and a quick overview found this stuff, I renamed the functions with my own naming convention.
* I have no idea what it's doing. I'll have to (possibly) patch the function signature too.
* There is also a lot of repetitive call to the same function pointer
* Then I'll have to trace back the references to the function pointers
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

* Tons of "CALL dword ptr [k_DLL_FP1/2/3]" and there isn't a single write directly referring to the FP's addresses
    * It must be part of a struct or an array
    * AND their addresses are : 0040c1e8, 0040c17c, 0040c1a8, they're quite close to each-others
    * AND there is a lot of 4 bytes data in there. possibly a huge list of (function?) pointer
    * if I scroll up a little bit I find the address "0040c040", an XREF find me a PUSH to this address.
    * and it leads directly to "k_DLL_loadfunction3?(0x21,0x54b7e774,&DAT_0040c040);"
    * Heh :)
    * If we look at all the &DAT_ in k_DLL_loadfunction3, the address are not that far to each other
    * And not far from the FP too.
    * I'll bet on an array of struct for now and take a closer look at k_DLL_loadfunction3?
    * (Btw, the 2nd argument may be a hash)


### k_DLL_loadfunction3

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

### Back to k_DLL_loadfunction?

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
* it would be way too inconvenient if it wasn't the case (but we never know with malware, I may be deep into a rabbit hole instead)
* I'll take a small break, rename stuff, explore some more.
* And it's getting late anyway.


The function calling k_DLL_loadfunction is this one, looks familiar ? I hope it does, or you haven't been paying attention

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



