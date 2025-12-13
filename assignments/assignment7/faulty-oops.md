# Kernel Oops Analysis: Faulty Module NULL Pointer Dereference

This document analyzes a kernel oops (segmentation fault) triggered by the aesd-faulty kernel module when writing to /dev/faulty. This is intentional behavior for demonstrating kernel error handling.

## Oops Summary

- Error type: NULL pointer dereference
- Location: faulty_write() in aesd_faulty module
- Trigger: Writing data to /dev/faulty
- Severity: Kernel oops (non-fatal, system continues running)


## Detailed Error Analysis


1- Error Message

`Unable to handle kernel NULL pointer dereference at virtual address 0000000000000000`

- Virtual address 0x0000000000000000 is NULL
- The kernel attempted to write to this address
- This is a level 1 translation fault (page table entry is invalid)

2- Exception Information

```
ESR = 0x0000000096000045
EC = 0x25: DABT (current EL), IL = 32 bits
FSC = 0x05: level 1 translation fault
WnR = 1: Write operation (not read)
```

- Data Abort (DABT) at current exception level
- Translation fault: no valid page table entry for address 0
- Write operation attempted

3- Fault Location

`pc : faulty_write+0x18/0x20 [aesd_faulty]`

- Program counter at faulty_write+0x18
- Function size: 0x20 (32 bytes)
- Offset 0x18 (24 bytes) into the function
- This corresponds to: *(int *)0 = 0;

4- Call Stack Analysis

```
Call trace:  
faulty_write+0x18/0x20 [aesd_faulty]    ← Our module's write function  
ksys_write+0x74/0x110                  ← Kernel syscall write handler  
__arm64_sys_write+0x24/0x30            ← ARM64 syscall wrapper  
invoke_syscall+0x5c/0x130              ← Syscall invocation  
el0_svc_common.constprop.0+0x4c/0x100  ← Exception level 0 service call  
do_el0_svc+0x4c/0xc0                    ← Handle EL0 syscall  
el0_svc+0x28/0x80                      ← EL0 syscall entry  
el0t_64_sync_handler+0xa4/0x130         ← Exception handler  
el0t_64_sync+0x1a0/0x1a4              ← Exception entry point
```

Flow:

- User space: echo "hello_world" > /dev/faulty
- System call: write() syscall
- VFS: vfs_write() → ksys_write()
- Driver: faulty_write() in the module
- Fault: NULL pointer dereference at *(int *)0 = 0;

```
root@qemuarm64:~# mknod /dev/faulty c 249 0
root@qemuarm64:~# ls -la /dev/faulty 
crw-r--r--    1 root     root      249,   0 Dec 13 12:01 /dev/faulty
root@qemuarm64:~# echo “hello_world” > /dev/faulty
[ 1164.709576] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000000
[ 1164.710090] Mem abort info:
[ 1164.710171]   ESR = 0x0000000096000045
[ 1164.710270]   EC = 0x25: DABT (current EL), IL = 32 bits
[ 1164.710383]   SET = 0, FnV = 0
[ 1164.710451]   EA = 0, S1PTW = 0
[ 1164.710523]   FSC = 0x05: level 1 translation fault
[ 1164.710630] Data abort info:
[ 1164.710723]   ISV = 0, ISS = 0x00000045
[ 1164.710833]   CM = 0, WnR = 1
[ 1164.711023] user pgtable: 4k pages, 39-bit VAs, pgdp=0000000043717000
[ 1164.711212] [0000000000000000] pgd=0000000000000000, p4d=0000000000000000, pud=0000000000000000
[ 1164.711695] Internal error: Oops: 0000000096000045 [#1] PREEMPT SMP
[ 1164.712068] Modules linked in: aesd_faulty(O) scull(O) hello(O) [last unloaded: aesd_faulty]
[ 1164.716742] CPU: 1 PID: 399 Comm: sh Tainted: G           O      5.15.194-yocto-standard #1
[ 1164.717138] Hardware name: linux,dummy-virt (DT)
[ 1164.717596] pstate: 80000005 (Nzcv daif -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[ 1164.718474] pc : faulty_write+0x18/0x20 [aesd_faulty]
[ 1164.722096] lr : vfs_write+0xf8/0x2a0
[ 1164.722528] sp : ffffffc00970bd80
[ 1164.722883] x29: ffffffc00970bd80 x28: ffffff80035b8000 x27: 0000000000000000
[ 1164.723368] x26: 0000000000000000 x25: 0000000000000000 x24: 0000000000000000
[ 1164.723772] x23: 0000000000000000 x22: ffffffc00970bdc0 x21: 000000558b676b90
[ 1164.724743] x20: ffffff800364d400 x19: 0000000000000012 x18: 0000000000000000
[ 1164.725153] x17: 0000000000000000 x16: 0000000000000000 x15: 0000000000000000
[ 1164.725525] x14: 0000000000000000 x13: 0000000000000000 x12: 0000000000000000
[ 1164.726135] x11: 0000000000000000 x10: 0000000000000000 x9 : ffffffc008272108
[ 1164.729007] x8 : 0000000000000000 x7 : 0000000000000000 x6 : 0000000000000000
[ 1164.729411] x5 : 0000000000000001 x4 : ffffffc000bb1000 x3 : ffffffc00970bdc0
[ 1164.729779] x2 : 0000000000000012 x1 : 0000000000000000 x0 : 0000000000000000
[ 1164.730300] Call trace:
[ 1164.730478]  faulty_write+0x18/0x20 [aesd_faulty]
[ 1164.731179]  ksys_write+0x74/0x110
[ 1164.731387]  __arm64_sys_write+0x24/0x30
[ 1164.731592]  invoke_syscall+0x5c/0x130
[ 1164.731810]  el0_svc_common.constprop.0+0x4c/0x100
[ 1164.732071]  do_el0_svc+0x4c/0xc0
[ 1164.732245]  el0_svc+0x28/0x80
[ 1164.734310]  el0t_64_sync_handler+0xa4/0x130
[ 1164.734914]  el0t_64_sync+0x1a0/0x1a4
[ 1164.735753] Code: d2800001 d2800000 d503233f d50323bf (b900003f) 
[ 1164.736411] ---[ end trace 742f4a708059319c ]---
Segmentation fault

Poky (Yocto Project Reference Distro) 4.0.31 qemuarm64 /dev/ttyAMA0
```