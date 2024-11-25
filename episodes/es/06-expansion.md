---
title: Procesamiento de listas de entradas
teaching: 50
exercises: 30
---



::: questions


- "¿Cómo puedo procesar varios archivos a la vez?"
- "¿Cómo combino varios archivos?"

:::



::: objectives


- "Usa Snakemake para procesar todas nuestras muestras a la vez"
- "Haz un gráfico de escalabilidad que agrupe nuestros resultados"

:::


Hemos creado una regla que puede generar un único archivo de salida, pero no vamos a
crear varias reglas para cada archivo de salida. Queremos generar todos los archivos de
ejecución con una sola regla si pudiéramos, bueno Snakemake puede efectivamente tomar
una lista de archivos de entrada:

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_runs.txt"
    input:  "p_{parallel_proportion}/runs/amdahl_run_2.json", "p_{parallel_proportion}/runs/amdahl_run_6.json"
    shell:
        "echo {input} done > {output}"
```

Eso está muy bien, pero no queremos tener que listar todos los archivos que nos
interesan individualmente. ¿Cómo podemos hacerlo?

## Definición de una lista de muestras a procesar

Para ello, podemos definir algunas listas como **variables globales** de Snakemake.

Las variables globales deben añadirse antes de las reglas en el Snakefile.

```python
# Task sizes we wish to run
NTASK_SIZES = [1, 2, 3, 4, 5]
```

- A diferencia de lo que ocurre con las variables en los scripts de shell, podemos poner
  espacios alrededor del signo `=`, pero no son obligatorios.
- Las listas de cadenas entrecomilladas van entre corchetes y separadas por comas. Si
  sabes algo de Python reconocerás esto como sintaxis de listas de Python.
- Una buena convención es utilizar nombres en mayúsculas para estas variables, pero no
  es obligatorio.
- Aunque éstas se denominan variables, en realidad no se pueden cambiar los valores una
  vez que el flujo de trabajo se está ejecutando, por lo que las listas definidas de
  esta manera son más como constantes.

## Usando una regla de Snakemake para definir un lote de salidas

Ahora vamos a actualizar nuestro Snakefile para aprovechar la nueva variable global y
crear una lista de archivos:

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_runs.txt"
    input:  expand("p_{{parallel_proportion}}/runs/amdahl_run_{count}.json", count=NTASK_SIZES)
    shell:
        "echo {input} done > {output}"
```

La función `expand(...)` de esta regla genera una lista de nombres de archivo, tomando
como plantilla lo primero que aparece entre paréntesis y sustituyendo `{count}` por
todos los `NTASK_SIZES`. Dado que hay 5 elementos en la lista, esto producirá 5 archivos
que queremos hacer. Tenga en cuenta que tuvimos que proteger nuestro comodín en un
segundo conjunto de paréntesis para que no fuera interpretado como algo que necesitaba
ser expandido.

En nuestro caso actual seguimos dependiendo del nombre de fichero para definir el valor
del comodín `parallel_proportion`, por lo que no podemos llamar a la regla directamente,
seguimos necesitando solicitar un fichero específico:

```bash
snakemake --profile cluster_profile/ p_0.999_runs.txt
```

Si no especifica un nombre de regla de destino o cualquier nombre de archivo en la línea
de comandos cuando se ejecuta Snakemake, el valor predeterminado es utilizar ** la
primera regla ** en el Snakefile como el objetivo.


::: callout

## Reglas como destinos

Dar el nombre de una regla a Snakemake en la línea de comandos sólo funciona cuando esa
regla tiene *sin comodines* en las salidas, porque Snakemake no tiene manera de saber
cuáles podrían ser los comodines deseados. Verá el error "Las reglas de destino no
pueden contener comodines" Esto también puede ocurrir cuando no se proporciona ningún
objetivo explícito en la línea de comandos, y Snakemake intenta ejecutar la primera
regla definida en el archivo Snakefile.


:::


## Reglas que combinan varias entradas

Nuestra regla `generate_run_files` es una regla que toma una lista de archivos de
entrada. La longitud de esa lista no está fijada por la regla, sino que puede cambiar en
función de `NTASK_SIZES`.

En nuestro flujo de trabajo el paso final es tomar todos los archivos generados y
combinarlos en un gráfico. Para ello, es posible que haya oído que algunas personas
utilizan una biblioteca de Python llamada `matplotlib`. Está más allá del alcance de
este tutorial escribir el script de python para crear un gráfico final, así que te
proporcionamos el script como parte de esta lección. Puede descargarlo con

```bash
curl -O https://ocaisa.github.io/hpc-workflows/files/plot_terse_amdahl_results.py
```

El script `plot_terse_amdahl_results.py` necesita una línea de comandos parecida a:

```bash
python plot_terse_amdahl_results.py --output <output image filename> <1st input file> <2nd input file> ...
```

Introduzcámoslo en nuestra regla `generate_run_files`:

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_runs.txt"
    input:  expand("p_{{parallel_proportion}}/runs/amdahl_run_{count}.json", count=NTASK_SIZES)
    shell:
        "python plot_terse_amdahl_results.py --output {output} {input}"
```


::: challenge


Este script depende de `matplotlib`, ¿está disponible como módulo de entorno? Añada este
requisito a nuestra regla.


:::::: solution


```python
rule generate_run_files:
    output: "p_{parallel_proportion}_scalability.jpg"
    input:  expand("p_{{parallel_proportion}}/runs/amdahl_run_{count}.json", count=NTASK_SIZES)
    envmodules:
      "matplotlib"
    shell:
        "python plot_terse_amdahl_results.py --output {output} {input}"
```


::::::



:::


¡Por fin podemos generar un gráfico de escala! Ejecute el último comando de Snakemake:

```bash
snakemake --profile cluster_profile/ p_0.999_scalability.jpg
```


::: challenge


Genera el gráfico de escalabilidad para todos los valores de 1 a 10 núcleos.


:::::: solution


```python
NTASK_SIZES = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```


::::::



:::



::: challenge


Vuelva a ejecutar el flujo de trabajo para un valor `p` de 0.8


:::::: solution


```bash
snakemake --profile cluster_profile/ p_0.8_scalability.jpg
```


::::::



:::



::: challenge


## Ronda de bonificación

Cree una regla final que pueda llamarse directamente y genere un gráfico de escala para
3 valores diferentes de `p`.

:::



::: keypoints


- "Utilice la función `expand()` para generar listas de nombres de archivo que desee
  combinar"
- "Cualquier `{input}` de una regla puede ser una lista de longitud variable"

:::


