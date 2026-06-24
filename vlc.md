---
title:      Running VLC as root and the magic sed
summary:    binary
categories: blog
date:       Wed Jun 24 05:17:00 PM CEST 2026
thumbnail:  vlc
image:      /figs/vlc.png
layout:     post
author:     e__soriano
tags:
 - vlc
---

<div class="share-page">
    Share this on &rarr;
    [<a title="Share on Mastodon" href="https://tootpick.org/#text=Check%20out%20https://sysfatal.github.io{{  page.url }}%20by%20@esoriano@social.linux.pizza">Mastodon</a>]
    [<a href="https://twitter.com/intent/tweet?text={{ page.title }}&url={{ site.url }}{{ page.url }}&via=e__soriano&related=e__soriano" rel="nofollow" target="_blank" title="Share on Twitter">Twitter</a>]
    [<a href="https://facebook.com/sharer.php?u={{ site.url }}{{ page.url }}" rel="nofollow" target="_blank" title="Share on Facebook">Facebook</a>]
</div>
<br>

<center>
<figure class="image">
  <img src="figs/vlc.png">
  <figcaption> vlc as root </figcaption>
</figure>
</center>

Recently, a student told me that it wasn’t possible to run VLC as root… it seemed strange to me, so I tried it:

```
# id
uid=0(root) gid=0(root) groups=0(root)
# vlc
VLC is not supposed to be run as root. Sorry.
If you need to use real-time priorities and/or privileged TCP ports
you can use vlc-wrapper (make sure it is Set-UID root and
cannot be run by non-trusted users first).
#
```

Indeed, it does not allow itself to be run as root due to security reasons.
It appears that the application can be recompiled with an option to disable this check.

I would have thought that the quickest approach would be to patch the binary with radare2, for instance, to bypass the check. However, I came across this rather amusing solution: running a simple `sed` command to replace the string geteuid with the string getppid in the
ELF binary, like this:

```
# pwd
/tmp
# cp $(which vlc) vlc
# sed -i 's/geteuid/getppid/g' vlc
# ./vlc
VLC media player 3.0.20 Vetinari (revision 3.0.20-0-g6f0d0ab126b)
[000064c8dd5c3a80] vlcpulse audio output error: PulseAudio server connection failure: Connection refused
[000064c8dd505550] main libvlc: Running vlc with the default interface. Use 'cvlc' to use vlc without interface.
QStandardPaths: XDG_RUNTIME_DIR not set, defaulting to '/tmp/runtime-root'
[000064c8dd59d770] main playlist: playlist is empty
```

Well, it works! But, how? We are dealing with a typical case that surprises neither a novice (someone who does not know what an executable binary is, who has never heard of ELF or thinks that a binary is like a python program) nor an expert in systems and binary analysis, but rather those who fall somewhere in between these two extremes.

VLC checks the effective UID credential: if it is 0 (the UID of root), it aborts the execution. The trick here is to replace this call. Instead of calling the `geteuid` function, the program will call the `getppid` function. Both functions belong to the C standard library. Both have seven-character names. Both take no parameters. Both return an integer value. The key point is that getppid will never return 0. This function returns the PID of the parent process (which will not be 0). Therefore, VLC will assume that the user running the program is not root (0).

Is it calling the `getppid` function?

```
# strace ./vlc 2>&1 | grep getppid
getppid()                               = 99697
```
Yes, there is a `getppid` system call.

If we try with another function:

```
# sed -i 's/geteuid/getpid/g' vlc
# ./vlc
Segmentation fault (core dumped)
#
```
Why? The lenght of the name is not 7 chars :)

But how can that sed command achieve this simply by replacing one string with another? Why doesn’t it break the binary? Why it is now calling `getppid` instead of `geteuid`?

VLC is a stripped dynamically linked binary with full RELRO:

```
# cp $(which vlc) vlc
# file vlc
vlc: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=846755832ef8bb6ccc424ad2d6d30be209083a92, for GNU/Linux 3.2.0, stripped
# nm vlc
nm: vlc: no symbols
# checksec --file=./vlc
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH	Symbols		FORTIFY	Fortified	Fortifiable	FILE
Full RELRO      Canary found      NX enabled    PIE enabled     No RPATH   No RUNPATH   No Symbols	  Yes	2		3		./vlc
#
```

This means that, before running, all the relocations for dynamic libraries are resolved. 
If we dump the dynamic symbols table, we can see the `geteuid` symbol:

```
# objdump -T vlc | grep geteuid
0000000000000000      DF *UND*	0000000000000000 (GLIBC_2.2.5) geteuid
#
```

But, where is the `"geteuid"` string? Lets use `radare2`:

```
[0x000018e0]> / geteuid
Searching 7 bytes in [0x4010-0x4020]
hits: 0
Searching 7 bytes in [0x3c98-0x4010]
hits: 0
Searching 7 bytes in [0x2000-0x23e8]
hits: 0
Searching 7 bytes in [0x1000-0x1bbd]
hits: 0
Searching 7 bytes in [0x0-0xee8]
hits: 1
Searching 7 bytes in [0x100000-0x1f0000]
hits: 0
0x00000892 hit0_0 ._stack_chk_failgeteuidisattysigempty.
[0x000018e0]> px 128 @ 0x00000892
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x00000892  6765 7465 7569 6400 6973 6174 7479 0073  geteuid.isatty.s
0x000008a2  6967 656d 7074 7973 6574 0073 6967 6164  igemptyset.sigad
0x000008b2  6473 6574 0070 7468 7265 6164 5f73 656c  dset.pthread_sel
0x000008c2  6600 7074 6872 6561 645f 7369 676d 6173  f.pthread_sigmas
0x000008d2  6b00 6c69 6276 6c63 5f67 6574 5f63 6861  k.libvlc_get_cha
0x000008e2  6e67 6573 6574 006c 6962 766c 635f 6765  ngeset.libvlc_ge
0x000008f2  745f 7665 7273 696f 6e00 6d65 6d63 7079  t_version.memcpy
0x00000902  006c 6962 766c 635f 6e65 7700 6c69 6276  .libvlc_new.libv
[0x000018e0]> 0x00000892
[[0x00000892]> iS.
Current section

nth paddr        size vaddr       vsize perm name
―――――――――――――――――――――――――――――――――――――――――――――――――
0   0x00000798  0x271 0x00000798  0x271 -r-- .dynstr


[0x00000892]>

```

Here it is, together with other library function names, in the `.dynstr` section
of the ELF file.

What's this? According to the manual page `elf(5)`:

```
.dynstr
	 This section holds strings needed for dynamic linking, most
	 commonly the strings that represent the names associated
	 with symbol table entries.  This section is of type
	 SHT_STRTAB.  The attribute type used is SHF_ALLOC.
```

Those strings are used by the `.dynsym` table:

```
.dynsym
       This section holds the dynamic linking symbol table.  This
       section is of type SHT_DYNSYM.  The attribute used is
       SHF_ALLOC.
```

When a function is imported, the linker:

- stores its name as a string in the  `.dynstr` section
- creates a corresponding symbol in `.dynsym`. The offset for the corresponding
string is stored in the `st_name` field.
- adds a relocation entry in `.rel.plt` that refers to that symbol

The relocation specifies the destination through the `r_offset` field, 
which points to an entry in the GOT (Global Offset Table). 
This table, located in `.got.plt`, will be initialized 
by the dynamic loader as it resolves the relocations. The loader
will map the code of the libraries in the process memory and
patch the table with the corresponding addresses.     

As said before, with full RERLO, all imported symbols are resolved at
startup time. When the program starts, the .got.plt section is completely
initialized with the final addresses of the functions and the corresponding
memory pages will be marked as read only (as an exploiting mitigation).
Later, when the program calls the function, the trampolines will do the job
and the flow will be redirected to the corresponding function code (located
in the library's text pages).

The point is that the loader will use the name of the function
(`getppid` in this case) to resolve the symbol (that is, to search
the dynamic libraries used by the binary).
Thus, if the string is `getppid`, the code pointed by the
GOT will be the code of the `getppid` funcion of the libc. This code
is in fact the stub code for the `getppid` system call, which returns
the PID of the parent of the process; it will never be 0 (the UID for root).
This way, VLC is cheated.

Remember: don't run VLC as root.

<sub><sup>
    <b>(cc) Enrique Soriano-Salvador</b>
    Algunos derechos reservados. Este trabajo se entrega bajo la licencia
    Creative Commons Reconocimiento - NoComercial - SinObraDerivada (by-nc-nd).
    Creative Commons, 559 Nathan Abbott Way, Stanford,
    California 94305, USA.
</sup></sub>

