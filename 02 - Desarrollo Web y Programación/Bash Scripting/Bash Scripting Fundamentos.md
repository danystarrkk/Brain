---
aliases:
  - Bash Scripting
  - Scripting en Bash
  - Fundamentos Bash
tags:
  - scripting/bash
estado: 🟢 Terminado
---
## Estructura Básica de un Script

```bash
#!/bin/bash

function ctrl_c (){
 echo -e "\n\n[!] Saliendo....\n"
}

# CTRL + C
trap ctrl_c INT
```

> [!info] Manejo de Señales (`trap`)
> El comando `trap` se utiliza para ejecutar un comando o función cuando se recibe una señal específica. `INT` (o `SIGINT`) es la señal que se envía a un proceso cuando el usuario presiona `Ctrl+C`. Al usar `trap ctrl_c INT`, estamos indicando que cuando se reciba la señal `INT`, se debe ejecutar la función `ctrl_c`, permitiendo una salida limpia y controlada del script.

## Iterando sobre un Output

Esto se utiliza cuando se tiene varias líneas en la salida de un comando y se desea iterar sobre cada una de ellas.

```bash
| while read line; do echo "Hola $line "; done
```

> [!tip] Procesamiento de Líneas con `while read`
> El patrón `command | while read line; do ...; done` es fundamental para procesar la salida línea por línea de otro comando. `read` lee una línea de la entrada estándar (en este caso, la salida del comando anterior a través de la tubería `|`) y la asigna a la variable `line`. Este método es más robusto que usar `for line in $(command)` para manejar nombres de archivo con espacios o caracteres especiales, ya que evita problemas de expansión de palabras.

Esta es la base sobre la que normalmente se construirán los scripts.

## Variables

Definir variables en Bash no es complicado, pero se deben tener en cuenta las diferentes formas de hacerlo:

```bash
variable="Mi primer variable" # Esto es una variable de cualquier tipo, ya que no se define explícitamente.

# Para mostrar el valor de una variable:
echo "$variable"

declare -r noCambia="Hola" # Esta variable es de solo lectura, por lo que no se puede cambiar después de su inicialización.

# Tipo de variable entero
declare -i variableEntero=10 # Solo almacena valores enteros.

# Se pueden declarar arreglos (arrays)
declare -a myArray=(1 2 3 4 5)
echo ${#myArray[@]} # Muestra el total de elementos en el array.
echo ${myArray[@]} # Lista todos los elementos del array.
echo ${myArray[0]} # Muestra el valor del elemento en la posición 0.
echo ${myArray[-1]} # Muestra el último elemento del array (requiere Bash 4+).

myArray+=(5) # Agrega un elemento al final del array.

unset myArray[0] # Permite eliminar elementos del array por su índice.

# En ocasiones, al quitar elementos de un array, se pueden generar "huecos" en los índices. Para actualizar y reindexar el array:
myArray=(${myArray[@]})

# Iterar en uno (operación aritmética)
let posicion+=1
```

> [!info] Acceso y Manipulación de Arrays en Bash
> - `${#myArray[@]}` o `${#myArray[*]}`: Devuelve el número total de elementos en el array.
> - `${myArray[@]}` o `${myArray[*]}`: Expande a todos los elementos del array, cada uno como una palabra separada. Es crucial usar comillas dobles (`"${myArray[@]}"`) para preservar los espacios en los elementos si los hubiera.
> - `${myArray[indice]}`: Accede al elemento en la posición `indice`. Los índices comienzan en 0.
> - `myArray+=(elemento)`: Añade un nuevo elemento al final del array.
> - `unset myArray[indice]`: Elimina un elemento específico del array.

> [!warning] Reindexación de Arrays en Bash
> Cuando se elimina un elemento de un array indexado con `unset`, el índice de ese elemento se elimina, pero los índices de los elementos posteriores no se ajustan automáticamente. Para "reindexar" un array y eliminar los huecos, se puede reasignar el array a sí mismo usando `myArray=(${myArray[@]})`. Esto crea un nuevo array con los elementos restantes y los índices consecutivos, comenzando desde 0.

## Colores

Se pueden usar varios colores en la terminal mediante códigos de escape ANSI:

```bash
####### Colores ########
greenColour="\e[0;32m\033[1m"
endColour="\033[0m\e[0m"
redColour="\e[0;31m\033[1m"
blueColour="\e[0;34m\033[1m"
yellowColour="\e[0;33m\033[1m"
purpleColour="\e[0;35m\033[1m"
turquoiseColour="\e[0;36m\033[1m"
grayColour="\e[0;37m\033[1m"

## Extra
# Para ocultar o recuperar el cursor en la terminal:
tput civis # Oculta el cursor.
tput norm # Recupera el cursor.
```

