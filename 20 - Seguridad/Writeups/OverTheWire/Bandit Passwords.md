---
aliases:
  - Bandit Passwords
  - OverTheWire Bandit
tags:
  - hacking/laboratorios
  - scripting/bash
  - redes/comandos
estado: 🟢 Terminado
---
# Bandit Passwords

Para conectarse a un servidor que opera en el puerto 2220 a través del servicio SSH, se utiliza el siguiente comando:

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

> [!info] Conexión SSH a OverTheWire Bandit
> El comando `ssh bandit0@bandit.labs.overthewire.org -p 2220` inicia una sesión SSH.
> *   `ssh`: El cliente Secure Shell, un protocolo de red criptográfico para operar servicios de red de forma segura sobre una red no segura.
> *   `bandit0`: El nombre de usuario para autenticarse en el servidor. En los wargames de OverTheWire Bandit, cada nivel tiene un usuario asociado.
> *   `bandit.labs.overthewire.org`: La dirección del host (servidor) al que se intenta conectar.
> *   `-p 2220`: Especifica el puerto no estándar (2220) en lugar del puerto SSH predeterminado (22). Esto es común en entornos de laboratorio o CTF para evitar el escaneo masivo de puertos estándar.

| Users | Passwd |
|---|---|
| bandit0 | bandit0 |
| bandit1 | NH2SXQwcBdpmTEzi3bvBHMM9H66vVXjL |
| bandit2 | rRGizSaX8Mk1RTb1CNQoXTcYZWU6lgzi |
| bandit3 | aBZ0W5EmUfAf7kHTQeOwd8bauFJ2lAiG |
| bandit4 | 2EW7BBsr6aMMoJ2HjW067dm8EgX26xNe |
| bandit5 | lrIWWI6bB37kxfiCQZqUdOIYfr6eEeqR |
| bandit6 | P4L4vucdmLnm8I7Vl7jG1ApGSfjYKqJU |
| bandit7 | z7WtoNQU2XfjmMtWA8u5rN4vzqu4v99S |
| bandit8 | TESKZC0XvTetK0S9xNwm25STk5iWrBvP |
| bandit9 | EN632PlfYiZbn3PhVK3XOGSlNInNE00t |
| bandit10 | G7w8LIi6J3kTb8A7j9LgrywtEUlyyp6s |
| bandit11 | 6zPeziLdR2RKNdNYFNb6nVCKzphlXHBM |
| bandit12 | JVNBBFSmZwKKOP0XbFXOoW8chDz5yVRv |
| bandit13 | wbWdlBxEir4CaE8LaPhauuOo6pwRmrDw |
| bandit14 | fGrHPx402xGC7U7rXKDaxiWFTOiF0ENi |
| bandit15 | jN2kgmIXJ6fShzhT2avhotn4Zcka6tnt |
| bandit16 | JQttfApK4SeyHwDlI9SXGR50qclOAil1 |
| bandit17 | VwOSWtCA7lRKkTfbr2IDh6awj9RNZM5e |
| bandit18 | hga5tuuCLF6fFzUpnagiMN8ssu9LFrdg |
| bandit19 | awhqfNnAbc1naukrpqDYcF95h7HoMTrC |
| bandit20 | VxCazJaVykI6W36BkBU0mJTCM8rR95XT |
| bandit21 | NvEJF7oVjkddltPSrdKEFOllh9V1IBcq |
| bandit22 | WdDozAdTM2z9DiFEQ2mGlwngMfj4EZff |
| bandit23 | QYw0Y2aiA672PsMmh9puTQuhoz8SyR2G |
| bandit24 | VAfGXJ1PBSsPSnvsjI8p759leLZ9GGar |
| bandit25 | p7TaowMYrmu23Ol8hiZh9UvD0O9hpx8d |
| bandit26 | |
| bandit27 | YnQpBuifNMas1hcUFk70ZmqkhUU2EuaS |
| bandit28 | AVanL161y9rsbcJIsFHuw35rjaOM19nR |
| bandit29 | tQKvmcwNYcFS6vmPHIUSI3ShmsrQZK8S |
| bandit30 | xbhV3HpNGlTIdnjUrdAlPzc2L6y9EOnS |
| bandit31 | OoffzGDlzhAlerFJ2cAiz1D41JW1Mhmt |
| bandit32 | rmCBvG56y58BXzv98yZGdO7ATVL5dW8y |
| bandit33 | odHo63fHiFqcWWJG9rLiLDtPm45KzUKy |

