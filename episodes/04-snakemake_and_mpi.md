---
title: Aplicaciones MPI y Snakemake
teaching: 30
exercises: 20
---



::: objectives


- "Define reglas para ejecutar localmente y en el clúster"

:::



::: questions


- "¿Cómo puedo ejecutar una aplicación MPI a través de Snakemake en el cluster?"

:::


Ahora es el momento de volver a nuestro flujo de trabajo real. Podemos ejecutar un
comando en el cluster, pero ¿qué pasa con la ejecución de la aplicación MPI que nos
interesa? Nuestra aplicación se llama `amdahl` y está disponible como módulo de entorno.


::: challenge


Localiza y carga el módulo `amdahl` y luego _reemplaza_ nuestra regla `hostname_remote`
con una versión que ejecute `amdahl`. (No te preocupes por el MPI paralelo todavía,
ejecútalo con una sola CPU, `mpiexec -n 1 amdahl`).

¿Se ejecuta correctamente la regla? Si no es así, revise los archivos de registro para
averiguar por qué


::::::solution


```bash
module spider amdahl
module load amdahl
```

localizará y cargará el módulo `amdahl`. Podemos entonces actualizar/reemplazar nuestra
regla para ejecutar la aplicación `amdahl`:

```python
rule amdahl_run:
    output: "amdahl_run.txt"
    input:
    shell:
        "mpiexec -n 1 amdahl > {output}"
```

Sin embargo, cuando intentamos ejecutar la regla obtenemos un error (a menos que ya
tengas una versión diferente de `amdahl` disponible en tu ruta). Snakemake informa de la
ubicación de los logs y si miramos dentro podemos (eventualmente) encontrar

```output
...
mpiexec -n 1 amdahl > amdahl_run.txt
--------------------------------------------------------------------------
mpiexec was unable to find the specified executable file, and therefore
did not launch the job.  This error was first reported for process
rank 0; it may have occurred for other processes as well.

NOTE: A common cause for this error is misspelling a mpiexec command
      line parameter option (remember that mpiexec interprets the first
      unrecognized command line token as the executable).

Node:       tmpnode1
Executable: amdahl
--------------------------------------------------------------------------
...
```

Así que, aunque cargamos el módulo antes de ejecutar el flujo de trabajo, nuestra regla
Snakemake no encontró el ejecutable. Esto se debe a que la regla de Snakemake se está
ejecutando en un _entorno de ejecución_ limpio, y tenemos que decirle de alguna manera
que cargue el módulo de entorno necesario antes de intentar ejecutar la regla.


::::::


:::


## Snakemake y módulos de entorno

Nuestra aplicación se llama `amdahl` y está disponible en el sistema a través de un
módulo de entorno, por lo que necesitamos decirle a Snakemake que cargue el módulo antes
de que intente ejecutar la regla. Snakemake es consciente de los módulos de entorno, y
estos pueden ser especificados a través de (otra) opción:

```python
rule amdahl_run:
    output: "amdahl_run.txt"
    input:
    envmodules:
      "mpi4py",
      "amdahl"
    input:
    shell:
        "mpiexec -n 1 amdahl > {output}"
```

Sin embargo, añadir estas líneas no es suficiente para que la regla se ejecute. No sólo
tienes que decirle a Snakemake qué módulos cargar, sino que también tienes que decirle
que use módulos de entorno en general (ya que se considera que el uso de módulos de
entorno hace que tu entorno de ejecución sea menos reproducible, ya que los módulos
disponibles pueden diferir de un cluster a otro). Esto requiere que le des a Snakemake
una opción adicional

```bash
snakemake --profile cluster_profile --use-envmodules amdahl_run
```


::: challenge


Utilizaremos módulos de entorno durante el resto del tutorial, así que conviértalo en
una opción predeterminada de nuestro perfil (estableciendo su valor en `True`)


::::::solution


Actualiza el perfil de nuestro clúster a

