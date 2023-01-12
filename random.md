---
title:      El /dev/random de Linux es muy random
summary:    /dev/random y /dev/urandom (otra vez)
categories: blog
date:       Mon 28 Dec 2020 12:41:02 PM CET
thumbnail:  random
image:      https://github.com/sysfatal/figs/dices.png
layout:     post
thumbnail:  random
author:     e__soriano
tags:
 - random
 - csprng
---
<div class="share-page">
    Share this on &rarr;
    [<a title="Share on Mastodon" href="https://tootpick.org/#text=Check%20out%20https://sysfatal.github.io{{  page.url }}%20by%20@esoriano@social.linux.pizza">Mastodon</a>]
   [<a href="https://twitter.com/intent/tweet?text={{ page.title }}&url={{ site.url }}{{ page.url }}&via=e__soriano&related=e__soriano" rel="nofollow" target="_blank" title="Share on Twitter">Twitter</a>]
    [<a href="https://facebook.com/sharer.php?u={{ site.url }}{{ page.url }}" rel="nofollow" target="_blank" title="Share on Facebook">Facebook</a>]
</div>
<br>

---

<center>
<figure class="image">
  <img src="figs/dices.png">
 </figure>
</center>
<br>

La generación de claves, *nonces* y otros datos importantes para
asegurar las comunicaciones requiere conseguir datos lo más aleatorios
que se pueda. En esos casos, generar datos previsibles es un error fatal.
Por ejemplo, una versión antigua de Kerberos tenía una vulnerabilidad en
la generación de la clave de sesión: derivaba la clave de datos que eran
totalmente predecibles, como el UID, el identificador de la máquina
y la hora (en segundos) [1].

En general, conseguir números aleatorios en
un ordenador no es sencillo: *aleatorio*
significa que los números se eligen siguiendo
una distribución de probabilidad uniforme y no predecible.
Un ordenador siempre esta
en uno de sus estados (que son finitos), es una máquina determinista.

Para conseguir datos *que parezcan aleatorios*, podemos usar
un generador de números pseudo-aleatorios
(PRNG). Estos generadores crean una secuencia a partir de un valor
inicial, llamado *semilla* (*seed*).
Dicha secuencia *parece aleatoria*: no puede reproducir ningún
patrón.

Se puede usar un test de aleatoriedad estadístico para comprobar
la salida de un generador, por ejemplo:

```
$> rngtest -c 100 < /dev/urandom
rngtest 5
Copyright (c) 2004 by Henrique de Moraes Holschuh
This is free software; see the source for copying conditions.  There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

rngtest: starting FIPS tests...
rngtest: bits received from input: 2000032
rngtest: FIPS 140-2 successes: 100
rngtest: FIPS 140-2 failures: 0
rngtest: FIPS 140-2(2001-10-10) Monobit: 0
rngtest: FIPS 140-2(2001-10-10) Poker: 0
rngtest: FIPS 140-2(2001-10-10) Runs: 0
rngtest: FIPS 140-2(2001-10-10) Long run: 0
rngtest: FIPS 140-2(2001-10-10) Continuous run: 0
rngtest: input channel speed: (min=183.399; avg=447.629; max=1003.868)Mibits/s
rngtest: FIPS tests speed: (min=34.616; avg=55.527; max=120.718)Mibits/s
rngtest: Program run time: 38796 microseconds
$>
```

Un generador de números
pseudo-aleatorios tiene que cumplir ese requisito.
El problema es que siempre genera la misma secuencia para la misma semilla.

Por ejemplo, este script de AWK imprime 10 floats pseudoaleatorios entre 0 y 1:

```
$> awk 'BEGIN{srand(12);for(i=0;i<10;i++){print rand()};exit}'
0.420212
0.272851
0.798482
0.214592
0.527246
0.72186
0.0985599
0.939076
0.991433
0.460516
$> awk 'BEGIN{srand(12);for(i=0;i<10;i++){print rand()};exit}'
0.420212
0.272851
0.798482
0.214592
0.527246
0.72186
0.0985599
0.939076
0.991433
0.460516
$>
```

