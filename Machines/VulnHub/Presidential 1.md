----------------
**IP:** 192.168.1.111
**SO:** Linux (CentOS)
**Vulns:** [[Local File Inclusion (LFI)]] [[Log Poisoning (LFI - RCE)]] 

------------------------------------------
Vamos a comenzar con un escaneo de la red para identificar la maquina con ayuda de [[Arp-Scan]]:
```bash
arp-scan -I eth0 --localnet --ignoredups
```

![[Pasted image 20250830163745.png]]
como podemos observar la IP va a ser la `192.168.1.111`, vamos a realizarle un ping para tener una idea de que sistema puede estar corriendo:
```bash
ping -c 1 192.168.1.111
```

![[Pasted image 20250830164009.png]]

ya en este punto vamos a realizar un escaneo de puertos solo para identificar los puertos abiertos:
```bash
nmap -p- --open --min-rate 5000 -sS -v -n -Pn 192.168.1..111 -oG allPorts
```

![[Pasted image 20250830164214.png]]

Podemos observar que tenemos los puertos `80,2082` por lo que vamos a realizar un escaneo mas agresivo a a esos puertos para identificar mucha mas información:
```bash
nmap -p80,2082 -sCV 192.168.1.111 -oN target
```
![[Pasted image 20250830164418.png]]

Podemos observar que tenemos una web y que además algo un poco extraño es que tenemos el ssh corriendo en el puerto 2082, algo poco común, en este caso vamos a buscar en cual es la ultima versión de ssh:
![[Pasted image 20250830164636.png]]

Podemos ver que la ultima versión parece ser la 10.0, en este caso lo que vamos a hacer es ver si para esa versión tenemos alguna vulnerabilidad:
```bash
searchsploit openssh 
```

![[Pasted image 20250830164848.png]]

Podemos ver dos vulnerabilidades para esa versión que podemos usar, en este caso la primera nos habla de una escalada de privilegios y la otra la verdad tampoco es algo que podamos hacer estando fuera de la maquina.

En este punto lo que vamos a hacer es ver la web y realizar un fuzzing:
```bash
gobuster dir -u http://192.168.1.111/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

![[Pasted image 20250830170207.png]]

En este punto vemos un `cgi-bin` lo que nos puede indicar un [[Ataque Shell Shock]] pero en este caso el directorio esta vacío y no vamos a encontrar nada por aquí.
Nos quedamos sin muchas opciones por lo que se nos viene a la mente tratar de listar por subdominios:

antes del escaneo vamos a ingresar el dominio de la web que vamos aver en como nos indica los correos y por ayuda de [[Whatweb]]

![[Pasted image 20250830170801.png]]


```bash
gobuster vhost -u http://votenow.local/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 20 | grep -v "400"
```

![[Pasted image 20250830171234.png]]
Tampoco logramos encontrar nada en realidad vamos ahora a intentar pero con el diccionario de rutas para ver si nos descubre algo:
```bash
gobuster vhost -u http://votenow.local/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --append-domain | grep -v "400"
```

![[Pasted image 20250830171817.png]]

Podemos observar como logramos obtener un subdominio el cual vamos a ver que contiene:
![[Pasted image 20250830171937.png]]

tenemos al parecer un panel administrativo de php donde podemos intentar cosas como `root` y nada o contraseñas por defecto pero no vamos a lograr pasar.

Ya en este punto podemos comenzar a pensar de que algo no esta bien, algo no estamos detectando o algo estamos pasando por alto entonces podemos volver a intentar un escaneo exhaustivo para ver si talvez con extensiones de archivo encontramos algo en la otra web:

```bash
 gobuster dir -u http://192.168.1.111/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,zip,html
