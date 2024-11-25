---
title: Ejecutando comandos con Snakemake
teaching: 30
exercises: 30
---



::: questions


- "¿Cómo ejecuto un comando simple con Snakemake?"

:::



:::objectives


- "Crea una receta de Snakemake (un Snakefile)"

:::


## ¿Cuál es el flujo de trabajo que me interesa?

En esta lección haremos un experimento que toma una aplicación que corre en paralelo e
investigará su escalabilidad. Para ello necesitaremos recopilar datos, en este caso eso
significa ejecutar la aplicación varias veces con diferentes números de núcleos de CPU y
registrar el tiempo de ejecución. Una vez hecho esto, tenemos que crear una
visualización de los datos para ver cómo se compara con el caso ideal.

A partir de la visualización podemos decidir a qué escala tiene más sentido ejecutar la
aplicación en producción para maximizar el uso de nuestra asignación de CPU en el
sistema.

Podríamos hacer todo esto manualmente, pero existen herramientas útiles que nos ayudan a
gestionar pipelines de análisis de datos como el que tenemos en nuestro experimento. Hoy
vamos a aprender acerca de uno de ellos: Snakemake.

Con el fin de empezar con Snakemake, vamos a empezar por tomar un comando simple y ver
cómo podemos ejecutar que a través de Snakemake. Elijamos el comando `hostname` que
imprime el nombre del host donde se ejecuta el comando:

```bash
[ocaisa@node1 ~]$ hostname
```

```output
node1.int.jetstream2.hpc-carpentry.org
```

Eso imprime el resultado pero Snakemake se basa en archivos para conocer el estado de su
flujo de trabajo, así que vamos a redirigir la salida a un archivo:

```bash
[ocaisa@node1 ~]$ hostname > hostname_login.txt
```

## Creando un archivo Snakefile

Edita un nuevo archivo de texto llamado `Snakefile`.

Contenido de `Snakefile`:

```python
rule hostname_login:
    output: "hostname_login.txt"
    input:  
    shell:
        "hostname > hostname_login.txt"
```


::: callout


## Puntos clave sobre este archivo

1. El archivo se llama `Snakefile` - con mayúscula `S` y sin extensión de archivo.
1. Algunas líneas tienen sangría. Las sangrías deben ser con caracteres de espacio, no
   tabuladores. Vea la sección de configuración para saber cómo hacer que su editor de
   texto haga esto.
1. La definición de la regla comienza con la palabra clave `rule` seguida por el nombre
   de la regla, luego dos puntos.
1. Llamamos a la regla `hostname_login`. Puede utilizar letras, números o guiones bajos,
   pero el nombre de la regla debe comenzar con una letra y no puede ser una palabra
   clave.
1. Las palabras clave `input`, `output`, y `shell` van todas seguidas de dos puntos
   (":").
1. Los nombres de los archivos y el comando shell están todos en `"quotes"`.
1. El nombre del archivo de salida se da antes del nombre del archivo de entrada. De
   hecho, a Snakemake no le importa en qué orden aparecen, pero en este curso daremos
   primero el de salida. Pronto veremos por qué.
1. En este caso de uso no hay fichero de entrada para el comando por lo que lo dejamos
   en blanco.

:::


De vuelta en el shell ejecutaremos nuestra nueva regla. En este punto, si faltaran
comillas, sangrías incorrectas, etc., podríamos ver un error.

```bash
snakemake -j1 -p hostname_login
```


::: callout


## `bash: snakemake: command not found...`

Si tu shell te dice que no puede encontrar el comando `snakemake` entonces tenemos que
hacer que el software esté disponible de alguna manera. En nuestro caso, esto significa
buscar el módulo que necesitamos cargar:

```bash
module spider snakemake
```

```output
[ocaisa@node1 ~]$ module spider snakemake

--------------------------------------------------------------------------------------------------------
  snakemake:
--------------------------------------------------------------------------------------------------------
     Versions:
        snakemake/8.2.1-foss-2023a
        snakemake/8.2.1 (E)

Names marked by a trailing (E) are extensions provided by another module.


--------------------------------------------------------------------------------------------------------
  For detailed information about a specific "snakemake" package (including how to load the modules) use the module's full name.
  Note that names that have a trailing (E) are extensions provided by other modules.
  For example:

     $ module spider snakemake/8.2.1
--------------------------------------------------------------------------------------------------------
```

Ahora queremos el módulo, así que vamos a cargarlo para que el paquete esté disponible

```bash
[ocaisa@node1 ~]$ module load snakemake
```

y luego asegúrate de que tenemos el comando `snakemake` disponible

```bash
[ocaisa@node1 ~]$ which snakemake
```

```output
/cvmfs/software.eessi.io/host_injections/2023.06/software/linux/x86_64/amd/zen3/software/snakemake/8.2.1-foss-2023a/bin/snakemake
```

```bash
snakemake -j1 -p hostname_login
```

:::



::: challenge

## Ejecutando Snakemake

Ejecute `snakemake --help | less` para ver la ayuda de todas las opciones disponibles.
¿Qué hace la opción `-p` en el comando `snakemake` anterior?

1. Protege los archivos de salida existentes
1. Imprime en el terminal los comandos shell que se están ejecutando
1. Le dice a Snakemake que sólo ejecute un proceso a la vez
1. Solicita al usuario el fichero de entrada correcto

:::::: hint

Puedes buscar en el texto pulsando <kbd>/</kbd>, y salir de nuevo al shell con
<kbd>q</kbd>.

::::::



:::::: solution

(2) Imprime en el terminal los comandos shell que se están ejecutando

¡Esto es tan útil que no sabemos por qué no está por defecto! La opción `-j1` es lo que
le dice a Snakemake que sólo ejecute un proceso a la vez, y nos quedaremos con esto por
ahora ya que hace las cosas más simples. La respuesta 4 es una pista falsa, ya que
Snakemake nunca solicita interactivamente la entrada del usuario.

::::::


:::



::: keypoints


- "Antes de ejecutar Snakemake necesitas escribir un Snakefile"
- "Un Snakefile es un archivo de texto que define una lista de reglas"
- "Las reglas tienen entradas, salidas y comandos de shell a ejecutar"
- "Le dices a Snakemake qué archivo hacer y ejecutará el comando shell definido en la
  regla apropiada"

:::


