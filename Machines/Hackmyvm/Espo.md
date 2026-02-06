----------
**IP:** 
**Sistema 
**Distribución:** 
**Vulnerabilidades:** 

-----------

Comenzamos identificando la maquina victima con ayuda de [[Arp-Scan]]:

```bash
arp-scan -I ens33 --localnet --ignoredups
```

![[Pasted image 20260107004626.png]]

Ya con la IP identificada podemos intentar intuir con ayuda de ping el sistema operativo:
```bash
ping -c 1 192.168.1.87
```

![[Pasted image 20260107005218.png]]

como podemos observar el `ttl=64` nos permite intuir que tenemos una maquina Linux.

Vamos a comenzar con un escaneo de puertos para identificar los que se encuentran abiertos, esto con ayuda de [[Nmap]]:
```bash
nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.1.87 -oG allPorts
```

![[Pasted image 20260107005536.png]]

Como podemos observar tenemos los puertos `80 y 22` abiertos, vamos a tratar de con [[Nmap]]d obtener mas información sobre los mismos:

```bash
nmap -p80,22 -sVC 192.168.1.87 -oN target
```

![[Pasted image 20260107005759.png]]

Podemos observar como es que esta corriendo los servicios de SSH y HTTP con Nginx.

Lo primero es identificar que tenemos en HTTP y mientras se haber la web vamos a ver si logramos identificar algo especial con [[Whatweb]]:
```bash
whatweb http://192.168.1.87
```

![[Pasted image 20260107010245.png]]

Como podemos observar corre `php 8.2.7`, tenemos `nginx` y pues nada mas, en este punto veamos que tenemos en la web:

![[Pasted image 20260107010434.png]]

con ayuda de `wappalizer` podemos intentar ver información extra:

![[Pasted image 20260107010529.png]]

Podemos observar que tiene `Frameworks` y `Librerias` de Java Script.

Bueno en la web se intento con credenciales por defecto y se intento comprobar si se podía enumerar usuarios que no se logro, en este punto vamos a realizar una búsqueda por directorios con ayuda de [[Gobuster]]:
```bash
gobuster dir -u http://192.168.1.87 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

![[Pasted image 20260107133038.png]]

vemos algunas rutas al parecer no podemos visitar, ahora un concepto rápido y es que en `nginx` tenemos o se suelen definir **Alias** para algunas rutas donde si esto no se realizo correctamente se podrían mediante un **Path Traversal** exponer archivos y descubrirlos mediante fuzzing.

Entonces con lo anterior claro encontramos un error al definir el alias de admin y lo podemos comprobar de la siguiente manera:

- Web `admin`:
![[Pasted image 20260107134633.png]]

- Con la inyección logramos recargar la misma web llamando directo al archivo:
![[Pasted image 20260107134740.png]]

Esto no es posible en las otras rutas.

Realizando Fuzzing tenemos que hacerlo lo mejor posible y para encontrar el archivo que necesitamos vamos a usar en la inyección un `_` de la siguiente manera:

```bash
ffuf -c -u http://192.168.1.87/admin../_FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

![[Pasted image 20260107161318.png]]

Vemos que acabamos de encontrar una ruta `_oldsite`, veamos que contiene:

![[Pasted image 20260107161414.png]]

al parecer nada, veamos si es algún directorio y busquemos archivos dentro de esta:

```bash
gobuster dir -u http://192.168.1.87/admin../_oldsite/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

![[Pasted image 20260107161540.png]]

Encontramos un archivo `info` veamos su contenido:

![[Pasted image 20260107161634.png]]

se descarga un archivo con el siguiente contenido:

![[Pasted image 20260107161811.png]]

Lo que en realidad nos sirve es que nos avisa que al parecer tenemos dentro de `/admin/_odsite` un archivo backup con formato zip, busquémoslo:

```bash
gobuster dir -u http://192.168.1.87/admin../_oldsite/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x zip
```

![[Pasted image 20260107162036.png]]

Encontramos un `backup.zip`, esto es bueno, vamos a descargarlo y descomprimirlo para ver que contiene:

![[Pasted image 20260107162458.png]]

observamos al parecer una copia de toda la web, vamos a intentar ver si en algunos archivos de configuración encontramos algo.

para evitar un poco el ruido general primero filtre por archivos `\*conf\*` con ayuda de find y luego a estas rutas le ejecute un `cat` donde por ultimo filtre por todos las coincidencias con las palabras `username|user|pass|password` quedando de la siguiente manera:
```bash
find . -name \*conf\* -exec /bin/cat {} \; | grep -iE 'user|username|password|pass'
```

![[Pasted image 20260107164644.png]]

Podemos ver dos posibles usuarios y sus claves de donde el cual el que me interesa es el de `admin` , por lo tanto vamos a probar este a ver si logramos acceder a la web:

![[Pasted image 20260107171922.png]]

como vemos logramos entrar como `admin`.

En este punto lo primero es identificar la versión del EspoCRM para buscar por vulnerabilidades conocidas.

![[Pasted image 20260107175239.png]]

vemos la versión `7.2.4` vayamos en busca de alguna vulnerabilidad conocida para esta verción:

![[Pasted image 20260107180331.png]]

encontramos el repositorio de [Github](https://github.com/josemlwdf/CVE-2023-5965) donde vamos a seguir los pasos que nos dice para explotar la vulnerabilidad, donde primero actualizamos y luego subimos la extensión y una vez echo eso obtendremos los siguiente en la ruta de `/espocrm/webshell.php`:

![[Pasted image 20260107181051.png]]

ya podemos ejecutar comandos, por lo tanto vamos a generar una Reverse Shell de la siguiente manera:

![[Pasted image 20260107181236.png]]

![[Pasted image 20260107181247.png]]

perfecto vamos a darle tratamiento y tendremos ya una bash:

![[Pasted image 20260107181346.png]]

Realizamos reconocimiento pero no encontramos nada pero vamos a revisar mediante un mini script que comandos se ejecutan a intervalos de tiempo:

![[Pasted image 20260107182305.png]]

con este script vamos a intentar detectar alguna cosa.

![[Pasted image 20260107182528.png]]

detectamos la ejecución del siguiente script done si lo revisamos encontramos lo siguiente:

![[Pasted image 20260107182954.png]]

De forma resumida el script lo que hace es copiar todo lo de `/var/shared_medias` a el `/home/madie` del usuario, esto nosotros lo podemos aprovechar y es que podemos intentar crear un archivo con el mismo nombre y que contenga el mismo script con un detalle y es que al final generemos una reverse shell. Al ejecutarse el script lo que sucedera es que remplazara el script en la carpeta y al volverse a ejecutar sin importar si tiene permisos de ejecución debido a que lo hace con `/bin/sh` se ejecutara obtendremos la reverse shell, entonces nuestro script malicioso será el siguiente:

![[Pasted image 20260107184137.png]]

antes de pasar esto al directorio `/var/shared_medias` vamos a dejar ya el puerto 444 en escucha y ahí si podemos ponerlo en el directorio y esperamos la conexión:

![[Pasted image 20260107185813.png]]

Tenemos ya conexión, al igual que con la anterior vamos a darle tratamiento:

![[Pasted image 20260107185944.png]]

ya estamos como el usuario `madie` y volvemos a hacer reconocimiento, donde con el comando `sudo -l` encontramos lo siguiente:

![[Pasted image 20260108102540.png]]

vemos que mediante el comando `sudo` podemos ejecutar el binario de `savelog` como `root` por lo tanto vamos a listar la ayuda a ver que es lo que se puede hacer:

![[Pasted image 20260108102641.png]]

podemos observar que tenemos una opción que nos permite ejecutar script, vamos a investigar la forma de usarla correctamente para intentar obtener una `bash`.

![[Pasted image 20260108103809.png]]
podemos observar que se están ejecutando comandos por lo que podríamos directamente ejecutar una `bash` desde allí de la siguiente manera:

```bash
sudo /usr/bin/savelog -x "bash" test3
```

![[Pasted image 20260108103913.png]]

Maquina Terminada.

![[Pasted image 20260108104134.png]]