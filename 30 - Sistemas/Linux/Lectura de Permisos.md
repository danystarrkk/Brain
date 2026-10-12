---
tags:
  - sistemas
  - linux
  - permisos
tipo: concepto
estado: terminado
---

# Lectura de Permisos

**Leer permisos** es identificar qué puede hacer quién con un archivo: si podemos escribir, leer o ejecutar, y con qué identidad (propietario, grupo u otros) lo decidimos. Todo el sistema de control de acceso discrecional (DAC) de Linux cabe en la cadena que devuelve `ls -l` — y entenderla a fondo es la base para administrar sistemas y para auditarlos (ver [[Abuso de Permisos en Linux y LSE]]).

Para ver los permisos de un archivo:

```bash
ls -l
```

![[Pasted image 20240608214316.png]]

Identificamos los permisos en la sección resaltada en rojo.

![[Pasted image 20240608214350.png]]

Primero, dividimos esto en tres pares:

![[Pasted image 20240608214417.png]]

Observamos tres pares de tres caracteres, cada uno con letras que significan lo siguiente:

- `r` → leer (read)
- `w` → escribir (write)
- `x` → ejecutar (execute)

> [!info] El primer carácter de los permisos: el tipo de archivo
> El primer carácter de la cadena (`-rwxrwxrwx`) indica el **tipo**:
> - `-`: archivo regular
> - `d`: directorio
> - `l`: enlace simbólico (symlink — los symlink no tienen permisos propios efectivos: el kernel los ignora y usa los del destino)
> - `c`: dispositivo de caracteres (ej. `/dev/tty`, `/dev/null`)
> - `b`: dispositivo de bloque (ej. `/dev/sda`)
> - `p`: tubería con nombre (FIFO)
> - `s`: socket de dominio Unix (en `ls -l` de un socket no aparece como tipo con permisos; `s` aparece **dentro** de los pares cuando hay SUID/SGID activos, ver más abajo)
> - `D`, `t`…: variantes raras (dir compartido en nfs, sticky)

## Qué significa `r`/`w`/`x` según el tipo de objeto

El significado **cambia** entre archivo y directorio — la distinción que más se olvida:

| Permiso | En un **archivo** regular | En un **directorio** |
|---|---|---|
| `r` | Leer el contenido (`cat`, `less`, `cp` origen) | **Listar** los nombres de sus entradas (`ls`) |
| `w` | Modificar el contenido (sobrescribir, truncar) | **Crear, borrar y renombrar** archivos **dentro** de él (no modifica el dir "en sí", sus entradas) |
| `x` | Ejecutarlo como programa / interpretarlo (`./script.sh`, `bash script.sh`) | **Atravesar** el directorio: poder acceder a sus contenidos y a los de sus subdirs (`cd`, `stat f`, ejecutar `dir/archivo`) |

> [!warning] `r` sin `x` en un directorio: puedes ver los nombres, no los archivos
> En un directorio `rw-` (sin `x`) puedes listar los nombres pero **ningún archivo dentro es accesible** (no hay search). Inversamente, sin `r` pero con `x` puedes acceder a archivos **si conoces sus nombres** — el patrón de los directorios de binarios con nombres predecibles. Para "entra pero no mires": `--x`.

En el caso de directorios, tanto `w` como `x` requieren atención especial. Si un directorio tiene permiso `w` (escritura), significa que se pueden crear, eliminar o renombrar archivos *dentro* de él. El permiso `x` (ejecución) es lo que nos permite ingresar (atravesar) al directorio. Para archivos, su significado es más intuitivo.

> [!tip] Permisos `w` y `x` en directorios
> - `w` en un directorio permite crear, eliminar y renombrar archivos *dentro* de él, no modificar el directorio en sí.
> - `x` en un directorio es esencial para *acceder* a su contenido, listarlo o atravesarlo. Sin `x`, no se puede entrar aunque se tenga `r`.

## Las tres secciones: User, Group, Others

Ahora, centrémonos en nuestras tres secciones principales, que se dividen en:

- Propietario (**User**, `u`)
- Grupo (**Group**, `g`)
- Otros (**Others**, `o`) — todo usuario que no sea el dueño ni esté en el grupo
- (**All**, `a`) — los tres a la vez (solo en notación simbólica)

![[Pasted image 20240608214458.png]]

El propietario y el grupo se identifican en la siguiente sección:

![[Pasted image 20240608214521.png]]

