---
aliases:
  - THC-Hydra
  - Hydra
tags:
  - hacking/herramientas
estado: 🟢 Terminado
---
# THC-Hydra

**THC-Hydra** es una herramienta de fuerza bruta paralelizada que admite numerosos protocolos de ataque. Es extremadamente rápida y flexible, permitiendo ataques de diccionario contra servicios de red.

> [!info] Paralelización y Eficiencia
> THC-Hydra está diseñada para maximizar la eficiencia en ataques de fuerza bruta y diccionario. Su capacidad de paralelización permite lanzar múltiples intentos de autenticación simultáneamente contra un servicio, lo que reduce drásticamente el tiempo necesario para encontrar credenciales válidas. Soporta una amplia gama de protocolos, incluyendo SSH, FTP, HTTP/S, SMB, RDP, MySQL, PostgreSQL, entre muchos otros.

## Tabla de Opciones Principales de Hydra

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-l` | Usuario específico | `hydra -l admin` |
| `-L` | Lista de usuarios | `hydra -L users.txt` |
| `-p` | Contraseña específica | `hydra -p password123` |
| `-P` | Lista de contraseñas | `hydra -P pass.txt` |
| `-C` | Archivo con usuario:contraseña | `hydra -C creds.txt` |
| `-e` | Opciones adicionales (n: null password, s: try login as pass, r: reverse login as pass) | `hydra -e nsr` |
| `-t` | Tareas paralelas (por defecto 16) | `hydra -t 16` |
| `-w` | Timeout (segundos) | `hydra -w 10` |
| `-f` | Parar al encontrar credenciales | `hydra -f` |
| `-v` | Modo detallado | `hydra -v` |
| `-V` | Modo muy detallado | `hydra -V` |
| `-s` | Puerto específico | `hydra -s 8080` |
| `-S` | Usar SSL/TLS | `hydra -S` |
| `-4` | Usar solo IPv4 | `hydra -4` |
| `-6` | Usar solo IPv6 | `hydra -6` |

> [!tip] Optimización de Ataques
> Para maximizar la eficiencia, considera ajustar el número de tareas paralelas (`-t`) según la capacidad de la red y del objetivo para evitar bloqueos o detecciones. El uso de `-f` es crucial para detener el ataque una vez que se encuentran las credenciales, ahorrando tiempo y recursos.

### Opciones de Salida y Registro

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-o` | Archivo de salida | `hydra -o results.txt` |
| `-b` | Formato de salida (ej. json, xml) | `hydra -b json` |
| `-I` | Ignorar archivo de restauración | `hydra -I` |
| `-O` | Usar formato de registro antiguo | `hydra -O` |

### Opciones de Protocolos Específicos

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-m` | Opciones específicas del módulo | `hydra -m "domain=LOCAL"` |
| `-M` | Archivo con múltiples objetivos | `hydra -M targets.txt` |
| `-u` | Iterar sobre usuarios (para cada usuario, probar todas las contraseñas) | `hydra -u` |
| `-U` | Información del módulo (muestra opciones específicas para un módulo, ej. `http-get`) | `hydra -U http-get` |

> [!warning] Uso Ético y Legal
> THC-Hydra es una herramienta potente que debe ser utilizada únicamente en entornos controlados y con la autorización explícita del propietario del sistema. Su uso no autorizado puede tener graves consecuencias legales y éticas. Siempre asegúrate de tener permiso antes de realizar cualquier prueba de penetración.
