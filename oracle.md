##  Qué es un ataque Padding Oracle y cómo funciona

Comencemos con una demostración. El siguiente video muestra
cómo un programa ejecutando en mi portátil es capaz de descifrar
un mensaje cifrado con AES-256 en modo CBC y padding PKCS7.
Tanto el cliente como el servidor son _correctos_: no tienen
bug explotables, usan correctamente las herramientas criptográficas,
etc. El tiempo para romper por fuerza bruta una clave AES-256
en un supercomputador actual es de aproximadamente *10^52* años.
Con  1000000000 GPUs de 2 Gigaflops, *60^25* años.
Además, necesitarías 150 Gigavatios para alimentarlas, esto es,
150 reactores nucleares.

Ahora mira:

// screencast


Ok. El único fallo que tiene el servidor es el siguiente: hace
visible al cliente cuando hay un error porque el padding de un
mensaje es incorrecto y cuando el error es por otro motivo.
Simplemente eso.

Un ataque de oráculo es un ataque de _side-channel_.
En general, un ataque de oráculo es cuando el protocolo o sistema
tiene un canal lateral que nos deja saber si ocurre o no una
cosa, dejando al adversario saber si está cerca o no de conseguir
un objetivo.

En este caso, el _side-channel_ consiste en saber si un mensaje
cifrado que envía el atacante tiene bien formado el padding o no.

Esto lo puede saber si el servidor devuelve un error
que permite diferenciar si hay un error de padding o si el error
es de otro tipo. El atacante puede sacar partido de esto para
descifrar cualquier mensaje sin tener la clave AES de 256 bits.
Esto se hace explotando un efecto del modo de operación CBC, que
vemos a continuación.

### CBC

Cuando queremos usar un cifrador de bloques como AES para cifrar
una mensaje más largo que un bloque (128 bits en el caso de AES),
necesitamos usar un **modo de operación**. Hasta hace poco, el modo
de operación estándar era CBC. En la actualidad se recomiendan
otros modos, pero CBC sigue en uso y no se considera inseguro (siempre
que se tomen precauciones).

Para cifrar un mensaje largo, se parte en
trozos del tamaño de bloque del algoritmo (128 bits para este ejemplo).
Cada trozo se cifra con AES usando la clave indicada. El modo indica cómo
se cifran los distintos trozos.

Si se cifran tal cual (modo llamado ECB), los bloques del mensaje cifrado
pueden terminar con patrones derivados del mensaje en claro. Piensa en
esto: si estás cifrando un pixmap y tienes la mala suerte de que cada
pixel ocupa un bloque completo, el pixmap cifrado será otro pixmap en el
que unos colores se han reemplazado por otros. Seguramente puedas reconocer
la imagen cifrada.

El modo CBC se pensó para evitar ese efecto y ocultar patrones del mensaje
en claro. Este modo consiste en hacer una operación XOR bit a bit entre
el bloque N del mensaje en claro y el bloque N-1 del mensaje cifrado,
antes de pasarlo por el cifrador AES.
De esta forma, se están _encadenando_
los bloques. Al resultado de ese XOR (esto es,
el bloque antes de pasar por el cifrador) se le llama **estado intermedio**.
Esto es importante, aparecerá más tarde.

Por tanto, en CBC un bloque _i_ se cifra así (siendo K la clave):


C<sub>i</sub> = AES<sub>K</sub>(Mi ⊕ C<sub>i-1</sub>)


Para el primer bloque del mensaje, que no tiene
anterior, se usa el _vector de inicialización_, que se debe proporcionar
junto con la clave a la hora de cifrar/descifrar el mensaje con AES en
modo CBC (y debe ser aleatorio, no predecible y no se debe reusar).

Un problema grave del modo CBC es que es **maleable**. Esto significa que
el atacante puede modificar el mensaje cifrado para provocar cambios en
el mensaje descifrado.

En CBC, el cambio del bit _i_ del bloque _N_ del mensaje cifrado supone
dos cambios en el mensaje descifrado:

1. El bloque _N_ del mensaje descifrado cambiará completamente (no tendrá
	ningún sentido).