> [!info] Dueño y grupo viven en el inodo, no en el archivo
> UID y GID propietarios son **campos del inodo** (junto a tamaño, timestamps y punteros a bloques). `ls -l` los lee de ahí. El **usuario propietario no siempre pertenece al grupo del archivo**: lo habitual es que el grupo sea el *grupo primario* del creador, pero `chown`/`chgrp`/SGID lo cambian. Si el UID ya no existe en el sistema, `ls` muestra el número crudo (huérfano — señal típica de cuentas borradas sin limpieza, detectable con `find / -nouser`, ver [[Búsquedas a nivel de sistema]]).

## El algoritmo de resolución: cómo decide el kernel

Cuando un proceso pide acceso (abrir, ejecutar, borrar), el kernel evalúa **en este orden exacto**:

1. **¿Es root (UID 0)?** → acceso concedido (con excepciones: los bits inmutables `chattr +i` y los mounts `ro`/`noexec` siguen aplicando, y el **capacitado** correspondiente — p. ej. `CAP_DAC_OVERRIDE` — es lo que "hace de root" aquí).
2. **¿El UID efectivo del proceso == UID dueño del inodo?** → se aplican los bits de la sección **user** y se para.
3. **¿El GID efectivo del proceso (o alguno de sus grupos suplementarios) == GID del inodo?** → se aplican los bits de **group** y se para.
4. **En otro caso** → se aplican los bits de **others**.

Implicaciones importantes:

- **Los grupos suplementarios cuentan**: `id` muestra todos; un usuario en el grupo `www-data` tiene los bits de grupo de todo archivo cuyo GID sea `www-data`, aunque su grupo primario sea otro.
- **No se "suman" secciones**: ser dueño ignora por completo grupo y otros (`chmod 400` deja al dueño sin lectura aunque "otros" tenga `r` — el dueño no cae a others).
- El **UID efectivo** (euid) manda, no el real: un binario SUID cambia el euid al del dueño, y por eso sus permisos se resuelven en la sección user del dueño (ver [[Privilegios SUID en Linux]]).
- En ACLs, el algoritmo se extiende: dueño → ACL de usuario → grupo/ACL de grupo → others (ver ACL más abajo).

## Asignación de permisos: `chmod`

Para cambiar los permisos de archivos y directorios usamos `chmod`:

### Notación simbólica

```bash
chmod <u,g,o,a> <+,-,=> <r,w,x,X,s,t> <archivo o carpeta>

chmod u+x file      # dueño puede ejecutar
chmod o-w file      # quita escritura a otros
chmod u-r,o+x,g+x file   # varios ajustes a la vez
chmod u-rx,o+xw,g+xr file   # también combinando permisos por categoría
chmod a=r f         # todos solo lectura (iguales los tres)
chmod -R g+w dirs/  # recursivo
```

| Letra especial | Significado |
|---|---|
| `X` | Ejecutable **solo si** es directorio o ya tiene `x` para alguien — el modo seguro en `chmod -R a+X` |
| `s` en `u` | SUID (`u+s`); en `g`: SGID |
| `s` en `o` | Ignorado |
| `t` | Sticky bit (`+t`) — solo tiene efecto en directorios |
| `=` | Fija el conjunto (borra lo demás); `+`/`-` ajustan |

> [!info] Notación simbólica vs octal
> - **Simbólica** (`u+x`, `g-w`): intuitiva para cambios **incrementales** — no toca los demás bits.
> - **Octal** (`755`, `644`): concisa para **establecer** un conjunto completo de una vez.
> Ambas son igualmente válidas. En scripts, la octal es determinista; en mano, la simbólica evita accidentes.

### Notación octal

Convertimos los permisos a binario (`1` concedido, `0` denegado):

![[Pasted image 20240608214543.png]]

Asignamos índices a cada posición, de derecha a izquierda:

![[Pasted image 20240608214603.png]]

Consideramos solo las posiciones con `1` y elevamos 2 al índice correspondiente:

![[Pasted image 20240608214620.png]]

Sumamos los valores resultantes para obtener el número octal de esa categoría:

![[Pasted image 20240608214639.png]]

Ya con esto claro, aplicamos el mismo procedimiento a cada tercio:

![[Pasted image 20240608214700.png]]

La forma corta: `r`=4, `w`=2, `x`=1; si el permiso está concedido, se suma:

| Permiso | Valor | Binario |
|---|---|---|
| `r` (read) | 4 | `100` |
| `w` (write) | 2 | `010` |
| `x` (execute) | 1 | `001` |
| ausente | 0 | `000` |

