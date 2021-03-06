---
layout: post
title:  "PDF Architect 6 - pdmodel.dll NULL Pointer Dereference"
date:   2018-09-19 03:39:03 +0700
categories: [security, null]
---

Vulnerability Details
---------------------
Vulnerability: NULL Pointer Dereference
Product: PDF Architect 6 - pdmodel.dll

Open the fuzz_14.pdf file, the application architect.exe crashed will trigger crash. The crash landed at pdmodel.dll which is 
the root cause of the issue:
```
(a98.b0c): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\pdmodel.dll - 
pdmodel!PDMODELProvidePDModelHFT+0x6e034:
00007ffd`5bffe404 81790800080000  cmp     dword ptr [rcx+8],800h ds:00000000`00000008=????????
```

Crash triage:
Looking into the exception address shows that the issue indeed are NULL pointer. It attempts to read from address 0xffffffffffffffff. 
```
0:026> .exr -1
ExceptionAddress: 00007ffd5c85e404 (pdmodel!PDMODELProvidePDModelHFT+0x000000000006e034)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: ffffffffffffffff
Attempt to read from address ffffffffffffffff
```

Disassesmble of the crash path:
```
0:028> u
pdmodel!PDMODELProvidePDModelHFT+0x6e034:
00007ffd`5bffe404 81790800080000  cmp     dword ptr [rcx+8],800h
00007ffd`5bffe40b 488bc2          mov     rax,rdx
00007ffd`5bffe40e 750e            jne     pdmodel!PDMODELProvidePDModelHFT+0x6e04e (00007ffd`5bffe41e)
00007ffd`5bffe410 488bd1          mov     rdx,rcx
00007ffd`5bffe413 488bc8          mov     rcx,rax
00007ffd`5bffe416 e8851c0400      call    pdmodel!PDMODELProvidePDModelHFT+0xafcd0 (00007ffd`5c0400a0)
00007ffd`5bffe41b 488bc8          mov     rcx,rax
00007ffd`5bffe41e 4885c9          test    rcx,rcx
```

Crash dump details:
```
0:028> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************

*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\bl-views.dll - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\ui.dll - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\architect.exe - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\bl.dll - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\plugins\plugin-user-management.dll - 
*** WARNING: Unable to verify checksum for C:\Program Files\PDF Architect 6\htmlayout.dll
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\htmlayout.dll - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\plugins\plugin-eSign2.dll - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\ui-main-frame.dll - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\ui-engine.dll - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\win-specific-services.dll - 
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Program Files\PDF Architect 6\notification-service.dll - 
GetUrlPageData2 (WinHttp) failed: 12002.

KEY_VALUES_STRING: 1

TIMELINE_ANALYSIS: 1

Timeline: !analyze.Start
    Name: <blank>
    Time: 2018-05-24T07:54:00.934Z
    Diff: 65 mSec

Timeline: Dump.Current
    Name: <blank>
    Time: 2018-05-24T07:54:01.0Z
    Diff: 0 mSec

Timeline: Process.Start
    Name: <blank>
    Time: 2018-05-24T07:53:25.0Z
    Diff: 36000 mSec

Timeline: OS.Boot
    Name: <blank>
    Time: 2018-05-24T03:02:52.0Z
    Diff: 17469000 mSec


DUMP_CLASS: 2

DUMP_QUALIFIER: 0

FAULTING_IP: 
pdmodel!PDMODELProvidePDModelHFT+6e034
00007ffd`5bffe404 81790800080000  cmp     dword ptr [rcx+8],800h

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ffd5bffe404 (pdmodel!PDMODELProvidePDModelHFT+0x000000000006e034)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: 0000000000000008
Attempt to read from address 0000000000000008

FAULTING_THREAD:  00000b0c

DEFAULT_BUCKET_ID:  NULL_CLASS_PTR_READ

PROCESS_NAME:  architect.exe

FOLLOWUP_IP: 
pdmodel!PDMODELProvidePDModelHFT+6e034
00007ffd`5bffe404 81790800080000  cmp     dword ptr [rcx+8],800h

READ_ADDRESS:  0000000000000008 
ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.
EXCEPTION_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.
EXCEPTION_CODE_STR:  c0000005
EXCEPTION_PARAMETER1:  0000000000000000
EXCEPTION_PARAMETER2:  0000000000000008
WATSON_BKT_PROCSTAMP:  5aa6dc1c
WATSON_BKT_PROCVER:  6.0.27.37336
PROCESS_VER_PRODUCT:  PDF Architect 6
WATSON_BKT_MODULE:  pdmodel.dll
WATSON_BKT_MODSTAMP:  5aa6bd34
WATSON_BKT_MODOFFSET:  12e404
WATSON_BKT_MODVER:  6.0.27.37336
MODULE_VER_PRODUCT:  PDF Architect 6
BUILD_VERSION_STRING:  10240.17443.amd64fre.th1.170602-2340
MODLIST_WITH_TSCHKSUM_HASH:  145e850624abc2c57fded8cf0b65db3364cd623a
MODLIST_SHA1_HASH:  f872f18894c0de0d204e2bd2bfed332bad99fa8d
NTGLOBALFLAG:  400
PROCESS_BAM_CURRENT_THROTTLED: 0
PROCESS_BAM_PREVIOUS_THROTTLED: 0
APPLICATION_VERIFIER_FLAGS:  0
PRODUCT_TYPE:  1
SUITE_MASK:  272
DUMP_TYPE:  fe
ANALYSIS_SESSION_HOST:  DESKTOP-1IBEKMI
ANALYSIS_SESSION_TIME:  05-24-2018 15:54:00.0934
ANALYSIS_VERSION: 10.0.17134.1 amd64fre

THREAD_ATTRIBUTES: 
OS_LOCALE:  ENU

PROBLEM_CLASSES: 

    ID:     [0n309]
    Type:   [@ACCESS_VIOLATION]
    Class:  Addendum
    Scope:  BUCKET_ID
    Name:   Omit
    Data:   Omit
    PID:    [Unspecified]
    TID:    [0xb0c]
    Frame:  [0] : pdmodel!PDMODELProvidePDModelHFT

    ID:     [0n281]
    Type:   [INVALID_POINTER_READ]
    Class:  Primary
    Scope:  BUCKET_ID
    Name:   Add
    Data:   Omit
    PID:    [Unspecified]
    TID:    [0xb0c]
    Frame:  [0] : pdmodel!PDMODELProvidePDModelHFT

    ID:     [0n306]
    Type:   [NULL_CLASS_PTR_READ]
    Class:  Primary
    Scope:  DEFAULT_BUCKET_ID (Failure Bucket ID prefix)
            BUCKET_ID
    Name:   Add
    Data:   Omit
    PID:    [0xa98]
    TID:    [0xb0c]
    Frame:  [0] : pdmodel!PDMODELProvidePDModelHFT

BUGCHECK_STR:  APPLICATION_FAULT_NULL_CLASS_PTR_READ_INVALID_POINTER_READ
PRIMARY_PROBLEM_CLASS:  APPLICATION_FAULT
LAST_CONTROL_TRANSFER:  from 00007ffd5c16aa64 to 00007ffd5bffe404

STACK_TEXT:  
000000a4`3822d0d0 00007ffd`5c16aa64 : 00000000`00000000 000000a4`331b3ba0 000000a4`33738ab0 000000a4`331b45c0 : pdmodel!PDMODELProvidePDModelHFT+0x6e034
000000a4`3822d130 00007ffd`5c129e9c : 000000a4`334b0bc0 000000a4`331b33c0 000000a4`331b33c0 000000a4`00000078 : pdmodel!PDMODELProvidePDModelHFT+0x1da694
000000a4`3822d200 00007ffd`5c068025 : 000000a4`00000026 000000a4`331b33c0 000000a4`330bc4e0 000000a4`000000b7 : pdmodel!PDMODELProvidePDModelHFT+0x199acc
000000a4`3822d340 00007ffd`5c16a8d5 : 000000a4`334b0bc0 000000a4`000001be 000000a4`332c66c0 000000a4`3822d3e8 : pdmodel!PDMODELProvidePDModelHFT+0xd7c55
000000a4`3822d3c0 00007ffd`5c0eaff3 : 000000a4`332c66c0 00007ffd`00000054 000000a4`332c66c0 000000a4`3822d4a0 : pdmodel!PDMODELProvidePDModelHFT+0x1da505
000000a4`3822d470 00007ffd`5c068025 : 000000a4`0000007a 00000000`ffffffff 000000a4`3320bd00 000000a4`2da99700 : pdmodel!PDMODELProvidePDModelHFT+0x15ac23
000000a4`3822d680 00007ffd`5c04cd6e : 000000a4`332b8750 000000a4`000001be 000000a4`332c66c0 000000a4`3822d728 : pdmodel!PDMODELProvidePDModelHFT+0xd7c55
000000a4`3822d700 00007ffd`5c005a79 : 000000a4`3822d8e8 000000a4`3822d810 000000a4`331b39c0 000000a4`3822d790 : pdmodel!PDMODELProvidePDModelHFT+0xbc99e
000000a4`3822d760 00007ffd`5c00c548 : 000000a4`3320bd00 000000a4`3822d800 000000a4`3822d8e8 000000a4`3822d829 : pdmodel!PDMODELProvidePDModelHFT+0x756a9
000000a4`3822d7c0 00007ffd`5c00a92b : 000000a4`327e0000 000000a4`3822d8e8 00000000`00000294 000000a4`3822d8c8 : pdmodel!PDMODELProvidePDModelHFT+0x7c178
000000a4`3822d890 00007ffd`5c00a00c : 000000a4`3330cb00 000000a4`3822da10 000000a4`332c68f8 000000a4`3822d970 : pdmodel!PDMODELProvidePDModelHFT+0x7a55b
000000a4`3822d930 00007ffd`5c003708 : 000000a4`332c68f8 000000a4`332c66c8 000000a4`332c66c0 000000a4`332c66c0 : pdmodel!PDMODELProvidePDModelHFT+0x79c3c
000000a4`3822d9c0 00007ffd`5bfafd7b : 000000a4`33332db0 000000a4`3822db88 000000a4`332c66c8 000000a4`3822db50 : pdmodel!PDMODELProvidePDModelHFT+0x73338
000000a4`3822da50 00007ffd`5bfaee61 : 000000a4`3330eb80 000000a4`32b18670 000000a4`000001fa 00000000`00000000 : pdmodel!PDMODELProvidePDModelHFT+0x1f9ab
000000a4`3822dbf0 00007ffd`5c0a2c4e : 00000000`00000000 000000a4`3822dca0 00000000`000001fa 00007ffd`5c338cf1 : pdmodel!PDMODELProvidePDModelHFT+0x1ea91
000000a4`3822dc60 00007ffd`5c0a6a1a : 000000a4`00000025 00007ffd`000001fa 00000000`00000000 00000000`00000000 : pdmodel!PDMODELProvidePDModelHFT+0x11287e
000000a4`3822dce0 00007ffd`5c0a5b87 : 000000a4`332d42f0 000000a4`000001fa 000000a4`3822df10 00000000`00000000 : pdmodel!PDMODELProvidePDModelHFT+0x11664a
000000a4`3822de20 00007ffd`5c0a85bb : 000000a4`3822df10 000000a4`00000000 000000a4`332d42f0 000000a4`3822e248 : pdmodel!PDMODELProvidePDModelHFT+0x1157b7
000000a4`3822deb0 00007ffd`5c057c0a : 000000a4`3822e248 000000a4`3822ded0 000000a4`3822e6e0 000000a4`33332e90 : pdmodel!PDMODELProvidePDModelHFT+0x1181eb
000000a4`3822e1c0 00007ffd`5c04eb57 : 000000a4`3822e418 000000a4`3330e580 000000a4`3822e3f0 000000a4`3320bd00 : pdmodel!PDMODELProvidePDModelHFT+0xc783a
000000a4`3822e3b0 00007ffd`5c04d787 : 000000a4`30b6bf10 00007ffd`5bf8a3d1 000000a4`30b6bf10 000000a4`3330e680 : pdmodel!PDMODELProvidePDModelHFT+0xbe787
000000a4`3822e4e0 00007ffd`5bf618e3 : 000000a4`334ae4b0 00000000`00000038 ffffffff`fffffffe 00007ffd`00000002 : pdmodel!PDMODELProvidePDModelHFT+0xbd3b7
000000a4`3822e540 00007ffd`5bf61d7b : 000000a4`3330e680 00000000`00000001 000000a4`3822e7e8 00000000`00000000 : pdmodel!PDMODELProvidePDTextBlockHFT+0x8463
000000a4`3822e5a0 00007ffd`5a0a24b6 : 000000a4`3330e680 000000a4`3822e7e0 00000000`00000000 00000000`00000000 : pdmodel!PDMODELProvidePDTextBlockHFT+0x88fb
000000a4`3822e5e0 00007ffd`5a086f01 : 000000a4`330bb300 000000a4`3822e7e0 00000000`00000000 00000000`00000001 : bl_views!CreateServiceObject+0x308f6
000000a4`3822e710 00007ffd`5a087ace : 000000a4`2d042eb0 000000a4`331821e0 000000a4`2d042eb0 000000a4`3822f0d0 : bl_views!CreateServiceObject+0x15341
000000a4`3822e870 00007ffd`5a088481 : 000000a4`32c126f0 000000a4`32c126f0 000000a4`33121040 000000a4`00000026 : bl_views!CreateServiceObject+0x15f0e
000000a4`3822e9d0 00007ffd`5a0a1768 : 000000a4`32c126f0 00007ffd`6ff4e7dc 000000a4`3822f0c0 00000000`00000021 : bl_views!CreateServiceObject+0x168c1
000000a4`3822ea40 00007ffd`5a0a1c5d : 000000a4`33443d68 000000a4`3822ebc8 00000000`00000000 00000000`00000000 : bl_views!CreateServiceObject+0x2fba8
000000a4`3822eb10 00007ffd`5a05230b : 000000a4`2d82c0b0 00000000`00000000 00000000`00000000 00000000`00000000 : bl_views!CreateServiceObject+0x3009d
000000a4`3822eb50 00007ffd`5a056e8a : 000000a4`32b18990 00007ffd`5a05f4f0 000000a4`32daf230 000000a4`3822f548 : bl_views!ServiceObjectModuleOnFree+0xbc41b
000000a4`3822f2b0 00007ffd`59f9ca4f : 00000000`00000030 000000a4`33443d30 ffffffff`fffffffe 000000a4`330bb1a0 : bl_views!ServiceObjectModuleOnFree+0xc0f9a
000000a4`3822f2e0 00007ffd`59f9c33b : 000000a4`32c13620 000000a4`33166410 000000a4`3822f330 000000a4`3822f338 : bl_views!ServiceObjectModuleOnFree+0x6b5f
000000a4`3822f330 00007ffd`5a05aafa : 00000000`00000000 00000000`00000000 000000a4`3345a570 00000000`00000001 : bl_views!ServiceObjectModuleOnFree+0x644b
000000a4`3822f370 00007ffd`5a05b059 : 000000a4`32c0c338 00007ffd`5a3a57ad 000000a4`3822f568 000000a4`3822f548 : bl_views!ServiceObjectModuleOnFree+0xc4c0a
000000a4`3822f510 00007ffd`5a05cd2e : 000000a4`33480c40 00007ffd`5a48b5e0 000000a4`3345a900 00000000`00000b0c : bl_views!ServiceObjectModuleOnFree+0xc5169
000000a4`3822f5d0 00007ffd`5a05ca39 : 000000a4`3822f6e0 00007ffd`5a48b5a1 000000a4`3345ad20 000000a4`3330b280 : bl_views!ServiceObjectModuleOnFree+0xc6e3e
000000a4`3822f6c0 00007ffd`5a3a46e3 : 000000a4`32b18490 000000a4`32b18490 00007ffd`5a3a46a0 000000a4`3345ad20 : bl_views!ServiceObjectModuleOnFree+0xc6b49
000000a4`3822f780 00007ffd`77ee829d : 00000000`00000000 00007ffd`77ee8240 000000a4`3345ad20 00000000`00000000 : bl_views!CreateServiceObject+0x332b23
000000a4`3822f7c0 00007ffd`7ec42d92 : 00007ffd`77ee8240 000000a4`3345ad20 00000000`00000000 00000000`00000000 : ucrtbase!thread_start<unsigned int (__cdecl*)(void * __ptr64)>+0x5d
000000a4`3822f7f0 00007ffd`7f1d9f64 : 00007ffd`7ec42d70 00000000`00000000 00000000`00000000 00000000`00000000 : KERNEL32!BaseThreadInitThunk+0x22
000000a4`3822f820 00000000`00000000 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x34


THREAD_SHA1_HASH_MOD_FUNC:  fd99fba49920b9945d29f403b0cd998e5fb12059
THREAD_SHA1_HASH_MOD_FUNC_OFFSET:  d23174f428ffb033899771085f7f08609fc306a3
THREAD_SHA1_HASH_MOD:  eaa06e4d44662fa454bcfa636a5a191bf6e363db
FAULT_INSTR_CODE:  87981
SYMBOL_STACK_INDEX:  0
SYMBOL_NAME:  pdmodel!PDMODELProvidePDModelHFT+6e034
FOLLOWUP_NAME:  MachineOwner
MODULE_NAME: pdmodel
IMAGE_NAME:  pdmodel.dll
DEBUG_FLR_IMAGE_TIMESTAMP:  5aa6bd34
STACK_COMMAND:  ~28s ; .cxr ; kb
FAILURE_BUCKET_ID:  NULL_CLASS_PTR_READ_c0000005_pdmodel.dll!PDMODELProvidePDModelHFT
BUCKET_ID:  APPLICATION_FAULT_NULL_CLASS_PTR_READ_INVALID_POINTER_READ_pdmodel!PDMODELProvidePDModelHFT+6e034
FAILURE_EXCEPTION_CODE:  c0000005
FAILURE_IMAGE_NAME:  pdmodel.dll
BUCKET_ID_IMAGE_STR:  pdmodel.dll
FAILURE_MODULE_NAME:  pdmodel
BUCKET_ID_MODULE_STR:  pdmodel
FAILURE_FUNCTION_NAME:  PDMODELProvidePDModelHFT
BUCKET_ID_FUNCTION_STR:  PDMODELProvidePDModelHFT
BUCKET_ID_OFFSET:  6e034
BUCKET_ID_MODTIMEDATESTAMP:  5aa6bd34
BUCKET_ID_MODCHECKSUM:  866b74
BUCKET_ID_MODVER_STR:  6.0.27.37336
BUCKET_ID_PREFIX_STR:  APPLICATION_FAULT_NULL_CLASS_PTR_READ_INVALID_POINTER_READ_
FAILURE_PROBLEM_CLASS:  APPLICATION_FAULT
FAILURE_SYMBOL_NAME:  pdmodel.dll!PDMODELProvidePDModelHFT
WATSON_STAGEONE_URL:  http://watson.microsoft.com/StageOne/architect.exe/6.0.27.37336/5aa6dc1c/pdmodel.dll/6.0.27.37336/5aa6bd34/c0000005/0012e404.htm?Retriage=1

TARGET_TIME:  2018-05-24T07:55:10.000Z
OSBUILD:  10240
OSSERVICEPACK:  17113
SERVICEPACK_NUMBER: 0
OS_REVISION: 0
OSPLATFORM_TYPE:  x64
OSNAME:  Windows 10
OSEDITION:  Windows 10 WinNt SingleUserTS
USER_LCID:  0
OSBUILD_TIMESTAMP:  2016-09-07 12:12:10
BUILDDATESTAMP_STR:  170602-2340
BUILDLAB_STR:  th1
BUILDOSVER_STR:  10.0.10240.17443.amd64fre.th1.170602-2340
ANALYSIS_SESSION_ELAPSED_TIME:  161dd
ANALYSIS_SOURCE:  UM
FAILURE_ID_HASH_STRING:  um:null_class_ptr_read_c0000005_pdmodel.dll!pdmodelprovidepdmodelhft
FAILURE_ID_HASH:  {582d441e-69c7-d099-1bc5-bb628400ba2e}

Followup:     MachineOwner
---------
```