> [!info] Códigos de Escape ANSI para Colores y Formato
> Los códigos de escape ANSI son secuencias de caracteres especiales que permiten controlar el formato del texto en terminales compatibles. Por ejemplo:
> - `\e[0;32m`: Establece el color de primer plano a verde (0;32m es para color normal, 1m para negrita).
> - `\033[1m`: Establece el texto en negrita.
> - `\033[0m`: Restablece todos los atributos a sus valores predeterminados (fin de color/formato).
> `tput` es una utilidad que permite manipular las capacidades del terminal de forma portable, como ocultar (`civis`) o mostrar (`norm`) el cursor, lo que es útil para mejorar la experiencia del usuario en scripts interactivos.

## Manejo de Argumentos (`getopts`)

La sintaxis básica para la creación de un panel de opciones (manejo de argumentos) es la siguiente:

```bash
# Indicadores
declare -i parametercounter=0  # Declara un contador para reconocer los parámetros recibidos.

###### Menú ######
while getopts "m:h" arg; do # 'm' y 'h' son parámetros. 'm' al requerir un argumento extra, lleva dos puntos.
    case $arg in
    m)
        machineName=$OPTARG         # Gracias a la variable OPTARG, se obtiene el valor que se le pasa a la opción 'm'.
        let parametercounter+=1     # Aumenta en uno el valor del contador de parámetros.
        ;;
    h) helpPanel ;;
    esac
done

# Ejecutando parámetro
if [ $parametercounter -eq 1 ]; then # Define condicionales para ejecutar las diferentes funciones del script.
    searchMachine $machineName
else
    helpPanel
fi

###### En el caso de querer tener más de un comando, se usan "flags" o "chivatos" de la siguiente manera: #####
declare -i parametercounter=0  # Declara un contador para reconocer los parámetros.

### Flags/Chivatos ####
declare -i chivato=0 # Inicializa un flag.

###### Menú ######
while getopts "m:h" arg; do # 'm' y 'h' son parámetros. 'm' al requerir argumentos extras, lleva dos puntos.
    case $arg in
    m)
        machineName=$OPTARG;         # Gracias a la variable OPTARG, se obtiene el valor que se le pasa a la opción 'm'.
        let parametercounter+=1;     # Aumenta en uno el valor del contador de parámetros.
        chivato=1
        ;;
    h) helpPanel; chivato=1 ;;
    esac
done

# Ejecutando parámetro
if [ $parametercounter -eq 1 ]; then # Define condicionales para ejecutar las diferentes funciones del script.
    searchMachine $machineName
elif [ $chivato -eq 1 ] && [ $chivato -eq 1 ] # Esta condición parece redundante, se debería usar para dos flags diferentes.
	funcionparabuscarpordosparametros $parametro1 $parametro2
else
    helpPanel
fi
```

> [!info] `getopts` para Parsing de Argumentos
> `getopts` es una utilidad integrada de Bash para parsear opciones de línea de comandos de manera estándar y eficiente. Es ideal para scripts que necesitan aceptar argumentos con formato `-o` o `-o argumento`.
> - La cadena de opciones (ej. `"m:h"`) define las opciones válidas. Un colon (`:`) después de una letra (ej. `m:`) indica que esa opción requiere un argumento.
> - `OPTARG` es una variable especial que contiene el argumento de la opción actual.
> - `OPTIND` es otra variable especial que contiene el índice del siguiente argumento a procesar.
> - El bloque `case $arg in ... esac` maneja la lógica para cada opción. Este método es generalmente preferible a `getopt` (con una 't' al final) para scripts simples, ya que `getopt` es un comando externo y tiene una sintaxis ligeramente diferente.

## Condicionales

Existen dos tipos principales de condicionales en Bash: `if` y `case`.

```bash
if [ condicion ]; then ## Si se quiere comprobar la existencia de un archivo, se usa -f seguido por la ruta: [ -f data.txt ]
	# Código que se ejecuta si la condición se cumple.
else
	# Código que se ejecuta si la condición no se cumple.
fi

### Para 'case', se necesita una variable:

variable=1
case $variable in
	1) # Código si 'variable' es 1 ;;
	2) # Código si 'variable' es 2 ;;
	3) # Código si 'variable' es 3 ;;
esac
```