```bash
chmod 755 <archivo>   # rwxr-xr-x
chmod 644 <archivo>   # rw-r--r--
chmod 600 <archivo>   # rw-------
```

### Patrones comunes y qué significan realmente

| Modo | Cadena | Significado práctico | Uso típico |
|---|---|---|---|
| `644` | `rw-r--r--` | Dueño edita, mundo solo lee | Config, HTML, scripts ejecutados vía intérprete |
| `755` | `rwxr-xr-x` | Dueño controla, todos ejecutan/listan | Directorios públicos, binarios |
| `600` | `rw-------` | Totalmente privado | `~/.ssh/id_rsa`, `.pgpass`, `shadow` |
| `700` | `rwx------` | Privado y ejecutable | `~/.ssh/`, `~/bin` |
| `640` | `rw-r-----` | Dueño + un grupo dedicado | Logs (`root:adm`), secretos compartidos con un servicio |
| `660` | `rw-rw----` | Dos usuarios/grupo de trabajo | Bases de datos, spool de impresión |
| `777` | `rwxrwxrwx` | **Todos pueden todo** — casi siempre un error o un hallazgo de auditoría | Solo tolerable en `/tmp` y `/dev/shm` (con sticky); en ficheros: ver [[Abuso de Permisos en Linux y LSE]] |
| `1777` | sticky + 777 | Mundial escribible pero cada quien borra lo suyo | `/tmp` (el único lugar donde es correcto) |
| `4755` | SUID + 755 | Ejecuta como el **dueño** | `passwd`, `sudo` (ver [[Privilegios SUID en Linux]]) |
| `2755` | SGID + 755 | Ejecuta con el **grupo** del archivo / hereda grupo en dir | Juegos antiguos, dirs de equipo |

## Cambiar propietario: `chown` y `chgrp`

```bash
chown usuario archivo            # solo dueño
chown :grupo archivo             # solo grupo (nota los dos puntos)
chown usuario:grupo archivo      # ambos
chown -R usuario:grupo dir/      # recursivo (¡cuidado con SUID heredados!)
chgrp grupo archivo              # cambiar solo grupo (cualquier miembro del grupo puede hacerlo)
```

Regla del kernel: para cambiar el dueño hace falta **root** (o ser dueño en algunos casos con `chown` restringido); para cambiar el grupo, pertenecer a él. `chown` también borra el **SUID/SGID** al cambiar de dueño (por seguridad: el nuevo dueño no hereda privilegios arbitrarios) — detalle que sorprende cuando "desaparece" el SUID tras un `chown`.

## `umask`: los permisos con que nacen los archivos

Los archivos **nuevos** no nacen con permisos arbitrarios: el kernel parte de `777` (dirs) o `666` (archivos) y **resta** la máscara `umask`:

```bash
umask                    # ver (0022 típico)
# Archivo: 666 & ~022 = 644     Directorio: 777 & ~022 = 755
umask 077                # resto de sesión: archivos 600, dirs 700
```

| Umask | Nace archivo | Nace directorio | Contexto |
|---|---|---|---|
| `022` | `644` | `755` | Default de usuarios |
| `002` | `664` | `775` | Entornos con grupo cooperativo (users) |
| `077` | `600` | `700` | Privacidad total (usado por servicios) |
| `000` | `666` | `777` | **Peligroso**: todo nace abierto |

El umask se define en `/etc/profile`, `/etc/login.defs` (`UMASK`), `~/.bashrc`, o por servicio (`UMask=` en systemd). **No afecta** a archivos creados por `mv`/`cp` (esos conservan los permisos originales salvo `-p`) ni a `install` (que lleva su propio `--mode`).

## ACL: permisos más allá de las tres secciones

Las **listas de control de acceso** (`POSIX ACL`, soportadas por ext4, XFS, Btrfs, tmpfs…) permiten dar permisos **por usuario o por grupo concretos**, adicionales a la triada clásica. En `ls -l` la señal es un `+` al final de los permisos.

```bash
getfacl archivo                     # leer la ACL
setfacl -m u:otrousuario:rw archivo      # dar rw a un usuario específico
setfacl -m g:equipo:r-x dir         # dar rx a un grupo
setfacl -m d:u:otrousuario:rw dir   # ACL por defecto: heredan los archivos nuevos
setfacl -x u:otrousuario archivo    # quitar esa entrada
setfacl -b archivo                  # borrar TODA la ACL (volver al modo clásico)
setfacl -R -m u:www-data:rX /var/www   # recursivo
```

