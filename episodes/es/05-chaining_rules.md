---
title: Encadenamiento de reglas
teaching: 40
exercises: 30
---



::: questions


- "¿Cómo combino reglas en un flujo de trabajo?"
- "¿Cómo hago una regla con múltiples entradas y salidas?"

:::



::: objectives


- ""

:::


## Una cadena de múltiples reglas

Ahora tenemos una regla que puede generar una salida para cualquier valor de `-p` y
cualquier número de tareas, sólo tenemos que llamar a Snakemake con los parámetros que
queremos:

```bash
snakemake --profile cluster_profile p_0.999/runs/amdahl_run_6.json
```

Aunque eso no es exactamente conveniente, para generar un conjunto de datos completo
tenemos que ejecutar Snakemake muchas veces con diferentes objetivos de archivos de
salida. En lugar de eso, vamos a crear una regla que puede generar los archivos para
nosotros.

Encadenar reglas en Snakemake es cuestión de elegir patrones de nombres de archivos que
conecten las reglas. Hay algo de arte en ello - la mayoría de las veces hay varias
opciones que funcionarán:

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_runs.txt"
    input:  "p_{parallel_proportion}/runs/amdahl_run_6.json"
    shell:
        "echo {input} done > {output}"
```


::: challenge


La nueva regla no está haciendo ningún trabajo, sólo se está asegurando de que creamos
el archivo que queremos. No vale la pena ejecutarla en el cluster. ¿Cómo asegurarse de
que se ejecuta sólo en el nodo de inicio de sesión?


:::::: solution


Necesitamos añadir la nueva regla a nuestro `localrules`:

```python
localrules: hostname_login, generate_run_files
```


:::



:::


Ahora vamos a ejecutar la nueva regla (recuerda que tenemos que solicitar el archivo de
salida por su nombre ya que `output` en nuestra regla contiene un patrón comodín):

```bash
[ocaisa@node1 ~]$ snakemake --profile cluster_profile/ p_0.999_runs.txt
```

```output
Using profile cluster_profile/ for setting default command line arguments.
Building DAG of jobs...
Retrieving input from storage.
Using shell: /cvmfs/software.eessi.io/versions/2023.06/compat/linux/x86_64/bin/bash
Provided remote nodes: 3
Job stats:
job                   count
------------------  -------
amdahl_run                1
generate_run_files        1
total                     2

Select jobs to execute...
Execute 1 jobs...

[Tue Jan 30 17:39:29 2024]
rule amdahl_run:
    output: p_0.999/runs/amdahl_run_6.json
    jobid: 1
    reason: Missing output files: p_0.999/runs/amdahl_run_6.json
    wildcards: parallel_proportion=0.999, parallel_tasks=6
    resources: mem_mb=1000, mem_mib=954, disk_mb=1000, disk_mib=954,
               tmpdir=<TBD>, mem_mb_per_cpu=3600, runtime=2, mpi=mpiexec, tasks=6

mpiexec -n 6 amdahl --terse -p 0.999 > p_0.999/runs/amdahl_run_6.json
No SLURM account given, trying to guess.
Guessed SLURM account: def-users
Job 1 has been submitted with SLURM jobid 342 (log: /home/ocaisa/.snakemake/slurm_logs/rule_amdahl_run/342.log).
[Tue Jan 30 17:47:31 2024]
Finished job 1.
1 of 2 steps (50%) done
Select jobs to execute...
Execute 1 jobs...

[Tue Jan 30 17:47:31 2024]
localrule generate_run_files:
    input: p_0.999/runs/amdahl_run_6.json
    output: p_0.999_runs.txt
    jobid: 0
    reason: Missing output files: p_0.999_runs.txt;
            Input files updated by another job: p_0.999/runs/amdahl_run_6.json
    wildcards: parallel_proportion=0.999
    resources: mem_mb=1000, mem_mib=954, disk_mb=1000, disk_mib=954,
               tmpdir=/tmp, mem_mb_per_cpu=3600, runtime=2

