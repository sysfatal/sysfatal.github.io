---
title:      Bypassing Frida dynamic function call tracing
summary:    Bypassing Frida dynamic function call tracing
categories: blog
date:       Tue 10 Aug 2021 12:54:49 PM CEST
thumbnail:  frida-trace
image:      https://github.com/sysfatal/figs/frida.jpg
layout:     post
author:     e__soriano
tags:
 - reversing
 - evasion
 - malware
---

<div class="share-page">
    Share this on &rarr;
    [<a href="https://twitter.com/intent/tweet?text={{ page.title }}&url={{ site.url }}{{ page.url }}&via=e__soriano&related=e__soriano" rel="nofollow" target="_blank" title="Share on Twitter">Twitter</a>]
    [<a href="https://facebook.com/sharer.php?u={{ site.url }}{{ page.url }}" rel="nofollow" target="_blank" title="Share on Facebook">Facebook</a>]
</div>
<br>

<center>
<figure class="image">
  <img src="figs/frida.jpg" alt="{{frida calo painting image}}">
</figure>
</center>

-------

# UPDATE: Paper!!!!

This post has been converted into an [article](https://link.springer.com/article/10.1007/s11416-022-00458-7) about antifrida techniques.

**How to cite this work:** Soriano-Salvador, E., Guardiola-Múzquiz, G. Detecting
and bypassing frida dynamic function call tracing: exploitation and mitigation. J
Comput Virol Hack Tech 19, 503–513 (2023). DOI: [10.1007/s11416-022-00458-7](https://link.springer.com/article/10.1007/s11416-022-00458-7)

Open version: PDF available [here](https://burjcdigital.urjc.es/bitstream/handle/10115/25911/2023-frida-bypass-repositorio.pdf)


-------

# Old blog entry ("preprint" content)


Frida is an amazing dynamic analysis tool, it's quite powerful.
Lately, I've been learning how to use it and wondering
how it works. What mechanisms are used?

There are different modes (embedded, injected and preloaded). The default
mode is the *injected* one. In summary, it uses ```ptrace``` to hijack the
control flow of the analyzed process. Then, it allocates some memory in
the process memory space and injects a library (.so), *the agent*.
This agent uses IPC mechanisms (fifo, signals, etc.)
to communicate with external tools. The code of the process is manipulated
to intercept the function calls and analyze them. In order to do the job, it
also uses ```mmap```, ```dlopen```, ```dlsym```, etc.

I figured that Frida was using ```dlsym``` to hook and intercept
the library functions, as usual.  Later, I read their
[OSDC 2015 presentation](https://www.frida.re/slides/osdc-2015-the-engineering-behind-the-reverse-engineering.pdf)
and realized that Frida uses another mechanism to intercept calls.

Frida rewrites the prelude of traced functions. The beginning of the
function is replaced by their code, that jumps to a trampoline (a mechanism
similar to that in the PLT/GOT for lazy binding).
Once the call is intercepted, the flow is redirected
to the agent and the magic is done. The agent can execute whatever it needs:
incrementally, for each branch in the function,
the *stalker* rewrites the basic blocks and stores them in a
new executable page.

## Detection

After reading the description, it's easy to see how a malicious binary could
detect Frida. It can inspect the memory space of the process and find the
regions for the mapped library (the agent).
If we execute a simple program and inspect the memory maps:

```
$>  cat /proc/12555/maps | grep frida
$>
```

Now, if we run ```frida-trace``` in another terminal (as root)
to intercept the calls for ```open``` (the
libc function for the ```open``` syscall):

```
$> frida-trace --version
14.2.18
$> frida-trace -i open a.out
```

and dump the maps again:

```
$> cat /proc/12555/maps | grep frida
7fea7168a000-7fea72c53000 r-xp 00000000 103:04 1968850                   /tmp/frida-81be93df197f7fb45dea328c7237c270/frida-agent-64.so
7fea72c53000-7fea72c54000 ---p 015c9000 103:04 1968850                   /tmp/frida-81be93df197f7fb45dea328c7237c270/frida-agent-64.so
7fea72c54000-7fea72cde000 r--p 015c9000 103:04 1968850                   /tmp/frida-81be93df197f7fb45dea328c7237c270/frida-agent-64.so
7fea72cde000-7fea72cf6000 rw-p 01653000 103:04 1968850                   /tmp/frida-81be93df197f7fb45dea328c7237c270/frida-agent-64.so
$>
```

we find a library that should not be there.

In order to do that, the malicious binary should inspect the address space,
performing suspicious system calls that would alert the analyst.

Nevertheless, given the mechanisms used by Frida to intercept the calls,
there is a more subtle way to detect that the binary is being analyzed:
**the binary can check if the original preludes have been changed**.

This method does not require any external function or syscall.
You only need to check some bytes.
If the preludes have been changed, Frida is around.

## Evasion

If the analyzed binary detects Frida, it can try to bypass the interception.
How? The binary can restore the original prelude before calling the function.
Moreover, it can hide just the important calls (those that perform
the suspicious malicious actions) and let Frida intercept the rest of calls.

Suppose the following example:

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

void
dumpprelude(unsigned char *f)
{	int i;
	int c;

	for(i=0; i<32; i++){
		fprintf(stderr, "%02x", *(f+i));
	}
	fprintf(stderr, "\npress enter...\n");
	read(0, &c, 1);
}

int
f(int x)
{
	fprintf(stderr, "f says hi!\n");
	return ++x;
}

int
main(int argc, char *argv[])
{
	int x;

	dumpprelude((unsigned char*)f);
	dumpprelude((unsigned char*)f);
	x = f(13);
	fprintf(stderr, "x is %d\n", x);
	exit(EXIT_SUCCESS);
}
```

If we compile and execute (pressing enter when prompted):

```
$> gcc -o ex1 ex1.c
$> ./ex1
f30f1efa554889e54883ec10897dfc488b05942d00004889c1ba0b000000be01
press enter...

f30f1efa554889e54883ec10897dfc488b05942d00004889c1ba0b000000be01
press enter...

f says hi!
x is 14
$>
```

we can see that the prelude of the function doesn't change.

If we execute it again:

```
$> ./ex1
f30f1efa554889e54883ec10897dfc488b05942d00004889c1ba0b000000be01
press enter...
```

and, before pressing enter, we run ```frida-trace``` as root in another
terminal like this (in order to trace ```f```, with offset 0x1276 in the
binary text):

```
$> nm ex1 | grep ' f$'
0000000000001276 T f
$> frida-trace -a ex1\!0x1276 ex1
Instrumenting...                                                        
sub_1276: Auto-generated handler at "/tmp/__handlers__/ex1/sub_1276.js"
Started tracing 1 function. Press Ctrl+C to stop.
```

then we press enter in the other terminal

```
e98d4d00004889e54883ec10897dfc488b05942d00004889c1ba0b000000be01
press intro...

f says hi!
x is 14
$>
```

```frida-trace``` intercepts one call (```f```
is named ```sub_1276()```):

```

           /* TID 0x38f4 */
  7393 ms  sub_1276()
Process terminated
$>
```

We can observe that the prelude has changed (the first 5 bytes):

**f30f1efa55**4889e54883ec10897dfc488b05942d00004889c1ba0b000000be01

**e98d4d0000**4889e54883ec10897dfc488b05942d00004889c1ba0b000000be01


The first two instructions of function ```f``` have been replaced
for jumping to the Frida trampoline:

```
$> r2 ex1
 -- ♥ --
 [0x000010e0]> pd 5 @ 0x1276
             ;-- f:
             0x00001276      f30f1efa       endbr64
             0x0000127a      55             pushq %rbp
             0x0000127b      4889e5         movq %rsp, %rbp
             0x0000127e      4883ec10       subq $0x10, %rsp
             0x00001282      897dfc         movl %edi, -4(%rbp)
 [0x000010e0]>
```

Now, lets try to modify the example to restore the original prelude:

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

void
dumpprelude(unsigned char *f)
{	int i;
	int c;

	for(i=0; i<32; i++){
		fprintf(stderr, "%02x", *(f+i));
	}
	fprintf(stderr, "\npress enter...\n");
	read(0, &c, 1);
}

int
f(int x)
{
	fprintf(stderr, "f says hi!\n");
	return ++x;
}

int
main(int argc, char *argv[])
{
	int x;

	dumpprelude((unsigned char*)f);
	dumpprelude((unsigned char*)f);
	memcpy((void*)f, "\xf3\x0f\x1e\xfa\x55", 5);
	dumpprelude((unsigned char*)f);
	x = f(13);
	fprintf(stderr, "x is %d\n", x);
	exit(EXIT_SUCCESS);
}
```

If we execute it (running ```frida-trace``` in another terminal
as in the previos example; note
that the offset has changed, now it's 0x1296):

```
$> ./ex1
f30f1efa554889e54883ec10897dfc488b05742d00004889c1ba0b000000be01
press enter...

e96d4d00004889e54883ec10897dfc488b05742d00004889c1ba0b000000be01
press enter...

f30f1efa554889e54883ec10897dfc488b05742d00004889c1ba0b000000be01
press enter...

f says hi!
x is 14
$>
```

Now Frida says:

```
$> frida-trace -a ex1\!0x1296 ex1
Instrumenting...                                                        
sub_1296: Auto-generated handler at "/tmp/__handlers__/ex1/sub_1296.js"
Started tracing 1 function. Press Ctrl+C to stop.                       
Process terminated
$>
```
**The call has not been intercepted**.

Note that we don't need to use ```memcpy``` to restore the prelude, we
could use a simple loop. This way, we don't depend on any library function
or syscall to restore the prelude.

Wait, we wrote a ```.text``` page and there was not a segmentation fault?
Why?

Frida leaves the write permission for the page enabled after
modifying the code:

```
56550bc57000-56550bc58000 rwxp 00001000 103:02 4724567          /tmp/ex1

```

So, if we run the example *without* running ```frida-trace``` for tracing the
process, it receives the ```SIGSEGV```:

```
$> ./ex1
f30f1efa554889e54883ec10897dfc488b05742d00004889c1ba0b000000be01
press enter...

f30f1efa554889e54883ec10897dfc488b05742d00004889c1ba0b000000be01
press enter...

Segmentation fault (core dumped)
$>
```

Frida leaves this permission enabled
and that's very convenient for the evasion:
We don't need to use ```mprotect``` to enable it. Note that
using ```mprotect``` is a striking clue of malicious activity.

Thus, we are able to perform silent calls to selected functions by
using a simple C macro with the following parameters:

* Variable to store the return value
* Function to be called
* Array with the original prelude for the function
* Arguments for the function (variadic)

For example:

```c
#define HIDECALL(RET,FUNC,PRELUDE,...) \
	{   int i; \
		int presz = sizeof(PRELUDE); \
		unsigned char aux[presz]; \
 		for(i = 0; i < presz; i++) {\
			if(*(((unsigned char*)FUNC)+i) != PRELUDE[i]){\
				break;\
			} \
		} \
		if(i != presz){ \
			for(i = 0; i < presz; i++){ \
				aux[i] = *(((unsigned char*)FUNC)+i); \
				*(((unsigned char*)FUNC)+i) = PRELUDE[i]; \
			} \
			RET = FUNC(__VA_ARGS__); \
			for(i = 0; i < presz; i++){ \
				*(((unsigned char *)FUNC)+i) = aux[i]; \
			} \
		}else{ \
			RET = FUNC(__VA_ARGS__); \
		} \
	}
```

This macro:

1. Checks if the prelude is the original one
2. If not, it copies the original one, calls the function and restores the
Frida prelude
3. Else, it calls the function normally (Frida is not around)

In general, this is  dirty code, but in this case it's just what we need
to do the job.
Alternatively, it could be implemented as an inline function.

A malicious binary could use the macro to hide the important calls (i.e. evidences of malicious activity).

## Example

The following program dumps a file (the path is passed as an argument).
If the ```-s``` flag is used, it dumps the file silently:
calls to the libc functions ```open```, ```read```, ```write``` and ```close```
are hidden (i.e. they will not be intercepted).
These functions will perform the required syscalls to read and write the file.
Function ```silentreadall``` implements that.

If the flag is not used, those libc functions are called
normally (```readall``` function).

The original preludes for ```open```, ```read```, ```write``` and ```close```
are stored in global variables:

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <err.h>
#include <unistd.h>
#include <fcntl.h>

enum{
	Bsz = 1024,
};

#define HIDECALL(RET,FUNC,PRELUDE,...) \
	{	int i; \
		int presz = sizeof(PRELUDE); \
		unsigned char aux[presz]; \
 		for(i = 0; i < presz; i++) {\
			if(*(((unsigned char*)FUNC)+i) != PRELUDE[i]){\
				break;\
			} \
		} \
		if(i != presz){ \
			for(i=0; i<presz; i++){ \
				aux[i] = *(((unsigned char*)FUNC)+i); \
				*(((unsigned char*)FUNC)+i) = PRELUDE[i]; \
			} \
			RET = FUNC(__VA_ARGS__); \
			for(i=0; i<presz; i++){ \
				*(((unsigned char *)FUNC)+i) = aux[i]; \
			} \
		}else{ \
			RET = FUNC(__VA_ARGS__); \
		} \
	}

unsigned char openprelude[] = {
				0xf3, 0x0f, 0x1e, 0xfa, 0x41, 0x54,
				0x41, 0x89,
				};

unsigned char readprelude[] = {
				0xf3, 0x0f, 0x1e, 0xfa, 0x64, 0x8b,
				0x04, 0x25, 0x18, 0x00, 0x00, 0x00,
				};

unsigned char writeprelude[] = {
				0xf3, 0x0f, 0x1e, 0xfa, 0x64, 0x8b,
				0x04, 0x25, 0x18, 0x00, 0x00, 0x00,
				};

unsigned char closeprelude[] = {
				0xf3, 0x0f, 0x1e, 0xfa, 0x64, 0x8b,
				0x04, 0x25, 0x18, 0x00, 0x00, 0x00,
				};

void
usage(void)
{
	fprintf(stderr, "usage: readfile [-s] path\n");
	exit(EXIT_FAILURE);
}

void
readall(char *path)
{
	int fd;
	int nr;
	char buf[Bsz];

	fd = open(path, O_RDONLY);
	if(fd < 0){
		err(EXIT_FAILURE, "open failed");
	}
	while((nr = read(fd, buf, Bsz)) > 0){
		if(write(1, buf, nr) != nr){
			err(EXIT_FAILURE, "write failed");
		}
	}
	if(nr < 0){
		err(EXIT_FAILURE, "read failed");
	}
	close(fd);
}

void
silentreadall(char *path)
{
	int fd;
	int nr;
	char buf[Bsz];
	int ret;

	HIDECALL(ret, open, openprelude, path, O_RDONLY);
	fd = ret;
	if(fd < 0){
		err(EXIT_FAILURE, "open failed");
	}
	for(;;){
		HIDECALL(ret, read, readprelude, fd, buf, Bsz);
		nr = ret;
		if(nr <= 0)
			break;
		HIDECALL(ret, write, writeprelude, 1, buf, nr);
		if(ret != nr){
			err(EXIT_FAILURE, "write failed");
		}
	};
	if(nr < 0){
		err(EXIT_FAILURE, "read failed");
	}
	HIDECALL(ret, close, closeprelude, fd);
}

int
main(int argc, char *argv[])
{
	char c;

	fprintf(stderr, "press enter...\n");
	read(0, &c, 1);

	argc--;
	argv++;
	if(argc < 1 || argc > 2){
		usage();
	}
	if(argc == 2){
		if(strcmp(argv[0], "-s") != 0){
			usage();
		}
		silentreadall(argv[1]);
	}else{
		readall(argv[0]);
	}
	exit(EXIT_SUCCESS);
 }
```

If this program is executed:

```
$> seq -w 1 1000 > /tmp/f
$> ./readfile-simple /tmp/f
press enter...
0001
0002
...
0996
0997
0998
0999
1000
$>
```

If we use the ```-s``` flag:

```
$> ./readfile-simple -s /tmp/f
press enter...
0001
0002
...
0996
0997
0998
0999
1000
$>
```
Ok, it works.
Now, we run it *without* the ```-s``` flag and execute
```frida-trace``` before pressing enter:

```
$> frida-trace -i open -i read -i write -i close readfile-simple
Instrumenting...                                                        
open: Loaded handler at "/tmp/__handlers__/libc_2.31.so/open.js"
read: Loaded handler at "/tmp/__handlers__/libc_2.31.so/read.js"
write: Loaded handler at "/tmp/__handlers__/libc_2.31.so/write.js"
close: Loaded handler at "/tmp/__handlers__/libc_2.31.so/close.js"
open: Loaded handler at "/tmp/__handlers__/libpthread_2.31.so/open.js"
read: Loaded handler at "/tmp/__handlers__/libpthread_2.31.so/read.js"
write: Loaded handler at "/tmp/__handlers__/libpthread_2.31.so/write.js"
close: Loaded handler at "/tmp/__handlers__/libpthread_2.31.so/close.js"
Started tracing 8 functions. Press Ctrl+C to stop.                      
           /* TID 0x4445 */
  1986 ms  open(pathname="/tmp/f", flags=0x0)
  1986 ms  read(fd=0x4, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  write(fd=0x1, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  read(fd=0x4, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  write(fd=0x1, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  read(fd=0x4, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  write(fd=0x1, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  read(fd=0x4, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  write(fd=0x1, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  read(fd=0x4, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  write(fd=0x1, buf=0x7ffcb03292f0, count=0x388)
  1986 ms  read(fd=0x4, buf=0x7ffcb03292f0, count=0x400)
  1986 ms  close(fd=0x4)
Process terminated
$>
```

```frida-trace``` is able to intercept them.

If we do the same with the ```-s``` flag:

```
$> frida-trace -i open -i read -i write -i close readfile-simple
Instrumenting...                                                        
open: Loaded handler at "/tmp/__handlers__/libc_2.31.so/open.js"
read: Loaded handler at "/tmp/__handlers__/libc_2.31.so/read.js"
write: Loaded handler at "/tmp/__handlers__/libc_2.31.so/write.js"
close: Loaded handler at "/tmp/__handlers__/libc_2.31.so/close.js"
open: Loaded handler at "/tmp/__handlers__/libpthread_2.31.so/open.js"
read: Loaded handler at "/tmp/__handlers__/libpthread_2.31.so/read.js"
write: Loaded handler at "/tmp/__handlers__/libpthread_2.31.so/write.js"
close: Loaded handler at "/tmp/__handlers__/libpthread_2.31.so/close.js"
Started tracing 8 functions. Press Ctrl+C to stop.                      
Process terminated
$>
```

**The hidden calls have not been intercepted** and the program
works fine:

```
$> ./readfile-simple -s /tmp/f
press enter...
0001
0002
...
0996
0997
0998
0999
1000
$>
```

## Conclusions

We can easily  detect and bypass the ```frida-trace``` interception mechanism
without depending on any library function or syscall. Moreover, the code
generated by the macro is more complex, so it's harder to
analyze the disassembly.

**I'm probably not the first one to notice this issue**. Anyway, I haven't read anything related (but I didn't do a proper research before writing this post).
I understand that Frida developers are aware of it.

We should study the Frida code (which is not trivial) before searching specific solutions. A possible solution may be disabling the write permission of code pages  after the instrumentation is done and monitor ```mprotect``` syscalls. Another approach could be detecting changes in the modified preludes (maybe by using  [umwait](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=UMWAIT&expand=5980) and [umonitor](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=Umo&expand=5980,5979,7252) ? I used the kernel space vesions time ago and they were tricky; I don't know the userspace versions).

**UPDATE:** Frida devs are aware of this post and commented: (about restoring the original page permissions) *"it's probably something we should do as part of enhancing Interceptor so it's able to suspend other threads to make it safe to patch functions that might have calls in progress."*

**UPDATE 2:** I'm testing new methods to detect modifications of the preludes, ```umwait``` and ```umonitor``` are not suitable (these instructions need to change the state of the core; that would be an overkill). Now, I'm playing with the ```soft dirty``` bit of the page table entries (PTE). Stay tuned.

### Comments

You can comment this post in [twitter](https://twitter.com/e__soriano/status/1425069650329604096)

<sub><sup>
    <b> Excluding linked files, only for the blog content:</b>
    <b>(cc) Enrique Soriano-Salvador</b>
    Creative Commons  (by-nc-nd).
    Creative Commons, 559 Nathan Abbott Way, Stanford,
    California 94305, USA.
    <br>
</sup></sub>