| Herramienta | Función |
|---|---|
| `getfacl -R dir` | ACL recursiva (auditoría) |
| `setfacl -m` | Modifica/crea entrada |
| `setfacl -x` | Elimina entrada |
| `setfacl -b` | Elimina toda la ACL |
| `setfacl -n` | No recalcular máscara (uso experto) |

> [!info] El campo máscara de la ACL
> Cada ACL lleva un **mask** que recorta el máximo efectivo para grupo/otros/mask. `setfacl -m g:…` lo ajusta automáticamente. El kernel resuelve: dueño → ACL usuario (match exacto) → ACL grupo/mask → others. Si el dueño quiere "solo bit clásico" usar `-b`.

> [!warning] ACL en auditorías
> `getfacl -R /` revela permisos **invisibles** para `ls -l`: un `+` sospechoso en `/etc/shadow` o en un binario puede ser una puerta trasera de un atacante (grant a su UID). `find` no filtra por ACL directamente, pero `getfacl -R` + grep es una revisión estándar post-incidente.

## Archivos especiales y permisos de dispositivo

Los nodos en `/dev` (ver [[Estructura de Directorios de Linux]]) se controlan con los **mismos bits**: `ls -l /dev/sda` → `brw-rw---- 1 root disk` = solo root y el grupo `disk` pueden leer/escribir el disco crudo. Grupos del sistema típicos: `disk` (acceso crudo a discos), `kvm` (`/dev/kvm`), `audio`, `video`, `tty`, `lp`, `shadow` (lectura de `/etc/shadow` en algunas configs). En auditorías: pertenecer a `disk` equivale casi a root (puede volcar `/etc/shadow` desde el dispositivo crudo) — patrón de escalada documentado en [[Abuso de Permisos en Linux y LSE]].

> [!warning] `noexec`, `nosuid`, `nodev` en los mounts
> Los bits pueden estar bien y el acceso seguir bloqueado: los **flags de montaje** (`/etc/fstab`) añaden capas: `noexec` (no ejecuta binarios), `nosuid` (ignora SUID/SGID), `nodev` (ignora nodos de dispositivo). `mount | grep noexec` explica muchos "Permission denied" inexplicables (típico en `/tmp` y `/dev/shm` de sistemas endurecidos). `find … -perm -4000` en un `nosuid` no es escalada real.

## Permisos especiales: SUID, SGID y Sticky Bit

### Tabla semántica completa

| Permiso | En archivo ejecutable | En directorio | Octal (bit) | `ls -l` muestra |
|---|---|---|---|---|
| **SUID** (Set User ID) | Se ejecuta con el **UID del dueño** del archivo, no del ejecutor | Sin efecto | `4` (bit 8) | `s` en user: `-rwsr-xr-x` |
| **SGID** (Set Group ID) | Se ejecuta con el **GID del dueño** del archivo | Los archivos/subdirs creados **heredan el grupo** del directorio | `2` (bit 4) | `s` en group: `-rwxr-sr-x` |
| **Sticky Bit** | Sin efecto moderno (históricamente "mantener en swap") | Solo el **dueño del archivo** (o root) puede borrarlo/renombrarlo dentro | `1` (bit 0) | `t` en otros: `drwxrwxrwt` |

### SUID (Set User ID)

Permite que un binario se ejecute **con los privilegios del usuario propietario** del archivo, no del que lo lanza. Así funciona `passwd` (edita `/etc/shadow` como root sin dar shell root).

```bash
chmod u+s <binario>       # simbólico
chmod 4755 <binario>      # octal: 4 + 755
find / -perm -4000 -ls    # listar todos (auditoría — ver [[Privilegios SUID en Linux]])
```

### SGID (Set Group ID)

- **En un archivo ejecutable**: ejecuta con el **grupo** propietario del archivo.
- **En un directorio**: todo archivo o subdirectorio creado dentro **hereda el grupo** del directorio — el mecanismo de los directorios de trabajo compartidos de equipo.

```bash
chmod g+s <archivo_o_dir>   # simbólico
chmod 2755 <archivo_o_dir>  # octal: 2 + 755
```

### Sticky Bit

Impide que usuarios con permiso `w` en el directorio borren o renombren archivos **que no son suyos**. Es lo que hace seguro el `1777` de `/tmp`: cualquiera crea, cada quien borra lo suyo.

