---
title:      Gopper, keep a close eye on this proc!
summary:    Gopper
categories: blog
date:       Thu 27 Jan 2022 07:35:51 PM CET
thumbnail:  gopper
image:      https://github.com/sysfatal/figs/gopher-copper.jpg
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
  <img src="figs/gopher-copper.jpg" alt="{{gopher copper image}}">
</figure>
</center>

## Gopper

I have written a mini tool named Gopper (*gopher copper* :)) in Go.
It implements the procedure explained above.
In addition, it also detects the following suspicious actions:

- Modifications of the permissions of the pages. It also
warns about dangerous permissions (i.e. write+exec). To do that, it polls
/proc/pid/maps.

- Calls to the mprotect syscall, used to change page permissions. This is
done by receiving events from the Linux kernel tracepoints through the
synthetic files located in /sys/kernel/debug/tracing.
See [events.txt](https://www.kernel.org/doc/Documentation/trace/events.txt)
for more info.

- Calls to other syscalls defined by the user.

Gopper can be used together with Frida-trace or any other
analysis tool. It does not interfere with the watched process.

Gopper git:

[https://gitlab.etsit.urjc.es/esoriano/gopper](https://gitlab.etsit.urjc.es/esoriano/gopper)


## Comments

You can comment this post in [twitter](https://twitter.com/e__soriano/status/1486817363802607616)



<sub><sup>
    <b>(cc) Enrique Soriano-Salvador</b>
    Algunos derechos reservados. Este trabajo se entrega bajo la licencia
    Creative Commons Reconocimiento - NoComercial - SinObraDerivada (by-nc-nd).
    Creative Commons, 559 Nathan Abbott Way, Stanford,
    California 94305, USA.
</sup></sub>