> [!tip] Operadores de Prueba Comunes en Condicionales Bash
> Además de los operadores numéricos (`-eq`, `-lt`, etc.), Bash ofrece otros operadores para pruebas de archivos y cadenas dentro de `[ ]` o `[[ ]]`:
> - `[ -f archivo ]`: Verdadero si `archivo` existe y es un archivo regular.
> - `[ -d directorio ]`: Verdadero si `directorio` existe y es un directorio.
> - `[ -z "$cadena" ]`: Verdadero si `cadena` está vacía (longitud cero).
> - `[ -n "$cadena" ]`: Verdadero si `cadena` no está vacía.
> - `[ "$cadena1" = "$cadena2" ]`: Verdadero si las cadenas son iguales.
> - `[ "$cadena1" != "$cadena2" ]`: Verdadero si las cadenas son diferentes.
> Para expresiones más complejas, se pueden usar `[[ ... ]]` (que permite `&&`, `||`, `==` para patrones) o `(( ... ))` para aritmética. Es importante recordar que `[ ]` es un comando externo (`test`), mientras que `[[ ]]` es una palabra clave de Bash que ofrece más funcionalidades y es más segura para manejar cadenas con espacios.

## Recibir Valores en Bash

Para solicitar valores al usuario por Bash, se puede hacer de la siguiente manera:

```bash
echo -n "Digitar su número de cédula: " && read numeroCedula # Se usa '-n' para evitar un salto de línea, y luego se concatena 'read' con la variable donde se almacenará la información.
```

> [!tip] Entrada de Usuario con `read`
> El comando `read` se utiliza para leer una línea de la entrada estándar y asignarla a una o más variables.
> - `read variable`: Lee la entrada y la almacena en `variable`.
> - `echo -n "Prompt: " && read variable`: Muestra un prompt sin nueva línea y luego lee la entrada.
> - `read -p "Prompt: " variable`: Es una forma más concisa y común de mostrar un prompt y leer la entrada en una sola operación, ya que `-p` permite especificar el prompt directamente.

## Números Aleatorios

Se pueden generar números aleatorios a través de un módulo ya existente en Bash:

```bash
echo $(($RANDOM % 37)) # Los números generados irán del 0 al 36. Para cambiar el rango, se modifica el 37, teniendo en cuenta que el resultado siempre será uno menos que el valor del módulo.
```

> [!info] Generación de Números Aleatorios con `$RANDOM`
> `$RANDOM` es una variable especial de Bash que devuelve un entero pseudoaleatorio entre 0 y 32767 cada vez que se referencia. Para generar un número aleatorio dentro de un rango específico (por ejemplo, de 1 a N), se usa el operador módulo (`%`).
> - `$(($RANDOM % N))` generará un número entre 0 y N-1.
> - Si se necesita un rango de 1 a N, se puede usar `$(($RANDOM % N + 1))`.

## Validaciones

Para realizar validaciones, se hace uso de condicionales a través de los cuales se verifica el contenido de una variable. Si esta está vacía, se ejecutará cierto código; de lo contrario, se ejecutará otro.

```bash
variable="cat /etc/passwd"
if [ $variable ]; then # Se lee: si 'variable' tiene contenido, entonces ejecuta 'código1'; si no, ejecuta 'código2'.
	código1
else
	código2
fi
```

> [!warning] Validación de Variables en Bash
> La expresión `if [ $variable ]` verifica si la variable `$variable` no está vacía. Si la variable está vacía, la expresión se evalúa como falsa. Es una forma concisa de verificar si una variable tiene algún contenido. Para mayor claridad y robustez, especialmente si la variable puede contener espacios o caracteres especiales, es recomendable usar comillas dobles y operadores específicos:
> - `if [ -n "$variable" ]`: Verdadero si la variable no está vacía.
> - `if [ -z "$variable" ]`: Verdadero si la variable está vacía.

## Operadores

| Operador | Descripción                                            |
| -------- | ------------------------------------------------------ |
| -eq      | Es el equivalente al “==” pero se usa para números.    |
| -ne      | Es el equivalente al “!=” pero se usa para números.    |
| -le      | Es el equivalente al “≤” pero se usa para números.     |
| -ge      | Es el equivalente al “≥” pero se usa para números.     |
| -gt      | Es el equivalente al “>” pero se usa para números.     |
| -lt      | Es el equivalente al “<” pero se usa para números.     |

> [!info] Operadores de Comparación Numérica y Lógica
> Los operadores `-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge` se usan para comparar valores numéricos dentro de `[ ]` o `[[ ]]`.
> Para operaciones lógicas dentro de `[ ]`:
> - `-a`: AND lógico
> - `-o`: OR lógico
> Para operaciones lógicas dentro de `[[ ]]` o `(( ))`:
> - `&&`: AND lógico
> - `||`: OR lógico
> - `!`: NOT lógico
> Es importante recordar que `[ ]` es un comando externo (`test`), mientras que `[[ ]]` es una palabra clave de Bash que ofrece más funcionalidades y es más segura para manejar cadenas con espacios. `(( ))` se utiliza para aritmética entera.