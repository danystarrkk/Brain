# Enumeración

Comenzamos con la enumeración general, sobre todos los puertos primero detectando los que se encuentran abiertos con ayuda de `nmap`:
```bash
nmap -p- --open -sS --min-rate 5000 -v -n -Pn 10.67.159.182 -oG allPorts
```

![[Pasted image 20261107203745.png]]
El resultado nos muestra que tenemos los puertos `22, 8080, 9100` abiertos, por lo tanto, vamos a realizar un escaneo específico sobre estos puertos de la siguiente manera:

```bash
nmap -p22,8080,9100 -sVC 10.67.159.182 -oN target
```

![[Pasted image 20261107204422.png]]
Excelente, ahora tenemos no solo los puertos y servicios, sino también tenemos sus versiones que pueden ser de ayuda.

En este punto vamos a comenzar a investigar un poco más sobre los puertos. Comenzaré analizando lo que es el servicio de `HTTP` por el puerto `8080` ingresando directamente desde el navegador.

![[Pasted image 20261107204835.png]]

Observamos que, como se nos reportó en el escaneo en verdad es Python aunque no podemos asegurar del todo su versión podemos asegurar que la tecnología sí lo es, por lo tanto, podemos sacar una lista de posibles frameworks usados como lo son Flask o Django pero no podemos asegurar nada. Además, la web nos revela mucha información que concuerda con la descripción que se nos proporcionó y observamos un mensaje que nos habla de puertas remotas mediante el servicio CEN/XFS el cual es un estándar internacional de software utilizado en los cajeros esto porque antes los bancos fabricaban software sobre marcas específicas impidiendo el funcionamiento en otras marcas, para esto se desarrolló CEN/XFS que funciona como un traductor de software a hardware para las diferentes marcas de cajero traduciendo instrucciones de forma específica para la marca del cajero.

Ahora verificamos su código fuente desde el navegador:

![[Pasted image 20261107205226.png]]]

Como observamos encontramos una ruta a la que un script incrustado en el propio HTML hace referencia, veamos qué contiene:

![[Pasted image 20261107205352.png]]

Observamos que son los datos representados en la web en realidad, donde el que resaltamos es el `session code`.

Bueno si regresamos a la web principal tenemos un archivo el cual podemos descargar, vamos a hacerlo:

![[Pasted image 20261107210702.png]]
Si analizamos el archivo es un binario compilado el cual podemos ejecutar:
![[Pasted image 20261107210750.png]]
Al intentar ejecutar como podemos observar, nos pide un `session code` y si recordamos tenemos uno en la web así que vamos a enviárselo a ver qué sucede:

```bash
./jackpot 148947
```
![[Pasted image 20261107211027.png]]
Esto nos devuelve un código de activación, aún no sabemos para qué, pero puede servirnos luego así que vamos a tenerlo en cuenta.

No logré encontrar nada más por aquí, así que pasamos directamente con otro de los puertos que es el `9100`, el cual se nos representó con el servicio de `jetdirect`, pero no logramos obtener una versión y esto es extraño así que por qué no realizamos una conexión y verificamos de qué se trata:

```bash
nc 10.67.159.182 9100
```
![[Pasted image 20261107211725.png]]
# Explotación

Si leemos un poco en la respuesta de la conexión observamos que habla sobre `XFS` lo que me hace pensar que en realidad mediante ese puerto se está comunicando este servicio.

Si continuamos leyendo miramos que nos da un formato sobre cómo enviar el mensaje el cual tenemos que seguir, luego tenemos los tipos de mensajes que podemos utilizar y por último una pista que, en específico nos dice intentar ejecutar el comando `0x0304` perteneciente a `WFS_INF_CDM_CAPABILITIES`.

Con esto claro, a partir de aquí voy a realizar el procedimiento correcto exacto necesario para no alargar mucho el writeup.

Primero vamos a armar correctamente el formato de las peticiones en formato hexadecimal utilizando `GETINFO`, donde su valor es `0x01` y lo que vamos a utilizar como `command id` es el `0x0304` que nos sugirió.

```bash
printf '\x01\x00\x00\x03\x04\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo
```
![[Pasted image 20261107213034.png]]
Listo lo correcto a partir de aquí es ejecutar todos los comandos para comprender qué hace y que errores o mensajes nos dan en específico cada uno y lo voy a poner directamente:

```bash
printf '\x01\x00\x00\x03\x01\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo

 printf '\x01\x00\x00\x03\x03\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo
 
 printf '\x01\x00\x00\x04\x01\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo
 
 printf '\x01\x00\x00\x04\xf0\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo
 
 printf '\x01\x00\x00\x04\xf1\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo
 
 printf '\x01\x00\x00\x04\xff\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo
```
![[Pasted image 20261107213501.png]]
Lo que estamos observando son las salidas respectivas de cada comando y aquí aunque parece todo un desorden en realidad esto nos ayudará a comprender el panorama completo y comprender cada conclusión. El primer comando con un poco de atención observamos que es el mismo contenido de la web así que de alguna forma la web puede que se esté comunicando directa o indirectamente con el servicio CEN/XFS. Luego el segundo comando nos lista algo parecido yo diría que algo resumido por cuenta pero nada más. El tercer comando es más interesante, si nos fijamos con atención nos habla de dispensar, es decir este en específico es el comando que envía las solicitudes de retiros. El cuarto comando podemos observar como nos especifica algo de código inválido eso quiere decir que en realidad espera un código. El quinto directamente nos habla de no armado, puede que tenga que ver algo con el código anterior que no enviamos. Por último tenemos el que al parecer será el comando responsable de darnos la flag.

Perfecto ahora con esto comenzamos el proceso enviando primero el código de armado y para esto recordemos que tenemos un código de activación que nos proporcionó el binario que descargamos:

```bash
printf '\x02\x00\x00\x04\xf0\x00\x0880036065' | nc -q 1 10.67.159.182 9100 ; echo
```

![[Pasted image 20261107214927.png]]
Como observamos ya activamos el modo supervisor y está como armado, por lo tanto, por qué no intentamos hacer un retiro:
![[Pasted image 20261107215046.png]]
La primera vez sin problema, pero luego nos bloqueó y si observamos con más atención vemos que nos habla de una anomalía y nos bloquea, al terminar el bloque vamos a tener que volver a realizar el proceso sin intentar nuevamente retiros aún.

Una vez estemos armados vamos a ejecutar el quinto comando nuevamente porque este especificaba algo sobre el armado y que tenía que estar armado porque no lo intentamos a ver qué obtenemos ahora que si lo está:

```bash
printf '\x01\x00\x00\x04\xf1\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo
```
![[Pasted image 20261107215634.png]]
Como observamos el mensaje es diferente, nos habla de un monitor que puede ser lo que detectó la anomalía así que vamos a intentar desactivarlo, recordemos que `1 -> True y 0 -> False `, así que vamos a intentarlo de la siguiente manera:

```bash
printf '\x02\x00\x00\x04\xf1\x00\x010' | nc -q 1 10.67.159.182 9100 ; echo
```
![[Pasted image 20261107215846.png]]
Como observamos lo desactivamos sin problemas así que intentemos ahora sí retirar el dinero nuevamente:
![[Pasted image 20261107220020.png]]
Como observamos logramos retirar todo el dinero dentro restante y si nos fijamos dice que la flag ya está disponible ahora ¿por qué no intentamos verla a ver si es verdad?, recordemos que el último comando era el que nos lo permitía:

```bash
printf '\x01\x00\x00\x04\xff\x00\x00' | nc -q 1 10.67.159.182 9100 ; echo
```

![[Pasted image 20261107220201.png]]
Esto es perfecto ya observamos la flag.

Laboratorio Terminado.