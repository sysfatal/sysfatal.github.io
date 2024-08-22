---
title:      Empty Capabilities (not really]
summary:    capabilities
categories: blog
date:       Thu Aug 22 13:35:04 CEST 2024
thumbnail:  capabilities
image:      /figs/caps.jpg
layout:     post
author:     e__soriano
tags:
 - capabilities
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
  <img src="figs/caps.jpg">
</figure>
</center>

It's said that a file with empty capabilies runs as a root setuid executable.
For example, [hacktricks states](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities):

```
The special case of "empty" capabilities

From the docs: Note that one can assign empty capability
sets to a program file, and thus it is possible to create a
set-user-ID-root program that changes the effective and saved
set-user-ID of the process that executes the program to 0,
but confers no capabilities to that process. Or, simply put,
if you have a binary that:

    is not owned by root

    has no SUID/SGID bits set

    has empty capabilities set (e.g.: getcap myelf returns
    myelf =ep)

then that binary will run as root.
```

Let's try it. The *setcap* command can change the
capabilities of a file.  
For example, this command:

```
$ sudo setcap cap_sys_admin+ep a.out
```

adds the *cap_sys_admin* capability to the sets
*effective* (e) and *permitted* (p).

Now, we create a copy of the `cat` command
and set *"empty capabilities"* :

```
$ cp /bin/cat mycat
$ sudo setcap =ep mycat
```
Let's check it:

```
$ getcap mycat
mycat =ep
$
```
We can see that the sets are empty, right?
If we execute this file to dump */etc/shadow* (only root can read
this file):

```
$ ls -l
total 36
-rwxr-xr-x 1 esoriano esoriano 35288 aug 20 14:02 mycat
$ ./mycat /etc/shadow | grep root
root:!:19391:0:99999:7:::
$
```
It works. So, it's true that this file is executing with
root privileges... with which UID?

Let's execute this custom C program:

```
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <err.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

enum{
	Bufsize = 8* 1024,
};

int
main(int argc, char *argv[])
{
	char buf[Bufsize];
	int nr;
	int fd;

	fprintf(stderr, "uid: %d euid: %d\n",
		getuid(),
		geteuid());

	fd = open("/etc/shadow", O_RDONLY);
	if(fd < 0) {
		err(1, "open failed");
	}
	while((nr = read(fd, buf, Bufsize)) != 0){
		if(nr < 0){
			err(EXIT_FAILURE, "can't read");
		}
		if(write(1, buf, nr) != nr){
			err(EXIT_FAILURE, "can't write");
		}
	}
	close(fd);
	exit(EXIT_SUCCESS);
}
```

This program just prints the UID (real and effective) and
dumps the corresponding file.
If we compile and execute it to dump */etc/shadow*:

```
$ gcc read.c
$ ./a.out
uid: 1000 euid: 1000
a.out: open failed: Permission denied
```

Now, with *empty* capabilities:

```
$ sudo setcap =ep ./a.out
$ ./a.out
uid: 1000 euid: 1000
root:!:19391:0:99999:7:::
...
$
```

It executes with the user's UID, 1000 (root is 0).

But, are the file capabilities empty?

# Extended attributes and capabilities encoding

File capabilities are extended
attributes. Extended attributes in Ext file systems are
metadata entries that are not stored within the i-node.
They are stored in a dedicated extra block. Extended attributes
have a key name. The name for the capabilities is
*security.capability*.
We can dump the values
of the extended attributes with the *getfattr* command:

```
$ getfattr -e hex -n security.capability mycat
# file: mycat
security.capability=0x01000002ffffffff00000000ff01000000000000
$
```

As showed above, the attribute exists and it is not empty.  
How are the capabilities encoded in this hexadecimal value?

There are 3 different versions of capabilities (the last is *VFS_CAP_REVISION_3*)
and they are backwards compatible.
V3 uses 64-bit values for each set, but they are not contiguous (due to backwards compatibility!).

The first 4 bytes form a "header" (that includes the
version and other bits).
The next 16 bytes are the permitted
and effective bits (64-bit each). There can be extra bytes.

Let's play with these values.
First, we delete all capabilities from a file:

```
$ cp /bin/echo echo
$ setcap all=-eip echo
$ getcap echo
echo =
$ getfattr --dump --match="." -e hex echo
# file: echo
security.capability=0x0000000200000000000000000000000000000000
```

