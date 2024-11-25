---
title: Marcadores de posición
teaching: 40
exercises: 30
---



::: questions


- "¿Cómo hago una regla genérica?"

:::



::: objectives


- "Mira cómo Snakemake trata algunos errores"

:::


Nuestro Snakefile tiene algunos duplicados. Por ejemplo, los nombres de los archivos de
texto se repiten en algunos lugares a lo largo de las reglas del Snakefile. Los
Snakefiles son una forma de código y, en cualquier código, la repetición puede dar lugar
a problemas (por ejemplo, renombramos un archivo de datos en una parte del Snakefile
pero olvidamos renombrarlo en otro lugar).


::: callout

## D.R.Y. (Don't Repeat Yourself) (No te repitas)

En muchos lenguajes de programación, la mayor parte de las características del lenguaje
están ahí para permitir al programador describir rutinas computacionales largas como
código corto, expresivo y bonito. Las características de Python, R o Java, como las
variables y funciones definidas por el usuario, son útiles en parte porque nos evitan
tener que escribir (o pensar) en todos los detalles una y otra vez. Este buen hábito de
escribir las cosas sólo una vez se conoce como el principio de "No te repitas" o D.R.Y.
(por sus siglas en inglés).

:::


Vamos a eliminar algunas de las repeticiones de nuestro archivo Snakefile.

## Marcadores de posición

Para hacer una regla más genérica necesitamos **marcadores de posición**. Veamos qué
aspecto tiene un marcador de posición

```python
rule hostname_remote:
    output: "hostname_remote.txt"
    input:
    shell:
        "hostname > {output}"

```

Como recordatorio, aquí está la versión anterior del último episodio:

```python
rule hostname_remote:
    output: "hostname_remote.txt"
    input:
    shell:
        "hostname > hostname_remote.txt"

```

La nueva regla ha reemplazado nombres de archivo explícitos con cosas en `{curly
brackets}`, específicamente `{output}` (pero también podría haber sido `{input}`...si
eso tuviera un valor y fuera útil).

### `{input}` y `{output}` son **marcadores de posición**

Los marcadores de posición se utilizan en la sección `shell` de una regla, y Snakemake
los reemplazará con los valores apropiados - `{input}` con el nombre completo del
archivo de entrada, y `{output}` con el nombre completo del archivo de salida - antes de
ejecutar el comando.

`{resources}` también es un marcador de posición, y podemos acceder a un elemento con
nombre del `{resources}` con la notación `{resources.runtime}` (por ejemplo).


:::keypoints


- "Las reglas de Snakemake se hacen más genéricas con marcadores de posición"
- "Los marcadores de posición en la parte shell de la regla se sustituyen por valores
  basados en los comodines elegidos"

:::


