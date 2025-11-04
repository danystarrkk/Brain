-------
# Búsquedas a nivel de sistema

Muchas de las veces nosotros vamos a querer filtrar o buscar archivos con ciertas características descriptivas, como permisos especiales y mas.
Para nosotros poder hacer este tipo de búsquedas uno de los mejores comandos va a ser find, el uso es sencillo y vamos a poner algunos ejemplos explicados para comprender mejor y para esto vamos a usar [[Comandos, Rutas y Operadores]] y [[Comandos Útiles]] con los cuales vamos a poder filtrar y obtener archivos con [[Permisos, Privilegios y Atributos]].

## Ejemplos:

```bash
find / -name passwd 2>/dev/null
```

El comando de manera general esta buscando desde la raíz todos los archivos que tengan el nombre passwd, a demás al no poder acceder a ciertas rutas lo que hacemos es redirigir el stderr al /dev/null y evitar el que se vea en pantalla.

```bash
find / -name passwd 2>/dev/null | xargs ls -l
```

Lo único que cambia es el uso del comando en paralelo de ls -l

```bash
find / -group stark -type d 2>/dev/null 
```

Nos permite buscar por todos los archivos que pertenezcan al grupo stark y sean de tipo directorio, si queremos que sean archivos vamos a usar la letra f(file).

```bash
find / -user root -writable 2>/dev/null
```

Buscamos por archivos y carpetas que pertenezcan a root y que tengan permisos de escritura.

```bash
find / -name dex\\* 2>/dev/null 
```

lo que estamos buscando es por algo que comience por dex y lo que sea.

```bash
find / -name \\*ex\\* 2>/dev/null
```

En este caso le decimos que queremos los archivos que comiencen por lo que se contengan ex y terminen en lo que sea.

Bueno una cosa a tener en cuenta que es find puede hacer millones de cosas por lo que siempre es bueno tener un manual sobre el comando como el siguiente:

[Cómo usar los comandos find y locate en Linux](https://www.hostinger.es/tutoriales/como-usar-comando-find-locate-en-linux/)
# Expresiones regulares

## Grep

|Parámetro|Descripción|
|---|---|
|-r|Nos permite hacer una búsqueda recursiva|
|“\w”|Nos permite filtrar por palabras|
|-v|Nos permite quitar output|
|-vE|Nos permite quitar múltiples cosas|
|-n|Nos permite ver el numero de linea|
|-i|Nos permite ser indiferente de mayúsculas y minúsculas.|
## Tail and head

|Parámetro|Descripción|
|---|---|
|-n <numero>|Nos permite filtrar por el numero de líneas comenzando por el final siendo la ultima la linea 1.|
|||
|||
|||
|||
|||
## Cut

|Parámetro|Descripción|
|---|---|
|-d “valor”|definimos un delimitador|
|-f <numero>|Nos permite filtrar por un argumento(palabra)|
|||
|||
|||
|||
## Awk

|Parámetro|Descripción|
|---|---|
|‘{print $<valor>}|Nos permite filtrar por el numero de palabra|
|FS=”<valor>”|Nos permite definir un delimitador|
|‘NF{print $NF}’|Ultimo argumento(palabra).|
|NR==<numero>|Nos permite filtrar por lineas|
|||
|||
## Tr

|Parámetro|Descripción|
|---|---|
|‘algo’ ‘cambio’|Nos permite cambiar o sustituir algo por un cambio.|
|||
|||
|||
|||
|||
## Find

| Parámetro   | Descripción                                                                        |
| ----------- | ---------------------------------------------------------------------------------- |
| -type       | Nos permite buscar por tipo de archivo como: f, d, b.                              |
| -name       | Nos permite buscar por nombre                                                      |
| -readable   | busca por archivos que se puedan leer                                              |
| -writable   | busca por archivos que se puedan escribir                                          |
| -executable | buscar por archivos que se puedan ejecutar.                                        |
| -size       | busca por archivos con un peso de lo que sea, pero se especifica el tipo: byte(c), |
| -user       | busca por usuario asociados                                                        |
| -group      | busca por grupos asociados                                                         |
|             |                                                                                    |
|             |                                                                                    |
|             |                                                                                    |
## Sed
|                            |                                        |
| -------------------------- | -------------------------------------- |
| ‘s/palabra/palabranueva/’  | cambia solo en la primer coincidencia. |
| ‘s/palabra/palabranueva/g’ | cambia en todas las coincidencias.     |