```bash
chmod +t <directorio>       # simbólico
chmod 1755 <directorio>     # octal: 1 + 755
ls -ld /tmp                 # drwxrwxrwt  ← el 't' final
```

> [!warning] SUID/SGID: potencia y riesgo en la misma medida
> Un binario SUID/SGID es un **canal de elevación**: si tiene una vulnerabilidad (o simplemente permite ejecutar shell, interpretar scripts o manipular archivos), quien lo ejecute actúa como el dueño — normalmente root. Auditorías básicas: `find / -perm -4000 -o -perm -2000`, comprobar dueño (`root` es esperado), y revisar cada hallazgo contra [[Privilegios SUID en Linux]] y las técnicas de [[Abuso de Permisos en Linux y LSE]]. El SUID **no hereda** en copias (`cp -a` lo preserva pero `cp` simple no) ni sobrevive a `chown`.

## Capabilities: el SUID granular

Las **capabilities** dividen el privilegio de root en unidades pequeñas: un binario puede hacer *una cosa* de root sin ser root completo (p. ej. `CAP_NET_BIND_SERVICE` para escuchar en puertos <1024).

```bash
getcap -r / 2>/dev/null              # auditar: ¿qué binarios tienen caps?
setcap 'cap_net_bind_service=+ep' /usr/bin/nginx   # conceder (effective+permitted)
getcap /usr/bin/ping                 # ping moderno: cap_net_raw en vez de SUID
```

| Flag de setcap | Significado |
|---|---|
| `=+ep` | Añade effective + permitted |
| `=+ei` | Añade inheritable (se transmite por exec a hijos) |
| `=-ep` | Quita |

> [!info] Modelo de capabilities
> Principio de mínimo privilegio: en vez de dar root a un proceso que solo necesita un socket crudo, se le concede solo esa capacidad. Reduce la superficie de ataque: comprometer el proceso ya no da root completo. Los nombres completos están en `man 7 capabilities`. En auditorías, `getcap -r /` es el equivalente moderno de `find -perm -4000` — ver [[Linux Capabilities]].

## Atributos de ficheros: `chattr` y `lsattr`

Los atributos son **metadatos del filesystem** que controlan el comportamiento del kernel sobre el archivo, por encima de los bits (¡incluso por encima de root!).

```bash
chattr +i -V <archivo>     # inmutable: no se puede modificar, borrar, renombrar ni enlazar (ni root)
chattr -i <archivo>        # quitar el atributo (única forma de editar después)
lsattr <archivo_o_dir>     # listar atributos (-R recursivo)
```

| Atributo | Efecto |
|---|---|
| `+i` | **Inmutable**: bloquea escritura, borrado, renombrado, hardlinks — incluso para root |
| `+a` | **Solo append**: solo puede añadirse al final (logs que no se pueden reescribir) |
| `+c` | Sin compresión automática |
| `+u` | Datos conservados al borrar (undelete) |
| `+s` | Borrado seguro (bloques llenados de ceros) |
| `+S` | Actualizaciones síncronas (escribe inmediatamente a disco) |
| `+e` | Extents (formato ext4, por defecto) |

> [!warning] Implicaciones de seguridad de `chattr +i`
> Extremadamente potente: un archivo `+i` no se modifica, borra ni renombra **ni siendo root**. Útil para proteger `/etc/passwd` o binarios críticos… y también usado por atacantes para bloquear la limpieza del sistema o proteger su persistencia. Si un `rm`/`mv` falla como root con `Operation not permitted`, sospechá de `lsattr` — `chattr -i` devuelve el control.

## Tabla de troubleshooting

