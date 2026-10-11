# Enumeración

Comenzamos con un escaneo en red para determinar la IP de la máquina víctima mediante `arp-scan`:

```bash
sudo arp-scan -I vboxnet0 --localnet --ignoredups
```

![[Pasted image 20261009171650.png]]

La IP es la `192.168.56.3` en este caso, así que para intentar intuir el sistema vamos a utilizar el comando `ping`:

```bash
ping -c 1 192.168.56.3
```

![[Pasted image 20261009171854.png]]

Observamos que tenemos `ttl=64` eso quiere decir que el sistema es posiblemente un base UNIX.

Con esta básica información vamos a comenzar con el escaneo general mediante `nmap`:

```bash
nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.56.3 -oG allPorts
```

![[Pasted image 20261009172255.png]]

Observamos dos puertos abiertos que son el 80 y el 2121 que aparentemente son de http y ccproxy-ftp.

Vamos a realizar sobre estos dos puertos un escaneo más exhaustivo para identificar versiones y datos extras:

```bash
sudo nmap -p80,2121 -sVC 192.168.56.3 -oN target
```

![[Pasted image 20261009172700.png]]

Podemos observar las versiones y gracias a algunos scripts extras de reconocimiento que ejecuto `nmap` logramos observar que mediante el usuario `anonymous` logro listar los archivos contenidos dentro del servidor FTP, además tenemos las versiones de los dos servicios mediante las cuales podemos intuir que posiblemente es un sistema Linux.

Ahora vamos a conectarnos al servidor FTP para intentar extraer los archivos que vemos listados:

```bash
ftp 192.168.56.3 -p 2121
```

![[Pasted image 20261009173235.png]]

Ahora ya que estamos dentro vamos a descargar estos archivos:

```bash
mget *
```

![[Pasted image 20261009225047.png]]

Como podemos observar descargamos todos los archivos sin ningún problema y ahora, podemos comenzar a analizarlos.

Lo primero es observar el contenido del archivo `note.txt`:

![[Pasted image 20261011131147.png]]

Vemos que esta nos dice que tiene que cambiar la contraseña del WordPress, de aquí podemos sacar una conclusión y es que posiblemente tenemos alojado un WordPress.

Ahora en cuanto a los archivos, pues vamos a analizarlos mediante el comando `file` de la siguiente manera:

```bash
ls | grep "?*.jpg$" | while read file; do file $file; done
```

![[Pasted image 20261011131733.png]]

Si observamos todos están correctos, no vemos nada, pero que tal si analizamos los metadatos de estos archivos a ver que encontramos:

