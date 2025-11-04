----------------------------

## PortSwigger

### Lab: Information disclosure in error messages

El propósito es acontecer un error que nos permita filtrar por información.

![[{89BB39D5-7691-4A6F-86B0-D5F8AB2CF1FC}.png]]

vamos a ver que podemos controlar los id de los productos, podemos intentar cambiar los valores numéricos por texto a ver si ocurren errores que filtren información:

![[{781AB5B1-489A-4D73-99D7-3A7D14154138}.png]]

vemos que si, con esto vamos a pasarle la version y veamos si se resuelve:

![[{DB119A49-F169-430A-B4CF-540383A5DC58}.png]]

Lab Terminado.


### Lab: Information disclosure on debug page

En este laboratorio vamos a tener una pagina de debug que filtra información, el propósito es encontrar la Secret_KEY.

![[{BA410643-FA5D-4C0B-BBDB-E8032F104595}.png]]

en el código fuente de la web vemos una ruta donde parece ser el `phpinfo` vamos a ver:

![[{8C423E80-F934-47C3-A751-7A9219B5460D}.png]]

Listo con esto podemos enviar la respuesta:

![[{4ED99445-2343-4C99-AC33-1585CB5916A1}.png]]

 Lab Terminado.


### Lab: Source code disclosure via backup files

al parecer vamos a encontrar archivos backup mediante los cuales filtra información.


Bueno para esto vamos a pasar las peticiones por el BurpSuite y vamos a ver mediante el `Site Map` que podemos encontrar:

![[{FAC31A64-AD02-4913-B732-69BEB9782672}.png]]

no encontramos nada por ahora y lo que podemos hacer es buscar por rutas conocidas como `robots.txt`:

![[{E3E1A7B3-4F4F-4AAB-86B5-703950141C5D}.png]]

al parecer contamos con la ruta `/backup`:

![[{A4392F45-3527-499C-8C61-F5973AD0AF19}.png]]

perfecto encontramos el archivo y podemos ahora si pasarle la contraseña que nos pide:

![[{5CD3F0DA-CE47-4CA2-B36D-1E14DA1C0217}.png]]

Lab Terminado.


### Lab: Authentication bypass via information disclosure

El objetivo es obtener el control de admin comenzando como wiener y luego eliminar el usuario `carlos`.

Vamos a ver como se esta tramitando primero al hacer login:

![[{AD7ED10E-3DDF-46A5-9480-09B20C17D97E}.png]]

No vamos a encontrar nada interesante pero podemos intentar ver que sucede o como se tramita el momento en el que tratamos de ingresar a la interfaz administrativa.

![[{5AE0F7D6-3D7B-401A-BCAF-C082196EC173}.png]]

bueno lo que podemos hacer es utilizar ciertas cabeceras para intentar burlar la protección haciendo que la maquina crea que las peticiones vienen del mismo servidor, pero no funciona ninguna conocida y por fuerza bruta al ser cabeceras extrañas demoraremos mucho, pero podemos intentar cambiando por diferentes métodos como POST, HEAD y en este caso uno nuevo que es `TRACE` el cual es un método usando solo en depuraciones y pruebas que devuelve una copia de la web solicitada incluyendo cabeceras, vamos a intentarlo:

![[{3CCA8A7C-5EFF-47CD-BE91-58D525570D66}.png]]

vemos que si a funcionado y vemos muchas mas cabeceras, donde podemos ver que destaca la ultima `X-Custom_IP-Authorization` lo que parece ser que si enviamos esto talvez intentando con la IP local nos permita pasar:

![[{2A1B5854-D2DC-428C-B6E7-8DEC840E50CD}.png]]\

Con esta cabecera ya logramos pasar, en este punto lo que vamos a hacer es completar el lab eliminado el usuario  `Carlos`.

![[{D028E237-30C7-442F-A3D2-3E8E3261085B}.png]]

Lab Terminado.


### Lab: Information disclosure in version control history

Al parecer tenemos que intentar conseguir la contraseña del usuario `adminstrator` mediante el historial de control de versiones.


En este punto vamos a intentar un pequeño ataque de fuerza bruta con ayuda de [[Nmap]] para encontrar rutas:

![[{16EE65DB-0000-4085-A1AF-3EA945379B46}.png]]

no encuentra nada por lo que podemos intentar ahora con [[Gobuster]]:

![[{EF3B21C6-7B19-42F1-A65D-0AC08B38D8C7}.png]]

vemos que encuentra un directorio `/.git`, recordemos que cuando construimos un proyecto usual mente se usa un software de control de versiones como git y este para su gestión crea el `.git` donde almacena todo, lo que vamos a hacer es primero descargarnos todo el proyecto dentro de `.git`:

![[{27C7BABC-D2C6-460B-AF78-A5B6DCA1608C}.png]]

esto con `wget -r y la ruta`

ahora vamos a entrar y vamos a hacer un reconocimiento de los commits:

![[{105B7E9B-4E5E-492C-9834-82273AE369B8}.png]]

vemos que tenemos un commit que nos dice que se elimino la contraseña de la configuración, esto quiere decir que en el commit anterior se tenia, vamos a regresar al anterior a ver que encontramos:

![[{83B56613-C13F-41A8-A5B3-C68E9AA37FF8}.png]]

ya tenemos la contraseña del admin, vamos ahora si a ver si completamos el lab:

![[{71FA9D5F-7A15-4BDC-8B64-613FCA83E67A}.png]]

Lab Terminado.

