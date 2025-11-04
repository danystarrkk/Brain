-----
# Bandint passwords

Tenemos que conectarnos a un servidor el cual corre por el puerto 2220, además que a través del servicio ssh. Para esto realizamos el siguiente comando:

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

|Users|Passwd|
|---|---|
|bandit0|bandit0|
|bandit1|NH2SXQwcBdpmTEzi3bvBHMM9H66vVXjL|
|bandit2|rRGizSaX8Mk1RTb1CNQoXTcYZWU6lgzi|
|bandit3|aBZ0W5EmUfAf7kHTQeOwd8bauFJ2lAiG|
|bandit4|2EW7BBsr6aMMoJ2HjW067dm8EgX26xNe|
|bandit5|lrIWWI6bB37kxfiCQZqUdOIYfr6eEeqR|
|bandit6|P4L4vucdmLnm8I7Vl7jG1ApGSfjYKqJU|
|bandit7|z7WtoNQU2XfjmMtWA8u5rN4vzqu4v99S|
|bandit8|TESKZC0XvTetK0S9xNwm25STk5iWrBvP|
|bandit9|EN632PlfYiZbn3PhVK3XOGSlNInNE00t|
|bandit10|G7w8LIi6J3kTb8A7j9LgrywtEUlyyp6s|
|bandit11|6zPeziLdR2RKNdNYFNb6nVCKzphlXHBM|
|bandit12|JVNBBFSmZwKKOP0XbFXOoW8chDz5yVRv|
|bandit13|wbWdlBxEir4CaE8LaPhauuOo6pwRmrDw|
|bandit14|fGrHPx402xGC7U7rXKDaxiWFTOiF0ENq|
|bandit15|jN2kgmIXJ6fShzhT2avhotn4Zcka6tnt|
|bandit16|JQttfApK4SeyHwDlI9SXGR50qclOAil1|
|bandit17|VwOSWtCA7lRKkTfbr2IDh6awj9RNZM5e|
|bandit18|hga5tuuCLF6fFzUpnagiMN8ssu9LFrdg|
|bandit19|awhqfNnAbc1naukrpqDYcF95h7HoMTrC|
|bandit20|VxCazJaVykI6W36BkBU0mJTCM8rR95XT|
|bandit21|NvEJF7oVjkddltPSrdKEFOllh9V1IBcq|
|bandit22|WdDozAdTM2z9DiFEQ2mGlwngMfj4EZff|
|bandit23|QYw0Y2aiA672PsMmh9puTQuhoz8SyR2G|
|bandit24|VAfGXJ1PBSsPSnvsjI8p759leLZ9GGar|
|bandit25|p7TaowMYrmu23Ol8hiZh9UvD0O9hpx8d|
|bandit26||
|bandit27|YnQpBuifNMas1hcUFk70ZmqkhUU2EuaS|
|bandit28|AVanL161y9rsbcJIsFHuw35rjaOM19nR|
|bandit29|tQKvmcwNYcFS6vmPHIUSI3ShmsrQZK8S|
|bandit30|xbhV3HpNGlTIdnjUrdAlPzc2L6y9EOnS|
|bandit31|OoffzGDlzhAlerFJ2cAiz1D41JW1Mhmt|
|bandit32|rmCBvG56y58BXzv98yZGdO7ATVL5dW8y|
|bandit33|odHo63fHiFqcWWJG9rLiLDtPm45KzUKy|

## Script Creados en Bandit

Decompresor.sh:

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

  echo -e "\\n[+] Nuvo archivo descomprimido: $last_file_name"
  7z x $last_file_name &>/dev/null 
  last_file_name="$(7z l $last_file_name 2>/dev/null | tail -n 3 | head -n 1 | xargs | awk 'NF{print $NF}')"

done
```

Scanner de puertos:

```bash
#!/bin/bash

function ctrl_c(){
 echo -e "\\n[+] Saliendo.....\\n" 
 exit 1
}

#CTRL + C
trap ctrl_c INT

for port in $(seq 1 65635); do
  (echo '' > /dev/tcp/127.0.0.1/$port) 2>/dev/null && echo -e "[+] $port - OPEN" &
done; wait
```

Scanner de Hosts:

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

Recordemos que en estos scripts hacemos uso de algunos [[Comandos Útiles]] y [[Comandos, Rutas y Operadores]]