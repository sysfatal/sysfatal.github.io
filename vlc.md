---
title:      VLC as root and the magic sed
summary:    elf, binary
categories: blog
date:       Wed Jun 24 05:17:00 PM CEST 2026
thumbnail:  vlc
image:      /figs/vlc.png
layout:     post
thumbnail:  vlc
author:     e__soriano
tags:
 - tty
 - terminals
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

Well, it works! But, how?

We are dealing with a typical case that surprises neither a novice (someone who does not know what an executable binary is, who has never heard of ELF or thinks that a binary is like a python program) nor an expert in systems and binary analysis, but rather those who fall somewhere in between these two extremes.

This document describes why this seemingly unusual solution works.