## Scripts Creados en Bandit

### Decompresor.sh

```bash
#!/bin/bash

function ctrl_c (){
 echo -e "\\n\\n[!] Saliendo....\\n" 
 exit 1
}

#CTRL + C
trap ctrl_c INT

first_file_name="data.gz"
last_file_name="$(7z l data.gz | tail -n 3 | head -n 1 | xargs | awk 'NF{print $NF}')"

7z x $first_file_name &>/dev/null 

while [ $last_file_name ]; do 

  echo -e "\\n[+] Nuevo archivo descomprimido: $last_file_name"
  7z x $last_file_name &>/dev/null 
  last_file_name="$(7z l $last_file_name 2>/dev/null | tail -n 3 | head -n 1 | xargs | awk 'NF{print $NF}')"

done
```

> [!info] Análisis del Script `Decompresor.sh`
> Este script Bash está diseñado para descomprimir archivos anidados, una tarea común en desafíos de CTF o wargames como OverTheWire Bandit.
> *   **`ctrl_c` function**: Define una función para manejar la interrupción del script (Ctrl+C), mostrando un mensaje de salida y terminando el proceso.
> *   **`trap ctrl_c INT`**: Captura la señal `INT` (interrupción) y ejecuta la función `ctrl_c` cuando se recibe. Esto asegura una salida limpia.
> *   **`first_file_name="data.gz"`**: Inicializa el nombre del primer archivo a descomprimir.
> *   **`last_file_name="$(7z l data.gz | tail -n 3 | head -n 1 | xargs | awk 'NF{print $NF}')"`**: Esta es una cadena de comandos para extraer el nombre del archivo contenido dentro del archivo comprimido.
>     *   `7z l data.gz`: Lista el contenido del archivo `data.gz`.
>     *   `tail -n 3`: Toma las últimas 3 líneas de la salida (donde suele estar la información del archivo).
>     *   `head -n 1`: Toma la primera de esas 3 líneas.
>     *   `xargs`: Elimina espacios en blanco y maneja la entrada para el siguiente comando.
>     *   `awk 'NF{print $NF}'`: Imprime el último campo (`$NF`) de la línea si tiene campos (`NF`). Esto es robusto para extraer el nombre del archivo.
> *   **`7z x $first_file_name &>/dev/null`**: Descomprime el primer archivo.
>     *   `7z x`: Comando para extraer archivos.
>     *   `&>/dev/null`: Redirige tanto la salida estándar (`stdout`) como la salida de error estándar (`stderr`) a `/dev/null`, suprimiendo cualquier mensaje de 7-Zip.
> *   **`while [ $last_file_name ]`**: Un bucle `while` que continúa mientras la variable `last_file_name` no esté vacía (es decir, mientras se encuentre un nuevo archivo para descomprimir).
> *   **`echo -e "\\n[+] Nuevo archivo descomprimido: $last_file_name"`**: Muestra el nombre del archivo que se acaba de descomprimir.
> *   **`last_file_name="$(7z l $last_file_name 2>/dev/null | tail -n 3 | head -n 1 | xargs | awk 'NF{print $NF}')"`**: Dentro del bucle, este comando intenta listar el contenido del archivo recién descomprimido para encontrar el siguiente archivo anidado. `2>/dev/null` se usa aquí para suprimir errores si el archivo no es un archivo 7z válido o no contiene más archivos.

### Scanner de puertos

```bash
#!/bin/bash

function ctrl_c(){
 echo -e "\\n[+] Saliendo.....\\n" 
 exit 1
}

#CTRL + C
trap ctrl_c INT

for port in $(seq 1 65535); do
  (echo '' > /dev/tcp/127.0.0.1/$port) 2>/dev/null && echo -e "[+] $port - OPEN" &
done; wait
```

