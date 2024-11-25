---
title: Ejecutando Snakemake en el cluster
teaching: 30
exercises: 20
---


::: objectives

- "Define reglas para ejecutar localmente y en el cluster"

:::

::: questions

- "¿Cómo ejecuto mi regla de Snakemake en el cluster?"

:::

¿Qué ocurre cuando queremos que nuestra regla se ejecute en el cluster en lugar de en el
nodo de inicio de sesión? El cluster que estamos usando utiliza Slurm, y sucede que
Snakemake tiene soporte integrado para Slurm, solo necesitamos decirle que queremos
usarlo.

Snakemake utiliza la opción `executor` para permitirte seleccionar el plugin que deseas
que ejecute la regla. La forma más rápida de aplicar esto a tu Snakefile es definirlo en
la línea de comandos. Vamos a probarlo

```bash
[ocaisa@node1 ~]$ snakemake -j1 -p --executor slurm hostname_login
```

```output
Building DAG of jobs...
Retrieving input from storage.
Nothing to be done (all requested files are present and up to date).
```

¡No ha pasado nada! ¿Por qué? Cuando se le pide que construya un objetivo, Snakemake
comprueba la 'última hora de modificación' tanto del objetivo como de sus dependencias.
Si alguna dependencia ha sido actualizada desde el objetivo, entonces las acciones se
vuelven a ejecutar para actualizar el objetivo. Usando este enfoque, Snakemake sabe que
sólo debe reconstruir los archivos que, directa o indirectamente, dependen del archivo
que cambió. Esto se llama una _construcción incremental_.

::: callout
## Las construcciones incrementales mejoran la eficiencia

Al reconstruir archivos sólo cuando es necesario, Snakemake hace que su procesamiento
sea más eficiente. :::

::: challenge
## Ejecutando en el cluster

Ahora necesitamos otra regla que ejecute el `hostname` en el _cluster_. Crea una nueva
regla en tu Snakefile e intenta ejecutarla en el cluster con la opción `--executor
slurm` a `snakemake`.

:::::: solution The rule is almost identical to the previous rule save for the rule name
and output file:

```python
rule hostname_remote:
    output: "hostname_remote.txt"
    input:
    shell:
        "hostname > hostname_remote.txt"
```

A continuación, puede ejecutar la regla con

```bash
[ocaisa@node1 ~]$ snakemake -j1 -p --executor slurm hostname_remote
```

```output
Building DAG of jobs...
Retrieving input from storage.
Using shell: /cvmfs/software.eessi.io/versions/2023.06/compat/linux/x86_64/bin/bash
Provided remote nodes: 1
Job stats:
job                count
---------------  -------
hostname_remote        1
total                  1

Select jobs to execute...
Execute 1 jobs...

[Mon Jan 29 18:03:46 2024]
rule hostname_remote:
    output: hostname_remote.txt
    jobid: 0
    reason: Missing output files: hostname_remote.txt
    resources: tmpdir=<TBD>

hostname > hostname_remote.txt
No SLURM account given, trying to guess.
Guessed SLURM account: def-users
No wall time information given. This might or might not work on your cluster.
If not, specify the resource runtime in your rule or as a reasonable default
via --default-resources. No job memory information ('mem_mb' or 
'mem_mb_per_cpu') is given - submitting without.
This might or might not work on your cluster.
Job 0 has been submitted with SLURM jobid 326 (log: /home/ocaisa/.snakemake/slurm_logs/rule_hostname_remote/326.log).
[Mon Jan 29 18:04:26 2024]
Finished job 0.
1 of 1 steps (100%) done
Complete log: .snakemake/log/2024-01-29T180346.788174.snakemake.log
```

Observe todas las advertencias que Snakemake nos está dando sobre el hecho de que la
regla puede no ser capaz de ejecutarse en nuestro cluster ya que puede que no hayamos
dado suficiente información. Por suerte para nosotros, esto funciona en nuestro cluster
y podemos echar un vistazo al archivo de salida que crea la nueva regla,
`hostname_remote.txt`:

```bash
[ocaisa@node1 ~]$ cat hostname_remote.txt
```

```output
tmpnode1.int.jetstream2.hpc-carpentry.org
```

::::::

:::

## Perfil de Snakemake

Adaptar Snakemake a un entorno particular puede implicar muchas banderas y opciones. Por
lo tanto, es posible especificar un perfil de configuración que se utilizará para
obtener las opciones por defecto. Esto se parece a

```bash
snakemake --profile myprofileFolder ...
```

La carpeta del perfil debe contener un archivo llamado `config.yaml` que es el que
almacenará nuestras opciones. La carpeta también puede contener otros archivos
necesarios para el perfil. Vamos a crear el archivo `cluster_profile/config.yaml` e
insertar algunas de nuestras opciones existentes:

```yaml
printshellcmds: True
jobs: 3
executor: slurm
```

Ahora debemos ser capaces de volver a ejecutar nuestro flujo de trabajo apuntando al
perfil en lugar de la lista de las opciones. Para obligar a nuestro flujo de trabajo a
volver a ejecutarse, primero tenemos que eliminar el archivo de salida
`hostname_remote.txt`, y luego podemos probar nuestro nuevo perfil

```bash
[ocaisa@node1 ~]$ rm hostname_remote.txt
[ocaisa@node1 ~]$ snakemake --profile cluster_profile hostname_remote
```

El perfil es extremadamente útil en el contexto de nuestro cluster, ya que el ejecutor
Slurm tiene muchas opciones, y a veces necesitas usarlas para poder enviar trabajos al
cluster al que tienes acceso. Desafortunadamente, los nombres de las opciones en
Snakemake no son _exactamente_ los mismos que los de Slurm, por lo que necesitamos la
ayuda de una tabla de traducción:

| SLURM             | Snakemake         | Description                                                    |
| ----------------- | ----------------- | -------------------------------------------------------------- |
| `--partition`     | `slurm_partition` | the partition a rule/job is to use                             |
| `--time`          | `runtime`         | the walltime per job in minutes                                |
| `--constraint`    | `constraint`      | may hold features on some clusters                             |
| `--mem`           | `mem, mem_mb`     | memory a cluster node must                                     |
|                   |                   | provide (mem: string with unit), mem_mb: int                   |
| `--mem-per-cpu`   | `mem_mb_per_cpu`  | memory per reserved CPU                                        |
| `--ntasks`        | `tasks`           | number of concurrent tasks / ranks                             |
| `--cpus-per-task` | `cpus_per_task`   | number of cpus per task (in case of SMP, rather use `threads`) |
| `--nodes`         | `nodes`           | number of nodes                                                |

Las advertencias dadas por Snakemake dieron a entender que podríamos necesitar
proporcionar estas opciones. Una forma de hacerlo es proporcionarlas como parte de la
regla de Snakemake usando la palabra clave `resources`, por ejemplo,

```python
rule:
    input: ...
    output: ...
    resources:
        partition: <partition name>
        runtime: <some number>
```

y también podemos usar el perfil para definir valores por defecto para estas opciones
para usar con nuestro proyecto, usando la palabra clave `default-resources`. Por
ejemplo, la memoria disponible en nuestro cluster es de unos 4GB por núcleo, así que
podemos añadir eso a nuestro perfil:

```yaml
printshellcmds: True
jobs: 3
executor: slurm
default-resources:
  - mem_mb_per_cpu=3600
```

:::challenge We know that our problem runs in a very short time. Change the default
length of our jobs to two minutes for Slurm.

::::::solution

```yaml
printshellcmds: True
jobs: 3
executor: slurm
default-resources:
  - mem_mb_per_cpu=3600
  - runtime=2
```

::::::

:::

Existen varias opciones `sbatch` no soportadas directamente por las definiciones de
recursos de la tabla anterior. Puede utilizar el recurso `slurm_extra` para especificar
cualquiera de estas opciones adicionales a `sbatch`:

```python
rule myrule:
    input: ...
    output: ...
    resources:
        slurm_extra="--mail-type=ALL --mail-user=<your email>"
```

## Ejecución local de reglas

Nuestra regla inicial era obtener el nombre de host del nodo de inicio de sesión.
Siempre queremos ejecutar esa regla en el nodo de inicio de sesión para que tenga
sentido. Si le decimos a Snakemake que ejecute todas las reglas a través del ejecutor
Slurm (que es lo que estamos haciendo a través de nuestro nuevo perfil) esto ya no
sucederá. Entonces, ¿cómo forzamos la regla para que se ejecute en el nodo de inicio de
sesión?

Bueno, en el caso de que una regla de Snakemake realice una tarea trivial, el envío de
trabajos podría ser excesivo (por ejemplo, menos de 1 minuto de tiempo de computación).
Similar a nuestro caso, sería una mejor idea hacer que estas reglas se ejecuten
localmente (es decir, donde se ejecuta el comando `snakemake`) en lugar de como un
trabajo. Snakemake le permite indicar qué reglas deben ejecutarse siempre localmente con
la palabra clave `localrules`. Vamos a definir `hostname_login` como una regla local
cerca de la parte superior de nuestro `Snakefile`.

```python
localrules: hostname_login
```

::: keypoints

- "Snakemake genera y envía sus propios scripts por lotes para su planificador."
- "Puedes almacenar parámetros de configuración por defecto en un perfil de Snakemake"
- "`localrules` define reglas que son ejecutadas localmente, y nunca enviadas a un
  cluster."

:::

