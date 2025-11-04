------
Para comenzar con la creación de scripts tenemos que tener claro los  [[Comandos Útiles]], [[Comandos, Rutas y Operadores]] y tener claro el [[Uso de Vim]]
Estructura básica de un script:

```bash
#!/bin/bash

function ctrl_c (){
 echo -e "\\n\\n[!] Saliendo....\\n" 
}

#CTRL + C
trap ctrl_c INT
```

## Iterando sobre un output
Esto lo usamos cuando tenemos varias cosas en nuestro output y queremos iterar sobre las mismas.

```bash
| while read line; do echo "Hola $line "; done
```

Esta es la base en la que normalmente crearemos nuestros scripts.

## Variables
El definir variables en Bash no es algo complicado pero que debemos tener en cuenta ya que tenemos varias formas:

```bash
varialble="Mi primer variable" -> esto es una variabla de cualquier tipo ya que no la definimos.

#Para definir una variable lo hacemos de la siguiente manera:
echo "$variable"

declare -r noCambia="Hola" # esta variable es de solo lecturar por lo que no podemos cambiarla despues.

# Tipo de variable entero
declare -i variableEntero=10 # Solo almacena valores enteros.

# Podemos declarar arreglos
declare -a myArray=(1 2 3 4 5)
echo ${#myArray[@]} #Vemos el total de elementos
echo ${myArray[@]} #lista todos los elementos del array\\
echo ${myArray[0]} #Mostramos el valor de la posición
echo ${myArray[-1]}  #Mostramos el ultimo elemento del arraglo

myArray+=(5) # agregamos un elemento al array, en este caso agregamos el 5

unset myArray[0] #Nos permite eleminar elementos del array.

#En ocaciones cuando quitamos elementos del array entramos en conflicto por lo que vamos a actualizar el array
# De la siguiente manera:

myArray=(${myArray[@]})

# Iterar en uno
let posicion+=1
```

## Colores
tenemos varios colores los cuales podemos usar y son los siguientes:

```bash
####### Colores ########
greenColour="\\e[0;32m\\033[1m"
endColour="\\033[0m\\e[0m"
redColour="\\e[0;31m\\033[1m"
blueColour="\\e[0;34m\\033[1m"
yellowColour="\\e[0;33m\\033[1m"
purpleColour="\\e[0;35m\\033[1m"
turquoiseColour="\\e[0;36m\\033[1m"
grayColour="\\e[0;37m\\033[1m"

## Extra
# Tenemos una forma de ocultar el cursor y esto lo podemos hacer con:
tput civis # Ocultamos
tput norm # Los recuperamos
```

## Panel
la sintaxis básica para la creación de una panel es la siguiente:

```bash
# Indicadores
declare -i parametercounter=0  # Declaramos un indicador para reconocer a los parametros

###### Menu ######
while getopts "m:h" arg; do # Tanto m y h son parametros pero m al recivir argumentos extras lleva dos puntos.
    case $arg in
    m)
        machineName=$OPTARG         # Gracias a la variable OPTARG podemos obtener el valor que se le pasa a la variable.
        let parametercounter+=1     # Aumentamos en uno el valor de parameter counter.
        ;;
    h) helpPanel ;;
    esac
done

# Ejecutando parametro
if [ $parametercounter -eq 1 ]; then # Definimos condicionales para ejecutar las diferentes funciones del panel
    searchMachine $machineName
else
    helpPanel
fi

###### En el caso de querer tener mas de un comando se usan chivatos de la siguiente manera #####
declare -i parametercounter=0  # Declaramos un indicador para reconocer a los parametros

### Chivatos ####
declare -i chivato=1

###### Menu ######
while getopts "m:h" arg; do # Tanto m y h son parametros pero m al recivir argumentos extras lleva dos puntos.
    case $arg in
    m)
        machineName=$OPTARG;         # Gracias a la variable OPTARG podemos obtener el valor que se le pasa a la variable.
        let parametercounter+=1;     # Aumentamos en uno el valor de parameter counter.
        chivato=1
        ;;
    h) helpPanel; chivato=1 ;;
    esac
done

# Ejecutando parametro
if [ $parametercounter -eq 1 ]; then # Definimos condicionales para ejecutar las diferentes funciones del panel
    searchMachine $machineName
elif [ $chivato -eq 1 ] && [ $chivato -eq 1]
	funcionparabuscarpordosparametros $parametro1 $parametro2
else
    helpPanel
fi
```

## Condicionales
Tenemos dos tipos de condiciónales que son el if el case:

```bash
if [ condicion ]; then ## Si queremos comprobar por la existencia de un archivo escribimos -f seguido por la ruta [ -f data.txt ]
	lo que se ejecuta si se cumple
else
	lo que se ejecuta si no se cumple
fi
### Para case vamos a nesesitar de una variable

variable=1
case $variable in
	1);;  #En el caso de variable ser 1
	2);;  #En el caso de variable ser 2
	3);;  #En el caso de varible ser 3
esac
```

## Recibir valores en bash
Vamos a ver cual es la manera correcta para poder pedir al usuario valores por bash, esto lo vamos a hacer de la siguiente manera:

```bash
echo -n "Digitar su numero de cedual" && read numeroCedula # Como vemos aumentamos un -n y luego concatenamos un read y la variabla en la que se almacenara la información que se digite.
```

## Números random
Podemos generar números random a través de un modulo ya en bash:

```bash
echo $(($RANDOM % 37)) # Los numeros van del 1 al 36, para cambiar el rango cambiamos el 37, tomando encuenta que siempre llega a uno menos.
```

## Validaciones
Para realizar validaciones vamos a hacer uso de condicionales a través de los cuales verificaremos el contenido de una variable y si esta esta vacía ejecutara cierto código:

```bash
variable="cat /etc/passwd"
if [ $variable ]; then # se lee: si variable tiene contenido, entonces ejecuta código1 y si no ejecuta código2
	código1
else
	código2
fi
```

## Operadores

| Operador | Descripción                                            |
| -------- | ------------------------------------------------------ |
| -eq      | es el equivalente al “==” pero lo usamos para números. |
| -le      | es el equivalente al “≤” pero lo usamos para números.  |
| -ge      | es el equivalente al “≥” pero lo usamos para números.  |
| -gt      | es el equivalente al “>” pero lo usamos para números.  |
| -lt      | es el equivalente al “<” pero lo usamos para números.  |
|          |                                                        |
|          |                                                        |
|          |                                                        |