2. Se provoca un _flip_ del bit _i_ del bloque _N+1_ del mensaje descifrado:
ese bit habrá cambiado (será 0 si antes era 1, y viceversa).

Mucha gente piensa que una modificación en un mensaje cifrado en modo
CBC destroza el mensaje descifrado desde el bloque modificado hasta
el final del mensaje. **No es así**. El bloque _N+2_, y los siguientes,
no se ven afectados por esa modificación del mensaje cifrado.

### Padding

Cuando tenemos un mensaje que no es múltiplo del tamaño del bloque,
el último bloque del mensaje se tiene que _rellenar_, ya que al
cifrador hay que darle bloques completos.

Hay varias formas
de hacer esto. Para este ejemplo, estamos usando el estándar de
padding PKCS7 que consiste en rellenar con todos los bytes puestos al
número de bytes del padding:

 ```
      01
      02 02
      03 03 03
      04 04 04 04
      05 05 05 05 05
      etc.
```

Por ejemplo, si en el último bloque tienes que rellenar con 7 bytes,
se pondrían con un valor _0x07_. Si el mensaje es múltiplo del tamaño
de bloque, se debe meter un bloque completo de padding.

### Ataque

Para realizar el ataque, el adversario necesita:

1. Capturar el tráfico cifrado legítimo de un cliente.

2. Poder enviar peticiones espurias al servidor.

3. Diferenciar entre los fallos provocados por un padding incorrecto del
	resto de fallos. **Esta es la fuga de información necesaria
	para realizar este ataque de side-channel.**


#### Ataque I: conocer el tamaño del mensaje cifrado

El primer ataque, más sencillo, consiste en descubrir el tamaño
del mensaje cifrado (esto es, saber cuánto padding se ha usado).
Esto es algo que un atacante no debería poder descubrir.

Para ello:

1. Se envían versiones modificadas del mensaje cifrado original, _C_.

2. Por cada intento, se modifica un único byte del *penúltimo*
	bloque de _C_.

3. Se empieza modificando el último byte de ese bloque,
	y se va iterando hacia atrás.

4. Paramos cuando el servidor *no* de un error de padding (dará
	otro tipo de error).

5. Si eso pasa en N-ésimo byte del penúltimo bloque, ya sabemos que el 	
	mensaje original tiene una longitud en bytes de:

<center>
	_LEN(M) = BLOCKSIZE * (NBLOCKS - 1) + N_
</center>

#### Ataque II: descifrar un mensaje interceptado previamente

Ahora el objetivo es *descifrar un mensaje del cliente*, que
mucho más interesante.

Llamemos _C[i]_ al bloque del mensaje cifrado _C_  
que queremos descifrar.

Idea:

1. Usaremos _C[i]_ como *último* bloque de un mensaje espurio
que llamaremos _fake_.

2. Modificaremos el penúltimo bloque del mensaje
	espurio, _fake[j]_, para descubrir
	cuándo el servidor *no* retorna un error de padding.

3. Así descubriremos el **valor intermedio** de cada byte de _C[i]_.

4. Con el valor intermedio, reconstruimos cada byte de _M[i]_.

5. Esto lo haremos por cada bloque de _C_ hasta conseguir el
	mensaje en claro _M_.

Por ejemplo, para descifrar un único byte:

```
val := 0
found := false
while val <=  255 and not found
    fake[j][last] := val
    sendFakeMessage()
    if not paddingError then
         found := true
    else
         val := val + 1
    end if
end while
```

Obtenemos el valor intermedio correspondiente a dicho byte:

```
 $I := val ⊕ 0x01$
```

\item Obtenemos el valor del byte en el texto claro, $M[i][last]$,
\textbf{usando el bloque cifrado anterior del mensaje original},
$C[i-1][last]$:

~\\
 $M[i][last] := C[i-1][last] \oplus I $
~\\


<sub><sup>
    <b>(cc) Enrique Soriano-Salvador</b>
    Algunos derechos reservados. Este trabajo se entrega bajo la licencia
    Creative Commons Reconocimiento - NoComercial - SinObraDerivada (by-nc-nd).
    Creative Commons, 559 Nathan Abbott Way, Stanford,
    California 94305, USA.
</sup></sub>
