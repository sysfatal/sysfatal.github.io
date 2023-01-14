---
title:      ChatGPT aprueba el examen
summary:    ChatGPT aprueba el examen
categories: blog
date:       Fri Jan 13 06:55:33 PM CET 2023
thumbnail:  ChatGPT
image:      https://github.com/sysfatal/figs/asimov.png
layout:     post
author:     e__soriano
tags:
 - ChatGPT
 - linux
 - programación
 - IA
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
  <img src="figs/asimov.png">
</figure>
</center>

<br>

# ChatGPT aprueba el examen

<br>

Muchos llevamos unas semanas revolucionados con ChatGPT,
la IA especializada en generación de diálogo
de [OpenAI](https://openai.com/). Supongo que a estas alturas
no hace falta explicar [qué es](https://es.wikipedia.org/wiki/ChatGPT).
A mí me gusta decir que es una *charlatana excelente*:
habla mucho pero no entiende nada, sólo dice lo que ha escuchado por ahí :)

Este post describe un experimento:
he dado a ChatGPT el enunciado del examen de prácticas
que pusimos esta semana en la asignatura *Sistemas Operativos* (tercer curso
de los grados Ing. Telemática / Ing. Tecnologías de Teleco.).

Lo ha hecho bien.
ChatGPT aprueba el examen, como el 30% de los alumnos que se han presentado
al examen (el 20% de los matriculados en la asignatura).

### Enunciado

El enunciado es muy sencillo:

```
Escribe un programa llamado concattxt en C para Linux que admita un
único argumento, que será una ruta.

El programa debe escribir el contenido de todos los ficheros del
directorio de trabajo actual cuyo nombre termine en .txt en el fichero
indicado por el argumento. No importa el orden en el que se recorra los
ficheros correspondientes (como estén en el directorio). Si el fichero
ya existe, se debe truncar. En otro caso, se debe crear.

El programa debe avisar y fallar si se ejecuta sin argumentos o con más
de un argumento.

Los ficheros pueden tener cualquier tamaño.

No se puede ejecutar ningún programa externo.

Por ejemplo:

$ echo xxx > uno.txt
$ echo yyy > dos.txt
$ echo zzz > tres.txt
$ ls *.txt
dos.txt  tres.txt  uno.txt
$ ./concattxt /tmp/salida
$ cat /tmp/salida
yyy
zzz
xxx
$
```

La prueba es presencial.
Los alumnos tienen una hora para hacer este programa en un sistema GNU Linux
Ubuntu 22.04 con las páginas de manual instaladas, sin material (apuntes, etc.)
ni conexión a internet.
Como se puede ver, no es una prueba muy difícil.

### Nuestra solución

Esta es la solución de los profesores de la asignatura:

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <err.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <dirent.h>

enum {
	BufSize = 32 * 1024,
};

int
dump(char *path, int fdout)
{
	int fdin;
	int nr;
	char buf[BufSize];

	fdin = open(path, O_RDONLY);
	if (fdin < 0) {
		warn("can't open %s", path);
		return -1;
	}
	while ((nr = read(fdin, buf, BufSize)) > 0) {
		if (write(fdout, buf, nr) != nr) {
			warn("can't write");
			return -1;
		}
	}
	if (nr < 0) {
		warn("can't read file %s", path);
		return -1;
	}
	close(fdin);
	return 0;
}

int
istxt(struct dirent *ent)
{
	char *p;

	p = strrchr(ent->d_name, '.');
	if (p == NULL) {
		return 0;
	}
	return strcmp(p, ".txt") == 0 && ent->d_type == DT_REG;
}

int
concattxt(int fdout)
{
	DIR *dir;
	struct dirent *ent;
	int sts;

	sts = 0;
	dir = opendir(".");
	if (dir == NULL) {
		warn("can't open .");
		return 1;
	}
	while ((ent = readdir(dir)) != NULL) {
		if (ent->d_name[0] != '.' && istxt(ent)) {
			if (dump(ent->d_name, fdout) < 0) {
				warnx("can't process file: %s", ent->d_name);
				sts++;
			}
		}
	}
	closedir(dir);
	return sts;
}

int
main(int argc, char *argv[])
{
	int fdout;
	int sts;

	if (argc != 2) {
		fprintf(stderr, "usage: concattxt file\n");
		exit(EXIT_FAILURE);
	}
	fdout = open(argv[1], O_WRONLY | O_CREAT | O_TRUNC, 0660);
	if (fdout < 0) {
		err(EXIT_FAILURE, "can't create file %s", argv[1]);
	}
	sts = concattxt(fdout);
	close(fdout);
	exit(sts);
}
```

### Primer intento

**yo:** Hola

**ChatGPT:** ¡Hola! ¿En qué puedo ayudarte hoy?

**yo:** Quiero hacer un programa en C para linux

**ChatGPT:** ¡Genial! Para comenzar a programar en C en Linux, necesitarás un compilador de C, como GCC (GNU Compiler Collection). La mayoría de las distribuciones de Linux vienen con GCC preinstalado, pero si no está instalado en tu sistema, puedes instalarlo usando el administrador de paquetes de tu distribución.
Una vez que tengas GCC instalado, puedes crear un archivo de código fuente con un editor de texto (como nano o vi) y guardarlo con la extensión ".c". Luego puedes compilar el archivo fuente utilizando el comando "gcc" seguido del nombre del archivo.  

**yo:** Escribe un programa llamado concattxt en C que admita un único argumento, que será una ruta. El programa debe escribir el contenido de todos los ficheros del directorio de trabajo actual cuyo nombre termine en .txt en el fichero indicado por el argumento. No importa el orden en el que se recorra los ficheros correspondientes (como estén en el directorio). Si el fichero ya existe, se debe truncar. En otro caso, se debe crear. El programa debe avisar y fallar si se ejecuta sin argumentos o con más de un argumento. Los ficheros pueden tener cualquier tamaño. No se puede ejecutar ningún programa externo.

**ChatGPT:** Aquí te dejo un ejemplo de código de un programa en C que cumple con los requisitos especificados:

```c
#include <stdio.h>
#include <stdlib.h>
#include <dirent.h>
#include <string.h>

int main(int argc, char* argv[]) {
    if (argc != 2) {
        printf("Uso: concattxt [archivo de salida]\n");
        return 1;
    }

    char* outputFile = argv[1];
    FILE* fp = fopen(outputFile, "w");
    if (!fp) {
        printf("No se pudo abrir el archivo de salida: %s\n", outputFile);
        return 1;
    }

    DIR* dir = opendir(".");
    if (!dir) {
        printf("No se pudo abrir el directorio actual\n");
        return 1;
    }

    struct dirent* ent;
    while ((ent = readdir(dir)) != NULL) {
        if (ent->d_type != DT_REG) {
            continue;
        }

        int len = strlen(ent->d_name);
        if (len < 4 || strcmp(ent->d_name + len - 4, ".txt") != 0) {
            continue;
        }

        FILE* file = fopen(ent->d_name, "r");
        if (!file) {
            printf("No se pudo abrir el archivo: %s\n", ent->d_name);
            continue;
        }

        char buffer[4096];
        size_t bytesRead;
        while ((bytesRead = fread(buffer, 1, sizeof(buffer), file)) > 0) {
            fwrite(buffer, 1, bytesRead, fp);
        }

        fclose(file);
    }

    closedir(dir);
    fclose(fp);

    return 0;
}
```
___

Ok. Este programa está bien. Hace lo que pide el enunciado y funciona.
Sin embargo, no es del todo correcto (ignoraremos todo lo que tiene que ver
con el estilo de código que utilizamos en la asignatura,
porque ChatGPT no lo puede saber):

- No escribe los mensajes de error/diagnóstico en la salida de errores, los escribe
en la salida estándar. Eso no está bien.

- En lugar de usar las llamadas al sistema (`open`, `read`, `write`, `close`) usa `stdio`.
En este caso, `stdio` no aporta nada, pero tampoco está mal usarla. En el examen,
se permitía implementar el programa de cualquiera de las dos formas.
El problema es que comprueba errores en las aperturas, pero no lo hace
en las lecturas y escrituras: las funciones `fread` y `fwrite`
pueden fallar y no se está comprobando. Ya que ha decidido usar `stdio` para leer y
escribir, debería usar `ferror` y `feof` para comprobar los posibles errores.

- Todo está implementado en el procedimiento principal (main). Aunque sea
un programa pequeño, debería estar partido en funciones
(i.e. cada subproblema en un subprograma).

- El código no es muy estructurado: esos `continue` son una guarrada. Nótese que
en esta asignatura se permite el uso de `continue` y `break`, pero tiene que
estar justificado. Esto se puede escribir mucho más limpio.

- Debería definir al menos una constante para el tamaño del buffer.

- No debería declarar variables en cualquier parte de la función. Esto tiene que ver
con la comprensión de los registros de activación en la pila, la depuración y
los posibles problemas de ocultación (*shadowing*).

¿Estaría este programa aprobado aplicando los mismos criterios de corrección? Sin duda.
Teniendo en cuenta las entregas del examen, **la nota de ChatGPT estaría
en el primer cuartil.**

___

### Mejorando: partir el programa

**yo:** Está todo implementado en la función principal, ¿puedes partir en funciones?

**ChatGPT**: Claro, aquí te muestro cómo podría dividir el código en varias funciones para una mejor legibilidad y reutilización:

```c
#include <stdio.h>
#include <stdlib.h>
#include <dirent.h>
#include <string.h>

int checkArguments(int argc, char* argv[]) {
    if (argc != 2) {
        printf("Uso: concattxt [archivo de salida]\n");
        return 1;
    }
    return 0;
}

FILE* openOutputFile(char* outputFile) {
    FILE* fp = fopen(outputFile, "w");
    if (!fp) {
        printf("No se pudo abrir el archivo de salida: %s\n", outputFile);
        exit(1);
    }
    return fp;
}

DIR* openCurrentDir() {
    DIR* dir = opendir(".");
    if (!dir) {
        printf("No se pudo abrir el directorio actual\n");
        exit(1);
    }
    return dir;
}

void concatenateTXT(FILE* fp, DIR* dir) {
    struct dirent* ent;
    while ((ent = readdir(dir)) != NULL) {
        if (ent->d_type != DT_REG) {
            continue;
        }

        int len = strlen(ent->d_name);
        if (len < 4 || strcmp(ent->d_name + len - 4, ".txt") != 0) {
            continue;
        }

        FILE* file = fopen(ent->d_name, "r");
        if (!file) {
            printf("No se pudo abrir el archivo: %s\n", ent->d_name);
            continue;
        }

        char buffer[4096];
        size_t bytesRead;
        while ((bytesRead = fread(buffer, 1, sizeof(buffer), file)) > 0) {
            fwrite(buffer, 1, bytesRead, fp);
        }

        fclose(file);
    }
}

int main(int argc, char* argv[]) {
    if (checkArguments(argc, argv))
        return 1;
    char* outputFile = argv[1];
    FILE* fp = openOutputFile(outputFile);
    DIR* dir = openCurrentDir();
    concatenateTXT(fp, dir);
    closedir(dir);
    fclose(fp);
    return 0;
}
```

**ChatGPT:** De esta manera el código queda más organizado, se puede entender mejor y es más fácil de mantener y extender. Cada función tiene un propósito específico, es fácil de entender y testear por separado, y se pueden reutilizar en otros programas.

___

Como ella misma dice, partir el programa mejora la legibilidad y la reutilización del código.
Vemos que ha partido el código, como se ha pedido. No está muy bien partido,
porque básicamente está recubriendo las funciones de biblioteca y
sacando parte del programa principal a otra función. Al menos, lo ha intentado.

___

### Los errores, por stderr

**yo:** Los errores deberian salir por la salida de errores, ¿no?

**ChatGPT:** Sí, tienes razón. Es recomendable que los errores y mensajes de advertencia se escriban en la salida de error estándar (stderr) en lugar de la salida estándar (stdout), ya que la salida de error estándar suele ser redirigida a un archivo o a una consola separada, lo que facilita su monitoreo y registro. Para escribir en la salida de error estándar, se puede usar la función fprintf() con el puntero de archivo "stderr" en lugar de "stdout". Por ejemplo:

fprintf(stderr, "Error: No se pudo abrir el archivo de salida: %s\n", outputFile);

**yo:** ¿Puedes reescribir el código de esa manera?

**ChatGPT:** Claro, aquí te dejo una versión actualizada del código donde se escriben los mensajes de error en la salida de error estándar:

```c
#include <stdio.h>
#include <stdlib.h>
#include <dirent.h>
#include <string.h>

int checkArguments(int argc, char* argv[]) {
    if (argc != 2) {
        fprintf(stderr, "Uso: concattxt [archivo de salida]\n");
        return 1;
    }
    return 0;
}

FILE* openOutputFile(char* outputFile) {
    FILE* fp = fopen(outputFile, "w");
    if (!fp) {
        fprintf(stderr, "Error: No se pudo abrir el archivo de salida: %s\n", outputFile);
        exit(1);
    }
    return fp;
}

DIR* openCurrentDir() {
    DIR* dir = opendir(".");
    if (!dir) {
        fprintf(stderr, "Error: No se pudo abrir el directorio actual\n");
        exit(1);
    }
    return dir;
}

void concatenateTXT(FILE* fp, DIR* dir) {
    struct dirent* ent;
    while ((ent = readdir(dir)) != NULL) {
        if (ent->d_type != DT_REG) {
            continue;
        }

        int len = strlen(ent->d_name);
        if (len < 4 || strcmp(ent->d_name + len - 4, ".txt") != 0) {
            continue;
        }

        FILE* file = fopen(ent->d_name, "r");
        if (!file) {
            fprintf(stderr, "Error: No se pudo abrir el archivo: %s\n", ent->d_name);
            continue;
        }

        char buffer[4096];
        size_t bytesRead;
        while ((bytesRead = fread(buffer, 1, sizeof(buffer), file)) > 0) {
            fwrite(buffer, 1, bytesRead, fp);
        }

        fclose(file);
    }
}

int main(int argc, char* argv[]) {
    if (checkArguments(argc, argv))
        return 1;
    char* outputFile = argv[1];
    FILE* fp = openOutputFile(outputFile);
    DIR* dir = openCurrentDir();
    concatenateTXT(fp, dir);
    closedir(dir);
    fclose(fp);
    return 0;
}
```

___

Correcto, ahora está mejor.

___

### Constantes

**yo:** ¿Puedes definir constantes cuando sea apropiado?

**ChatGPT:** Sí, se pueden definir constantes en C para mejorar la legibilidad y mantenibilidad del código.
Una constante es un valor que no puede ser modificado una vez que se asigna. En C, se pueden definir constantes utilizando la palabra clave "const" delante del tipo de datos. Por ejemplo, se puede definir una constante entera llamada "BUFFER_SIZE" de la siguiente manera:
const int BUFFER_SIZE = 4096;
En el código anterior se puede observar como se usa una constante BUFFER_SIZE para definir el tamaño del buffer, esto tiene varias ventajas:

1. Facilita el mantenimiento del código, ya que si se necesita cambiar el tamaño del buffer solo se tiene que modificar una sola línea de código.

2. Mejora la legibilidad del código, ya que se puede entender que el tamaño del buffer es un valor constante.

3. Aumenta la seguridad del código, ya que al ser una constante no se puede modificar accidentalmente.

También se pueden utilizar constantes en otros lugares donde sea apropiado, como en los argumentos de las funciones, o para definir el tamaño de un arreglo.

**yo:** ¿Puedes reescribir el código con constantes cuando hagan falta?

**ChatGPT:** Claro, aquí te muestro cómo podría reescribir el código utilizando constantes cuando sea apropiado:

```c
#include <stdio.h>
#include <stdlib.h>
#include <dirent.h>
#include <string.h>

const int BUFFER_SIZE = 4096;

int checkArguments(int argc, char* argv[]) {
    if (argc != 2) {
        fprintf(stderr, "Uso: concattxt [archivo de salida]\n");
        return 1;
    }
    return 0;
}

FILE* openOutputFile(char* outputFile) {
    FILE* fp = fopen(outputFile, "w");
    if (!fp) {
        fprintf(stderr, "Error: No se pudo abrir el archivo de salida: %s\n", outputFile);
        exit(1);
    }
    return fp;
}

DIR* openCurrentDir() {
    DIR* dir = opendir(".");
    if (!dir) {
        fprintf(stderr, "Error: No se pudo abrir el directorio actual\n");
        exit(1);
    }
    return dir;
}

void concatenateTXT(FILE* fp, DIR* dir) {
    struct dirent* ent;
    while ((ent = readdir(dir)) != NULL) {
        if (ent->d_type != DT_REG) {
            continue;
        }

        int len = strlen(ent->d_name);
        if (len < 4 || strcmp(ent->d_name + len - 4, ".txt") != 0) {
            continue;
        }

        FILE* file = fopen(ent->d_name, "r");
        if (!file) {
            fprintf(stderr, "Error: No se pudo abrir el archivo: %s\n", ent->d_name);
            continue;
        }

        char buffer[BUFFER_SIZE];
        size_t bytesRead;
        while ((bytesRead = fread(buffer, 1, sizeof(buffer), file)) > 0) {
            fwrite(buffer, 1, bytesRead, fp);
        }

        fclose(file);
    }
}

int main(int argc, char* argv[]) {
    if (checkArguments(argc, argv))
        return 1;
    char* outputFile = argv[1];
    FILE* fp = openOutputFile(outputFile);
    DIR* dir = openCurrentDir();
    concatenateTXT(fp, dir);
    closedir(dir);
    fclose(fp);
    return 0;
}
```
**ChatGPT:** Espero que esta nueva versión del código cumpla con tus necesidades.

___

Aquí está usando `const` para definir la constante. Entramos en un viejo
debate: ¿cuál es la mejor forma de definir constantes en C?

Tenemos tres formas para definir constantes enteras: `define`, `const`
y `enum`.

`define` es una orden para el preprocesador de C que realiza
una sustitución textual en el fuente antes de pasar al compilador.
Eso trae muchos problemas. `const` es un cualificador que
indica que dicha **"variable"** es inmutable,
que no es lo mismo que definir un valor que sea constante durante toda
la ejecución del programa:
la memoria de dicha variable puede estar en una región
de memoria que sea modificable y un bug puede provocar un cambio en el valor de
la constante (cosa que nunca debería pasar), se puede conseguir su dirección de memoria
con &, etc. Puedes ver un ejemplo [aquí](https://twitter.com/e__soriano/status/1398573039256756225?s=20&t=GXIS8vrp9Q6yYs6H9zVVtg).
Para definir constantes enteras en C, lo más apropiado es usar `enum`, que sirve para
crear tipos enumerados y permite asignar nombres a constantes enteras.

___

### Usar las llamadas al sistema para I/O

**yo:** ¿Puedes cambiar el programa de C para que no use stdio y use las llamadas al sistema read y write?

**ChatGPT:** Sí, se puede reescribir el programa para que utilice las llamadas al sistema read() y write() en lugar de las funciones de la biblioteca estándar stdio. Sin embargo, es importante tener en cuenta que estas llamadas al sistema son más bajas nivel y requieren un mayor cuidado y precisión al manejar los errores y los tamaños de los datos.
Claro, aquí te muestro el programa completo reescrito para utilizar las llamadas al sistema read() y write():

```c
#include <unistd.h>
#include <stdlib.h>
#include <dirent.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

#define BUFFER_SIZE 4096

int checkArguments(int argc, char* argv[]) {
    if (argc != 2) {
        write(STDERR_FILENO, "Uso: concattxt [archivo de salida]\n", 34);
        return 1;
    }
    return 0;
}

int openOutputFile(char* outputFile) {
    int fd = open(outputFile, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        char error[50];
        snprintf(error, 50, "Error: No se pudo abrir el archivo de salida: %s\n", outputFile);
        write(STDERR_FILENO, error, strlen(error));
        exit(1);
    }
    return fd;
}

DIR* openCurrentDir() {
    DIR* dir = opendir(".");
    if (!dir) {
        write(STDERR_FILENO, "Error: No se pudo abrir el directorio actual\n", 42);
        exit(1);
    }
    return dir;
}

void concatenateTXT(int fd, DIR* dir) {
    struct dirent* ent;
    while ((ent = readdir(dir)) != NULL) {
        if (ent->d_type != DT_REG) {
            continue;
        }

        int len = strlen(ent->d_name);
        if (len < 4 || strcmp(ent->d_name + len - 4, ".txt") != 0) {
            continue;
        }

        int file = open(ent->d_name, O_RDONLY);
        if (file == -1) {
            char error[50];
            snprintf(error, 50, "Error: No se pudo abrir el archivo: %s\n", ent->d_name);
            write(STDERR_FILENO, error, strlen(error));
            continue;
        }

        char buffer[BUFFER_SIZE];
        ssize_t bytesRead;
        while ((bytesRead = read(file, buffer, sizeof(buffer))) > 0) {
            write(fd, buffer, bytesRead);
        }

        close(file);
    }
}

int main(int argc, char* argv[]) {
    if (checkArguments(argc, argv))
        return 1;
    char* outputFile = argv[1];
    int fd = openOutputFile(outputFile);
    DIR* dir = openCurrentDir();
    concatenateTXT(fd, dir);
    closedir(dir);
    close(fd);
    return 0;
}
```

---

Aquí se empieza a torcer el asunto.

En realidad, yo quería que usara las
llamadas al sistema `open`, `read`, `write` y `close` en lugar
de `fopen`, `fread`, `fwrite` y `fclose` para la lectura/escritura
de los ficheros, no para escribir los mensajes de error/diagnóstico
en `stderr` (eso de podía quedar con `fprintf`). De todas formas,
es natural que ChatGPT no me haya entendido bien y haya cambiado todo.

El problema es que
ahora el código **no compila**: ha quitado `includes` pero sigue usando
funciones que los necesitan.

Sigue sin comprobar los errores en las lecturas y las escrituras.

---

### Un script de shell

**yo:** ¿Puedes reescribir el programa en shell scripting?

**ChatGPT:** Claro, aquí te muestro una posible implementación del programa en shell scripting:

```sh
#!/bin/bash

if [[ $# -ne 1 ]]
then
    echo "Uso: concattxt [archivo de salida]" >&2
    exit 1
fi

outputFile=$1

> $outputFile
for file in *.txt
do
    cat $file >> $outputFile
done
```

**ChatGPT:** El script comienza verificando el número de argumentos pasados al script mediante el uso de la expresión if [[ $# -ne 1 ]]. El uso de >&2 en el echo, redirige la salida a la salida de error, de esta forma se diferencian los mensajes de error de los normales.
Luego, se asigna el primer argumento (nombre del archivo de salida) a una variable outputFile.
Por último, se utiliza el comando for para recorrer todos los archivos con extensión .txt en el directorio actual. Dentro del ciclo, se utiliza el comando cat para concatenar el contenido de cada archivo al archivo de salida. El operador >> se utiliza para agregar el contenido al final del archivo de salida en lugar de sobreescribirlo.
Espero que esta implementación sea de ayuda.

---

El script es correcto, incluso escribe los errores por la salida de errores.

___

###  Opinión (la de hoy :) )

Estas herramientas van a cambiar la forma de programar, eso es lo único
que tengo claro.


Llevo escuchando que los programadores
dejarán de existir desde que empecé a estudiar en los 90s. Entonces, se decía
por las herramientas CASE. En la última década, por la IA.
Matt Welsh declaró hace pocos días el [*fin de la programación*](https://cacm.acm.org/magazines/2023/1/267976-the-end-of-programming/fulltext) en Communications of the ACM.
El exprofesor de Harvard (y exdirector de ingeniería en Apple y Google)
es el fundador y CEO de Fixie.ai, una startup de IA para desarrollo software.
Puede que eso tenga algo que ver con su opinión.

Los avances de los
últimos años parecen indicar que los programadores menos cualificados, los
dedicados a copiar y pegar código de *stack overflow*,
ya están en peligro. Sin embargo, en mi opinión, en el futuro
se seguirán necesitando programadores expertos
para, entre otras cosas, auditar, modificar e integrar la salida de estas
herramientas.
Si fuese un estudiante TIC, ahora mismo estaría bastante preocupado e
intentaría esforzarme para ser uno de ellos.

Como nos recuerda Rodney Brooks, exdirector
del MIT Computer Science and Artificial Intelligence Lab,
[*hay que calmarse*](https://rodneybrooks.com/predictions-scorecard-2023-january-01/).
Hasta el propio CEO de OpenAI, Sam Altman, [dice](https://twitter.com/sama/status/1601731295792414720?s=20&t=mGoYALxCL6wwHKJ4KzIrlA) que ChatGPT tiene
muchas limitaciones pero es
lo suficientemente buena como para dar una falsa impresión de grandeza y es un
error depender de ella a día de hoy.

Esta vez ha quedado entre los mejores de la clase,  pero podría haber fallado.
De hecho, he repetido el experimento y
ha generado una solución peor que usa `fgets` y `fputs`
(no son apropiadas para este programa).
En otras ocasiones me ha dado soluciones y explicaciones totalmente incorrectas.

Ya hay [estudios](https://arxiv.org/abs/2211.03622) que indican que el código
creado con asistentes de este tipo es más inseguro.
Apuesto a que este *hype* nos traerá muchos problemas durante los próximos años.

A largo plazo, puede que los humanos olvidemos las matemáticas y
la programación, y no sepamos cómo funcionan las máquinas.
Ya nos avisó Asimov (o eso dice ChatGPT).

### Comentarios

Puedes comentar este post en Mastodon: [https://social.linux.pizza/@esoriano/109684708021342526](https://social.linux.pizza/@esoriano/109684708021342526)

<sub><sup>
    <b>(cc) Enrique Soriano-Salvador</b>
    Creative Commons (by-nc-nd).
    Creative Commons, 559 Nathan Abbott Way, Stanford,
    California 94305, USA.
</sup></sub>