The first 64-bit value, 0x00000002, is the header. This
is version 0x02, and the rest of bits are set to 0.

Now, we enable all the permitted capabilities:

```
$ setcap all=-eip echo
$ setcap all=p echo
$ getcap echo
echo =p
$ getfattr --dump --match="." -e hex echo
# file: echo
security.capability=0x00000002ffffffff00000000ff01000000000000
```

* 00000002 <- header

* ffffffff <- first 32 bits for permitted

* 00000000 <- first 32 bits for inherited

* ff010000 <- last 32 bits for permitted

* 00000000 <- last 32 bits for inherited


Now, we enable only the inherited capabilities:

```
$ setcap all=-eip echo
$ setcap all=i echo
$ getfattr --dump --match="." -e hex echo
# file: echo
security.capability=0x0000000200000000ffffffff00000000ff010000
```

* 00000002 <- header

* 00000000 <- first 32 bits for permitted

* ffffffff <- first 32 bits for inherited

* 00000000 <- last 32 bits for permitted

* ff010000 <- last 32 bits for inherited

In this case, we enable the effective capabilities.
Note that in file capabilities, the effective set is
just one bit:

```
$ setcap all=-eip echo
$ setcap all=e echo
$ getcap echo
echo =
$ getfattr --dump --match="." -e hex echo
# file: echo
security.capability=0x0100000200000000000000000000000000000000
```

* 01000002 <- header, effective bit set to 1

* 00000000 <- first 32 bits for permitted

* 00000000 <- first 32 bits for inherited

* 00000000 <- last 32 bits for permitted

* 00000000 <- last 32 bits for inherited

Last, we enable all and add a root UID for a namespace:

```
$ setcap -n 255 all=eip echo
$ getcap -n echo
echo =eip [rootid=255]
$ getfattr --dump --match="." -e hex echo
# file: echo
security.capability=0x01000003ffffffffffffffffff010000ff010000ff000000
```

* 01000003 <- header, the version has changed to V3

* ffffffff <- first 32 bits for permitted

* ffffffff <- first 32 bits for inherited

* ff010000 <- last 32 bits for permitted

* ff010000 <- last 32 bits for inherited

* ff000000 <- extra bytes for the root UID 255 (0xff)



# No empty capabilities

Therefore, the capabilities are not *empty*:

```
$ getfattr -e hex -n security.capability mycat
# file: mycat
security.capability=0x01000002ffffffff00000000ff01000000000000
$
```

The effective bit is set to 1, the inherited set is empty
and the permitted has all the capabilities enabled.

What's happening here?

The problem is the textual representation of the capabilities,
described in the manual page *cap_from_text(3)*  and used
by the *getcap* and *setcap* commands. It states:

```
In the case that the leading operator is '=', and no list of
capabilities is provided, the action-list is assumed to refer to 'all'
capabilities. For example, the following three clauses are equivalent
to each other (and indicate a completely empty capability set): "all=";
"="; "cap_chown,<every-other-capability>=".
```

Let's check if this is true:

```

$ cp /bin/cat c1
$ cp /bin/cat c2
$ sudo setcap =ep c1
$ sudo setcap all=ep c2
$ getfattr --dump --match="." -e hex c1
# file: c1
security.capability=0x01000002ffffffff00000000ff01000000000000
$ getfattr --dump --match="." -e hex c2
# file: c2
security.capability=0x01000002ffffffff00000000ff01000000000000
$ getcap c1
c1 =ep
$ getcap c2
c2 =ep
```

Yes, it's true:
_"all"_ and the empty string are equivalent in this syntax. Why? **¯\\\_(ツ)_/¯**

In my opinion, that was an unfortunate decision. It leads
to confusion.

Capabilities in Linux are obscure and too complex. Remember, complexity is the enemy of security.

<sub><sup>
    <b>(cc) Enrique Soriano-Salvador</b>
    Algunos derechos reservados. Este trabajo se entrega bajo la licencia
    Creative Commons Reconocimiento - NoComercial - SinObraDerivada (by-nc-nd).
    Creative Commons, 559 Nathan Abbott Way, Stanford,
    California 94305, USA.
</sup></sub>
