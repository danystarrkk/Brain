---
aliases:
  - CrackMapExec
  - CME
tags:
  - hacking/activedirectory
  - hacking/herramientas
estado: 🟢 Terminado
---
# CrackMapExec: La Navaja Suiza para Pentesting de Active Directory

**CrackMapExec** (CME) es una herramienta de post-explotación que ayuda a automatizar la evaluación de seguridad en grandes redes de Active Directory. Es conocida como "la navaja suiza para pentesting de Active Directory".

> [!info] ¿Qué es CrackMapExec?
> CME es una herramienta versátil escrita en Python diseñada para enumerar, explotar y realizar acciones post-explotación en entornos de Active Directory. Utiliza protocolos como SMB, WinRM, SSH y LDAP para interactuar con los sistemas, permitiendo a los pentesters identificar vulnerabilidades, extraer credenciales y ejecutar comandos de forma remota. Su eficiencia radica en su capacidad para operar en múltiples objetivos simultáneamente.

## Tabla de Opciones Principales de CrackMapExec

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-h` | Muestra la ayuda de la herramienta. | `crackmapexec -h` |
| `-id` | Credenciales en formato `[dominio/]usuario:contraseña`. | `crackmapexec -id usuario:contraseña` |
| `-u` | Especifica el nombre de usuario. | `crackmapexec -u usuario` |
| `-p` | Especifica la contraseña. | `crackmapexec -p contraseña` |
| `-d` | Especifica el dominio de Active Directory. | `crackmapexec -d DOMINIO` |
| `-H` | Hash NTLM para autenticación Pass-the-Hash. | `crackmapexec -H aad3b435b51404eeaad3b435b51404ee:NT_HASH` |
| `-k` | Habilita la autenticación Kerberos. | `crackmapexec -k` |
| `--aesKey` | Clave AES para autenticación Kerberos (Pass-the-Key). | `crackmapexec --aesKey [AES key]` |

> [!tip] Autenticación en Active Directory
> CME soporta múltiples métodos de autenticación. El uso de hashes NTLM (`-H`) es fundamental para ataques "Pass-the-Hash", donde no se necesita la contraseña en texto claro. La autenticación Kerberos (`-k`, `--aesKey`) es crucial en entornos donde NTLM está deshabilitado o para ataques "Pass-the-Ticket" o "Pass-the-Key", aprovechando tickets o claves AES obtenidas previamente.

### Opciones de Objetivo

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-t` | Número de hilos (threads) a utilizar (por defecto: 100). | `crackmapexec -t 50` |
| `--timeout` | Tiempo de espera para las conexiones (por defecto: 20s). | `crackmapexec --timeout 30` |
| `--jitter` | Retraso aleatorio entre solicitudes para evadir detección. | `crackmapexec --jitter 2` |
| `--delay` | Retraso fijo entre solicitudes. | `crackmapexec --delay 1` |
| `--source-ip` | Especifica la dirección IP de origen para las conexiones. | `crackmapexec --source-ip 192.168.1.10` |
| `--port` | Puerto específico a utilizar para la conexión (ej. 445 para SMB). | `crackmapexec --port 445` |

> [!warning] Consideraciones de Rendimiento y Detección
> Ajustar los parámetros `--threads`, `--timeout`, `--jitter` y `--delay` es vital. Un número alto de hilos puede acelerar el escaneo, pero también puede generar ruido en la red y activar alertas de seguridad. `--jitter` y `--delay` son útiles para simular un comportamiento más humano y evadir sistemas de detección de intrusiones (IDS/IPS) o honeypots.

### Opciones de Salida

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-v` | Habilita el modo verbose (más detallado). | `crackmapexec -v` |
| `--verbose` | Habilita un modo aún más verbose. | `crackmapexec --verbose` |
| `-o` | Especifica un archivo de salida para guardar los resultados. | `crackmapexec -o resultados.txt` |
| `--log` | Especifica un archivo de registro para la actividad de CME. | `crackmapexec --log cme.log` |
| `--json` | Genera la salida en formato JSON. | `crackmapexec --json output.json` |
| `--csv` | Genera la salida en formato CSV. | `crackmapexec --csv output.csv` |

> [!info] Formatos de Salida para Análisis Post-Explotación
> La capacidad de CME para generar resultados en formatos como JSON y CSV es extremadamente útil para el análisis post-explotación. Estos formatos estructurados facilitan la importación de datos a otras herramientas, bases de datos o scripts personalizados para un procesamiento y correlación más avanzados, lo que es crucial en evaluaciones de seguridad de gran escala.