```

![[Pasted image 20250830172316.png]]
Vemos que en efecto tenemos un `config.php` vamos a ver de que se trata este archivo:
![[Pasted image 20250830172447.png]]
vemos que carga pero a la vez no logramos ver ningún contenido dentro.

Vamos a ver nosotros cuando enumeramos tenemos que enumerar a muerte por todo tipo de extensiones y tenemos algunas no tan comunes pero que siempre tenemos que probar como lo son `bak,bakcup,backups` y mas además podemos intentar si tenemos ya archivos como el de `config.php` que no se ve nada podemos intentar por mas extensiones como `php.bak, zip.bak, bak1` y mas por lo tanto vamos a intentar con estas a ver que encontramos:

```bash
gobuster dir -u http://192.168.1.111/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php.bak, bak
```

![[Pasted image 20250830173510.png]]
En este caso vamos a ver que tenemos un `config.php.bak` en este punto vamos a ver que contenemos allí:
![[Pasted image 20250830173649.png]]

No vemos nada pero si revisamos el código fuente veremos lo siguiente:
![[Pasted image 20250830173749.png]]
Vemos por alguna  razón  credenciales como `votebox` -> `casoj3FFASPsbyoRP` con esto podemos intentar ingresar a la web de phpMyAdmin:
![[Pasted image 20250830174024.png]]
como podemos observar ya logramos ingresar a ella sin problema.

Podemos observar que la versión de phpMyAdmin que se esta empleando es la `4.8.1` por lo que podemos buscar exploits a ver si encontramos alguno válido:
```bash
searchsploit phpmyadmin 4.8.1
```

![[Pasted image 20250830174441.png]]
Vemos que tenemos un script de python mediante el cual al parecer podemos ejecutar comandos, vamos a ver como lo hace:
![[Pasted image 20250830180725.png]]
Podemos ver que tenemos una ruta la cual al parecer esta usando para un Path Traversal, lo único que parece estar mal es que en la ruta no es `sessions` sino `session` por lo que vamos primero a intentar el [[Local File Inclusion (LFI)]] y con este vamos a intentar enumerar el sistema:
![[Pasted image 20250830181114.png]]

vemos que nos lista el `etc/passwd` y algo que bueno ya podemos hacer es ver los usuario validos en el sistema:
![[Pasted image 20250830181230.png]]

parece ser que tenemos solo el usuario admin por lo que podemos intentar ver si tiene claves ssh de las cuales podamos aprovecharnos:
![[Pasted image 20250830181424.png]]
al parecer no podemos listar, parece ser que no tenemos permisos para eso por lo tanto podemos intentar un [[Log Poisoning (LFI - RCE)]] vemos si podemos listar los logs de `apache`:
![[Pasted image 20250830181527.png]]

Podemos ver que tampoco nos deja ver este archivo o no existe que no es muy lógico, vamos ahora a intentar enumerar servicios internos en este caso puertos abiertos de forma interna de la siguietne manera:
![[Pasted image 20250830181656.png]]
Con esto podemos intentar darnos una idea de que puertos están abiertos y en donde nos encontramos, en un contenedor o en la maquina directamente. Vamos a copiar eso y ponerlo en un archivo ya que recordemos que necesitamos darle tratamiento dado que eso esta en hexadecimal:
```bash
for port in $(cat data | awk '{print $2}' | awk '{print $2}' FS=":" | sort -u); do echo "[+] Puerto $port -> $((0x$port)) "; done
```

![[Pasted image 20250830182232.png]]

Vemos que tenemos los puertos `80,2082 y 3306` abiertos los cuales corresponde al http, ssh y el ultimo es casi siempre usado por sql.
Podemos enumerar si algún servicio sql esta corriendo enumerado la ruta del `proc/sched_debug`:
![[Pasted image 20250830182441.png]]

Podemos ver el servicio corriendo es `mysql` por lo que si seria ese servicio.
En el caso de querer saber la IP de la maquina victima para confirmar que son los archivos de la misma y no de un docker podemos hacer lo siguiente con la información guardada de `/proc/net/tcp`:
```bash
 echo "$((0xC0)).$((0xA8)).$((0x01)).$((0x6F))" 