```yaml
printshellcmds: True
jobs: 3
executor: slurm
default-resources:
  - mem_mb_per_cpu=3600
  - runtime=2
use-envmodules: True
```

Si quieres probarlo, tienes que borrar el archivo de salida de la regla y volver a
ejecutar Snakemake.


::::::



:::


## Snakemake y MPI

En realidad no hemos ejecutado una aplicación MPI en la última sección, ya que sólo lo
hemos hecho en un núcleo. ¿Cómo solicitamos que se ejecute en varios núcleos para una
única regla?

Snakemake tiene soporte general para MPI, pero el único ejecutor que actualmente soporta
explícitamente MPI es el ejecutor Slurm (¡por suerte para nosotros!). Si volvemos a
mirar nuestra tabla de traducción de Slurm a Snakemake nos daremos cuenta de que las
opciones relevantes aparecen cerca de la parte inferior:

| SLURM             | Snakemake       | Description                                                    |
| ----------------- | --------------- | -------------------------------------------------------------- |
| ...               | ...             | ...                                                            |
| `--ntasks`        | `tasks`         | number of concurrent tasks / ranks                             |
| `--cpus-per-task` | `cpus_per_task` | number of cpus per task (in case of SMP, rather use `threads`) |
| `--nodes`         | `nodes`         | number of nodes                                                |

El que nos interesa es `tasks` ya que sólo vamos a aumentar el número de rangos. Podemos
definirlos en una sección `resources` de nuestra regla y referirnos a ellos usando
marcadores de posición:

```python
rule amdahl_run:
    output: "amdahl_run.txt"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi='mpiexec',
      tasks=2
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl > {output}"
```

Ha funcionado, pero ahora tenemos un pequeño problema. Queremos hacer esto para algunos
valores diferentes de `tasks` que significaría que necesitaríamos un archivo de salida
diferente para cada ejecución. Sería genial si de alguna manera podemos indicar en el
`output` el valor que queremos utilizar para `tasks` ... y que Snakemake lo recoja.

Podríamos utilizar un _wildcard_ en el `output` para permitirnos definir el `tasks` que
deseamos utilizar. La sintaxis de este comodín es la siguiente

```python
output: "amdahl_run_{parallel_tasks}.txt"
```

donde `parallel_tasks` es nuestro comodín.


::: callout

## Comodines

Los comodines se utilizan en las líneas `input` y `output` de la regla para representar
partes de nombres de archivo. Al igual que el patrón `*` del intérprete de comandos, el
comodín puede sustituir a cualquier texto para formar el nombre de archivo deseado. Al
igual que con el nombre de sus reglas, puede elegir cualquier nombre que desee para sus
comodines, así que aquí hemos utilizado `parallel_tasks`. El uso de los mismos comodines
en la entrada y la salida es lo que le dice a Snakemake cómo hacer coincidir los
archivos de entrada con los archivos de salida.

Si dos reglas usan un comodín con el mismo nombre entonces Snakemake las tratará como
entidades diferentes - las reglas en Snakemake son autocontenidas de esta manera.

En la línea `shell` puede hacer referencia al comodín con `{wildcards.parallel_tasks}`

:::


## Orden de operaciones de Snakemake

Sólo estamos empezando con algunas reglas simples, pero vale la pena pensar en lo que
Snakemake está haciendo exactamente cuando se ejecuta. Hay tres fases distintas:

1. Prepara la ejecución:
    1. Lee todas las definiciones de reglas del archivo Snakefile
1. Planea qué hacer:
    1. Ve qué fichero(s) le estás pidiendo que haga
    1. Busca una regla coincidente mirando las `output`s de todas las reglas que conoce
    1. Rellena los comodines para calcular el `input` de esta regla
    1. Comprueba que este archivo de entrada (si es necesario) está disponible
1. Ejecuta los pasos:
    1. Crea el directorio para el fichero de salida, si es necesario
    1. Elimina el archivo de salida antiguo si ya está ahí
    1. Sólo entonces, ejecuta el comando de shell con los marcadores de posición
       sustituidos
    1. Comprueba que el comando se ejecuta sin errores *y* crea el nuevo archivo de
       salida según lo esperado

::: callout

## Modo de ejecución en seco (`-n`)

A menudo es útil ejecutar sólo las dos primeras fases, de modo que Snakemake planificará
los trabajos a ejecutar, y los imprimirá en la pantalla, pero nunca los ejecutará
realmente. Esto se hace con la bandera `-n`, eg:

```bash
> $ snakemake -n ...
```


:::


La cantidad de comprobaciones puede parecer pedante ahora mismo, pero a medida que el
flujo de trabajo gane más pasos esto nos resultará realmente muy útil.

## Usando comodines en nuestra regla

Nos gustaría utilizar un comodín en `output` para poder definir el número de `tasks` que
deseamos utilizar. Basándonos en lo que hemos visto hasta ahora, se podría imaginar que
esto podría tener el siguiente aspecto

```python
rule amdahl_run:
    output: "amdahl_run_{parallel_tasks}.txt"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      tasks="{parallel_tasks}"
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl > {output}"
```

pero hay dos problemas con esto:

- La única forma de que Snakemake conozca el valor del comodín es que el usuario
  solicite explícitamente un archivo de salida concreto (en lugar de llamar a la regla):

```bash
  snakemake --profile cluster_profile amdahl_run_2.txt
```

Esto es perfectamente válido, ya que Snakemake puede averiguar que tiene una regla que
puede coincidir con ese nombre de archivo.

- El mayor problema es que incluso haciendo eso no funciona, parece que no podemos usar
  un comodín para `tasks`:

  ```output
  WorkflowError:
  SLURM job submission failed. The error message was sbatch:
  error: Invalid numeric value "{parallel_tasks}" for --ntasks.
  ```

Por desgracia para nosotros, no hay forma directa de acceder a los comodines de `tasks`.
La razón de esto es que Snakemake intenta utilizar el valor de `tasks` durante su etapa
de inicialización, que es antes de que sepamos el valor del comodín. Necesitamos aplazar
la determinación de `tasks` para más adelante. Esto se puede conseguir especificando una
función de entrada en lugar de un valor para este escenario. La solución entonces es
escribir una función de un solo uso para manipular Snakemake para que haga esto por
nosotros. Dado que la función es específicamente para la regla, podemos utilizar una
función de una sola línea sin nombre. Este tipo de funciones se llaman funciones
anónimas o funciones lamdba (ambas significan lo mismo), y son una característica de
Python (y otros lenguajes de programación).

Para definir una función lambda en python, la sintaxis general es la siguiente:

```python
lambda x: x + 54
```

Dado que nuestra función _puede_ tomar los comodines como argumentos, podemos utilizarla
para establecer el valor de `tasks`:

```python
rule amdahl_run:
    output: "amdahl_run_{parallel_tasks}.txt"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      # No direct way to access the wildcard in tasks, so we need to do this
      # indirectly by declaring a short function that takes the wildcards as an
      # argument
      tasks=lambda wildcards: int(wildcards.parallel_tasks)
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl > {output}"
```

Ahora tenemos una regla que puede utilizarse para generar la salida de ejecuciones de un
número arbitrario de tareas paralelas.


::: callout


## Comentarios en Snakefiles

En el código anterior, la línea que comienza `#` es una línea de comentario. Esperemos
que ya tengas el hábito de añadir comentarios a tus propios scripts. Los buenos
comentarios hacen que cualquier script sea más legible, y esto es igual de cierto con
los Snakefiles.


:::


Dado que nuestra regla es ahora capaz de generar un número arbitrario de ficheros de
salida las cosas podrían llenarse mucho en nuestro directorio actual. Probablemente sea
mejor entonces poner las ejecuciones en una carpeta separada para mantener las cosas
ordenadas. Podemos añadir la carpeta directamente a nuestro `output` y Snakemake se
encargará de crear el directorio por nosotros:

```python
rule amdahl_run:
    output: "runs/amdahl_run_{parallel_tasks}.txt"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      # No direct way to access the wildcard in tasks, so we need to do this
      # indirectly by declaring a short function that takes the wildcards as an
      # argument
      tasks=lambda wildcards: int(wildcards.parallel_tasks)
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl > {output}"
```


::: challenge


Crea un fichero de salida (en la carpeta `runs`) para el caso en que tengamos 6 tareas
paralelas

(SUGERENCIA: Recuerde que Snakemake tiene que ser capaz de hacer coincidir el archivo
solicitado con el `output` de una regla)


:::::: solution


```bash
snakemake --profile cluster_profile runs/amdahl_run_6.txt
```


::::::



:::


Otra cosa sobre nuestra aplicación `amdahl` es que en última instancia queremos procesar
la salida para generar nuestro gráfico de escala. La salida en este momento es útil para
la lectura, pero hace que el procesamiento sea más difícil.`amdahl` tiene una opción que
realmente hace esto más fácil para nosotros. Para ver las opciones de `amdahl` podemos
utilizar

```bash
[ocaisa@node1 ~]$ module load amdahl
[ocaisa@node1 ~]$ amdahl --help
```

```output
usage: amdahl [-h] [-p [PARALLEL_PROPORTION]] [-w [WORK_SECONDS]] [-t] [-e]

options:
  -h, --help            show this help message and exit
  -p [PARALLEL_PROPORTION], --parallel-proportion [PARALLEL_PROPORTION]
                        Parallel proportion should be a float between 0 and 1
  -w [WORK_SECONDS], --work-seconds [WORK_SECONDS]
                        Total seconds of workload, should be an integer greater than 0
  -t, --terse           Enable terse output
  -e, --exact           Disable random jitter
```

La opción que buscamos es `--terse`, que hará que `amdahl` imprima la salida en un
formato mucho más fácil de procesar, JSON. El formato JSON en un archivo normalmente
utiliza la extensión de archivo `.json` así que vamos a añadir esa opción a nuestro
comando `shell` _y_ cambiar el formato de archivo de la `output` para que coincida con
nuestro nuevo comando:

```python
rule amdahl_run:
    output: "runs/amdahl_run_{parallel_tasks}.json"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      # No direct way to access the wildcard in tasks, so we need to do this
      # indirectly by declaring a short function that takes the wildcards as an
      # argument
      tasks=lambda wildcards: int(wildcards.parallel_tasks)
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl --terse > {output}"
```

Había otro parámetro para `amdahl` que me llamó la atención. `amdahl` tiene una opción
`--parallel-proportion` (o `-p`) que nos puede interesar cambiar, ya que cambia el
comportamiento del código, y por tanto tiene un impacto en los valores que obtenemos en
nuestros resultados. Vamos a añadir otra capa de directorio a nuestro formato de salida
para reflejar una elección particular para este valor. Podemos utilizar un comodín para
no tener que elegir el valor de inmediato:

```python
rule amdahl_run:
    output: "p_{parallel_proportion}/runs/amdahl_run_{parallel_tasks}.json"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      # No direct way to access the wildcard in tasks, so we need to do this
      # indirectly by declaring a short function that takes the wildcards as an
      # argument
      tasks=lambda wildcards: int(wildcards.parallel_tasks)
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl --terse -p {wildcards.parallel_proportion} > {output}"
```


::: challenge


Crea un fichero de salida para un valor de `-p` de 0.999 (el valor por defecto es 0.8)
para el caso en que tengamos 6 tareas paralelas.


:::::: solution


```bash
snakemake --profile cluster_profile p_0.999/runs/amdahl_run_6.json
```


::::::



:::



::: keypoints


- "Snakemake elige la regla apropiada mediante la sustitución de comodines de tal manera
  que la salida coincide con el objetivo"
- "Snakemake comprueba varias condiciones de error y se detendrá si ve un problema"

:::