Los datos parecen aleatorios, pero ¡son
siempre los mismos!
Si inicializamos el generador con otra semilla distinta a 12, la secuencia cambia:

```
$ awk 'BEGIN{srand(13);for(i=0;i<10;i++){print rand()};exit}'
0.378928
0.548198
0.264116
0.496156
0.907806
0.0257171
0.446959
0.853342
0.0397394
0.74097
$>
```

Si tenemos que usar un generador como este,
debemos reiniciar el generador constantemente
con nuevas semillas (con la mayor entropía que podamos
conseguir). Hay que evitar que se pueda adivinar
o alterar el estado actual del generador.

Hay que tener cuidado con los generadores
por omisión en los lenguajes/bibliotecas que estamos
usando. **Muchas veces no son
criptográficamente seguros**.

Un *generador de números pseudo-aleatorios criptográficamente seguro*
(CSPRNG) crea una secuencia no predecible ni reproducible de forma
fiable.
Es computacionalmente inviable obtener ninguna información
sobre la semilla observando la salida del CSPRNG.

## /dev/random y /dev/urandom

En Linux hay dos ficheros sintéticos de los que podemos
leer datos datos pseudoaleatorios, que crean bastante
discordia [2]: */dev/random* y */dev/urandom*.

Al principio sólo existía */dev/random*, luego se metió */dev/urandom*
para evitar bloqueos cuando el sistema se quedaba sin entropía.

La existencia de dos ficheros distintos ha sido un problema durante mucho tiempo. Muchos seguían
usando */dev/random* sin entender su comportamiento bloqueante ni los datos que ofrecía.
Otros confundían los dos ficheros o pensaban
que eran iguales, etc. También
existía el mito de que *urandom*
era un PRNG y *random* era un CSPRNG (y otros, ver [3]).
La recomendación usual era: **usa urandom**.

La implementación de estos ficheros ha cambiado varias veces.
Recientemente han sufrido cambios severos.

Con tantos cambios y *creencias* sobre estos dos ficheros,
no es fácil encontrar información clara sobre cómo está organizado
el sistema.

En Linux 5.6, el esquema es el siguiente [4].
El sistema recolecta aleatoriedad de distintas fuentes,
como por ejemplo los bits de menos peso en el
tiempo en nanosegundos en el que ocurre una interrupción hardware (p. ej. IRQ).
En las CPUs modernas hay hardware para conseguir
entropía (p. ej. con la instrucción *rdrand*), aunque
Linux no lo usa. Los desarrolladores de Linux no se fían.
Hay que ser cuidadoso a la hora de extraer esos datos:
en las máquinas virtuales el hardware está virtualizado,
no se puede acceder a ciertos registros,
hay documentados bugs en estas
instrucciones, etc.

A partir de esa aleatoriedad (que pasa un filtrado), se mantiene
un *pool de entropía* de 4096 bits.
De ese *pool* salen las semillas
para un CSPRNG basado en el algoritmo de cifrado de stream **ChaCha20**.
Ese CSPRNG es el que proporciona los datos de los dos ficheros.
La semilla se renueva (*reseed*) periódicamente (cada 5 minutos).

En Linux 5.6, hay poca diferencia entre los dos ficheros (ojo, antes
**no era así**):

- ***/dev/random***

	- En versiones anteriores a Linux 5.6, se bloqueaba
	cuando *se estimaba* que
	no había suficiente entropía en el *pool*.
	Esto podía ocurrir en momentos poco adecuados
	y podía provocar parones largos.

	- En Linux 5.6 sólo se bloquea cuando el CSPRNG no
	está inicializado correctamente
	(en el arranque). Esto es importante
	para los primeros procesos (p. ej. *init*).
	Una vez que está inicializado,
	ya no se bloquea [5] aunque se
	estime que hay poca entropía en el *pool*.
	En ese caso, no se cambia la semilla del CSPRNG.

- ***/dev/urandom***

	- Nunca se bloquea, aunque el CSPRNG no esté
	correctamente
	inicializado.
	Hay que tener cuidado con esto (ver la página de manual
	*random(4)*, aunque
	esta página de manual es conocida por la confusión que
	genera y a día de hoy no está actualizada con los últimos
	cambios).

Las escrituras en estos ficheros también alimentan el *pool*,
*mezclándose* con sus datos.
Por ejemplo, podemos escribir
datos extraídos de un dispositivo
USB que ofrezca entropía basada
en el *efecto  avalancha*
en un semiconductor (*avalanche breakdown*).
Hay que tener en cuenta que el cambio de semilla (*reseed*)
se hace cada 5 minutos. Por tanto, pasará un tiempo hasta que se renueve
la semilla del CSPRNG.

En */proc/sys/kernel/random* tenemos ficheros sintéticos
con información sobre */dev/random*, entre ellos:

- ***entropy_avail***: te dice la cantidad de bits de
entropía del pool (de 0 a 4096).
Grosso modo, tener N bits de entropía significa que los datos
del *pool* son unos datos escogidos entre
2 elevado a N posibilidades distintas.

- ***poolsize***: te dice el tamaño máximo del *pool* en bits (4096).

- ***write_wakeup_threshold***: contiene el número
de bits de entropía al que hay que llegar
para permitir escribir en el pool (por si hay procesos
que lo alimentan).

- ***uuid*** y ***boot_id***: dan identificadores
aleatorios, el primero se actualiza cada vez que se lee, el
segundo se genera una vez.

## ¿Cómo de lento es leer de */dev/urandom*?

Bueno, esta prueba rápida y
sucia (habría que hacer un estudio más serio para verlo de verdad)
indica que no es mucho más lento que leer ceros de */dev/zero*:

```
$> dd if=/dev/zero of=/tmp/z bs=512 count=$((1024*100)) oflag=sync,direct
102400+0 records in
102400+0 records out
52428800 bytes (52 MB, 50 MiB) copied, 226.873 s, 231 kB/s
$> dd if=/dev/urandom of=/tmp/r bs=512 count=$((1024*100)) oflag=sync,direct
102400+0 records in
102400+0 records out
52428800 bytes (52 MB, 50 MiB) copied, 238.146 s, 220 kB/s
```
La penalización para copiar 50 MiB de datos a un fichero en un disco
SSD, saltando la cache, es de aprox. 4.9%.

También hay una llamada al sistema para conseguir los
datos, *getrandom(2)*, y una función de biblioteca, *getentropy(3)*.
Para más detalles sobre la implementación, se recomienda acudir a la
referencia [4].

Así están las cosas ahora mismo, pero pueden cambiar en unos meses...
con */dev/random* en Linux ***nunca se sabe***, es muy *random* ;)

### Referencias
[1]  W. Venema, “Murphy’s law and computer security,” in Proceedings of the 6th
conference on USENIX Security Symposium, Focusing on Applications of Crypto-
graphy - Volume 6, pp. 19–19, USENIX Association, 1996.

[2] T. Pornin, “On linux’s random number generation.”
[link](https://research.nccgroup.com/2019/12/19/on-linuxs-random-number-generation/)

[3] T. Hühn, “Myths about /dev/urandom.”. [link](https://www.2uo.de/myths-about-urandom/)

[4] S. Müller, “Documentation and analysis of the linux random number generator,
prepared for germany federal office of information security (BSI) by atsec
information security gmbh”, 2020.
[link](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/Studies/LinuxRNG/LinuxRNG_EN_V4_3.html).

[5] J. Edge, “Removing the linux /dev/random blocking pool.”
[link](https://lwn.net/Articles/808575)

<sub><sup>
    <b>(cc) Enrique Soriano-Salvador</b>
    Algunos derechos reservados. Este trabajo se entrega bajo la licencia
    Creative Commons Reconocimiento - NoComercial - SinObraDerivada (by-nc-nd).
    Creative Commons, 559 Nathan Abbott Way, Stanford,
    California 94305, USA.
</sup></sub>