```

![[Pasted image 20250830182844.png]]
podemos verificar que efectivamente si estamos en la maquina victima.

Podemos igual mente tratar de enumerar el `/proc/net/fib_trie` que es un fichero que nos permite ver las IP:
![[Pasted image 20250830183039.png]]
Bueno ya en este punto vamos a intentar ejecutar comandos que es lo que se quiere:
![[Pasted image 20250830183507.png]]
en esa línea lo que se intenta hacer es con el [[Local File Inclusion (LFI)]] que tenemos cargar un archivo temporal que se crea sobre la sesión, es por esto que tenemos que pasarle nuestro id de sesión (la cookie) y con eso completamos la ruta además recordemos que no se `sessions` sino `session` que eso tenemos que corregir:
![[Pasted image 20250830183645.png]]
Si lo hacemos correctamente vemos un montón de información que nos esta listando.
Esto corresponde una parte a lo que se esta haciendo en la web por lo que así vamos a hacer un [[Log Poisoning (LFI - RCE)]] en este caso del archivo de sesión temporal del usuario registrado:

Vamos a SQL y vamos a poner una Query sencilla solo para probar el punto de la inyección:
![[Pasted image 20250830183905.png]]

Vemos que allí nos muestra el Hola, si nosotros ahora regresamos y recargamos vamos a ver lo siguiente:
![[Pasted image 20250830184004.png]]

se inyecto el `Hola` y mas abajo vemos el selecto "hola", entonces nuestro objetivo aquí es inyectar código php de la siguiente manera:
![[Pasted image 20250830184134.png]]

vemos si nos representa el comando inyectado:
![[Pasted image 20250830184218.png]]

Podemos ver que si se esta interpretando, entonces en este punto lo que vamos a intentar hacer es obtener directamente una reverse shell de la siguiente manera:
![[Pasted image 20250830184803.png]]

vamos a recargar la pagina anterior pero antes de eso vamos a dejar el puerto 443 en escucha para que si se interpreta la shell nos llegue:
![[Pasted image 20250830184828.png]]
ya tenemos la shell pero algo que tenemos que tener en cuenta es que no podemos usar la web ahora que tenemos la conexión, esta se queda inhabilitada por lo que vamos a hacer lo siguiente, en este caso vamos a cancelar la conexión y enumeraremos primero la base de datos en búsqueda de información por si encontramos algo:
![[Pasted image 20250830185506.png]]

vemos que tenemos un hash del usuario `admin` y si  recordamos tenemos un usuario `admin` dentro del sistema por lo que si logramos romper la contraseña podría ser util dentro del sistema:
```bash
john -w:/usr/share/wordlists/rockyou.txt hash
```

en este punto al ser un ataque de fuerza bruta suele tardar mucho por lo que nos resta esperar a ver si tenemos la contraseña en el rockyou.txt así que en este punto nos toca esperar.
![[Pasted image 20250830190814.png]]

vemos que si logramos rompiendo la contraseña, en este punto volvamos a generar la reverse shell y una vez dentro intentemos escalar al usuario admin mediante esa contraseña:
![[Pasted image 20250830191246.png]]

ya estamos como el usuario admin por lo que podemos ir viendo cosas:
![[Pasted image 20250830191347.png]]
Podemos ver en esa nota que nos dice que se usaron nuevo comando y se comprimieron los archivos sensibles, continuando vamos a ver que sistema es:
```bash
cat /etc/os-release
```

![[Pasted image 20250830191603.png]]

En este punto ver la la version del kernel:
![[Pasted image 20250830191650.png]]

Vemos que el kernel es algo antiguo pero no lo suficiente, en este punto vamos a realizar un escaneo para ver cosas como son `sudo -l` y `find / -perm -4000 2>/dev/mull` pero no tenemos nada demasiado interesante.

vamos a buscar por capabilities:
```bash
getcap -r / 2>/dev/null
```

![[Pasted image 20250830192344.png]]

Tenemos un archivo especifico que llama mucho la atención el cual es `tarS` que tiene asignada la capability `cap_dac_read_search` que permite leer y listar cualquier archivo en el sistema si importar los privilegios, puede que esto tenga algo que ver con la nota que nos dejaron.

intentemos ejecutarlo a ver que conseguimos:
![[Pasted image 20250830193456.png]]

si nos deja ejecutar podemos ver la salida del `--help` por lo que con esta capability vamos a intentar comprimir un archivo del cual no tenemos permisos para ver que sucede:
![[Pasted image 20250830193648.png]]

vemos que si se puedo y se obtuvo el comprimido, vamos a descomprimirlo y ver que tenemos:
![[Pasted image 20250830193745.png]]

vemos que nos dice que permiso denegado pero si revisamos los propietarios:
![[Pasted image 20250830193830.png]]
vemos que somos los propietarios, entonces si asignamos permisos y veremos el contenido:
![[Pasted image 20250830193929.png]]
perfecto en este punto podemos listar cualquier archivo del sistema gracias a esa capability.

Ahora recordemos que tenemos el servicio de ssh corriendo en el puerto `2082` y como admin no tiene permisos puede que root si por lo que vamos a intentar comprimir el contenido de `/root/.ssh/id_rsa` y luego descomprimirla para ver su contenido:
![[Pasted image 20250830194326.png]]
![[Pasted image 20250830194341.png]]
perfecto ya tenemos la clave privada de ssh del usuario root que podemos usar para conectarnos como root:
```bash
ssh -i id_rsa root@localhost -p 2082
```

![[Pasted image 20250830194552.png]]
recordemos que la las claves privadas tienen que tener el permiso `600` y  listo ya estamos como root.

![[Pasted image 20250830194645.png]]

Maquina Terminada :)


