---
aliases: ["Ghidra", "Reverse Engineering", "Ingeniería Inversa", "Decompilador", "Ghidra Headless"]
tags: [tool/ghidra]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[GHex Editor Hexadecimal]]"
  - "[[Elevación de Privilegios mediante Explotación de Binarios]]"
  - "[[Privilegios SUID en Linux]]"
  - "[[Msfvenom - Herramienta de Generación de Payloads]]"
  - "[[Introducción a la Explotación de Vulnerabilidades]]"
fuente: "https://ghidra-sre.org/"
---

# Ghidra: Herramienta de Ingeniería Inversa

**Ghidra** es un framework de ingeniería inversa (SRE) desarrollado por la **NSA** y liberado como código abierto en 2019: desensambla, **decompila a pseudo-C**, reconstruye funciones, permite edición de tipos y scripting, y funciona sin conexión. Es la alternativa gratuita al IDA Pro con decompilador incluido, y la herramienta por defecto para analizar binarios en CTF, malware y vulnerabilidades (ver [[Introducción a la Explotación de Vulnerabilidades]]).

> [!info] Qué resuelve Ghidra
> Un binario compilado es código máquina sin nombres: no hay `main`, no hay variables, solo direcciones. Ghidra automatiza tres tareas que a mano tardan días: **recuperar el flujo de control** (funciones, bucles, llamadas), **reconstruir tipos** (firmas de función, structs, punteros) y **decompilar a C legible** para razonar sobre la lógica. El analista dedica su tiempo a *entender*, no a reconstruir estructuras.

## Ghidra vs alternativas

| Criterio | **Ghidra** | IDA (Free/Pro) | radare2 / Rizin+Cutter | Binary Ninja |
|---|---|---|---|---|
| Licencia | Gratuito, open source (NSA) | Free limitado / Pro carísimo | Libre | Comercial |
| Decompilador | **Sí, incluido** | Sí (Pro; Free sin decompilador) | `pdd`/`pdg` (r2ghidra) | Sí (docker/intermedio) |
| Plataformas | x86/x64, ARM/AArch64, MIPS, PPC, RISC-V, SPARC… | Muy amplia | Muy amplia | Amplia |
| Scripting | Java + Python (Jython/PyGhidra) | IDC/Python | Python (r2pipe) | Python/C++ |
| Colaboración | Proyectos compartidos, patch tracking | Team (caro) | — | — |
| Curva de aprendizaje | Media (GUI amable) | Baja-Media | **Alta** (CLI) | Baja |
| Ideal para | Análisis profundo, CTF, batch/headless | Flujos profesionales ya montados | Rápidez en CLI / integración | Análisis guiado moderno |

**Regla práctica:** empezar siempre en Ghidra; radare2 para queries rápidas en scripts; IDA si la organización lo paga.

## Instalación

```bash
# Kali Linux
sudo apt update && sudo apt install ghidra

# Cualquier distro: paquete ZIP oficial (requiere JDK 17+)
# Descargar de https://ghidra-sre.org/ y:
unzip ghidra_11.x_PUBLIC_YYYYMMDD.zip -d /opt/
sudo /opt/ghidra/support/launch4jconfigure.sh   # sólo si usas launchers

# Lanzar GUI / headless
/opt/ghidra/ghidraRun
/opt/ghidra/support/analyzeHeadless --help
```

> [!warning] Dependencias típicas
> - **JDK 17 o superior** (Ghidra 11.x): sin él, `ghidraRun` falla con errores de `java`/`class version`.
> - Para decompilar se necesita que el **Decompiler** del paquete esté presente (viene incluido).
> - Headless necesita que las rutas de `support/` estén ejecutables (`chmod +x`).

## Modelo de trabajo: proyecto → programa → análisis

| Concepto | Qué es |
|---|---|
| **Project (.gpr)** | Contenedor de carpetas con los binarios importados; guarda anotaciones |
| **Program** | Un binario importado (con su lenguaje/arquitectura/endian) |
| **Listing** | Vista de ensamblador con direcciones (fuente de verdad) |
| **Decompiler** | Pseudo-C generado a partir del código máquina |
| **Function** | Rango de código con nombre, firma y variables (creada por análisis o a mano) |
| **Symbols** | Nombres: imports, exports, labels, clases |
| **P-code** | IR intermedia de Ghidra: instrucciones normalizadas entre máquina y C |
| **Xref** | Referencia cruzada: quién llama a qué, quién usa qué dirección |

## Flujo completo (walkthrough de CTF)

### 1. Importar y analizar

```text
File > New Project            → crear proyecto (binario → no usar "extract from archive")
File > Import                 → elegir el crackme ELF/PE
  · Ghidra detecta arquitectura, bits y endianness; si falla: "Set Language" manual
Analysis > Auto Analyze (S)   → aceptar todos los analizadores por defecto
```

El **Auto Analyze** hace el trabajo pesado: crea funciones en cada llamada, resuelve strings y sus referencias, interpreta el demangled de C++, detecta estructuras de control y reconstruye tablas de saltos. Es el paso que convierte "direcciones" en "programa".

### 2. Localizar la lógica

| Objetivo | Cómo |
|---|---|
| Punto de entrada | `Symbol Tree > Functions > main` o `Search > Program Symbols` |
| Cadena interesante | `Search > For Strings` → doble clic → ver **xrefs** (`Ctrl+Shift+F` en la lista de referencias) |
| Función que usa una cadena | Doble clic en la referencia → caes en el código que la usa |
| Comparaciones/flags | Buscar en pseudo-C `strcmp`, `==`, `memcmp`; seguir llamadas con doble clic |
| TODOs del analista | Renombrar y comentar según avanzas |

```text
Search > For Strings   →  filtrar "flag", "password", "/home", "usage"
  → seleccionar el string → ver xrefs → ir a la función que lo referencia
```

### 3. Entender el decompilador

```c
int32_t main(int32_t arg1, char **arg2) {
    char local_20 [16];
    printf("Introduce la clave: ");
    __isoc99_scanf("%15s", local_20);
    if (strcmp(local_20, "s3cr3tK3y") == 0) {
        puts("OK");                       // ← objetivo: esta rama
        return 0;
    }
    puts("KO");
    return 1;
}
```

Qué mirar en el pseudo-C:

| Elemento | Interpretación |
|---|---|
| `local_20` | Variable de pila (offset negativo). Renombrarla a `buffer` con `L` |
| `arg1`, `arg2` | Argumentos aún sin tipar → `Edit > Function Signature` para fijarlos |
| `strcmp(...) == 0` | La condición que hay que cumplir o parchear |
| `(int32_t)` casts | Ghidra no sabe el tipo: definirlo mejora todo el resto |
| Códigos extraños | Decompilación imperfecta: corregir con stack editor / tipos / `F` crear función |

> [!warning] El decompilador miente (a veces)
> Errores clásicos: llamadas con argumentos desplazados (firma mal recuperada), bucles con `goto`, código sin funciones creadas ("undefined"), y descompilación fallida en funciones gigantes. Cuando no cuadre, **mira el Listing (ensamblador)**: es la verdad. Corrige con: `Edit > Function Signature`, `Edit > Stack Variables`, `Function > Edit Function` (frame/registro) y `Analysis > One Shot` para re-analizar ese punto.

### 4. Editar, parchear y exportar

```text
· Renombrar:        L (etiqueta/variable)      · Comentar:  ;
· Crear función:    F en la primera instrucción· Tipos:     Edit > Data Type / Signature
· Modificar bytes:  seleccionar en Listing → menú contextual de edición / Byte Viewer
· Exportar:         File > Export Program → Binary (imagen parcheada) o C
```

Para parchear, sigue la regla de tamaño de [[GHex Editor Hexadecimal]]: mismo número de bytes, o recompila/recalcula offsets.

### 5. Scriptear lo repetitivo

```text
Window > Script Manager       → lista scripts Java/Python
Analysis > Run Script         → ejecutar uno concreto
Window > Python Interactive?  → consola interactiva (versión según instalación)
```

```java
// Ejemplo mínimo (Java) — listar funciones con "flag" en el nombre
import ghidra.app.script.GhidraScript;
public class ListFlagFuncs extends GhidraScript {
    @Override public void run() throws Exception {
        currentProgram.getFunctionManager().getFunctions(true).forEach(f -> {
            if (f.getName().toLowerCase().contains("flag")) println(f.getName() + " @ " + f.getEntryPoint());
        });
    }
}
```

| Dónde | Ruta |
|---|---|
| Scripts del usuario | `~/ghidra_scripts/` |
| Scripts incluidos | `$GHIDRA_INSTALL_DIR/ghidra_scripts/` |
| Python | **Jython 2.7** (sintaxis Python 2) en versiones previas; **PyGhidra (Python 3)** desde Ghidra 11.3+ |
| Java | Compilados en tiempo de ejecución desde el Script Manager |

### 6. Headless: análisis por lotes (el superpoder de Ghidra)

```bash
# Importar + analizar + correr script, sin GUI
/opt/ghidra/support/analyzeHeadless ~/ghidra_projects CTF \
    -import ./crackme \
    -scriptPath ~/ghidra_scripts -postScript extraer_funciones.py \
    -analysisTimeoutPerFile 300 -log ghidra.log
```

| Opción headless | Función |
|---|---|
| `-import <file>` | Importa y analiza el fichero |
| `-process <name>` | Trabaja sobre un programa **ya** importado |
| `-noanalysis` | Importa sin analizar (si vas a analizar tú) |
| `-preScript` / `-postScript` | Script antes/después del análisis |
| `-scriptPath <dir>` | Rutas extra de scripts |
| `-analysisTimeoutPerFile <s>` | Corta análisis eternos |
| `-deleteProject` | Borra el proyecto al terminar (pipeline limpio) |
| `-log <file>` | Log de ejecución |

> [!tip] Patrón de pipeline para CTF
> Analizar 50 binarios de una carpeta: `for f in *.elf; do analyzeHeadless ./proj P -import $f -postScript dump.py -deleteProject; done` → el script imprime funciones/strings interesantes y solo abres en GUI los que prometen.

## Tabla de comandos y atajos

> [!note] Los atajos son los **defaults de Ghidra 11.x** y son editables en `Edit > Keybindings`; el menú siempre es la referencia válida.

### Proyecto y análisis

| Categoría | Comando/Función | Descripción | Atajo |
|---|---|---|---|
| Proyecto | `File > New Project` | Crear proyecto | `Ctrl+N` |
| Proyecto | `File > Import` | Importar binario | `Ctrl+I` |
| Proyecto | `File > Save Project` | Guardar proyecto | `Ctrl+S` |
| Análisis | `Analysis > Auto Analyze` | Análisis automático completo (funciones, xrefs, tipos, CFG) | `S` |
| Análisis | `Analysis > One Shot` | Ejecutar analizadores concretos de forma individual | — |

### Vistas y navegación

| Categoría | Comando | Descripción | Atajo |
|---|---|---|---|
| Navegación | `Window > CodeBrowser` | Ventana principal de código | `Ctrl+Shift+C` |
| Navegación | `Window > Symbol Tree` | Jerarquía de funciones/imports/labels/clases | `Ctrl+Shift+S` |
| Navegación | `Window > Listing` | Desensamblador con direcciones | `Ctrl+Shift+L` |
| Visualización | `Window > Function Graph` | CFG gráfico de la función actual | — |
| Vistas | `Window > Decompiler` | Pseudo-C de alto nivel | `F5` (configurable) |
| Vistas | `Window > Memory Map` | Secciones y permisos (`.text` r-x, `.data` rw-) | — |
| Vistas | `Window > Byte Viewer` | Bytes crudos (hex) del programa | — |

### Edición y análisis de código

| Categoría | Comando | Descripción | Atajo |
|---|---|---|---|
| Edición | `Edit > Undo/Redo` | Deshacer/rehacer | `Ctrl+Z` / `Ctrl+Y` |
| Edición | `Edit > Rename` (Label) | Renombrar variable/función/tipo | `L` |
| Edición | `Edit > Comment` | Comentario en la línea | `;` |
| Funciones | `Function > Create Function` | Definir manualmente una función | `F` |
| Funciones | `Function > Edit Function` | Nombre, retorno, parámetros, frame | `Ctrl+F` (verificar) |
| Funciones | `Function > Stack` | Editor de pila/variables locales | — |

> [!warning] Editor de pila
> Es crucial para corregir la interpretación de Ghidra sobre **variables locales y parámetros**: si el frame se calcula mal, el pseudo-C muestra argumentos inexistentes. Corregirlo (o fijar la firma con `Edit > Function Signature`) es lo que separa pseudo-C correcto de pseudo-C plausible-but-wrong.

### Búsqueda

| Categoría | Comando | Descripción | Atajo |
|---|---|---|---|
| Búsqueda | `Search > For Strings` | Cadenas en el binario (+ xrefs) | `Ctrl+F` |
| Búsqueda | `Search > For Functions` | Funciones por nombre/patrón | `Ctrl+Shift+F` |
| Búsqueda | `Search > For Patterns` | Secuencias de bytes (hex/regex) | — |
| Búsqueda | `Search > Memory` | Bytes en rangos de memoria | — |
| Datos | `Window > Data Type Manager` | Structs, enums, typedefs | — |
| Scripting | `Window > Script Manager` | Gestionar/ejecutar scripts | — |
| Scripting | `Analysis > Run Script` | Ejecutar script seleccionado | — |

## Errores comunes (mensaje → arreglo)

1. **`analyzeHeadless: command not found`** — no está en PATH.
   → Usar la ruta completa `$GHIDRA_INSTALL_DIR/support/analyzeHeadless` (o `/opt/ghidra/support/...`).
