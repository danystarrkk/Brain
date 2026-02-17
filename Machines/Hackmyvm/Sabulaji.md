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

```