---
aliases: ["Msfvenom", "Payload Generation"]
tags:
  - hacking/herramientas
  - hacking/explotacion
estado: 🟢 Terminado
---
# Msfvenom - Herramienta de Generación de Payloads

Msfvenom es una herramienta fundamental en el framework Metasploit, utilizada para generar payloads maliciosos y shellcode. Combina las funcionalidades de `msfpayload` y `msfencode`, ofreciendo una solución robusta para la creación de ejecutables, scripts y código en diversos formatos, con opciones de codificación y evasión.

| Categoría | Comando/Opción | Descripción | Ejemplo |
|---|---|---|---|
| **Sintaxis Básica** | `msfvenom [opciones]` | | |
| **Parámetros Principales** | | | |
| `-p, --payload` | Payload a usar | `-p windows/meterpreter/reverse_tcp` |
| `-f, --format` | Formato de salida | `-f exe` |
| `-o, --out` | Archivo de salida | `-o payload.exe` |
| `-a, --arch` | Arquitectura | `-a x86` |
| `--platform` | Plataforma | `--platform windows` |

> [!info] **Payloads y Arquitecturas**
> Es crucial seleccionar el `payload` y la `arquitectura` (`-a`) correctos para el sistema objetivo. Un `payload` de `windows/meterpreter/reverse_tcp` para una arquitectura `x64` no funcionará correctamente en un sistema `x86` sin la configuración adecuada, o viceversa. La `plataforma` (`--platform`) ayuda a `msfvenom` a optimizar el payload para el sistema operativo específico.

| Parámetro | Descripción | Ejemplo de Uso |
| :--- | :--- | :--- |
| **LHOST** | IP de la máquina atacante (Local Host). | LHOST=192.168.1.100 |
| **LPORT** | Puerto de escucha en la máquina atacante. | LPORT=4444 |
| **EXITFUNC** | Método de salida del proceso tras la ejecución. | EXITFUNC=thread |
| **PrependMigrate** | Migración automática del proceso tras la ejecución. | PrependMigrate=true |

> [!tip] **EXITFUNC y Estabilidad**
> La opción `EXITFUNC` determina cómo el payload se comporta al finalizar o al encontrar un error. `thread` (hilo) es comúnmente usado para payloads que se ejecutan en un proceso existente, ya que intenta salir limpiamente sin terminar el proceso anfitrión. Otras opciones incluyen `process` (termina el proceso), `seh` (Structured Exception Handling) o `none`. Elegir el `EXITFUNC` adecuado puede mejorar la estabilidad y la discreción del payload.

> [!info] **PrependMigrate para Persistencia**
> `PrependMigrate=true` es una opción avanzada que instruye al payload a migrar automáticamente a otro proceso (generalmente `explorer.exe` o un proceso estable del sistema) inmediatamente después de su ejecución. Esto es útil para mantener la sesión de Meterpreter incluso si el proceso inicial del payload es terminado, aumentando la persistencia y la robustez del ataque.

| Formato de Salida | Descripción / Sistema | Parámetro de Comando |
| :--- | :--- | :--- |
| **Windows EXE** | Ejecutable estándar de Windows. | -f exe |
| **Linux ELF** | Binario ejecutable para sistemas Linux. | -f elf |
| **Raw** | Shellcode en estado puro (sin formato). | -f raw |
| **Python** | Script compatible con Python. | -f python |
| **Código C** | Formato de array para incluir en código C. | -f c |
| **PowerShell** | Script para entornos Windows PowerShell. | -f ps1 |
| **ASP** | Script para servidores web ASP clásicos. | -f asp |
| **ASPX** | Script para aplicaciones .NET/ASPX. | -f aspx |
| **WAR Java** | Archivo de despliegue para servidores Tomcat/Java. | -f war |

> [!info] **Versatilidad de Formatos**
> Msfvenom soporta una amplia gama de formatos de salida, lo que permite adaptar el payload a casi cualquier escenario de explotación. Desde ejecutables nativos para Windows y Linux, hasta scripts para lenguajes como Python y PowerShell, o formatos web como ASP/ASPX y WAR para servidores Java. Esta flexibilidad es clave para la fase de entrega del payload.

| Flag / Parámetro | Función (Opción Avanzada) | Ejemplo de Uso |
| :--- | :--- | :--- |
| **-e, --encoder** | Selecciona el codificador para evadir firmas simples. | -e x86/shikata_ga_nai |
| **-i, --iterations** | Número de veces que se codifica el payload. | -i 5 |
| **-b, --bad-chars** | Lista de bytes prohibidos (ej. caracteres nulos). | -b "\x00\x0a\x0d" |
| **-n, --nopsled** | Añade una "rampa" de NOPs al inicio del shellcode. | -n 100 |
| **-k, --keep** | Intenta preservar la funcionalidad del template original. | -k |
| **-c, --add-code** | Incluye un archivo de shellcode externo adicional. | -c /ruta/codigo |
| **-x, --template** | Usa un ejecutable legítimo como base para el payload. | -x /ruta/legitimo.exe |

> [!warning] **Evasión con Encoders y Bad Characters**
> Los `encoders` (`-e`) se utilizan para transformar el shellcode, haciéndolo más difícil de detectar por soluciones antivirus y para evitar `bad characters` (caracteres "malos") que podrían romper el shellcode en ciertos contextos (ej. `\x00` para terminadores de cadena, `\x0a` para nueva línea, `\x0d` para retorno de carro). `shikata_ga_nai` es un encoder polimórfico muy conocido por su efectividad. Las `iteraciones` (`-i`) aumentan la complejidad del encoding.

> [!tip] **Uso de Templates para Evasión AV**
> La opción `-x, --template` permite incrustar el payload generado en un ejecutable legítimo existente. Esto puede ser una técnica efectiva para evadir la detección de antivirus, ya que el archivo resultante tendrá la apariencia y, a menudo, la funcionalidad del programa original, mientras ejecuta el payload en segundo plano. Es crucial que la arquitectura del template coincida con la del payload.

| Comando de Listado | Descripción | Ejemplo de Uso |
| :--- | :--- | :--- |
| **--list payloads** | Muestra todos los payloads disponibles en el framework. | msfvenom --list payloads |
| **--list encoders** | Lista los codificadores para intentar evadir firmas. | msfvenom --list encoders |
| **--list formats** | Muestra los formatos de salida soportados (exe, elf, raw, etc.). | msfvenom --list formats |
| **--list archs** | Muestra las arquitecturas disponibles (x86, x64, arm, mips). | msfvenom --list archs |
| **--list platforms** | Lista los sistemas operativos y plataformas compatibles. | msfvenom --list platforms |

> [!info] **Descubrimiento de Opciones**
> Las opciones `--list` son extremadamente útiles para explorar las capacidades de `msfvenom`. Permiten al usuario descubrir todos los payloads disponibles, encoders, formatos de salida, arquitecturas y plataformas soportadas, lo cual es esencial para adaptar la generación de payloads a escenarios específicos.