2. **`Unsupported file format / No loadable program`** — formato no soportado (p. ej. script, fichero de datos).
   → Importarlo como *raw binary* y fijar arquitectura/endian a mano (`Set Language`), o extraer el binario embebido primero.
3. **`java.lang.OutOfMemoryError` durante el análisis** — heap por defecto insuficiente en binarios grandes.
   → Activar override en `support/launch.properties` (`vm.overrides=true`) y subir `-Xmx8g`.
4. **Decompiler: `Failed to decompile` / función vacía** — función corrupta o demasiado grande.
   → Aumentar *Decompiler options → timeout*, o leer el Listing de esa función directamente.
5. **`Project is locked` / no abre el proyecto** — otra instancia de Ghidra lo tiene abierto.
   → Cerrar la otra instancia (o borrar el lock si no hay proceso vivo: revisar antes con `ps`).
6. **Scripts Python fallan con `SyntaxError`** — estás usando sintaxis Python 3 sobre **Jython 2.7**.
   → Adaptar a Python 2 o usar **PyGhidra** (Ghidra 11.3+) para Python 3 real.
7. **Pseudo-C con argumentos desplazados / `undefined` heap** — firma o stack mal inferidos.
   → `Edit > Function Signature` + `Edit > Stack Variables`; re-analizar con `Analysis > One Shot`.
8. **Archivos PE/ELF sin símbolos ("stripped")** — no hay `main` visible.
   → Buscar por strings y xrefs; identificar `entry`; `F` para crear funciones manualmente; `Function Graph` para ver estructura.
9. **Análisis interminable** — analizadores costosos (demangler, FID) en binarios enormes.
   → `-analysisTimeoutPerFile` en headless, o desactivar analizadores en `Analysis > One Shot` y analizar por partes.

## Detección y notas de seguridad

- Ghidra trabaja **offline**: no genera tráfico hacia el objetivo (salvo comprobaciones de actualización, desactivables). Analizar malware aquí es seguro desde el punto de vista de red.
- Los **proyectos** (`.gpr`/`.rep`) contienen el binario y tus anotaciones: no compartirlos con información de clientes y no versionarlos sin querer.
- Parchear/exportar un binario modificado sí tiene impacto externo (firma, comportamiento) — ver [[GHex Editor Hexadecimal]] para la disciplina de edición.

## Cheatsheet — los 10 gestos del día a día

```text
1.  File > Import  (Ctrl+I)                → cargar binario
2.  S  (Auto Analyse)                      → analizar todo
3.  Search > For Strings → xrefs           → localizar lógica
4.  L                                     → renombrar (¡siempre!)
5.  ;                                      → comentar la hipótesis
6.  F5 / Window > Decompiler               → pseudo-C
7.  Ctrl+Shift+L (Listing)                 → verificar contra ensamblador
8.  Edit > Function Signature              → arreglar tipos/argumentos
9.  Function Graph                         → ver el flujo de la función
10. analyzeHeadless -import X -postScript  → repetir en lote
```

| Tarea | Comando/Acción |
|---|---|
| Abrir de golpe un string | `Search > For Strings` → doble clic |
| Ir a una dirección | `G` / `Navigation > Go To` |
| Renombrar símbolo | `L` |
| Comentar | `;` |
| Crear función | `F` |
| Tipar un puntero | `D` (cycle data type) / `A` (array) |
| Reanalizar | `S` (Auto Analyse) |
| headless mínimo | `analyzeHeadless dir P -import bin -deleteProject` |
| Exportar binario parcheado | `File > Export Program → Binary` |
| Ayuda de scripting | `Window > Script Manager` |

## Referencias

- [Sitio oficial Ghidra](https://ghidra-sre.org/) — descargas y release notes
- [Repositorio NSA (GitHub)](https://github.com/NationalSecurityAgency/ghidra) — issues, docs y `docs/`
- [Ghidra book (curso abierto)](https://ghidrabook.com/) — guía de uso paso a paso
- Man/changelog local tras instalar: `$GHIDRA_INSTALL_DIR/Ghidra/Features/...`
- GTFOBins/HackTricks — contexto de abuso de binarios una vez entendido el programa

## Relacionadas

- [[GHex Editor Hexadecimal]] — edición byte a byte y offsets fuera de código
- [[Elevación de Privilegios mediante Explotación de Binarios]] — aplicar lo encontrado en local
- [[Privilegios SUID en Linux]] — binarios SUID como objetivos de análisis
- [[Msfvenom - Herramienta de Generación de Payloads]] — estructura de payloads/shellcode que Ghidra ayuda a leer
- [[Introducción a la Explotación de Vulnerabilidades]] — del hallazgo al exploit