> [!info] Análisis del Script `Scanner de puertos`
> Este script Bash realiza un escaneo básico de puertos TCP en el localhost (`127.0.0.1`).
> *   **`ctrl_c` function y `trap`**: Similar al script anterior, maneja la salida limpia al presionar Ctrl+C.
> *   **`for port in $(seq 1 65535)`**: Itera a través de todos los puertos TCP posibles, desde 1 hasta 65535.
> *   **`(echo '' > /dev/tcp/127.0.0.1/$port)`**: Esta es una característica de Bash (y Zsh) que permite abrir conexiones TCP/UDP directamente desde el shell.
>     *   `echo ''`: Envía una cadena vacía al puerto.
>     *   `> /dev/tcp/127.0.0.1/$port`: Intenta abrir una conexión TCP al puerto especificado en `127.0.0.1`. Si la conexión es exitosa, el puerto se considera abierto.
> *   **`2>/dev/null`**: Redirige los errores (por ejemplo, "Connection refused" si el puerto está cerrado) a `/dev/null` para mantener la salida limpia.
> *   **`&& echo -e "[+] $port - OPEN"`**: Si el comando anterior (la conexión TCP) se ejecuta con éxito (código de salida 0), entonces se imprime un mensaje indicando que el puerto está abierto.
> *   **`&`**: Ejecuta cada intento de conexión en segundo plano. Esto permite que el escaneo se realice de forma concurrente, acelerando el proceso.
> *   **`wait`**: Espera a que todos los procesos en segundo plano (`&`) terminen antes de que el script finalice.
> > [!warning] Limitaciones de este escáner
> > Este método es rudimentario y no es tan robusto o rápido como herramientas dedicadas como `nmap`. No detecta servicios, versiones, ni tipos de filtrado de firewall. Además, puede ser ruidoso en la red y fácilmente detectado por sistemas de detección de intrusiones (IDS/IPS).

### Scanner de Hosts

```bash
#!/bin/bash

function ctrl_c(){
  echo -e "\\n[!] Saliendo.....\\n"
  exit 1
}

#CTRL+C
trap ctrl_c INT

for host in $(seq 1 254);do
  timeout 1 bash -c "ping -c 1 192.168.1.$host &>/dev/null" && echo -e "[+] Host - 192.168.1.$host - Active" &
done; wait
```

> [!info] Análisis del Script `Scanner de Hosts`
> Este script Bash realiza un escaneo de hosts activos en una subred local (`192.168.1.0/24`) utilizando el comando `ping`.
> *   **`ctrl_c` function y `trap`**: Similar a los scripts anteriores, asegura una salida limpia.
> *   **`for host in $(seq 1 254)`**: Itera a través de las direcciones IP desde `192.168.1.1` hasta `192.168.1.254`.
> *   **`timeout 1 bash -c "ping -c 1 192.168.1.$host &>/dev/null"`**:
>     *   `timeout 1`: Limita la ejecución del comando `ping` a 1 segundo. Esto evita que el script se quede colgado esperando respuestas de hosts inactivos.
>     *   `bash -c "..."`: Ejecuta el comando `ping` dentro de una nueva instancia de Bash.
>     *   `ping -c 1 192.168.1.$host`: Envía un solo paquete ICMP Echo Request (`-c 1`) a la dirección IP del host actual.
>     *   `&>/dev/null`: Redirige tanto la salida estándar como la de error a `/dev/null`, suprimiendo la salida de `ping`.
> *   **`&& echo -e "[+] Host - 192.168.1.$host - Active"`**: Si el comando `ping` se ejecuta con éxito (recibe una respuesta ICMP), se imprime un mensaje indicando que el host está activo.
> *   **`&`**: Ejecuta cada comando `ping` en segundo plano, permitiendo el escaneo concurrente de múltiples hosts.
> *   **`wait`**: Espera a que todos los procesos en segundo plano terminen.
> > [!tip] Alternativas y Consideraciones
> > *   **Escaneo ARP**: Para redes locales, el escaneo ARP (`arp-scan`) es a menudo más rápido y fiable para identificar hosts activos, ya que opera a nivel de Capa 2.
> > *   **Nmap**: Para un escaneo de hosts más avanzado y con detección de sistemas operativos/servicios, `nmap` es la herramienta estándar de la industria.
> > *   **Firewalls**: Algunos firewalls pueden bloquear o ignorar paquetes ICMP, lo que podría llevar a falsos negativos con este método de escaneo.

Recordemos que en estos scripts hacemos uso de algunos  y 