echo p_0.999/runs/amdahl_run_6.json done > p_0.999_runs.txt
[Tue Jan 30 17:47:31 2024]
Finished job 0.
2 of 2 steps (100%) done
Complete log: .snakemake/log/2024-01-30T173929.781106.snakemake.log
```

Mira los mensajes de registro que Snakemake imprime en la terminal. ¿Qué ha ocurrido
aquí?

1. Snakemake busca una regla para hacer `p_0.999_runs.txt`
1. Determina que "generate_run_files" puede hacer esto si `parallel_proportion=0.999`
1. Por lo tanto, ve que la entrada necesaria es `p_0.999/runs/amdahl_run_6.json`
1. Snakemake busca una regla para hacer `p_0.999/runs/amdahl_run_6.json`
1. Determina que "amdahl_run" puede hacer esto si `parallel_proportion=0.999` y
   `parallel_tasks=6`
1. Ahora que Snakemake ha alcanzado un archivo de entrada disponible (en este caso, no
   se requiere ningún archivo de entrada), ejecuta ambos pasos para obtener la salida
   final

Así, en pocas palabras, es como construimos flujos de trabajo en Snakemake.

1. Definir reglas para todos los pasos de procesamiento
1. Elija `input` y `output` patrones de nomenclatura que permitan a Snakemake vincular
   las reglas
1. Indicar a Snakemake que genere los archivos de salida finales

Si usted está acostumbrado a escribir scripts regulares esto toma un poco de tiempo para
acostumbrarse. En lugar de enumerar los pasos en orden de ejecución, siempre se está
**trabajando hacia atrás** desde el resultado final deseado. El orden de las operaciones
viene determinado por la aplicación de las reglas de concordancia de patrones a los
nombres de archivo, no por el orden de las reglas en el archivo Snakefile.


::: callout


## ¿Salidas primero?

El enfoque de Snakemake de trabajar hacia atrás desde la salida deseada para determinar
el flujo de trabajo es la razón por la que estamos poniendo las líneas `output` primero
en todas nuestras reglas - ¡para recordarnos que esto es lo que Snakemake mira primero!

Muchos usuarios de Snakemake, y de hecho la documentación oficial, prefieren tener el
`input` primero, por lo que en la práctica se debe utilizar cualquier orden que tenga
sentido para usted.


:::



::: callout


## `log` salidas en Snakemake

Snakemake tiene un campo de regla dedicado para las salidas que son [archivos de
registro](https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html#log-files), y
estos son tratados en su mayoría como salidas regulares, excepto que los archivos de
registro no se eliminan si el trabajo produce un error. Esto significa que usted puede
mirar el registro para ayudar a diagnosticar el error. En un flujo de trabajo real esto
puede ser muy útil, pero en términos de aprendizaje de los fundamentos de Snakemake nos
quedaremos con los campos regulares `input` y `output` aquí.


:::



::: callout


## Los errores son normales

No te desanimes si ves errores cuando pruebes por primera vez tus nuevos pipelines de
Snakemake. Hay muchas cosas que pueden salir mal al escribir un nuevo flujo de trabajo,
y normalmente necesitarás varias iteraciones para hacer las cosas bien. Una ventaja del
enfoque de Snakemake en comparación con los scripts regulares es que Snakemake falla
rápidamente cuando hay un problema, en lugar de seguir adelante y potencialmente
ejecutar cálculos basura sobre datos parciales o corruptos. Otra ventaja es que cuando
un paso falla, podemos reanudarlo con seguridad desde donde lo dejamos.


:::





::: keypoints


- "Snakemake enlaza reglas buscando iterativamente reglas que tengan entradas faltantes"
- "Las reglas pueden tener múltiples entradas y/o salidas con nombre"
- "Si un comando del shell no produce una salida esperada entonces Snakemake lo
  considerará como un fallo"

:::