| Síntoma | Causa probable | Verificación | Arreglo |
|---|---|---|---|
| `Permission denied` al leer un archivo | No coincidís ni como dueño ni con el grupo, y otros no tiene `r` | `ls -l`; `id` (¿grupos suplementarios?) | `chmod g+r` o pertenecer al grupo |
| `Permission denied` al `cd` a un dir con `r` | Falta `x` (search) | `ls -ld dir` | `chmod +x dir` |
| `Permission denied` ejecutando `./script` | Sin bit `x`, o `noexec` en el mount | `ls -l`; `mount \| grep $(df --output=target .)` | `chmod +x`; revisar fstab |
| `Operation not permitted` borrando como root | `chattr +i` o mount `ro` | `lsattr archivo` | `chattr -i` |
| SUID no funciona en `/tmp` | Montaje con `nosuid` | `mount \| grep nosuid` | No usar ese dir para binarios |
| Archivo sin dueño (UID numérico) | Dueño borrado del sistema | `ls -l`; `find / -nouser` | `chown` a una cuenta válida |
| `ls` muestra `+` o `.` tras los permisos | ACL activa | `getfacl archivo` | `setfacl -b` si no debe haber |
| Root no puede escribir un archivo | Inmutable, o FS montado `ro` | `lsattr`, `mount` | `chattr -i`, remontar rw |
| `chown` quitó el SUID "mágicamente" | Comportamiento de seguridad del kernel | Historial del comando | Reaplicar `chmod 4755` si es legítimo |
| Los archivos nuevos salen con permisos abiertos | `umask` en `000` o `002` mal usado | `umask`, `login.defs` | Corregir a `022`/`077` |
| Servicio no puede leer su config | Dueño/grupo incorrecto tras copia | `ls -l` del archivo y del proceso (`ps -o user=`) | `chown` al usuario del servicio |

## Recetas de auditoría

```bash
# Superficie SUID/SGID del sistema
find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -ls 2>/dev/null

# Capabilities (el SUID moderno)
getcap -r / 2>/dev/null

# World-writable: archivos y directorios
find / -xdev -type f -perm -0002 -not -path '/proc/*' -ls 2>/dev/null
find / -xdev -type d -perm -0002 -not -path '/proc/*' -ls 2>/dev/null

# Directorios sin sticky que lo necesitarían (cualquiera borra lo ajeno)
find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null

# Archivos con ACL inesperadas
find /etc /usr/bin -exec getfacl {} \; 2>/dev/null | grep -B2 '^user::' | head

# Permisos de archivos sensibles
ls -la /etc/shadow* /etc/sudoers /root/.ssh/ 2>/dev/null
stat -c '%a %U:%G %n' /etc/passwd /etc/shadow /etc/sudoers

# Resumen: cómo quedaron tras una instalación masiva
find /opt/miapp -type f -printf '%m %p\n' | sort | uniq -c | sort -rn
```

## Cheatsheet

```bash
ls -l          # -rw-r--r-- 1 owner group size date NOMBRE  (+ 'l' si ACL)
# r=4 w=2 x=1 por sección (u,g,o) → chmod 755 = rwxr-xr-x
chmod 755 f · chmod u+x,g-w f · chmod -R a+X dir     # permisos
chown u:g f · chgrp g f · chown -R u:g dir           # dueño/grupo
umask                                  # 022→644/755 · 077→600/700
getfacl f · setfacl -m u:u2:rw f · setfacl -x u:u2 f · setfacl -b f
lsattr f · chattr +i f                                 # atributos (por encima de root)
getcap -r / · setcap 'cap_net_bind_service=+ep' bin    # capabilities

# Especiales: 4xxx=SUID (ejecuta como dueño) · 2xxx=SGID (grupo; hereda en dirs) · 1xxx=sticky (borra solo su dueño)
find / -perm -4000 -o -perm -2000     # auditoría SUID/SGID
# Resolución: root → dueño → grupo(suplementarios incl.) → otros (ACL inserta tras dueño)
```

## Relacionadas

- [[Privilegios SUID en Linux]] — profundización y abuso de los binarios `4xxx`
- [[Linux Capabilities]] — el modelo granular que reemplaza a SUID
- [[Abuso de Permisos en Linux y LSE]] — world-writable, `/tmp`, cron y sudo como vectores
- [[Comandos Básicos de Sistema y Red en Linux]] — `ls`, `stat`, `umask` en el catálogo diario
- [[Comandos, Rutas y Operadores de Linux]] — `/etc/passwd`, `/etc/shadow`, `sudoers`
- [[Estructura de Directorios de Linux]] — por qué ciertos directorios tienen ciertos permisos
- [[Búsquedas a nivel de sistema]] — `find -perm` en profundidad

## Referencias

- `man 1 chmod`, `man 7 chmod` (semántica rwx), `man 1 chown`, `man 1 umask`, `man 1 setfacl`, `man 1 getfacl`, `man 1 chattr`, `man 7 capabilities`
- POSIX.1-2017 File System Access Control: <https://pubs.opengroup.org/onlinepubs/9699919799/>
- Debian Administrator's Handbook — gestión de usuarios/permisos: <https://debian-handbook.info/browse/stable/sect-managing-users.html>
