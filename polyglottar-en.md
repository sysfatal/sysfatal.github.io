---
title:      Polyglottar, an ISO+TAR+ELF polyglot
summary:    Polylgot for two structured formats
categories: blog
date:       Sun Apr 12 17:58:47 CEST 2020
thumbnail:  polyglottar
image:      https://github.com/sysfatal/figs/elf-in-tar.png
layout:     post
author:     e__soriano
tags:
 - malware
 - stego
 - unix
---

<div class="share-page">
    Share this on &rarr;
    [<a title="Share on Mastodon" href="https://tootpick.org/#text=Check%20out%20https://sysfatal.github.io{{  page.url }}%20by%20@esoriano@social.linux.pizza">Mastodon</a>]
    [<a href="https://twitter.com/intent/tweet?text={{ page.title }}&url={{ site.url }}{{ page.url }}&via=e__soriano&related=e__soriano" rel="nofollow" target="_blank" title="Share on Twitter">Twitter</a>]
    [<a href="https://facebook.com/sharer.php?u={{ site.url }}{{ page.url }}" rel="nofollow" target="_blank" title="Share on Facebook">Facebook</a>]
</div>
<br>

___

<center>
<figure class="image">
  <img src="figs/elf-in-tar.png">
</figure>
</center>


# UPDATE: Paper!!!!

This post has been converted into an [article](https://tmpout.sh/4/16.html) in tmp.0ut Volume 4, march 2025: [https://tmpout.sh/4/](https://tmpout.sh/4/).


<x class="a070">    00000000: 7f45 4c46 0201 0100 0</x><x class="a220">┏━━━┓</x><x class="a070">000 0000 6e73  .ELF..........ns    </x>
<x class="a220">┏━━━┓</x><x class="a255">██████████</x><x class="a070">3</x><x class="a255">████</x><x class="a070">70</x><x class="a255">████</x><x class="a070">0</x><x class="a255">██████████ ██████████ ██</x><x class="a070">00  ..</x><x class="a255">██</x><x class="a070">.</x><x class="a255">██████████</x><x class="a220">┏━━━┓</x>
<x class="a220">┃   ┃</x><x class="a255">██</x><x class="a220">━━</x><x class="a255">██</x><x class="a070">0:</x><x class="a255">██</x><x class="a070">0</x><x class="a255">██ </x><x class="a070">0</x><x class="a255">██</x><x class="a070">0</x><x class="a255"> ██</x><x class="a220">━</x><x class="a255">██</x><x class="a220">┓┃   ┃</x><x class="a255">██</x><x class="a220">━</x><x class="a255">██</x><x class="a220">┃   ┃┏</x><x class="a255">██</x><x class="a220">━</x><x class="a255">██</x><x class="a070">00  @.</x><x class="a255">██</x><x class="a070">.</x><x class="a255">██</x><x class="a070">..</x><x class="a255">██</x><x class="a220">━━</x><x class="a255">██</x><x class="a220">┃   ┃</x>
<x class="a220">┗━━━┛┃   </x><x class="a255">██</x><x class="a220">━━━┓</x><x class="a070">0</x><x class="a255">██ </x><x class="a070">0</x><x class="a220">┏━━━</x><x class="a255">██ ██</x><x class="a220">┃┗━━━┛</x><x class="a255">██</x><x class="a070">0</x><x class="a255">██</x><x class="a220">┗━</x><x class="a255">██</x><x class="a220">┛┃</x><x class="a255">██ ██</x><x class="a220">━━━┓</x><x class="a070">..</x><x class="a255">██</x><x class="a070">@</x><x class="a220">┏━━━</x><x class="a255">██</x><x class="a220">   ┃┗━━━┛</x>
<x class="a255">  ██</x><x class="a070">0</x><x class="a255">██</x><x class="a220">━━</x><x class="a255">██</x><x class="a220">   ┃┏</x><x class="a255">██</x><x class="a220">━┓┃   </x><x class="a255">██</x><x class="a220">━</x><x class="a255">██████████</x><x class="a070">0</x><x class="a255">██</x><x class="a070">000 0</x><x class="a220">┗</x><x class="a255">██</x><x class="a220">━</x><x class="a255">██   </x><x class="a220">┃┏━</x><x class="a255">██</x><x class="a220">┓┃   </x><x class="a255">██</x><x class="a220">━━</x><x class="a255">██</x><x class="a070">.</x><x class="a255">██  </x>
<x class="a255">  ██</x><x class="a070">0</x><x class="a255">██</x><x class="a070">00</x><x class="a255">██</x><x class="a220">━━━┛┃</x><x class="a255">██ </x><x class="a220">┃┗━━━</x><x class="a255">██</x><x class="a070">0</x><x class="a255">██</x><x class="a070">0000 4000</x><x class="a255">██████████ ██████████</x><x class="a220">┃┗━━━</x><x class="a255">██</x><x class="a070">..</x><x class="a255">██</x><x class="a070">.</x><x class="a255">██  </x>
<x class="a242">▄▄▄▄</x><x class="a070">00000060: f</x><x class="a220">┗━━━┛</x><x class="a070">000 0000 0000 746d 702e 3075 7434  </x><x class="a220">┗━━━┛</x><x class="a070">...........</x><x class="a242">▄▄▄▄</x>
<x class="a244">▄▄▄▄</x><x class="a148"> ━┓     ┏━┏━━━━━━━┓┏        ┏       ┓┏━━━━━━━┓┏━━━━┓     ┏       ┓ </x><x class="a244">▄▄▄▄</x>


