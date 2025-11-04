--------
Para esto tenemos que tener claro [[Comandos, Rutas y Operadores]] al igual que algunos [[Comandos Útiles]].
# Stderr
Cuando hablamos del stderr nos referimos a los mensajes de error que obtenemos al ejecutar algunos comandos a estos mensajes de error el sistema los reconoce con el numero dos y podemos eliminarlos de la siguiente manera:

```bash
whoam 2>/dev/null
```

Tenemos que pensar en la ruta /dev/null como un agujero negro en donde todo lo que entre allí se pierde. Ya con esto los errores no se mostraran.

# Stdout

Cuando hablamos del stdout nos referimos a lo que el comando Correctamente ejecutado nos muestre por consola, a esto el sistema lo referencia con nada por lo que para poder eliminar y que el comando no se vea tenemos dos formas:

```bash
whoami > /dev/null   #Con esto solo vemos los errores en caso de producirse.
whoami > /dev/null 2>&1  #Con esto comvertimos los errores en stdout y tampoco aparece.
whoami &>/dev/null  # Con esto ya no vemos ni stdout ni el stderr.
```

# Procesos en Segundo Plano

Para entender esto tenemos que comprender que los procesos pueden tener procesos hijos y procesos padres, un ejemplo es ejecutar Firefox desde nuestra consola, Firefox se habré pero este proceso será un proceso hijo de nuestra consola, la cual seria un proceso padre.

Cuando ejecutamos Firefox retomando el ejemplo nuestra consola queda inutilizable ya que esta ejecutando Firefox desde la misma para evitar esto tenemos que usar el signo de & al final del comando y el proceso pasara a segundo plano de la siguiente manera:

```bash
firefox &>/dev/null &
```

En este caso aumente el redireccionamiento tanto del stdout como del stderr para evitar ver algo por consola y con el & hacemos que el proceso pase a segundo plano, pero esto no quiere decir que deje de ser un proceso hijo si llegamos a cerrar la consola se cerrara el Firefox ya que el proceso padre desaparece, con esto en mente para independizar los procesos vamos a agregar un comando mas al final:

```bash
firefox &>/dev/null & disown
```

Con esto nuestro proceso hijo se independiza y podemos cerrar la consola sin problema.

---

# Descriptores de Archivos

Los descriptores de archivos no son mas valores numéricos enteros los cuales son asignados a diferentes archivos y apara esto podemos hacer:

```bash
exec 3<> file
```

creamos un archivo file.

bueno a través de este descriptor de archivo podemos redirigir tanto stderr como stdout a nuestro archivo de la siguiente manera:

```bash
whoami >&3
```

Como vemos el stdout se redirigió al archivo file y esto es porque este archivo tiene como descriptor el valor 3

Bueno los archivos normalmente se encuentran en modo append lo que quiere decir que estos no se van a sobre escribir solo se insertaran líneas debajo.

Para cerrar un descriptor de archivo vamos a hacer lo siguiente:

```bash
exec 3>&-
```

Claro esto solo cierra el descriptor mas no elimina el archivo.

Esto es la base de los descriptores pero con esto podemos hacer varias cosas y vamos a dejar algunos ejemplos:

```bash
exec 5<> data # Descriptor de archivo 5 para el archivo data con capacidad de <(lectura) y >(escritura)

exec 8>&5 #Vamos a crear una copia del descriptor de archivo 5 en el 8 a la vez que lo creamos, cuando creamos copias podemos enviar información a la copia y se incorpora tambien al archivo original
```

```bash
exec 5<> files #Creamos un descriptor de archivo 5 par el archivo file

exec 6>&5- #Con esto creamos la copia del escritor de archivo 5 que seria el 6 y por ultimo cerramos el descriptor de archivo 5.
```